# Requisitos de Auditoria e Observabilidade

- **Status:** Draft em revisão
- **Escopo:** Senior IAM Connector — MVP e produção
- **Objetivo:** definir evidências, rastreabilidade, monitoramento e alertas necessários para operação segura e suporte a auditorias de controles, incluindo SOC/SOC 2 conforme aplicável à organização.

> Este documento define requisitos do sistema. A matriz final de controles, período de retenção e evidências exigidas deve ser alinhada com Segurança, Compliance e o escopo formal da auditoria da organização.

## Princípios

- Toda alteração de lifecycle deve ser rastreável da origem ao resultado no AD DS.
- Falha silenciosa de provisionamento não é aceitável.
- Eventos de auditoria devem ser protegidos contra alteração ou exclusão indevida.
- Logs técnicos não devem expor dados de RH fora do escopo IAM.
- Exceções e intervenções manuais exigem mais evidência, não menos.
- Uma requisição aceita por um serviço intermediário não deve ser considerada sucesso até que o estado esperado seja confirmado ou reconciliado.
- Falhas devem ser isoladas por identidade: um colaborador em erro/retry não pode bloquear o processamento de outros colaboradores.

## Requisitos de auditoria

| ID | Requisito |
|---|---|
| AUD-001 | O sistema deve gerar um identificador de correlação para cada fluxo de lifecycle e propagá-lo entre leitura da Senior, processamento interno, envio ao serviço de provisioning e acompanhamento do resultado. |
| AUD-002 | O sistema deve registrar a identidade afetada usando identificador técnico estável, sem depender apenas de nome ou e-mail e sem expor CPF em texto aberto nos logs operacionais. |
| AUD-003 | Cada evento auditável deve registrar timestamp confiável, operação, estado anterior esperado, estado posterior esperado e resultado do processamento. |
| AUD-004 | O sistema deve registrar a origem da decisão de lifecycle, distinguindo evento/dado autoritativo da Senior, regra IAM, override autorizado e intervenção manual. |
| AUD-005 | Alterações de configuração que afetem lifecycle, mappings, elegibilidade, políticas de suspensão ou alertas devem possuir histórico de alteração e responsável identificável. |
| AUD-006 | Overrides individuais de férias/afastamento devem registrar responsável/autorizador do time de Gente e Gestão, justificativa, início/fim da vigência e valores anterior/posterior quando aplicável. |
| AUD-007 | O sistema deve registrar tentativas e resultados de create, update, disable, enable, suspension, reactivation, rehire, admission cancellation e reconciliation. |
| AUD-008 | O sistema deve registrar erros de forma suficiente para investigação, incluindo código/categoria, etapa, número da tentativa e dependência afetada, sem gravar secrets ou dados proibidos. |
| AUD-009 | O sistema deve correlacionar alertas com o evento de auditoria e com a operação que originou a falha. |
| AUD-010 | Quando houver reprocessamento ou intervenção operacional, o sistema deve preservar o histórico da falha original e registrar a ação que levou à recuperação. |
| AUD-011 | Registros de auditoria históricos devem ser append-only ou armazenados em mecanismo com controles equivalentes que impeçam alteração retroativa não autorizada. |
| AUD-012 | Acesso aos registros de auditoria deve ser restrito segundo Least Privilege e, quando aplicável, separado das permissões de operação do conector. |
| AUD-013 | O período de retenção dos registros de auditoria deve ser configurado conforme política corporativa e requisitos do escopo formal de compliance, sem retenção arbitrária definida pelo código. |
| AUD-014 | O sistema deve utilizar timestamps normalizados e infraestrutura com sincronização de horário suficiente para permitir reconstrução cronológica confiável dos eventos. |
| AUD-015 | O sistema deve permitir produzir evidência de uma amostra de lifecycle demonstrando origem, decisão, execução, resultado e eventuais exceções. |
| AUD-016 | Quando uma conta for desabilitada, o audit trail deve registrar o lifecycle e o motivo efetivo da desabilitação, permitindo distinguir `TERMINATED`, `ADMISSION_CANCELLED` e `TEMPORARILY_SUSPENDED`. |

## Campos mínimos de um evento de auditoria

Exemplo lógico, não vinculante à implementação:

```text
auditEventId
correlationId
timestampUtc
sourceSystem
identityId / targetObjectId
identityLifecycleStateBefore
identityLifecycleStateAfter
operation
decisionSource
policyId / ruleVersion
overrideId (quando aplicável)
disableReason (quando aplicável)
disabledAt (quando aplicável)
targetSystem
provisioningJobId / externalRequestId
attempt
result
errorCategory (quando aplicável)
alertId (quando aplicável)
actorType
actorId (para alterações administrativas/manuais)
```

Não registrar por padrão payloads completos quando identificadores e campos necessários forem suficientes.

## Política de tentativas

Cada operação com falha transitória possui **3 tentativas no total por ciclo de processamento, incluindo a tentativa inicial**.

Fluxo:

```text
Tentativa 1 -> imediata
    |
    | falhou
    v
Tentativa 2 -> após backoff
    |
    | falhou
    v
Tentativa 3 -> após backoff
    |
    | falhou
    v
Gerar alerta conforme severidade
```

Regras:

- o backoff deve ser progressivo e configurável;
- quando a dependência fornecer `Retry-After` ou mecanismo equivalente, essa orientação deve ser respeitada;
- os intervalos padrão, quando não houver `Retry-After`, ainda serão definidos na configuração operacional;
- tentativas de uma identidade não bloqueiam a fila/processamento das demais;
- após o alerta, a reconciliação pode iniciar posteriormente um novo ciclo de convergência, sem apagar o histórico anterior;
- erros classificados como não recuperáveis podem encerrar antecipadamente o ciclo quando novas tentativas não fizerem sentido, mantendo o alerta e a evidência da decisão.

## Requisitos de alertas

| ID | Requisito |
|---|---|
| ALT-001 | Falhas não recuperáveis de provisioning devem gerar alerta operacional. |
| ALT-002 | Falhas temporárias devem seguir a política de 3 tentativas. Caso não haja convergência após a terceira tentativa, devem gerar alerta. |
| ALT-003 | Falha ao desabilitar uma identidade cujo estado esperado seja `TERMINATED` deve ser classificada como `CRITICAL`. |
| ALT-004 | Falha ao aplicar suspensão obrigatória, manter conta desabilitada em admissão cancelada ou reativar uma pessoa elegível deve ser alertada conforme a matriz de severidade. |
| ALT-005 | Divergência persistente detectada por reconciliation deve gerar alerta quando exceder tolerância ou tempo máximo definidos. |
| ALT-006 | Indisponibilidade prolongada de dependências críticas, como Senior, Entra Provisioning ou Provisioning Agent, deve gerar alerta de saúde do serviço. |
| ALT-007 | Alertas devem conter correlation ID, operação, identidade técnica afetada, severidade, horário, resumo do erro, quantidade de tentativas e referência para troubleshooting. |
| ALT-008 | O sistema deve registrar criação, reconhecimento/acknowledgement quando disponível, resolução e eventual reabertura de alertas relevantes. |
| ALT-009 | O mecanismo de notificação deve ser desacoplado da regra de detecção e suportar integração com WhatsApp, e-mail e Microsoft Teams conforme a criticidade. |
| ALT-010 | O sistema deve evitar tempestade de alertas por uma mesma causa raiz por meio de agregação, deduplicação ou supressão controlada, preservando a quantidade de identidades impactadas. |
| ALT-011 | A falha e o retry de uma identidade não podem interromper a geração/processamento de eventos das demais identidades. |

## Matriz inicial de severidade

Baseline proposta para aprovação operacional:

| Severidade | Cenários principais | Canais propostos |
|---|---|---|
| **CRITICAL** | Falha após 3 tentativas ao desabilitar `TERMINATED`; reconciliation encontra conta que deveria estar desligada ainda habilitada | **WhatsApp + Teams + e-mail** |
| **HIGH** | Falha após 3 tentativas ao aplicar suspensão obrigatória; `ADMISSION_CANCELLED` permanece habilitado; falha de reativação que impede retorno; falha de criação/ativação muito próxima ou na data de admissão | **Teams + e-mail** |
| **MEDIUM** | Falha persistente de atualização de atributo organizacional; cadastro tardio abaixo do SLA de onboarding; divergência não crítica de atributos | **Teams** e registro no portal |
| **LOW / INFO** | Retry que convergiu automaticamente; eventos informativos e mudanças sem impacto de acesso | Audit log / portal, sem notificação humana obrigatória |

A severidade final de um evento pode ser elevada por contexto, quantidade de identidades impactadas ou duração da indisponibilidade.

## Canais e responsáveis

Canais definidos como desejados:

```text
CRITICAL -> WhatsApp + e-mail + Teams
HIGH     -> e-mail + Teams
MEDIUM   -> Teams
LOW/INFO -> portal/audit log
```

A integração técnica de WhatsApp, e-mail e Teams deve permanecer desacoplada do motor de alertas.

**Responsáveis/grupos receptores ainda precisam ser nominados.** Como baseline para discussão:

- `CRITICAL`: IAM/Infra + Segurança; Gente e Gestão pode ser incluído quando a confirmação de negócio for necessária;
- `HIGH`: IAM/Infra e equipe operacional responsável pelo onboarding/lifecycle;
- `MEDIUM`: fila/canal operacional de IAM;
- `LOW/INFO`: consulta no portal e observabilidade.

WhatsApp deve usar um canal corporativo/provedor aprovado; não deve depender de número pessoal hardcoded no conector.

## Evidência para auditoria

O sistema deve permitir responder, com evidências, perguntas como:

1. Quem ou qual fonte iniciou esta alteração?
2. Qual dado ou regra determinou a decisão?
3. Qual identidade foi afetada?
4. Qual era o estado esperado antes e depois?
5. Quando a alteração ocorreu?
6. O AD DS atingiu o estado esperado?
7. Houve falha ou retry? Quantas tentativas ocorreram?
8. Se houve falha, qual alerta foi gerado, em qual severidade e por quais canais?
9. Houve override manual ou exceção de política?
10. Quem autorizou/configurou a exceção e por qual período?
11. Por que a conta está desabilitada e desde quando?

## Controles operacionais a definir antes de produção

- ferramenta de centralização de logs;
- destino de métricas e traces;
- responsáveis/grupos exatos por severidade;
- integração/provedor corporativo de WhatsApp;
- integração de e-mail e Microsoft Teams;
- tempos esperados de tratamento por severidade;
- retenção de audit logs;
- acesso de leitura para Auditoria/Compliance;
- proteção contra exclusão/alteração indevida;
- processo de revisão periódica de overrides e exceções;
- evidências de revisão de acessos administrativos ao próprio conector;
- intervalos padrão de backoff quando a dependência não fornecer `Retry-After`.

## Critérios de aceite do MVP

- [ ] Create bem-sucedido possui trilha ponta a ponta por correlation ID.
- [ ] Update bem-sucedido possui trilha ponta a ponta.
- [ ] Disable bem-sucedido possui trilha ponta a ponta e motivo/data de desabilitação.
- [ ] Falha simulada de uma identidade não bloqueia outra identidade válida.
- [ ] Operação transitória é tentada no máximo 3 vezes por ciclo.
- [ ] Falha após a terceira tentativa gera alerta na severidade correta.
- [ ] Falha de desligamento gera `CRITICAL`.
- [ ] Alerta permite identificar a identidade e a operação sem expor dados sensíveis desnecessários.
- [ ] Reprocessamento preserva histórico anterior.
- [ ] Override individual de férias registra autorização de Gente e Gestão e vigência completa.
- [ ] Reconciliation consegue evidenciar divergência e posterior convergência ou alerta.
- [ ] Matriz de canais por severidade validada em laboratório.
