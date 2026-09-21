# Attribute Mapping — Senior → Entra Provisioning → AD DS

- **Status:** Draft — contrato Senior parcialmente validado em Swagger
- **Objetivo:** definir o contrato mínimo de dados usado pelo IAM sem expor informações desnecessárias de RH.

> O contrato de `GET /getEmployee` e `GET /getPerson` foi validado no Swagger do ambiente Senior X. Os campos abaixo marcados como confirmados refletem o schema observado. A resolução de `manager` pela hierarquia do posto está definida neste documento; permanece pendente apenas a política funcional para posições superiores existentes porém sem ocupante resolvível. Mapeamentos físicos de alguns atributos no AD DS e a situação efetiva do vínculo ainda permanecem em Discovery.

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
GET /hcm/employeejourney/queries/getEmployee
```

Endpoint complementar:

```text
GET /hcm/employeejourney/queries/getPerson
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
| Código do cargo | `jobPosition.codcar` | custom | `seniorIamJobCode` | Não | Senior | **Confirmado; atributo custom definido** |
| Departamento | `department.name` | `department` | `department` | Não | Senior | **Confirmado** |
| Código do departamento | `department.code` | custom | `seniorIamDepartmentCode` | Não | Senior | **Confirmado; atributo custom definido** |
| Empresa | `employer.companyName` | `organization`/custom | `company` | Não | Senior | **Confirmado** |
| Código da empresa | `employer.numemp` | custom | `seniorIamCompanyCode` | Não | Senior | **Confirmado; atributo custom definido** |
| Centro de custo | `costCenter.name` | custom | `seniorIamCostCenter` | Não | Senior | **Confirmado; atributo custom definido** |
| Código do centro de custo | `costCenter.codccu` | custom | `seniorIamCostCenterCode` | Não | Senior | **Confirmado; atributo custom definido** |
| Gestor | `workstation.hierarchyItem.id` como referência de hierarquia | resolver hierarquia → pessoa gestora → identidade IAM | `manager` | Não | Senior + IAM correlation | **Referência confirmada; resolução do gestor pendente** |
| Tipo de contrato | `contractType` | regra de elegibilidade / custom | `seniorIamContractType` | Não | Senior | **Confirmado; atributo custom definido** |
| Tipo de colaborador | `employeeType` | regra de elegibilidade / custom | `seniorIamEmployeeType` | Não | Senior | **Confirmado; atributo custom definido** |
| Relação empregatícia | `employmentrelationshiptype` | regra de elegibilidade / custom | `seniorIamEmploymentRelationshipType` | Não | Senior | **Confirmado; atributo custom definido** |
| Situação efetiva do vínculo | **Não há campo de status explícito confirmado no schema observado** | lifecycle input calculado a partir dos dados autoritativos e regras aprovadas | não projetar isoladamente como estado da conta | Sim | Senior + IAM policy | **Pendente de definição funcional/técnica** |
| Data de admissão | `hireDate` | `employeeHireDate` quando aplicável / lifecycle | custom/extension attribute se necessário | Sim | Senior | **Confirmado** |
| Data de desligamento | `dismissalDate` | `employeeLeaveDateTime` quando aplicável / lifecycle | custom/extension attribute se necessário | Condicional | Senior | **Confirmado** |
| Posto / unidade organizacional | `workstation.workstationGroup.name` | custom | `physicalDeliveryOfficeName` ou custom | Não | Senior | **Campo confirmado; semântica/target pendente** |
| Código do posto | `workstation.workstationGroup.postra` | custom | `seniorIamWorkstationCode` | Não | Senior | **Confirmado; atributo custom definido** |
| Telefone do vínculo | `phoneContact[]` | atributo de contato quando corporativo | `telephoneNumber` / `mobile` conforme tipo | Não | Senior | **Campo confirmado; filtro de tipo pendente** |
| Estado operacional IAM | Calculado a partir da Senior + política IAM | `employmentStatus` lógico | `seniorIamEmploymentStatus` | Sim | IAM policy | **Atributo custom definido** |
| Data/hora da última mudança de estado | Calculado no momento da transição | `statusChangedAt` lógico | `seniorIamStatusChangedAt` | Sim | IAM policy | **Atributo custom definido** |
| Data/hora da desabilitação | Calculado no momento da ação | `disabledAt` lógico | `seniorIamDisabledAt` | Condicional | IAM policy | **Atributo custom definido** |

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

## Atributos customizados do AD DS

Foi decidido estender o schema do AD DS em vez de reutilizar genericamente `extensionAttribute1..15`. Os nomes abaixo são os `ldapDisplayName` propostos para o projeto e devem receber OIDs próprios no processo de extensão de schema:

| Atributo AD DS | Tipo sugerido | Origem / finalidade |
|---|---|---|
| `seniorIamJobCode` | Directory String | `jobPosition.codcar` |
| `seniorIamDepartmentCode` | Directory String | `department.code` |
| `seniorIamCostCenter` | Directory String | `costCenter.name` |
| `seniorIamCostCenterCode` | Directory String | `costCenter.codccu` |
| `seniorIamCompanyCode` | Integer/String conforme validação de schema | `employer.numemp` |
| `seniorIamWorkstationCode` | Directory String | `workstation.workstationGroup.postra` |
| `seniorIamContractType` | Directory String | `contractType` |
| `seniorIamEmployeeType` | Directory String | `employeeType` |
| `seniorIamEmploymentRelationshipType` | Directory String | `employmentrelationshiptype` |
| `seniorIamEmploymentStatus` | Directory String | estado operacional projetado pelo IAM |
| `seniorIamStatusChangedAt` | Generalized Time | data/hora da última transição de estado |
| `seniorIamDisabledAt` | Generalized Time | data/hora efetiva em que a conta foi desabilitada |

Valores iniciais permitidos para `seniorIamEmploymentStatus`:

```text
PRE_PROVISIONED
ACTIVE
VACATION
LEAVE
ADMISSION_CANCELLED
TERMINATED
```

Esse atributo é a referência operacional para distinguir, no AD DS, conta ativa, pré-provisionada, em férias, afastada, com admissão cancelada ou desligada.

> A extensão de schema deve ser executada de forma controlada em laboratório antes de produção. Nome, OID, sintaxe, single/multi-valued e replicação devem ser validados com o time de AD DS.

## Tratamento do CPF

O CPF faz parte do contrato IAM **exclusivamente porque é a chave de identidade da pessoa**.

Para a primeira fase foi aceita a persistência do CPF em texto claro **somente no storage interno do IAM**, para simplificar a correlação inicial. Essa decisão é um risco conscientemente aceito para o MVP e deve ser reavaliada antes de ampliar o escopo ou a exposição do serviço.

Mesmo nessa fase:

- normalizar e validar antes do matching;
- não usar o CPF como correlation ID público;
- não registrar o valor em logs ou alertas;
- não exibir no portal de auditoria por padrão;
- não gravar como atributo do AD DS;
- não enviar ao Entra Provisioning quando não houver necessidade técnica.

Após a correlação, o IAM deve manter a associação com o objeto AD correspondente. Uma fase posterior poderá substituir a persistência em texto claro por HMAC/tokenização/criptografia sem alterar a regra funcional de correlação.

## Estado operacional no AD DS

O AD DS deve expor o estado funcional calculado pelo IAM por meio de `seniorIamEmploymentStatus`.

Mapeamento inicial:

```text
PRE_PROVISIONED      -> conta criada, ainda não liberada
ACTIVE               -> conta habilitada
VACATION             -> férias; conta desabilitada conforme política
LEAVE                -> afastamento; conta desabilitada conforme política
ADMISSION_CANCELLED  -> admissão cancelada; conta desabilitada
TERMINATED           -> desligado; conta desabilitada
```

`seniorIamStatusChangedAt` registra a data/hora da última transição e `seniorIamDisabledAt` registra o instante efetivo do disable quando aplicável.

A integração **não exclui contas do AD DS em nenhum estado**. Retenção e eventual exclusão pertencem a outro processo, fora desta integração.

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

O `getEmployee` não expõe diretamente um identificador de usuário do gestor. A hierarquia é representada pelo posto do colaborador:

```text
workstation.hierarchyItem.id
workstation.hierarchyItem.parent.id
```

Semântica adotada:

- `workstation.hierarchyItem.id` é o **ID da posição hierárquica ocupada pelo próprio colaborador**;
- `workstation.hierarchyItem.parent.id` é o **ID da posição hierárquica superior imediata**;
- nenhum desses valores é ID de pessoa, Employee ID, matrícula ou DN do Active Directory;
- o objeto `parent` pode ser recursivo e representar níveis superiores sucessivos, porém o MVP usa somente o **parent imediato** para resolver o gestor direto.

A resolução de `manager` deve ocorrer por correlação entre os próprios registros de colaboradores retornados pela Senior:

```text
employee
  |
  +--> workstation.hierarchyItem.parent.id
                    |
                    v
procurar outro employee onde
workstation.hierarchyItem.id == parent.id
                    |
                    v
employee ocupante da posição superior
                    |
                    v
correlacionar pessoa com identidade IAM
                    |
                    v
resolver distinguishedName da conta no AD DS
                    |
                    v
manager
```

Exemplo conceitual:

```text
Colaborador A
  hierarchyItem.id        = POS-100
  hierarchyItem.parent.id = POS-050

Colaborador B
  hierarchyItem.id        = POS-050

=> B é o candidato a gestor direto de A
=> após correlação inequívoca com a identidade IAM de B:
   A.manager = distinguishedName da conta AD de B
```

#### Estratégia de implementação

Durante a leitura/reconciliação dos colaboradores, o conector deve construir um índice local:

```text
employeeByHierarchyItemId[
    employee.workstation.hierarchyItem.id
] = employee
```

A resolução passa a ser:

```text
managerHierarchyItemId =
    employee.workstation.hierarchyItem.parent.id

managerEmployee =
    employeeByHierarchyItemId[managerHierarchyItemId]
```

Essa estratégia evita executar uma chamada adicional à Senior para cada colaborador e permite resolver gestores em tempo constante após a montagem do snapshot/index.

O índice deve considerar apenas valores não nulos e a implementação deve detectar inconsistências em que mais de um colaborador apareça como ocupante da mesma posição hierárquica.

#### Regras de segurança e consistência

O atributo `manager` só deve ser aplicado quando:

1. `workstation.hierarchyItem.parent.id` estiver presente;
2. existir **um único** colaborador resolvido para esse `parent.id`;
3. a pessoa desse colaborador estiver correlacionada de forma inequívoca com uma identidade IAM;
4. a conta AD correspondente possuir um `distinguishedName` válido.

A implementação não deve:

- escrever o UUID de `hierarchyItem.id` ou `parent.id` diretamente no atributo `manager`;
- tratar o ID de hierarquia como Person ID, Employee ID ou matrícula;
- criar uma regra alternativa de correlação de pessoas diferente da definida em `identity-correlation.md`;
- subir automaticamente para `parent.parent` quando a posição superior imediata estiver sem ocupante, até que essa política seja aprovada.

Se o próprio `parent` não existir, o colaborador está no topo da hierarquia retornada e não há gestor direto a resolver por esta regra.

Se `parent.id` existir mas não houver ocupante resolvível, houver mais de um ocupante, ou a identidade AD do gestor ainda não estiver correlacionada, o conector deve tratar o `manager` como **não resolvido**, registrar a condição para reconciliação/observabilidade e não inventar um gestor alternativo.

#### Pendência funcional remanescente

Ainda precisa ser decidido o comportamento quando a posição superior imediata existe, porém está sem ocupante. As opções funcionais a serem avaliadas são:

- manter `manager` sem alteração até que a posição seja ocupada;
- limpar `manager`;
- subir recursivamente na hierarquia até encontrar a primeira posição superior ocupada.

Nenhuma dessas alternativas deve ser implementada implicitamente antes da decisão funcional.


### Situação do vínculo

O schema observado do `getEmployee` não apresentou um campo simples equivalente a `status = ACTIVE/TERMINATED`.

O lifecycle não deve assumir que `dismissalDate == null` é suficiente para classificar todos os casos. A situação efetiva deve considerar as regras funcionais aprovadas e, quando necessário, fontes complementares para afastamento, férias, cancelamento e demais estados.

### Vínculo único

Gente & Gestão confirmou que o cenário desta organização não possui múltiplos vínculos simultâneos para a mesma pessoa. Para o escopo desta integração, cada pessoa terá no máximo um vínculo ativo relevante por vez.

Com isso, matrícula, cargo, departamento, gestor, empresa e centro de custo do vínculo elegível podem ser projetados diretamente para a identidade, sem regra de precedência entre vínculos.

A confirmação funcional foi fornecida por **Juliana Croda — Gente & Gestão**. Caso essa premissa mude no futuro, a arquitetura deverá ser revisada antes de aceitar múltiplos vínculos.

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
- Validar em dados reais que `workstation.hierarchyItem.parent.id` referencia a posição superior imediata e que o ocupante é localizado por `workstation.hierarchyItem.id == parent.id`.
- Definir a política para posição superior existente porém sem ocupante resolvível: manter, limpar ou subir recursivamente para outro nível.
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

- Senior X API Portal — Jornada do Colaborador / `getEmployee` (schema validado no Swagger do ambiente; hierarquia documentada com `hierarchyItem.id` e `parent` recursivo): https://api.xplatform.com.br/api-portal/pt-br/tutoriais/jornada-do-colaborador/colaborador
- Senior X — nota de descontinuação das rotas antigas `employeejourney/apis/*` e `employeejourney/entities/*` em favor de `employeejourney/queries/*`: https://documentacao.senior.com.br/seniorxplatform/notas-da-versao/hcm/seniorx-modulo/painel-de-gestao/
- Senior X API Portal — Jornada do Colaborador / `getPerson` (schema validado no Swagger do ambiente).
- Microsoft Learn — UserAccountControl property flags: https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties
- Microsoft Learn — Revoke user access in Microsoft Entra ID: https://learn.microsoft.com/en-us/entra/identity/users/users-revoke-access
