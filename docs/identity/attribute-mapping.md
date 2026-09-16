# Attribute Mapping — Senior → Entra Provisioning → AD DS

- **Status:** Draft
- **Objetivo:** definir o contrato mínimo de dados usado pelo IAM sem expor informações desnecessárias de RH.

> Os nomes exatos dos campos e endpoints do Senior ainda precisam ser confirmados durante o Discovery. Esta tabela descreve a semântica esperada e não deve ser usada como contrato de API definitivo.

## Regras

- Apenas atributos necessários para identidade e lifecycle entram no conector.
- O Senior é authoritative source para os campos definidos como `Senior`.
- A conta corporativa representa a **pessoa**, e não o vínculo trabalhista.
- O **CPF é a única chave autorizada para correlacionar a pessoa entre vínculos**.
- `Person ID`, matrícula, nome, e-mail, UPN e outros identificadores não podem ser usados como fallback de correlação da pessoa.
- Campos calculados pelo IAM devem possuir regra determinística e documentada.
- O CPF deve ser protegido e não deve ser propagado ao AD DS, Entra, logs ou portal quando não houver necessidade técnica.
- Dados ausentes obrigatórios geram erro de validação e não devem produzir alterações parciais inseguras.
- O AD DS deve possuir metadados suficientes para diferenciar uma conta desabilitada por desligamento/cancelamento de uma conta temporariamente suspensa.

Ver também:

- [`identity-correlation.md`](./identity-correlation.md)
- [`joiner-mover-leaver.md`](./joiner-mover-leaver.md)
- [`naming-policy.md`](./naming-policy.md)

## Mapping inicial

| Informação | Campo Senior | Uso no IAM / SCIM | AD DS | Obrigatório | Authority | Status |
|---|---|---|---|---|---|---|
| CPF | **A confirmar** | **Chave exclusiva de correlação da pessoa; representação protegida no IAM** | **Não persistir por padrão** | Sim | Senior | **Decisão funcional definida; campo técnico a validar** |
| Person ID / referência interna Senior | **A confirmar** | Referência técnica/auditoria; não usar para matching de pessoa | Não necessário por padrão | Não | Senior | A validar |
| Matrícula / identificador do vínculo | **A confirmar** | atributo enterprise/custom | `employeeNumber` | Sim | Senior | A validar |
| Primeiro nome | **A confirmar** | `name.givenName` | `givenName` | Sim | Senior | A validar |
| Sobrenome | **A confirmar** | `name.familyName` | `sn` | Sim | Senior | A validar |
| Nome completo | **A confirmar** | `displayName` | `displayName` | Sim | Senior | A validar |
| Nome preferido | **A confirmar** | custom/transform | a definir | Não | Senior | Opcional |
| Username | Calculado | `userName` | `sAMAccountName` | Sim | IAM policy | Política proposta em `naming-policy.md` |
| UPN | Calculado | mapping/transform | `userPrincipalName` | Sim | IAM policy | Política proposta em `naming-policy.md` |
| Cargo | **A confirmar** | `title` | `title` | Não | Senior | A validar |
| Departamento | **A confirmar** | `department` | `department` | Não | Senior | A validar |
| Empresa | **A confirmar** | `organization`/custom | `company` | Não | Senior | A validar |
| Centro de custo | **A confirmar** | custom | extension/custom attribute | Não | Senior | **Decisão aberta** |
| Gestor | **A confirmar** | `manager`/custom correlation | `manager` | Não | Senior | **Decisão aberta** |
| Tipo de vínculo | **A confirmar** | custom | custom/extension attribute | Não | Senior | A validar |
| Situação do vínculo | **A confirmar** | lifecycle input | não projetar isoladamente como estado da conta | Sim | Senior | Deve compor estado efetivo da pessoa |
| Data de admissão | **A confirmar** | `employeeHireDate` quando aplicável | custom/extension attribute | Sim | Senior | A validar |
| Data de desligamento | **A confirmar** | `employeeLeaveDateTime` quando aplicável | custom/extension attribute | Condicional | Senior | A validar |
| Localidade/unidade | **A confirmar** | custom | `physicalDeliveryOfficeName` ou custom | Não | Senior | A validar |
| Estado efetivo IAM | Calculado | `iamLifecycleState` lógico | **atributo custom/extension a definir** | Sim | IAM policy | Necessário para auditoria/retention |
| Data/hora da desabilitação | Calculado no momento da ação | `iamDisabledAt` lógico | **atributo custom/extension a definir** | Condicional | IAM policy | Necessário quando conta desabilitada |
| Motivo da desabilitação | Calculado pela regra de lifecycle | `iamDisableReason` lógico | **atributo custom/extension a definir** | Condicional | IAM policy | Necessário quando conta desabilitada |

## Tratamento do CPF

O CPF faz parte do contrato IAM **exclusivamente porque é a chave de identidade da pessoa**.

Seu uso deve seguir minimização de exposição:

- normalizar e validar antes do matching;
- não usar o valor em texto aberto como correlation ID;
- não registrar o valor em logs ou alertas;
- não exibir no portal de auditoria por padrão;
- não gravar como atributo visível no AD DS;
- não enviar ao Entra Provisioning se o provisioning técnico puder operar usando a correlação já mantida pelo IAM;
- proteger a representação persistida no banco do IAM conforme decisão de segurança específica.

O fato de o CPF ser a chave funcional não implica que o número precise circular por todos os componentes. Após a correlação, o IAM deve manter a associação com o objeto AD correspondente.

## Metadados de lifecycle no AD DS

O AD DS precisa permitir distinguir **o motivo da desabilitação** sem depender apenas de `userAccountControl`, pois esse atributo informa que a conta está desabilitada, mas não representa o motivo funcional nem a data original definida pelo IAM.

Modelo lógico:

```text
iamLifecycleState
iamDisabledAt
iamDisableReason
```

Valores iniciais esperados para `iamLifecycleState`:

```text
PRE_PROVISIONED
ACTIVE
TEMPORARILY_SUSPENDED
ADMISSION_CANCELLED
TERMINATED
```

Valores iniciais esperados para `iamDisableReason`:

```text
VACATION
LEAVE
LEAVE_NO_END_DATE
ADMISSION_CANCELLED
TERMINATION
```

### Finalidade

Esses metadados devem suportar:

- auditoria do estado efetivo da identidade;
- reconciliação;
- investigação de conta desabilitada;
- prevenção de exclusão indevida de conta em férias/afastamento;
- futura política de retenção/limpeza de contas desabilitadas.

### Regra de segurança para limpeza

Uma futura rotina que avalie contas desabilitadas há mais de 30 dias **não deve utilizar apenas a data da desabilitação**.

Exemplo:

```text
iamLifecycleState = TEMPORARILY_SUSPENDED
iamDisableReason  = LEAVE_NO_END_DATE
iamDisabledAt     = há 90 dias

=> NÃO excluir por idade
```

A elegibilidade para exclusão deverá depender de lifecycle/motivo aprovado pela política de retenção, por exemplo `TERMINATED` e, se aprovado, `ADMISSION_CANCELLED`.

A exclusão automática definitiva continua fora do MVP.

### Atributo físico

Os nomes `iamLifecycleState`, `iamDisabledAt` e `iamDisableReason` são **nomes lógicos**, não nomes de schema AD já aprovados.

Durante o POC/mapping deve ser definido se serão utilizados:

- atributos de extensão já disponíveis e aprovados no schema;
- atributos customizados existentes na organização;
- outra estratégia suportada pelo Entra Provisioning/AD DS.

Evitar reutilizar campos de texto livre destinados a uso humano se isso puder causar sobrescrita ou ambiguidade operacional.

## Estado técnico da conta

Para estados que exigem bloqueio (`TEMPORARILY_SUSPENDED`, `ADMISSION_CANCELLED` e `TERMINATED`), a política prevê conta desabilitada no AD DS.

A Microsoft documenta a flag `ACCOUNTDISABLE` em `userAccountControl` como o estado de conta desabilitada no AD DS.

Em ambiente híbrido, esse estado local não deve ser confundido com revogação imediata de todas as sessões/tokens já emitidos na nuvem. A necessidade de revogação adicional no Entra deve ser validada por cenário.

## Dados explicitamente fora do contrato IAM

O conector não deve solicitar, persistir ou registrar em logs os seguintes grupos de dados:

- salário e remuneração;
- folha de pagamento;
- dados bancários;
- benefícios;
- dados médicos ou de saúde;
- dependentes;
- informações fiscais não necessárias para identidade;
- documentos pessoais sem necessidade técnica comprovada.

O CPF é a exceção necessária para correlação de identidade, com finalidade limitada e controles específicos.

Caso algum outro dado sensível seja proposto futuramente, a inclusão deve passar por nova decisão arquitetural e revisão de segurança/privacidade.

## Regras de transformação

### `sAMAccountName` e `userPrincipalName`

As regras propostas estão documentadas em [`naming-policy.md`](./naming-policy.md).

Ainda precisam de aprovação final do time para:

- sufixo de UPN;
- limites/truncamento;
- fallback final de colisão;
- gatilho final para rename;
- comportamento de aliases.

### Manager

O manager deve ser aplicado somente quando a identidade correspondente da pessoa gestora puder ser resolvida de forma inequívoca. A implementação não deve criar uma segunda regra de correlação de pessoas diferente da definida em `identity-correlation.md`.

### Múltiplos vínculos

Quando uma pessoa possuir mais de um vínculo simultaneamente, não se deve assumir que matrícula, cargo, departamento, gestor, empresa ou centro de custo de qualquer registro isolado representa automaticamente a identidade.

A regra de precedência por vínculo/atributo ainda deve ser aprovada conforme Issue #8.

## Critério para aprovar um atributo

Cada atributo só entra em produção quando estas perguntas estiverem respondidas:

1. Quem é o owner do dado?
2. Qual é o campo exato no Senior?
3. O conector precisa realmente desse dado?
4. Qual é o target no AD DS?
5. O valor pode ser alterado manualmente no AD DS?
6. O que acontece quando o valor fica vazio?
7. O que acontece quando o valor muda?
8. Há dado pessoal/sensível envolvido?
9. Como o campo é testado?

## Próximas validações

- Identificar endpoints reais do Senior.
- Testar conta técnica com permissões mínimas.
- Confirmar o campo de CPF e sua disponibilidade para todos os vínculos elegíveis.
- Confirmar que matrícula identifica o vínculo e pode mudar sem representar nova pessoa.
- Definir normalização, validação e proteção da chave CPF no IAM.
- Definir a chave técnica usada pelo IAM para referenciar a conta já correlacionada no Entra/AD DS.
- Selecionar os atributos físicos do AD DS para lifecycle, data e motivo da desabilitação.
- Validar leitura/escrita desses atributos pelo Provisioning Agent.
- Validar que conta `TEMPORARILY_SUSPENDED` não entra em eventual limpeza por idade.
- Exportar os default attribute mappings da aplicação Entra de laboratório.
- Validar os atributos suportados no AD DS de destino.

## Referências

- Microsoft Learn — UserAccountControl property flags: https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties
- Microsoft Learn — Revoke user access in Microsoft Entra ID: https://learn.microsoft.com/en-us/entra/identity/users/users-revoke-access
