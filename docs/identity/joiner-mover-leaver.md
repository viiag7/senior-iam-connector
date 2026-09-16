# Joiner–Mover–Leaver (JML) e Suspensões Temporárias

- **Status:** Draft em revisão
- **Objetivo:** definir como eventos do vínculo no Senior afetam a identidade no AD DS.

## Princípio

O Senior é a fonte autoritativa para o lifecycle do colaborador. O conector transforma o estado de RH em decisões de identidade, enquanto o Microsoft Entra Provisioning Service executa as operações de provisioning configuradas.

O lifecycle deve ser baseado em **estado e regras determinísticas**, não em tickets ou interpretações manuais.

A conta corporativa representa a pessoa e pode estar relacionada a mais de um vínculo simultâneo. Portanto, o estado efetivo da identidade não pode ser calculado observando apenas um vínculo isoladamente.

O modelo funcional considera Joiner, Mover, Leaver, Rehire, cancelamento de admissão e também **suspensões temporárias de acesso**, como férias e afastamentos.

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
- quando o acesso efetivo é liberado.

O risco de cadastro tardio na Senior está documentado em `docs/requirements/process-risks-and-assumptions.md`.

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
Send record to provisioning flow
        |
        v
Provisioning performs matching/execution
        |
        +--> existing identity: update/reactivate according to lifecycle
        |
        +--> no match: create AD DS account
```

### Regras obrigatórias

- Não criar identidade sem CPF válido para correlação.
- Não criar identidade se houver conflito/ambiguidade de correlação.
- Username e UPN devem seguir política determinística.
- Reprocessar a mesma pessoa não pode criar uma segunda conta.
- Criação da identidade e habilitação de acesso são decisões separadas.
- O resultado deve ser confirmado por provisioning status/log/reconciliation; aceite assíncrono da requisição não representa sucesso final.
- Falha persistente deve ser registrada e alertada conforme severidade definida.

### Cancelamento de admissão

Quando uma admissão já pré-provisionada for confirmadamente excluída/cancelada na Senior:

```text
PRE_PROVISIONED
      |
      | admissão excluída/cancelada
      v
ADMISSION_CANCELLED
```

A conta deve permanecer ou ser colocada em estado **desabilitado no AD DS**, não deve ser ativada na data originalmente prevista e deve continuar correlacionada à pessoa para auditoria e eventual reutilização futura pelo mesmo CPF.

O detalhe da detecção da exclusão está em `docs/integration/senior-admission-deletion.md`.

---

## Mover

Mover cobre qualquer mudança relevante em uma pessoa já correlacionada.

Exemplos:

- cargo;
- departamento;
- gestor;
- centro de custo;
- empresa/filial;
- localidade;
- nome civil/preferido quando aplicável;
- tipo de vínculo;
- situação de um vínculo que não represente o estado final da pessoa.

### Resultado esperado

```text
Senior state changes
        |
        v
Connector reads current effective state
        |
        v
Provisioning
        |
        v
Update approved AD DS attributes
```

### Regras

- O MVP atualiza apenas atributos aprovados no `attribute-mapping.md`.
- Mudança de cargo/departamento não concede automaticamente privilégios administrativos.
- Groups, roles e entitlements ficam fora do MVP, salvo decisão posterior explícita.
- Mudanças manuais no AD DS em atributos authoritative do Senior podem ser sobrescritas pelo provisioning, exceto exceções de naming formalmente registradas.
- Mudança de nome não deve criar nova identidade.
- Falhas que impeçam convergência do estado esperado devem ser rastreadas e alertadas conforme política operacional.
- Com múltiplos vínculos simultâneos, o conector deve avaliar todos os vínculos relevantes antes de projetar lifecycle e atributos no AD DS.

---

## Suspensão temporária

Suspensão temporária representa uma condição em que a relação com a organização continua existindo, mas a política IAM exige bloqueio temporário de acesso.

Exemplos:

- férias;
- afastamento cuja política exija bloqueio;
- licença;
- suspensão disciplinar;
- outras situações cadastradas na Senior e explicitamente classificadas pela organização como relevantes para acesso.

### Política padrão para férias

A política padrão definida para férias é:

> **Durante todo o período vigente de férias, a conta deve permanecer desabilitada no AD DS, salvo override individual válido autorizado pelo time de Gente e Gestão.**

No AD DS, a desabilitação deve utilizar o mecanismo suportado de conta desabilitada (`ACCOUNTDISABLE` / equivalente executado pelo provisioning), preservando o objeto da identidade.

A Microsoft documenta a desabilitação da conta como forma suportada de impedir novos logons no AD DS. Em ambiente híbrido, sessões/tokens já existentes em serviços de nuvem podem não ser encerrados imediatamente apenas com a mudança local; se a organização exigir revogação imediata também em Microsoft Entra/Microsoft 365 durante férias, esse controle deverá ser validado separadamente.

Para férias, a política não pressupõe reset de senha nem exclusão da identidade.

### Override individual

Overrides de férias/suspensão somente podem ser autorizados pelo time de **Gente e Gestão**.

Para férias, a vigência do override corresponde ao **período completo das férias**.

```text
Evento de férias
      |
      v
Política padrão: suspender acesso
      |
      +--> sem override válido -> TEMPORARILY_SUSPENDED
      |
      +--> override de Gente e Gestão -> aplicar exceção durante toda a vigência
```

A exceção individual deve ser auditável e registrar no mínimo:

- pessoa afetada;
- decisão efetiva;
- responsável/autorizador;
- justificativa;
- data/hora da autorização;
- início e fim da vigência.

A exceção individual **não pode** sobrepor estados de maior precedência, especialmente desligamento.

### Alteração da data de retorno

A data vigente na Senior é autoritativa.

Se o retorno for alterado:

```text
Data de retorno anterior
        |
        | alteração na Senior
        v
Nova data de retorno
        |
        v
recalcular suspensão / reativação / vigência do override
```

O IAM deve seguir a nova data e cancelar qualquer reativação agendada com base em uma data anterior.

### Afastamento sem data final

Quando um afastamento cuja política exige bloqueio não possuir data final:

- a conta permanece desabilitada;
- o lifecycle permanece `TEMPORARILY_SUSPENDED`;
- não deve existir reativação automática por passagem de tempo;
- a conta somente pode voltar a `ACTIVE` após novo estado autoritativo da Senior permitir o retorno.

Esse cenário é especialmente importante para a política futura de limpeza de contas: uma conta afastada por longo período **não pode ser confundida com uma conta de desligado**.

### Metadados de desabilitação no AD DS

O AD DS deve armazenar metadados controlados pelo IAM suficientes para distinguir o motivo e a data da desabilitação.

Modelo conceitual:

```text
iamLifecycleState = TEMPORARILY_SUSPENDED | ADMISSION_CANCELLED | TERMINATED
iamDisabledAt     = timestamp da desabilitação
iamDisableReason  = VACATION | LEAVE | LEAVE_NO_END_DATE | ADMISSION_CANCELLED | TERMINATION
```

Os nomes físicos dos atributos serão definidos no `attribute-mapping.md`.

Esses metadados são necessários porque uma rotina de limpeza baseada apenas em “conta desabilitada há mais de 30 dias” poderia apagar indevidamente uma conta de férias ou afastamento prolongado.

Portanto:

> **idade da desabilitação, isoladamente, nunca é critério suficiente para exclusão.**

A exclusão automática definitiva continua fora do MVP até aprovação da política de retenção.

### Regras obrigatórias

- Suspensão temporária não deve ser interpretada como Leaver.
- Suspensão temporária não deve excluir nem recriar a conta.
- O retorno deve reativar **a mesma identidade**, preservando CPF, correlação e objeto AD.
- Datas vigentes da Senior devem ser respeitadas.
- Um desligamento efetivo tem precedência sobre férias, afastamento, retorno e qualquer override individual.
- Falha ao aplicar bloqueio esperado deve gerar alerta conforme severidade definida.
- Falha ao reativar uma pessoa que deveria voltar a `ACTIVE` deve gerar alerta conforme severidade definida.

---

## Leaver

### Trigger funcional

A pessoa deixa de possuir vínculo elegível que justifique manutenção do acesso, conforme a regra de agregação de múltiplos vínculos.

Os códigos e situações exatos do Senior devem ser documentados no Discovery.

Exemplo importante:

```text
Mesmo CPF
├── CLT = TERMINATED
└── PJ  = ACTIVE

=> a identidade NÃO é TERMINATED
```

Somente o encerramento de um vínculo não é suficiente para desabilitar a identidade quando outro vínculo elegível permanecer ativo.

### Resultado esperado do MVP

```text
Effective person state becomes terminated
        |
        v
Connector determines TERMINATED
        |
        v
Provisioning disables AD DS identity
        |
        v
Confirm expected state
        |
        +--> success: audit
        |
        +--> failure: CRITICAL alert + retry/escalation
```

### Regra segura inicial

**Disable antes de Delete.** O MVP não deve excluir automaticamente contas do AD DS.

Falha ao desabilitar conta cujo estado esperado é `TERMINATED` é evento **CRITICAL**.

Nenhum override de férias, afastamento ou outra suspensão temporária pode impedir o bloqueio causado por desligamento efetivo da pessoa.

---

## Rehire

Rehire deve reutilizar a identidade existente quando o CPF for o mesmo e a correlação não apresentar conflito.

O naming existente deve ser preservado por padrão, conforme `naming-policy.md`.

Exemplo:

```text
TERMINATED
    |
    | novo vínculo elegível com mesmo CPF
    v
reativar a mesma conta AD
```

Não criar nova conta apenas porque matrícula, vínculo ou tipo de contratação mudou.

---

## Precedência de estados

Quando mais de uma situação estiver vigente simultaneamente, o sistema deve aplicar regra determinística de precedência.

Baseline funcional:

```text
TERMINATED / ADMISSION_CANCELLED
        >
TEMPORARILY_SUSPENDED
        >
ACTIVE
        >
PRE_PROVISIONED
```

`ADMISSION_CANCELLED` é terminal para aquela admissão e mantém a conta desabilitada, mas não representa uma pessoa que efetivamente iniciou e depois foi desligada.

Com múltiplos vínculos, a precedência só deve ser aplicada **depois** de calcular o estado agregado da pessoa.

---

## Política de tentativas e isolamento

Para falhas transitórias de provisioning:

```text
Tentativa 1 -> imediata
Tentativa 2 -> backoff
Tentativa 3 -> backoff
Falha após tentativa 3 -> alerta
```

São **3 tentativas no total por ciclo**, incluindo a primeira.

O backoff é progressivo e configurável. Quando o serviço remoto informar `Retry-After` ou mecanismo equivalente, o valor fornecido deve prevalecer.

A falha de um colaborador deve ser isolada. O processamento das demais pessoas continua normalmente e não pode ficar bloqueado por uma operação individual em retry/falha.

A reconciliação posterior pode iniciar um novo ciclo de convergência, preservando o histórico da falha e do alerta anterior.

---

## Cenários de exceção

| Cenário | Comportamento esperado |
|---|---|
| CPF ausente/inválido | Rejeitar registro e alertar conforme política |
| Duplicidade/conflito de CPF | Bloquear processamento da identidade |
| Match ambíguo no target | Não alterar contas automaticamente |
| Senior indisponível | Retry com backoff; não assumir desligamento |
| Entra indisponível/429 | Respeitar `Retry-After` quando disponível; retry isolado |
| Provisioning Agent offline | Manter acompanhamento e alertar |
| Registro inválido | Isolar falha ao colaborador afetado |
| Evento duplicado | Processamento idempotente |
| Evento perdido | Recuperar por reconciliation |
| Retorno de férias alterado | Seguir nova data da Senior |
| Férias sem override | Conta desabilitada durante toda a vigência |
| Férias com override Gente e Gestão | Aplicar exceção durante toda a vigência e auditar |
| Override + desligamento | Ignorar override para acesso; desligamento prevalece |
| Afastamento sem data final | Manter conta desabilitada até novo estado autoritativo |
| Admissão postergada após pré-provisionamento | Recalcular liberação conforme nova data |
| Admissão cancelada após pré-provisionamento | Manter/desabilitar conta; `ADMISSION_CANCELLED` |
| CLT encerrado + PJ ativo | Manter identidade conforme vínculo ativo elegível |
| Falha de uma identidade | Não bloquear processamento das demais |
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
- operações aceitas de forma assíncrona que nunca atingiram o estado final esperado;
- contas desabilitadas cujo lifecycle/motivo não corresponde ao estado esperado;
- divergências causadas por múltiplos vínculos processados fora de ordem.

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

## Referências

- `docs/identity/attribute-mapping.md`
- `docs/identity/naming-policy.md`
- `docs/requirements/audit-observability.md`
- `docs/integration/senior-admission-deletion.md`
- Microsoft Learn — UserAccountControl property flags: https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties
- Microsoft Learn — Revoke user access in Microsoft Entra ID: https://learn.microsoft.com/en-us/entra/identity/users/users-revoke-access
