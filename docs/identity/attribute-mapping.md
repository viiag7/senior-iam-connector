# Attribute Mapping — Senior → Entra Provisioning → AD DS

- **Status:** Draft — contrato Senior parcialmente validado em Swagger
- **Objetivo:** definir o contrato mínimo de dados usado pelo IAM sem expor informações desnecessárias de RH.

> O contrato de `GET /getEmployee` e `GET /getPerson` foi validado no Swagger do ambiente Senior X. Os campos abaixo marcados como confirmados refletem o schema observado. Mapeamentos físicos de alguns atributos no AD DS, regras de manager, situação efetiva do vínculo e decisões de múltiplos vínculos ainda permanecem em Discovery.

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
- O `GET /getEmployee` deve ser a fonte preferencial do MVP, pois já agrega dados do vínculo, pessoa, cargo, departamento, centro de custo, empresa, contato e referência de hierarquia.
- O `GET /getPerson` deve ser usado apenas quando houver necessidade específica de enriquecer dados de pessoa não presentes no `getEmployee`.

Ver também:

- [`identity-correlation.md`](./identity-correlation.md)
- [`joiner-mover-leaver.md`](./joiner-mover-leaver.md)
- [`naming-policy.md`](./naming-policy.md)

## Contrato Senior validado

Endpoint principal do MVP:

```text
GET /hcm/employeejourney/getEmployee
```

Endpoint complementar:

```text
GET /hcm/employeejourney/getPerson
```

O schema observado de `getEmployee` contém no mesmo registro:

- identificadores e datas do vínculo;
- tipo de contrato e relação empregatícia;
- cargo;
- departamento;
- centro de custo;
- empresa;
- dados básicos de pessoa;
- e-mail e telefone do vínculo;
- posto de trabalho;
- referência do item de hierarquia.

## Mapping inicial

| Informação | Campo Senior | Uso no IAM / SCIM | AD DS | Obrigatório | Authority | Status |
|---|---|---|---|---|---|---|
| CPF | `person.cpf` | **Chave exclusiva de correlação da pessoa; representação protegida no IAM** | **Não persistir por padrão** | Sim | Senior | **Campo confirmado no Swagger** |
| Person ID / referência interna Senior | `person.id` | Referência técnica/auditoria; não usar para matching de pessoa | Não necessário por padrão | Não | Senior | **Campo confirmado** |
| Employee ID / referência interna do vínculo | `id` | Referência técnica do vínculo e auditoria | custom/extension somente se necessário | Não | Senior | **Campo confirmado** |
| Matrícula / identificador do vínculo | `registerNumber` | atributo enterprise/custom | `employeeNumber` **a validar no mapping Entra/AD** | Sim | Senior | **Campo confirmado; target físico pendente** |
| Primeiro nome | `person.firstname` | `name.givenName` | `givenName` | Sim | Senior | **Confirmado** |
| Sobrenome | `person.lastname` | `name.familyName` | `sn` | Sim | Senior | **Confirmado** |
| Nome completo | `person.fullName` | `displayName` | `displayName` | Sim | Senior | **Confirmado** |
| Nome social / preferido | `person.socialName` | custom/transform conforme política de naming | a definir | Não | Senior | **Campo confirmado; regra de uso pendente** |
| Apelido | `person.nickname` | opcional / não usar por padrão em naming | a definir | Não | Senior | **Campo confirmado; fora do mínimo inicial** |
| Username | Calculado | `userName` | `sAMAccountName` | Sim | IAM policy | Política proposta em `naming-policy.md` |
| UPN | Calculado | mapping/transform | `userPrincipalName` | Sim | IAM policy | Política proposta em `naming-policy.md` |
| E-mail corporativo | `emails[].email` | `emails.work` / atributo de contato | `mail` e/ou `proxyAddresses` conforme política | Condicional | Senior | **Campo confirmado; regra de seleção/target pendente** |
| Cargo | `jobPosition.name` | `title` | `title` | Não | Senior | **Confirmado** |
| Código do cargo | `jobPosition.codcar` | custom | extension/custom attribute | Não | Senior | **Confirmado; target físico pendente** |
| Departamento | `department.name` | `department` | `department` | Não | Senior | **Confirmado** |
| Código do departamento | `department.code` | custom | extension/custom attribute | Não | Senior | **Confirmado; target físico pendente** |
| Empresa | `employer.companyName` | `organization`/custom | `company` | Não | Senior | **Confirmado** |
| Código da empresa | `employer.numemp` | custom | extension/custom attribute | Não | Senior | **Confirmado; target físico pendente** |
| Centro de custo | `costCenter.name` | custom | extension/custom attribute | Não | Senior | **Confirmado; target físico pendente** |
| Código do centro de custo | `costCenter.codccu` | custom | extension/custom attribute | Não | Senior | **Confirmado; target físico pendente** |
| Gestor | `workstation.hierarchyItem.id` como referência de hierarquia | resolver hierarquia → pessoa gestora → identidade IAM | `manager` | Não | Senior + IAM correlation | **Referência confirmada; resolução do gestor pendente** |
| Tipo de contrato | `contractType` | regra de elegibilidade / custom | custom/extension attribute se necessário | Não | Senior | **Confirmado** |
| Tipo de colaborador | `employeeType` | regra de elegibilidade / custom | custom/extension attribute se necessário | Não | Senior | **Confirmado** |
| Relação empregatícia | `employmentrelationshiptype` | regra de elegibilidade / custom | custom/extension attribute se necessário | Não | Senior | **Confirmado** |
| Situação efetiva do vínculo | **Não há campo de status explícito confirmado no schema observado** | lifecycle input calculado a partir dos dados autoritativos e regras aprovadas | não projetar isoladamente como estado da conta | Sim | Senior + IAM policy | **Pendente de definição funcional/técnica** |
| Data de admissão | `hireDate` | `employeeHireDate` quando aplicável / lifecycle | custom/extension attribute se necessário | Sim | Senior | **Confirmado** |
| Data de desligamento | `dismissalDate` | `employeeLeaveDateTime` quando aplicável / lifecycle | custom/extension attribute se necessário | Condicional | Senior | **Confirmado** |
| Posto / unidade organizacional | `workstation.workstationGroup.name` | custom | `physicalDeliveryOfficeName` ou custom | Não | Senior | **Campo confirmado; semântica/target pendente** |
| Código do posto | `workstation.workstationGroup.postra` | custom | extension/custom attribute | Não | Senior | **Confirmado; target físico pendente** |
| Telefone do vínculo | `phoneContact[]` | atributo de contato quando corporativo | `telephoneNumber` / `mobile` conforme tipo | Não | Senior | **Campo confirmado; filtro de tipo pendente** |
| Estado efetivo IAM | Calculado | `iamLifecycleState` lógico | **atributo custom/extension a definir** | Sim | IAM policy | Necessário para auditoria/retention |
| Data/hora da desabilitação | Calculado no momento da ação | `iamDisabledAt` lógico | **atributo custom/extension a definir** | Condicional | IAM policy | Necessário quando conta desabilitada |
| Motivo da desabilitação | Calculado pela regra de lifecycle | `iamDisableReason` lógico | **atributo custom/extension a definir** | Condicional | IAM policy | Necessário quando conta desabilitada |

## Campos confirmados no schema que não entram no contrato IAM padrão

O `getEmployee` agrega vários campos de pessoa que não são necessários para provisionamento de identidade. Eles devem ser ignorados pelo conector por padrão e não devem ser persistidos ou registrados em logs:

```text
person.birthday
person.gender
person.maritalstatus
person.race
person.ethnicity
person.nis
person.nationality
person.naturality
person.educationDegree
person.address
person.disabilities
```

Também não entram no contrato mínimo inicial, salvo decisão específica:

```text
workShift
historicStability
employer.cnpj
employer.cnae
employer.address
department.address
costCenter.company
```

A existência desses campos na resposta reforça a necessidade de minimização no processamento e de evitar logging de payload completo.

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

O `getEmployee` não expõe diretamente o usuário gestor. O schema validado disponibiliza:

```text
workstation.hierarchyItem.id
```

Esse valor deve ser tratado como **referência de hierarquia**, não como valor direto de `manager`.

Fluxo esperado:

```text
workstation.hierarchyItem.id
        |
        v
resolver item de hierarquia / posição superior
        |
        v
identificar pessoa ou vínculo do gestor
        |
        v
correlacionar com identidade IAM
        |
        v
resolver distinguishedName no AD DS
        |
        v
manager
```

O manager só deve ser aplicado quando a pessoa gestora puder ser resolvida de forma inequívoca. A implementação não deve criar uma segunda regra de correlação de pessoas diferente da definida em `identity-correlation.md`.

Ainda deve ser identificado no Senior X qual endpoint/query permite resolver o `hierarchyItem.id` até a pessoa/vínculo gestor.

### Situação do vínculo

O schema observado do `getEmployee` não apresentou um campo simples equivalente a `status = ACTIVE/TERMINATED`.

O lifecycle não deve assumir que `dismissalDate == null` é suficiente para classificar todos os casos. A situação efetiva deve considerar as regras funcionais aprovadas e, quando necessário, fontes complementares para afastamento, férias, cancelamento e demais estados.

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

- Testar conta técnica Senior X com permissões mínimas apenas para `employeejourney/getEmployee` e recursos estritamente necessários.
- Confirmar a disponibilidade de `person.cpf` para todos os vínculos elegíveis e os impactos de abrangência/permissão.
- Confirmar que `registerNumber` identifica o vínculo e pode mudar sem representar nova pessoa.
- Definir normalização, validação e proteção da chave CPF no IAM.
- Definir a chave técnica usada pelo IAM para referenciar a conta já correlacionada no Entra/AD DS.
- Identificar a query/endpoint Senior que resolve `workstation.hierarchyItem.id` até o gestor.
- Definir regra de seleção de `emails[]` quando houver mais de um e-mail e confirmar qual representa e-mail corporativo.
- Definir regra de seleção de `phoneContact[]` quando houver múltiplos contatos.
- Validar a semântica de `workstation.workstationGroup.name` como posto/localidade/unidade.
- Identificar a fonte autoritativa da situação efetiva do vínculo para férias, afastamentos, cancelamento e demais estados não representados por um status explícito no schema observado.
- Selecionar os atributos físicos do AD DS para códigos organizacionais, lifecycle, data e motivo da desabilitação.
- Validar leitura/escrita desses atributos pelo Provisioning Agent.
- Validar que conta `TEMPORARILY_SUSPENDED` não entra em eventual limpeza por idade.
- Exportar os default attribute mappings da aplicação Entra de laboratório.
- Confirmar se matrícula deve mapear fisicamente para `employeeNumber`, `employeeID` ou outro atributo suportado pelo fluxo de inbound provisioning.
- Validar os atributos suportados no AD DS de destino.
- Testar que o papel técnico do Senior não possui acesso a remuneração, folha e demais dados fora do escopo IAM.

## Referências

- Senior X API Portal — Jornada do Colaborador / `getEmployee` (schema validado no Swagger do ambiente).
- Senior X API Portal — Jornada do Colaborador / `getPerson` (schema validado no Swagger do ambiente).
- Microsoft Learn — UserAccountControl property flags: https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties
- Microsoft Learn — Revoke user access in Microsoft Entra ID: https://learn.microsoft.com/en-us/entra/identity/users/users-revoke-access
