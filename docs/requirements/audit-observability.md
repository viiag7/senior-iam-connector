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

## Requisitos de auditoria

| ID | Requisito |
|---|---|
| AUD-001 | O sistema deve gerar um identificador de correlação para cada fluxo de lifecycle e propagá-lo entre leitura da Senior, processamento interno, envio ao serviço de provisioning e acompanhamento do resultado. |
| AUD-002 | O sistema deve registrar a identidade de origem afetada usando identificador técnico estável, sem depender apenas de nome ou e-mail. |
| AUD-003 | Cada evento auditável deve registrar timestamp confiável, operação, estado anterior esperado, estado posterior esperado e resultado do processamento. |
| AUD-004 | O sistema deve registrar a origem da decisão de lifecycle, distinguindo evento/dado autoritativo da Senior, regra IAM, override autorizado e intervenção manual. |
| AUD-005 | Alterações de configuração que afetem lifecycle, mappings, elegibilidade, políticas de suspensão ou alertas devem possuir histórico de alteração e responsável identificável. |
| AUD-006 | Overrides individuais, especialmente manutenção de acesso durante férias/afastamento, devem registrar responsável, justificativa, vigência e valores anterior/posterior quando aplicável. |
| AUD-007 | O sistema deve registrar tentativas e resultados de create, update, disable, enable, suspension, reactivation, rehire e reconciliation. |
| AUD-008 | O sistema deve registrar erros de forma suficiente para investigação, incluindo código/categoria, etapa, tentativa e dependência afetada, sem gravar secrets ou dados proibidos. |
| AUD-009 | O sistema deve correlacionar alertas com o evento de auditoria e com a operação que originou a falha. |
| AUD-010 | Quando houver reprocessamento ou intervenção operacional, o sistema deve preservar o histórico da falha original e registrar a ação que levou à recuperação. |
| AUD-011 | Registros de auditoria históricos devem ser append-only ou armazenados em mecanismo com controles equivalentes que impeçam alteração retroativa não autorizada. |
| AUD-012 | Acesso aos registros de auditoria deve ser restrito segundo Least Privilege e, quando aplicável, separado das permissões de operação do conector. |
| AUD-013 | O período de retenção dos registros de auditoria deve ser configurado conforme política corporativa e requisitos do escopo formal de compliance, sem retenção arbitrária definida pelo código. |
| AUD-014 | O sistema deve utilizar timestamps normalizados e infraestrutura com sincronização de horário suficiente para permitir reconstrução cronológica confiável dos eventos. |
| AUD-015 | O sistema deve permitir produzir evidência de uma amostra de lifecycle demonstrando origem, decisão, execução, resultado e eventuais exceções. |

## Campos mínimos de um evento de auditoria

Exemplo lógico, não vinculante à implementação:

```text
auditEventId
correlationId
timestampUtc
sourceSystem
sourcePersonId
identityLifecycleStateBefore
identityLifecycleStateAfter
operation
decisionSource
policyId / ruleVersion
overrideId (quando aplicável)
targetSystem
targetObjectId
provisioningJobId / externalRequestId
attempt
result
errorCategory (quando aplicável)
alertId (quando aplicável)
actorType
actorId (para alterações administrativas/manuais)
```

Não registrar por padrão payloads completos quando identificadores e campos necessários forem suficientes.

## Requisitos de alertas

| ID | Requisito |
|---|---|
| ALT-001 | Falhas não recuperáveis de provisioning devem gerar alerta operacional. |
| ALT-002 | Falhas temporárias devem seguir política de retry; caso não haja convergência dentro do limite definido, devem gerar alerta. |
| ALT-003 | Falha ao desabilitar uma conta que deveria estar `TERMINATED` deve possuir severidade superior a uma atualização não crítica de atributo. |
| ALT-004 | Falha ao aplicar `TEMPORARILY_SUSPENDED` ou ao reativar um colaborador elegível deve ser alertada conforme criticidade definida pela política operacional. |
| ALT-005 | Divergência persistente detectada por reconciliation deve gerar alerta quando exceder tolerância ou tempo máximo definidos. |
| ALT-006 | Indisponibilidade prolongada de dependências críticas, como Senior, Entra Provisioning ou Provisioning Agent, deve gerar alerta de saúde do serviço. |
| ALT-007 | Alertas devem conter correlation ID, operação, identidade técnica afetada, severidade, horário, resumo do erro e referência para troubleshooting. |
| ALT-008 | O sistema deve registrar criação, reconhecimento/acknowledgement quando disponível, resolução e eventual reabertura de alertas relevantes. |
| ALT-009 | O canal de notificação deve ser desacoplado da regra de detecção, permitindo integração futura com e-mail, Teams, sistema de incidentes ou ferramenta corporativa de observabilidade. |
| ALT-010 | O sistema deve evitar tempestade de alertas por uma mesma causa raiz por meio de agregação, deduplicação ou supressão controlada, preservando a quantidade de identidades impactadas. |

## Severidade inicial proposta

A classificação final será definida nas regras operacionais, mas a baseline deve considerar impacto de acesso:

| Exemplo | Severidade relativa esperada |
|---|---|
| Falha ao desabilitar colaborador desligado | Crítica/alta |
| Falha ao suspender acesso que deveria estar bloqueado | Alta |
| Falha ao reativar colaborador que deveria estar ativo | Alta/média |
| Falha ao criar identidade de novo colaborador próximo à admissão | Alta/média |
| Falha em atributo organizacional não crítico | Média/baixa |
| Retry isolado que convergiu automaticamente | Informacional/auditável, sem alerta humano obrigatório |

Os nomes exatos (`Critical`, `High`, etc.) serão definidos conforme a ferramenta corporativa de incidentes/monitoramento.

## Evidência para auditoria

O sistema deve permitir responder, com evidências, perguntas como:

1. Quem ou qual fonte iniciou esta alteração?
2. Qual dado ou regra determinou a decisão?
3. Qual identidade foi afetada?
4. Qual era o estado esperado antes e depois?
5. Quando a alteração ocorreu?
6. O AD DS atingiu o estado esperado?
7. Houve falha ou retry?
8. Se houve falha, quem foi alertado e como a situação foi resolvida?
9. Houve override manual ou exceção de política?
10. Quem aprovou/configurou a exceção e por qual período?

## Controles operacionais a definir antes de produção

- ferramenta de centralização de logs;
- destino de métricas e traces;
- canal ou plataforma de alertas;
- responsáveis/on-call por tipo de incidente;
- tempos esperados de tratamento por severidade;
- retenção de audit logs;
- acesso de leitura para Auditoria/Compliance;
- proteção contra exclusão/alteração indevida;
- processo de revisão periódica de overrides e exceções;
- evidências de revisão de acessos administrativos ao próprio conector.

## Critérios de aceite do MVP

- [ ] Create bem-sucedido possui trilha ponta a ponta por correlation ID.
- [ ] Update bem-sucedido possui trilha ponta a ponta.
- [ ] Disable bem-sucedido possui trilha ponta a ponta.
- [ ] Falha simulada de provisioning gera registro auditável.
- [ ] Falha persistente simulada gera alerta.
- [ ] Alerta permite identificar a identidade e a operação sem expor dados sensíveis desnecessários.
- [ ] Reprocessamento preserva histórico anterior.
- [ ] Override individual de férias gera evidência de quem/quando/por quê.
- [ ] Reconciliation consegue evidenciar divergência e posterior convergência ou alerta.
