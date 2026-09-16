# Attribute Mapping — Senior → Entra Provisioning → AD DS

- **Status:** Draft
- **Objetivo:** definir o contrato mínimo de dados usado pelo IAM sem expor informações desnecessárias de RH.

> Os nomes exatos dos campos e endpoints do Senior ainda precisam ser confirmados durante o Discovery. Esta tabela descreve a semântica esperada e não deve ser usada como contrato de API definitivo.

## Regras

- Apenas atributos necessários para identidade e lifecycle entram no conector.
- O Senior é authoritative source para os campos definidos como `Senior`.
- Campos calculados pelo IAM devem possuir regra determinística e documentada.
- O identificador de correlation não pode depender de nome, e-mail ou outro valor sujeito a alteração.
- Dados ausentes obrigatórios geram erro de validação e não devem produzir alterações parciais inseguras.

## Mapping inicial

| Informação | Campo Senior | SCIM / Inbound provisioning | AD DS | Obrigatório | Authority | Status |
|---|---|---|---|---|---|---|
| Identificador imutável da pessoa | **A confirmar** | `externalId` ou atributo dedicado | `employeeID` ou atributo dedicado | Sim | Senior | **Decisão aberta** |
| Matrícula | **A confirmar** | atributo enterprise/custom | `employeeNumber` | Sim | Senior | A validar |
| Primeiro nome | **A confirmar** | `name.givenName` | `givenName` | Sim | Senior | A validar |
| Sobrenome | **A confirmar** | `name.familyName` | `sn` | Sim | Senior | A validar |
| Nome completo | **A confirmar** | `displayName` | `displayName` | Sim | Senior | A validar |
| Nome preferido | **A confirmar** | custom/transform | a definir | Não | Senior | Opcional |
| Username | Calculado | `userName` | `sAMAccountName` | Sim | IAM policy | **Decisão aberta** |
| UPN | Calculado | mapping/transform | `userPrincipalName` | Sim | IAM policy | **Decisão aberta** |
| Cargo | **A confirmar** | `title` | `title` | Não | Senior | A validar |
| Departamento | **A confirmar** | `department` | `department` | Não | Senior | A validar |
| Empresa | **A confirmar** | `organization`/custom | `company` | Não | Senior | A validar |
| Centro de custo | **A confirmar** | custom | extension/custom attribute | Não | Senior | **Decisão aberta** |
| Gestor | **A confirmar** | `manager`/custom correlation | `manager` | Não | Senior | **Decisão aberta** |
| Tipo de vínculo | **A confirmar** | custom | custom/extension attribute | Não | Senior | A validar |
| Situação do vínculo | **A confirmar** | lifecycle input | account state | Sim | Senior | **Decisão aberta** |
| Data de admissão | **A confirmar** | `employeeHireDate` quando aplicável | custom/extension attribute | Sim | Senior | A validar |
| Data de desligamento | **A confirmar** | `employeeLeaveDateTime` quando aplicável | custom/extension attribute | Condicional | Senior | A validar |
| Localidade/unidade | **A confirmar** | custom | `physicalDeliveryOfficeName` ou custom | Não | Senior | A validar |

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

Caso algum dado sensível seja proposto futuramente, a inclusão deve passar por nova decisão arquitetural e revisão de segurança/privacidade.

## Regras de transformação a definir

### `sAMAccountName`

Precisamos definir:

- algoritmo de geração;
- tamanho máximo;
- normalização de acentos/caracteres;
- tratamento de nomes compostos;
- colisões;
- estabilidade após mudança de nome;
- comportamento em Rehire.

### `userPrincipalName`

Precisamos definir:

- suffix utilizado;
- relação com e-mail corporativo;
- alteração ou preservação após mudança de nome;
- tratamento de colisões.

### Manager

O manager deve ser resolvido usando uma chave estável da pessoa gestora e somente aplicado quando a identidade correspondente estiver correlacionada de forma inequívoca.

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
- Confirmar qual identificador sobrevive a transferência, desligamento e recontratação.
- Exportar os default attribute mappings da aplicação Entra de laboratório.
- Validar os atributos suportados no AD DS de destino.
