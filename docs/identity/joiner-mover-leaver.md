# Joiner–Mover–Leaver (JML)

- **Status:** Draft
- **Objetivo:** definir como eventos do vínculo no Senior afetam a identidade no AD DS.

## Princípio

O Senior é a fonte autoritativa para o lifecycle do colaborador. O conector transforma o estado de RH em registros de identidade, enquanto o Microsoft Entra Provisioning Service determina e executa as operações de provisioning configuradas.

O lifecycle deve ser baseado em **estado e regras determinísticas**, não em tickets ou interpretações manuais.

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

## Reconciliation

Mesmo que a Senior disponibilize eventos/webhooks, deve existir reconciliação periódica para identificar:

- registros não processados;
- alterações perdidas;
- falhas temporárias;
- divergências persistentes;
- identities no target sem estado esperado na fonte.

A frequência será definida após entendermos volume, limites das APIs e capacidades de delta da Senior.

## Definition of Done por cenário

Um cenário JML só está pronto quando:

- regra funcional está documentada;
- dados de entrada estão definidos;
- mapping está aprovado;
- teste automatizado existe;
- teste ponta a ponta foi executado em laboratório;
- logs e correlation ID permitem rastrear o processamento;
- falhas e retry foram testados;
- comportamento de rollback/reprocessamento está documentado.
