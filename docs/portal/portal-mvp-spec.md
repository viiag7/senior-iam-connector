# Portal de Auditoria e Configurações — Especificação MVP

- **Objetivo:** especificar a primeira versão funcional a ser implementada pelo Antigravity.
- **Stack alvo:** ASP.NET Core + frontend TypeScript/React + Azure Web App + Azure Storage Account.
- **Autenticação:** Microsoft Entra ID.
- **Autorização:** Entra ID App Roles.
- **Persistência:** Azure Table Storage.
- **Status:** pronto para scaffold/primeira implementação.

## 1. Escopo da primeira versão

A primeira versão deve permitir:

1. autenticar usuários corporativos pelo Microsoft Entra ID;
2. autorizar acesso por App Roles;
3. listar identidades IAM;
4. abrir o detalhe de uma identidade;
5. exibir CPF mascarado no formato `123.***.***-89`;
6. visualizar timeline de eventos de auditoria;
7. visualizar operações de provisioning;
8. visualizar falhas e tentativas;
9. permitir retry manual para roles autorizadas;
10. visualizar configurações correntes;
11. registrar histórico de qualquer alteração futura de configuração;
12. mostrar saúde básica das integrações;
13. operar inicialmente com dados mock/semente sem depender da API Senior real.

Não é necessário para a primeira versão:

- integrar de fato com o Senior X;
- enviar dados para o Entra `/bulkUpload`;
- implementar webhook;
- implementar férias/afastamentos reais;
- resolver manager real;
- criar/alterar schema do AD;
- enviar alertas reais por Teams/e-mail/WhatsApp.

A primeira versão deve deixar interfaces e abstrações preparadas para essas integrações.

## 2. Arquitetura da solução

Estrutura sugerida:

```text
src/
├── SeniorIam.Web/
│   ├── API ASP.NET Core
│   └── frontend build/static files
├── SeniorIam.Application/
├── SeniorIam.Domain/
├── SeniorIam.Infrastructure/
└── SeniorIam.Contracts/

tests/
├── SeniorIam.UnitTests/
└── SeniorIam.IntegrationTests/
```

É aceitável simplificar o número de projetos no primeiro scaffold, desde que Domain/Application não fiquem acoplados diretamente ao SDK de Azure Table Storage.

## 3. Azure Web App

A aplicação deve ser compatível com deployment em Azure App Service / Web App.

Requisitos:

- configuração por environment variables;
- health endpoint;
- logs estruturados;
- Managed Identity ready;
- nenhuma credencial hardcoded;
- configuração separada entre Development e Production.

Endpoints mínimos de infraestrutura:

```text
GET /health
GET /health/live
GET /health/ready
```

## 4. Microsoft Entra ID

Usar autenticação OIDC/OAuth do Microsoft Entra ID.

Configurações esperadas:

```text
Entra:TenantId
Entra:ClientId
Entra:Instance
```

Secrets não devem ser necessários para login interativo do portal quando o fluxo adotado não exigir.

### App Roles

Implementar suporte às seguintes roles:

```text
SeniorIam.Auditor
SeniorIam.Operator
SeniorIam.GenteGestao
SeniorIam.Admin
```

### Matriz de autorização

| Recurso | Auditor | Operator | GenteGestao | Admin |
|---|---:|---:|---:|---:|
| Dashboard | R | R | R | R |
| Identidades | R | R | R | R |
| Auditoria | R | R | R | R |
| Saúde | R | R | R | R |
| Retry de operação | - | W | - | W |
| Overrides | R | R | W | W |
| Configurações | R | R | R | W |
| Histórico de configuração | R | R | R | R |

A autorização deve ser validada na API.

## 5. Telas

### 5.1 Dashboard

Cards mínimos:

- identidades totais;
- `ACTIVE`;
- `PRE_PROVISIONED`;
- `VACATION`;
- `LEAVE`;
- `TERMINATED`;
- operações com falha;
- última reconciliação.

Também mostrar uma lista compacta de operações recentes.

### 5.2 Identidades

Tabela com:

```text
Nome
CPF mascarado
Matrícula
Cargo
Departamento
Gestor
EmploymentStatus
sAMAccountName
UPN
Última sincronização
```

Filtros:

- nome;
- matrícula;
- status;
- departamento;
- operação com falha sim/não.

Nunca permitir pesquisa pelo CPF completo no frontend do MVP.

### 5.3 Detalhe da identidade

Seções:

#### Dados Senior/IAM

```text
DisplayName
CPF mascarado
RegisterNumber
Title
Department
Company
CostCenter
Manager
HireDate
DismissalDate
EmploymentStatus
```

#### Diretório

```text
SamAccountName
UserPrincipalName
AdObjectGuid
AccountEnabled/Disabled
```

#### Sincronização

```text
LastSeniorSyncAt
LastProvisioningAt
LastCorrelationId
```

#### Timeline

Exibir eventos de auditoria em ordem cronológica.

### 5.4 Auditoria

Tabela com:

```text
Timestamp
Identity
Operation
Before
After
Result
Attempt
Source
Target
CorrelationId
Actor
```

Filtros:

- período;
- identidade;
- operação;
- resultado;
- correlationId.

Ao abrir um evento, mostrar os campos estruturados disponíveis.

Não mostrar payload completo da Senior.

### 5.5 Operações/Falhas

Tabela:

```text
OperationId
Identity
Operation
Status
Attempt
CreatedAt
LastAttemptAt
ErrorCategory
ErrorSummary
CorrelationId
```

Ação `Retry` somente para:

```text
SeniorIam.Operator
SeniorIam.Admin
```

O retry da primeira versão pode ser simulado e deve gerar novo evento de auditoria.

### 5.6 Configurações

Na primeira versão, permitir leitura de:

#### Lifecycle

```text
CreateWhenSeenInSenior = true
EnableDaysBeforeHire = 1
DeleteAccounts = false
```

#### Naming

```text
Pattern = first.last
UpnSuffix
```

#### Senior

```text
BaseUrl
EmployeeEndpoint
ReconciliationInterval
```

#### Alertas

```text
MaxAttempts = 3
```

A edição real pode ser implementada apenas para `SeniorIam.Admin`.

Toda alteração deve registrar `ConfigurationHistory` e `AuditEvent`.

Não permitir armazenar secrets nessa tela.

### 5.7 Overrides

Estrutura preparada para:

```text
IdentityId
Type
Reason
StartAt
EndAt
AuthorizedBy
CreatedAt
Status
```

Criação/alteração somente para:

```text
SeniorIam.GenteGestao
SeniorIam.Admin
```

Na primeira versão pode funcionar somente sobre dados simulados.

### 5.8 Saúde

Mostrar:

```text
Portal/API
Storage Account
Senior X
Microsoft Entra Provisioning
LastReconciliation
```

Senior/Entra podem aparecer como `NOT_CONFIGURED` enquanto a integração real não existir.

## 6. Modelo de domínio

### Identity

```text
IdentityId: string
SeniorEmployeeId: string?
SeniorPersonId: string?
Cpf: string
RegisterNumber: string?
DisplayName: string
GivenName: string?
Surname: string?
Title: string?
Department: string?
Company: string?
CostCenter: string?
ManagerIdentityId: string?
ManagerDisplayName: string?
SamAccountName: string?
UserPrincipalName: string?
AdObjectGuid: string?
AccountEnabled: bool?
EmploymentStatus: enum
HireDate: DateOnly?
DismissalDate: DateOnly?
LastSeniorSyncAt: DateTimeOffset?
LastProvisioningAt: DateTimeOffset?
LastCorrelationId: string?
```

### EmploymentStatus

```text
PRE_PROVISIONED
ACTIVE
VACATION
LEAVE
ADMISSION_CANCELLED
TERMINATED
```

### AuditEvent

```text
AuditEventId
CorrelationId
TimestampUtc
IdentityId
SourceSystem
Operation
LifecycleBefore
LifecycleAfter
DecisionSource
TargetSystem
Attempt
Result
ErrorCategory
ErrorSummary
ActorType
ActorObjectId
ActorUpn
ActorRoles
```

### ProvisioningOperation

```text
OperationId
IdentityId
CorrelationId
OperationType
Status
Attempt
MaxAttempts
CreatedAt
LastAttemptAt
CompletedAt
ExternalRequestId
ErrorCategory
ErrorSummary
```

### ConfigurationValue

```text
Key
Value
Version
UpdatedAt
UpdatedByObjectId
UpdatedByUpn
```

### ConfigurationHistory

```text
HistoryId
Key
Before
After
Version
ChangedAt
ActorObjectId
ActorUpn
ActorRoles
CorrelationId
```

## 7. CPF

Criar uma função central de domínio:

```text
MaskCpf("12345678909") => "123.***.***-09"
```

Comportamento:

- remover pontuação antes do processamento;
- aceitar somente CPF com 11 dígitos para exibição;
- retornar valor seguro/fallback quando inválido;
- nunca enviar CPF completo para DTOs utilizados pelas telas;
- criar DTO separado para portal contendo apenas `CpfMasked`.

Teste unitário obrigatório para a máscara.

## 8. Azure Table Storage

Usar `Azure.Data.Tables`.

Criar abstrações de repository, evitando acesso direto ao SDK nas controllers.

Tabelas:

```text
IamIdentities
AuditEvents
ProvisioningOperations
Alerts
Configuration
ConfigurationHistory
Overrides
SyncRuns
IntegrationHealth
```

### Chaves sugeridas

#### IamIdentities

```text
PartitionKey = "IDENTITY"
RowKey = IdentityId
```

#### AuditEvents

Para a primeira versão:

```text
PartitionKey = IdentityId
RowKey = reverseTimestamp_or_timestamp + "_" + AuditEventId
```

O objetivo principal é timeline rápida por identidade.

Se for necessário auditoria global por período no futuro, criar uma projeção adicional, sem alterar o modelo de domínio.

#### Configuration

```text
PartitionKey = "CONFIG"
RowKey = Key
```

#### ConfigurationHistory

```text
PartitionKey = Key
RowKey = timestamp + "_" + HistoryId
```

## 9. Dados seed

Criar dados fictícios suficientes para demonstrar:

1. usuário `ACTIVE`;
2. usuário `PRE_PROVISIONED`;
3. usuário `VACATION`;
4. usuário `LEAVE`;
5. usuário `TERMINATED`;
6. operação `FAILED` após tentativas;
7. Rehire com conta reutilizada;
8. alteração de cargo/departamento;
9. alteração de configuração;
10. override de férias.

Nenhum dado seed deve usar pessoa real.

## 10. API mínima

### Identidades

```text
GET /api/identities
GET /api/identities/{identityId}
GET /api/identities/{identityId}/audit
```

### Auditoria

```text
GET /api/audit/events
GET /api/audit/events/{auditEventId}
```

### Operações

```text
GET /api/operations
GET /api/operations/{operationId}
POST /api/operations/{operationId}/retry
```

### Configuração

```text
GET /api/configuration
PUT /api/configuration/{key}
GET /api/configuration/{key}/history
```

### Overrides

```text
GET /api/overrides
POST /api/overrides
PUT /api/overrides/{overrideId}
```

### Saúde

```text
GET /api/health/integrations
```

## 11. Requisitos de segurança

- nenhuma página protegida sem autenticação;
- API valida App Role em todas as operações administrativas;
- CPF completo nunca retorna nos DTOs do portal;
- secrets nunca armazenados em Table Storage;
- nenhum token em logs;
- nenhuma connection string com Account Key commitada;
- suporte a Managed Identity;
- logs estruturados com `correlationId`;
- alterações de configuração devem ser auditadas;
- audit events não podem ser alterados pela UI;
- frontend não decide autorização sozinho.

## 12. UX mínima

Layout:

```text
Sidebar
├── Dashboard
├── Identidades
├── Auditoria
├── Operações
├── Overrides
├── Configurações
└── Saúde
```

Topo:

```text
Nome do usuário
Roles efetivas
Ambiente
```

Usar interface limpa, corporativa e responsiva.

A aplicação deve funcionar bem em desktop; suporte mobile pode ser responsivo básico no MVP.

## 13. Critérios de aceite da primeira versão

A versão está pronta quando:

- [ ] executa localmente;
- [ ] possui README com instruções;
- [ ] possui configuração de Azure Table Storage;
- [ ] consegue rodar com Azurite para desenvolvimento;
- [ ] apresenta dados seed;
- [ ] autenticação Entra pode ser habilitada por configuração;
- [ ] roles são verificadas pela API;
- [ ] usuário sem role não consegue acessar operações administrativas;
- [ ] lista identidades;
- [ ] detalhe mostra CPF somente mascarado;
- [ ] timeline de auditoria funciona;
- [ ] tela de falhas funciona;
- [ ] retry simulado gera novo AuditEvent;
- [ ] configurações são exibidas;
- [ ] alteração de configuração por Admin gera histórico;
- [ ] Auditor não consegue alterar configuração;
- [ ] GenteGestao consegue criar override;
- [ ] audit trail mostra o ator da ação;
- [ ] health endpoints funcionam;
- [ ] nenhum secret está no repositório.

## 14. Instruções para o Antigravity

Gerar a primeira versão do projeto respeitando esta especificação e a documentação existente em `docs/`.

Prioridades:

1. solução compilável e executável;
2. domínio e segurança antes de integrações externas;
3. Azure Table Storage por abstração;
4. Entra App Roles como autorização;
5. portal navegável com dados seed;
6. testes para CPF masking e autorização;
7. sem integração real com Senior/Entra provisioning nesta primeira entrega.

Não inventar regras funcionais que conflitem com a documentação existente.

Quando uma dependência externa ainda não estiver definida, criar interface/adaptador com implementação mock em vez de hardcode.

Entregar também:

- README de desenvolvimento;
- arquivo de exemplo de configuração sem secrets;
- instrução de Azurite;
- estrutura de testes;
- lista clara de TODOs para integrações Senior/Entra/AD futuras.
