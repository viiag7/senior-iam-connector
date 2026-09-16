# Joiner–Mover–Leaver (JML) e Suspensões Temporárias

- **Status:** Draft em revisão
- **Objetivo:** definir como eventos do vínculo no Senior afetam a identidade no AD DS.

## Princípio

O Senior é a fonte autoritativa para o lifecycle do colaborador. O conector transforma o estado de RH em decisões de identidade, enquanto o Microsoft Entra Provisioning Service executa as operações de provisioning configuradas.

O lifecycle deve ser baseado em **estado e regras determinísticas**, não em tickets ou interpretações manuais.

O modelo funcional considera Joiner, Mover, Leaver, Rehire e também **suspensões temporárias de acesso**, como férias, determinados afastamentos e outras situações que a organização decidir mapear.

Exceções IAM não alteram o estado de RH no Senior. Elas apenas alteram a ação de acesso aplicada pelo conector dentro dos limites autorizados pela política.

---

## Joiner

### Trigger funcional

Colaborador elegível para provisionamento segundo os critérios de escopo aprovados.

Critérios a confirmar:

- situação do vínculo;
- data de admissão;
- tipo de vínculo elegível;
- empresa/filial elegível;
- antecedência permitida para pré-provisionamento.

### Pré-provisionamento

A identidade pode precisar ser criada **antes da data efetiva de admissão** para permitir preparação dos processos dependentes de onboarding.

A antecedência deve ser configurável e definida por regra de negócio. A criação antecipada da identidade não implica que a conta já esteja habilitada para uso.

Devem ser definidos separadamente:

- número de dias de antecedência;
- estado técnico da conta após criação;
- quando o acesso efetivo é liberado;
- tratamento de admissão cancelada ou postergada.

### Resultado esperado

```text
Senior employee eligible
        |
        v
Validate required attributes
        |
        v
Generate deterministic identity attributes
        |
        v
Send full record to Entra /bulkUpload
        |
        v
Provisioning Service performs matching
        |
        +--> existing identity: update according to Rehire/matching policy
        |
        +--> no match: create AD DS account
```

### Regras obrigatórias

- Não criar identidade sem correlation key válida.
- Não criar identidade se houver match ambíguo.
- Username e UPN devem seguir política determinística.
- Reprocessar o mesmo colaborador não pode criar uma segunda conta.
- Criação da identidade e habilitação de acesso são decisões separadas.
- O resultado deve ser confirmado por provisioning status/log/reconciliation; aceite assíncrono da requisição não representa sucesso final.
- Falha persistente deve ser registrada e alertada conforme severidade definida.

### Critério de aceite do MVP

Um colaborador fictício elegível deve resultar em exatamente uma conta no AD DS, no estado esperado, com atributos mínimos aprovados e audit trail ponta a ponta.

---

## Mover

Mover cobre qualquer mudança relevante em um colaborador já correlacionado.

Exemplos:

- cargo;
- departamento;
- gestor;
- centro de custo;
- empresa/filial;
- localidade;
- nome civil/preferido quando aplicável;
- tipo de vínculo;
- situação do vínculo que não represente desligamento.

### Resultado esperado

```text
Senior state changes
        |
        v
Connector reads current complete record
        |
        v
Entra inbound provisioning
        |
        v
Update approved AD DS attributes
```

### Regras

- O MVP atualiza apenas atributos aprovados no `attribute-mapping.md`.
- Mudança de cargo/departamento não concede automaticamente privilégios administrativos.
- Groups, roles e entitlements ficam fora do MVP, salvo decisão posterior explícita.
- Mudanças manuais no AD DS em atributos authoritative do Senior podem ser sobrescritas pelo provisioning.
- Mudança de nome não deve automaticamente criar nova identidade.
- Falhas que impeçam convergência do estado esperado devem ser rastreadas e alertadas conforme política operacional.

---

## Suspensão temporária

Suspensão temporária representa uma condição em que o vínculo continua existindo, mas a política IAM pode exigir bloqueio temporário de acesso.

Exemplos potenciais:

- férias;
- afastamento médico quando a política exigir bloqueio;
- licença;
- suspensão disciplinar;
- outras situações cadastradas na Senior e explicitamente classificadas pela organização como relevantes para acesso.

### Política padrão e personalização por colaborador

O sistema deve suportar uma **política padrão da organização** e uma **exceção individual autorizada por colaborador**.

Exemplo:

```text
Evento de férias
      |
      v
Consultar política padrão
      |
      +--> sem exceção individual: aplicar política padrão
      |
      +--> com exceção autorizada: aplicar política efetiva do colaborador
```

Isso permite, por exemplo, que a política padrão seja suspender acesso durante férias, enquanto um colaborador específico permaneça habilitado quando existir justificativa e autorização apropriadas.

A exceção individual deve ser auditável e registrar no mínimo:

- colaborador afetado;
- política anterior e decisão efetiva;
- responsável pela configuração/alteração;
- justificativa;
- data/hora da alteração;
- vigência ou período relacionado, quando aplicável.

A exceção individual **não pode** sobrepor estados de maior precedência, especialmente desligamento.

### Resultado esperado quando a política efetiva exigir suspensão

```text
ACTIVE
  |
  | início do evento + política efetiva exige bloqueio
  v
TEMPORARILY_SUSPENDED
  |
  | evento termina + política permite acesso + nenhum estado impeditivo
  v
ACTIVE
```

### Resultado esperado quando a política efetiva permitir acesso

```text
ACTIVE
  |
  | férias/afastamento com exceção autorizada para manter acesso
  v
ACTIVE
```

O evento de RH continua existindo e deve permanecer rastreável, mesmo quando não causar alteração técnica na conta.

### Regras obrigatórias

- Suspensão temporária deve desabilitar a conta apenas quando a política efetiva determinar bloqueio.
- O retorno deve reativar **a mesma identidade**, preservando a correlation key.
- Suspensão temporária não deve ser interpretada como Leaver.
- Suspensão temporária não deve excluir nem recriar a conta.
- Datas de início e fim devem ser respeitadas quando disponíveis.
- Alteração da data de retorno deve recalcular o comportamento previsto.
- Se não existir data de retorno e a política exigir bloqueio, a conta permanece suspensa até que o estado autoritativo permita reativação.
- Um desligamento efetivo tem precedência sobre férias, afastamento, retorno e qualquer override individual.
- Falha ao aplicar um bloqueio esperado deve gerar alerta conforme severidade definida.
- Falha ao reativar um colaborador que deveria voltar a `ACTIVE` deve gerar alerta conforme severidade definida.

### Casos que precisam ser decididos

- política padrão da organização para férias;
- quem pode criar/aprovar override individual;
- se override possui expiração obrigatória;
- quais tipos/códigos de afastamento desabilitam acesso;
- horário de início do bloqueio;
- horário de retorno;
- comportamento quando eventos se sobrepõem;
- revisão periódica de exceções existentes.

---

## Leaver

### Trigger funcional

O vínculo atinge um estado de desligamento reconhecido pelo contrato IAM.

Os códigos e situações exatos do Senior devem ser documentados no Discovery.

### Resultado esperado do MVP

```text
Senior employment becomes terminated
        |
        v
Connector determines TERMINATED
        |
        v
Provisioning rules disable the AD DS identity
        |
        v
Confirm expected state
        |
        +--> success: audit
        |
        +--> failure: audit + high-priority alert + retry/escalation
```

### Regras a definir antes de produção

- horário efetivo do disable;
- uso da data de desligamento versus alteração imediata de status;
- comportamento para desligamento futuro/agendado;
- grupos privilegiados e sessões — fora ou dentro de fase posterior;
- retenção da conta desabilitada;
- OU de contas desabilitadas;
- momento e autoridade para exclusão definitiva;
- severidade e tempo de tratamento de falha de desabilitação.

### Regra segura inicial

**Disable antes de Delete.** O MVP não deve excluir automaticamente contas do AD DS.

Nenhum override de férias, afastamento ou outra suspensão temporária pode impedir o bloqueio causado por desligamento efetivo.

---

## Rehire

Rehire deve ser tratado como cenário próprio, não como novo Joiner cego.

### Objetivo

Reutilizar ou reativar a identidade correta somente quando houver correlation inequívoca e a política organizacional permitir.

### Pontos a decidir

- a pessoa mantém o mesmo identificador imutável no Senior?
- matrícula é reutilizada ou nova?
- a conta AD anterior deve ser reativada ou uma nova deve ser criada?
- UPN e `sAMAccountName` antigos devem ser preservados?
- como tratar uma conta já deletada?

Até essas respostas existirem, Rehire deve falhar de forma segura quando houver ambiguidade.

---

## Precedência de estados

Quando mais de uma situação estiver vigente simultaneamente, o sistema deve aplicar regra determinística de precedência. Como baseline funcional:

```text
TERMINATED
        >
TEMPORARILY_SUSPENDED
        >
ACTIVE
        >
PRE_PROVISIONED
```

A política efetiva de férias/afastamento determina se o evento leva ou não a `TEMPORARILY_SUSPENDED`, mas não modifica essa precedência.

Exemplo: um colaborador em férias com override para manter acesso que seja desligado durante o período deve ser desabilitado pelo estado `TERMINATED`.

A tabela final de precedência será definida nas regras de negócio.

---

## Cenários de exceção

| Cenário | Comportamento esperado |
|---|---|
| Identificador ausente | Rejeitar registro e alertar conforme política |
| Duplicidade de correlation key | Bloquear processamento da identidade |
| Match ambíguo no target | Não alterar contas automaticamente |
| Senior indisponível | Retry com backoff; não assumir desligamento |
| Entra indisponível/429 | Retry respeitando throttling |
| Provisioning Agent offline | Manter acompanhamento e alertar |
| Registro inválido | Isolar falha ao colaborador afetado |
| Evento duplicado | Processamento idempotente |
| Evento perdido | Recuperar por reconciliation |
| Retorno de férias após desligamento | Manter conta desabilitada |
| Férias com override autorizado | Aplicar política individual e auditar decisão |
| Override de férias + desligamento | Ignorar override para acesso; `TERMINATED` prevalece |
| Afastamento sem data final | Aplicar política efetiva até novo estado autoritativo |
| Admissão postergada após pré-provisionamento | Recalcular liberação conforme política |
| Admissão cancelada após pré-provisionamento | Impedir ativação; tratar conta conforme regra e auditar |
| Provisioning aceito mas sem convergência | Manter acompanhamento; alertar quando exceder tolerância |

## Reconciliation

Mesmo que a Senior disponibilize eventos/webhooks, deve existir reconciliação periódica para identificar:

- registros não processados;
- alterações perdidas;
- falhas temporárias;
- divergências persistentes;
- identidades no target sem estado esperado na fonte;
- contas temporariamente suspensas que deveriam ter sido reativadas ou mantidas bloqueadas;
- contas mantidas ativas por override sem evidência/configuração válida;
- operações aceitas de forma assíncrona que nunca atingiram o estado final esperado.

Divergências persistentes devem ser alertadas conforme `docs/requirements/audit-observability.md`.

## Definition of Done por cenário

Um cenário de lifecycle só está pronto quando:

- regra funcional está documentada;
- dados de entrada estão definidos;
- mapping está aprovado;
- testes automatizados existem;
- teste ponta a ponta foi executado em laboratório;
- logs e correlation ID permitem rastrear o processamento;
- falhas, retries e alertas foram testados;
- comportamento de reprocessamento está documentado;
- exceções/overrides aplicáveis possuem trilha de auditoria;
- o estado final no target pode ser evidenciado.
