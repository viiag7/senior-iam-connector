# Joiner–Mover–Leaver (JML)

- **Status:** Draft
- **Objetivo:** definir como eventos do vínculo no Senior afetam a identidade no AD DS.

## Princípio

O Senior é a fonte autoritativa para o lifecycle do colaborador. O conector transforma o estado de RH em registros de identidade, enquanto o Microsoft Entra Provisioning Service determina e executa as operações de provisioning configuradas.

O lifecycle deve ser baseado em **estado e regras determinísticas**, não em tickets ou interpretações manuais.

O modelo funcional deve considerar não apenas Joiner, Mover, Leaver e Rehire, mas também **suspensões temporárias de acesso**, como férias, determinados afastamentos e outras situações que a organização decidir mapear.

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

A identidade pode precisar ser criada **antes da data efetiva de admissão** para permitir preparação de onboarding, por exemplo:

- criação antecipada da conta;
- configuração de equipamentos;
- preparação de aplicações corporativas;
- associação futura de recursos necessários ao primeiro dia de trabalho.

A antecedência deve ser configurável e definida por regra de negócio. A criação antecipada da identidade não implica necessariamente que a conta já possa ser utilizada pelo colaborador.

Devem ser definidos separadamente:

- número de dias de antecedência;
- se a conta nasce habilitada ou desabilitada;
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
- O resultado deve ser consultado nos provisioning logs; HTTP 202 significa apenas que o payload foi aceito para processamento.

### Critério de aceite do MVP

Um colaborador fictício elegível no Senior deve resultar em exatamente uma conta no AD DS com os atributos mínimos aprovados e audit trail correspondente.

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

---

## Suspensão temporária

Suspensão temporária representa uma indisponibilidade de acesso durante a qual o vínculo continua existindo.

Exemplos potenciais:

- férias;
- afastamento médico quando a política exigir bloqueio;
- licença;
- suspensão disciplinar;
- outras situações cadastradas na Senior e explicitamente classificadas pela organização como impeditivas de acesso.

Nem todo status de afastamento precisa necessariamente desabilitar a conta. O comportamento deve ser determinado por uma tabela de regras aprovada por RH/IAM/Segurança.

### Resultado esperado

```text
ACTIVE
  |
  | início de férias/afastamento/suspensão elegível
  v
TEMPORARILY_SUSPENDED
  |
  | retorno + vínculo ainda válido
  v
ACTIVE
```

### Regras obrigatórias

- Suspensão temporária deve desabilitar a conta quando a situação estiver configurada para isso.
- O retorno deve reativar **a mesma identidade**, preservando a correlation key.
- Suspensão temporária não deve ser interpretada como Leaver.
- Suspensão temporária não deve excluir nem recriar a conta.
- Datas de início e fim devem ser respeitadas quando disponíveis.
- Alteração da data de retorno deve alterar o momento previsto de reativação.
- Se não existir data de retorno, a conta permanece suspensa até que o estado autoritativo permita reativação.
- Um desligamento efetivo tem precedência sobre retorno de férias/afastamento e deve impedir reativação indevida.

### Casos que precisam ser decididos

- todas as férias desabilitam acesso ou não;
- quais tipos/códigos de afastamento desabilitam acesso;
- horário de início do bloqueio;
- horário de retorno;
- exceções aprovadas;
- comportamento quando eventos se sobrepõem.

---

## Leaver

### Trigger funcional

O vínculo atinge um estado de desligamento reconhecido pelo contrato IAM.

Os códigos e situações exatos do Senior devem ser documentados no Discovery.

### Resultado esperado do MVP

```text
Senior employment becomes ineligible/terminated
        |
        v
Connector submits authoritative current state
        |
        v
Provisioning rules disable the AD DS identity
        |
        v
Audit + provisioning status
```

### Regras a definir antes de produção

- horário efetivo do disable;
- uso da data de desligamento versus alteração imediata de status;
- comportamento para desligamento futuro/agendado;
- exceções aprovadas pelo RH/Segurança;
- grupos privilegiados e sessões — fora ou dentro de uma fase posterior;
- retenção da conta desabilitada;
- OU de contas desabilitadas;
- momento e autoridade para exclusão definitiva.

### Regra segura inicial

**Disable antes de Delete.** O MVP não deve excluir automaticamente contas do AD DS.

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

Quando mais de uma situação de RH estiver vigente simultaneamente, o sistema deve aplicar uma regra determinística de precedência. Como baseline funcional, considerar:

```text
TERMINATED / LEAVER
        >
TEMPORARILY_SUSPENDED
        >
ACTIVE
        >
PRE_PROVISIONED
```

Exemplo: um colaborador em férias que seja desligado durante o período não deve ser reativado automaticamente na data originalmente prevista para retorno das férias.

A tabela final de precedência será definida nas regras de negócio.

---

## Cenários de exceção

O fluxo deve prever ao menos:

| Cenário | Comportamento esperado |
|---|---|
| Identificador ausente | Rejeitar registro e alertar |
| Duplicidade de correlation key | Bloquear processamento da identidade |
| Match ambíguo no target | Não alterar contas automaticamente |
| Senior indisponível | Retry com backoff; não assumir desligamento |
| Entra indisponível/429 | Retry respeitando throttling |
| Provisioning Agent offline | Manter acompanhamento e alertar |
| Registro inválido | Isolar falha ao colaborador afetado |
| Evento duplicado | Processamento idempotente |
| Evento perdido | Recuperar por reconciliation |
| Retorno de férias após desligamento | Manter conta desabilitada |
| Afastamento sem data final | Permanecer suspenso até novo estado autoritativo |
| Admissão postergada após pré-provisionamento | Recalcular liberação conforme política |
| Admissão cancelada após pré-provisionamento | Bloquear/desabilitar conforme regra definida e auditar |

## Reconciliation

Mesmo que a Senior disponibilize eventos/webhooks, deve existir reconciliação periódica para identificar:

- registros não processados;
- alterações perdidas;
- falhas temporárias;
- divergências persistentes;
- identities no target sem estado esperado na fonte;
- contas temporariamente suspensas que deveriam ter sido reativadas ou mantidas bloqueadas.

A frequência será definida após entendermos volume, limites das APIs e capacidades de delta da Senior.

## Definition of Done por cenário

Um cenário JML/suspensão só está pronto quando:

- regra funcional está documentada;
- dados de entrada estão definidos;
- mapping está aprovado;
- teste automatizado existe;
- teste ponta a ponta foi executado em laboratório;
- logs e correlation ID permitem rastrear o processamento;
- falhas e retry foram testados;
- comportamento de rollback/reprocessamento está documentado.
