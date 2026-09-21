# ADR-002 — Portal de Auditoria e Configurações

- **Status:** Proposed for MVP
- **Data:** 2026-09-21
- **Decisão:** Azure Web App + Microsoft Entra ID App Roles + Azure Storage Account

## Contexto

O Senior IAM Connector precisa de uma interface administrativa para:

- consultar identidades e seus estados;
- visualizar eventos de auditoria;
- acompanhar operações de provisioning e falhas;
- consultar saúde das integrações;
- futuramente alterar regras operacionais e configurações de lifecycle;
- manter rastreabilidade das alterações administrativas.

O volume esperado é baixo na operação normal (poucas movimentações por dia), com picos ocasionais em alterações em massa, como troca de gestor durante reconciliação.

## Decisão de runtime

O portal será hospedado em **Azure App Service / Web App**.

Arquitetura inicial:

```text
Microsoft Entra ID
        |
        | OIDC / OAuth 2.0
        v
Azure Web App
        |
        +-- Frontend
        +-- ASP.NET Core API
        +-- processamento assíncrono simples quando necessário
        |
        +--> Azure Storage Account
        |
        +--> Azure Key Vault
        |
        +--> Senior X / Microsoft Entra
```

O MVP pode ser publicado como uma única aplicação, evitando microserviços prematuros.

## Autenticação e autorização

O Microsoft Entra ID será a fonte autoritativa para autenticação e autorização administrativa do portal.

O portal não manterá usuários, senhas ou catálogo próprio de roles.

As permissões serão modeladas com **App Roles** da App Registration do portal e atribuídas preferencialmente a grupos do Entra ID.

App Roles iniciais:

| App Role | Permissão |
|---|---|
| `SeniorIam.Auditor` | Consulta de identidades, eventos, histórico e saúde |
| `SeniorIam.Operator` | Auditor + reprocessamento de operações com falha |
| `SeniorIam.GenteGestao` | Consulta + overrides autorizados de férias/afastamento |
| `SeniorIam.Admin` | Administração e alteração de configurações IAM |

A API deve validar as roles no backend. Ocultar botões no frontend é apenas UX e nunca substitui a autorização da API.

Exemplo de política:

```text
GET /api/audit/events
  Auditor | Operator | GenteGestao | Admin

POST /api/operations/{id}/retry
  Operator | Admin

POST /api/overrides
  GenteGestao | Admin

PUT /api/configuration/*
  Admin
```

A Enterprise Application deve ser configurada para exigir atribuição quando aplicável, evitando acesso ao portal por usuários sem role.

## Persistência

O MVP utilizará **Azure Storage Account**.

Serviços previstos:

- **Azure Table Storage:** estado operacional, auditoria e configurações;
- **Azure Queue Storage:** fila simples de operações/retries quando necessário;
- **Azure Blob Storage:** opcional para exportações e evidências futuras.

A aplicação deverá acessar a Storage Account preferencialmente via **Managed Identity**, sem connection string com Account Key no código.

## Modelo inicial de tabelas

### `IamIdentities`

Estado corrente da identidade.

Exemplos de propriedades:

```text
PartitionKey
RowKey / IdentityId
SeniorEmployeeId
SeniorPersonId
Cpf
RegisterNumber
DisplayName
GivenName
Surname
Title
Department
Company
CostCenter
ManagerIdentityId
SamAccountName
UserPrincipalName
AdObjectGuid
EmploymentStatus
HireDate
DismissalDate
LastSeniorSyncAt
LastProvisioningAt
```

### `AuditEvents`

Eventos append-only de auditoria.

Campos mínimos:

```text
PartitionKey
RowKey
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
AlertId
ActorType
ActorId
ActorRoles
```

### `ProvisioningOperations`

Estado de operações enviadas ou aguardando processamento.

### `Alerts`

Alertas funcionais e técnicos.

### `Configuration`

Configuração corrente.

### `ConfigurationHistory`

Histórico imutável de alterações de configuração.

### `Overrides`

Exceções autorizadas de férias/afastamento.

### `SyncRuns`

Execuções de polling/reconciliação.

### `IntegrationHealth`

Estado conhecido das dependências.

## Estratégia para Azure Table Storage

O modelo deve ser orientado às consultas do portal, pois Table Storage não possui joins relacionais.

O desenho de `PartitionKey` e `RowKey` deve favorecer:

- busca de identidade por ID técnico;
- listagem de identidades por estado;
- timeline de auditoria por identidade;
- auditoria por período;
- operações com falha;
- configuração corrente e histórico.

Para o MVP, simplicidade e rastreabilidade têm prioridade sobre otimização prematura.

## CPF

O CPF completo continua sendo utilizado internamente como chave funcional de correlação e pode ser armazenado no storage interno conforme decisão atual do MVP.

O portal **nunca deve exibir o CPF completo**.

Formato obrigatório de exibição:

```text
123.***.***-89
```

Regra:

- exibir os 3 primeiros dígitos;
- mascarar os 6 dígitos centrais;
- exibir os 2 últimos dígitos.

A máscara deve ser calculada na API ou em componente de domínio e não depender exclusivamente do frontend.

Logs, alertas e correlation IDs continuam sem CPF em texto claro.

## Secrets

Segredos nunca devem ser persistidos em Table Storage.

Exemplos:

- Senior client secret;
- Microsoft Graph client secret;
- certificados privados;
- tokens;
- Storage Account keys.

A configuração deverá armazenar apenas referências de secret quando necessário.

Exemplo:

```text
SeniorClientSecretRef = kv://senior-iam/prod/senior-client-secret
```

O valor real fica no Azure Key Vault.

## Auditoria de alteração de configuração

Toda alteração administrativa deve gerar evento append-only contendo:

```text
actorObjectId
actorUpn
actorRoles
operation
configurationKey
before
after
timestampUtc
correlationId
```

O histórico de configuração nunca deve ser sobrescrito.

## Consequências

### Positivas

- baixo custo e pouca operação para o volume inicial;
- integração nativa com Entra ID;
- sem autenticação local;
- RBAC administrado por grupos/App Roles;
- Managed Identity reduz uso de secrets;
- Table Storage atende bem ao padrão de leitura simples e audit trail;
- arquitetura fácil de hospedar em Web App.

### Limitações

- Table Storage não oferece joins, foreign keys ou consultas SQL;
- índices secundários exigem modelagem/projeções específicas;
- mudanças futuras com consultas relacionais complexas podem justificar migração para SQL/PostgreSQL;
- background processing muito intenso pode futuramente justificar Worker/WebJob/Functions separado.

## Critério para reavaliar a persistência

Reavaliar Table Storage se surgirem:

- consultas relacionais complexas;
- necessidade forte de filtros ad hoc;
- alto volume analítico;
- transações envolvendo múltiplas entidades;
- necessidade de relatórios SQL avançados.

Até que um desses fatores exista, Azure Table Storage é a persistência aprovada para o MVP.
