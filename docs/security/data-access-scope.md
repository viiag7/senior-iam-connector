# Data Access Scope e Security Baseline

- **Status:** Draft em revisão
- **Objetivo:** definir os limites de acesso do Senior IAM Connector, impedir acesso desnecessário a dados de RH e estabelecer controles mínimos de segurança, auditoria e operação.

## Princípio de Least Privilege

A identidade técnica utilizada pelo conector deve possuir somente permissões de leitura necessárias aos recursos que fornecem os atributos aprovados no contrato IAM.

Não deve ser utilizado usuário pessoal, papel administrativo amplo ou credencial compartilhada com outros processos.

A Senior X permite controlar acesso a telas, APIs e processos por recursos de permissão e papéis. O Discovery deve identificar exatamente quais recursos são suficientes para consultar os dados IAM sem liberar rotinas de remuneração/folha.

## Dados permitidos

Somente após validação no `attribute-mapping.md`, o conector poderá consultar dados como:

- identificador estável da pessoa;
- matrícula;
- nome;
- nome preferido, se necessário;
- situação do vínculo;
- tipo de vínculo;
- cargo;
- departamento;
- empresa/filial;
- centro de custo;
- gestor;
- localidade/unidade;
- data de admissão;
- data de desligamento;
- eventos de férias/afastamento estritamente necessários para decisão de lifecycle.

## Dados proibidos no escopo inicial

O conector não deve possuir acesso funcional nem técnico, quando a plataforma permitir segregação, a:

- salário/remuneração;
- eventos e valores de folha;
- dados bancários;
- benefícios;
- informações médicas ou de saúde além do estritamente necessário para classificar um evento de lifecycle, e preferencialmente sem detalhamento clínico;
- dependentes;
- dados fiscais sem relação necessária com identidade;
- documentos pessoais sem justificativa IAM.

Também é proibido registrar esses dados em logs, traces, métricas, cache ou banco de estado.

## Conta técnica no Senior

Requisitos:

- conta exclusiva da integração;
- nome identificável e finalidade documentada;
- papel exclusivo para IAM;
- somente permissões `read/view` necessárias;
- filtros de abrangência aplicados sempre que disponíveis;
- sem acesso interativo humano, quando tecnicamente possível;
- credencial armazenada em secret store apropriado;
- rotação de credenciais documentada;
- auditoria de uso habilitada quando disponível.

## Microsoft Entra / Microsoft Graph

O cliente que envia dados ao inbound provisioning deve usar autenticação de aplicação, sem credenciais de usuário humano.

As permissões do Microsoft Graph devem seguir o mínimo necessário para o cenário de inbound provisioning. As permissões efetivas do ambiente devem ser registradas antes do deploy.

## Secrets

Nunca devem ser versionados no GitHub:

- client secrets;
- passwords;
- tokens;
- API keys;
- certificados privados;
- connection strings contendo credenciais.

Ambiente local deve usar mecanismo de secrets de desenvolvimento; ambientes compartilhados/produção devem usar um secret store corporativo aprovado.

## Logging, auditoria e privacidade

Logs devem privilegiar identificadores técnicos e correlation IDs.

Exemplo recomendado:

```text
correlationId=...
sourcePersonId=...
operation=Provision
status=Accepted
provisioningJobId=...
```

Evitar registrar payload completo por padrão.

Payloads somente poderão ser habilitados para troubleshooting controlado, com sanitização e retenção curta.

Os requisitos completos de auditoria e observabilidade estão definidos em `docs/requirements/audit-observability.md`.

Como baseline de segurança:

- falhas de provisioning não podem permanecer silenciosas;
- eventos de auditoria devem ser protegidos contra alteração/exclusão não autorizada;
- alterações administrativas em políticas de lifecycle e overrides devem possuir responsável identificável;
- alertas devem conter contexto suficiente para investigação sem expor dados sensíveis desnecessários;
- sucesso assíncrono intermediário não deve ser confundido com convergência final no AD DS;
- operações críticas de disable/enable devem ser confirmadas ou reconciliadas;
- histórico de falha não deve ser apagado após reprocessamento bem-sucedido.

## Overrides de férias e suspensões

O sistema pode permitir políticas individuais de acesso durante férias/afastamentos, conforme requisitos funcionais.

Essas exceções devem seguir controles adicionais:

- somente atores autorizados podem criar ou alterar overrides;
- toda alteração deve registrar responsável e justificativa;
- a vigência deve ser rastreável;
- overrides devem estar disponíveis para auditoria/revisão;
- um override não pode sobrepor estado `TERMINATED` ou outra condição de maior precedência;
- alteração manual diretamente no AD DS não deve ser utilizada como substituto de um override governado pelo IAM.

## Separação de funções

Idealmente:

- RH controla dados de vínculo no Senior;
- IAM/Segurança define mappings, regras de lifecycle e políticas de suspensão;
- gestores/atores autorizados podem solicitar ou aprovar exceções conforme processo que ainda será definido;
- Infra/Identity administra AD DS e Provisioning Agent;
- aplicação usa identidades técnicas próprias;
- alterações em regras críticas passam por Pull Request/review;
- acesso de leitura a evidências de auditoria pode ser segregado do acesso de alteração operacional.

## Controles para produção

Antes do go-live:

- [ ] Papel IAM exclusivo criado no Senior.
- [ ] Permissões efetivas documentadas.
- [ ] Teste comprovando ausência de acesso a remuneração/folha.
- [ ] Credenciais fora do código/repositório.
- [ ] Permissões Graph mínimas documentadas.
- [ ] Provisioning Agent saudável e monitorado.
- [ ] Ambiente de teste separado de produção.
- [ ] Logs sem dados sensíveis desnecessários.
- [ ] Rotação e revogação de credenciais testadas.
- [ ] Runbook para comprometimento da credencial.
- [ ] Centralização de audit logs definida.
- [ ] Proteção contra alteração/exclusão indevida de audit logs definida.
- [ ] Retenção de logs aprovada por Segurança/Compliance.
- [ ] Alertas de falha persistente de provisioning testados.
- [ ] Falha de desabilitação de Leaver testada com alerta de alta prioridade.
- [ ] Acesso aos logs/auditoria seguindo Least Privilege.
- [ ] Processo de revisão de overrides/exceções definido.

## Referências

- `docs/requirements/functional-requirements.md`
- `docs/requirements/audit-observability.md`
- Senior — Papéis e permissões HCM: https://documentacao.senior.com.br/seniorxplatform/manual-do-usuario/hcm/papeis-e-permissoes/
- Microsoft Learn — API-driven inbound provisioning concepts: https://learn.microsoft.com/en-us/entra/identity/app-provisioning/inbound-provisioning-api-concepts
