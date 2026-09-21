# ADR-001 — Arquitetura de HR-driven identity provisioning

- **Status:** Proposed
- **Data:** 2026-09-16
- **Decisão:** Senior → Senior IAM Connector → Microsoft Entra API-driven inbound provisioning → Provisioning Agent → AD DS

## Contexto

O processo atual de criação, alteração e desligamento de contas depende de solicitações manuais entre RH e TI. Isso aumenta tempo de atendimento, risco de divergência entre RH e diretório e possibilidade de manter acessos após mudanças ou desligamentos.

O Senior Gestão de Pessoas contém os dados de vínculo do colaborador e deve atuar como **authoritative source** para o lifecycle de identidades.

O Microsoft Entra oferece **API-driven inbound provisioning to on-premises Active Directory**, no qual um cliente envia registros de pessoas para o endpoint `/bulkUpload`; o Entra Provisioning Service aplica matching, scoping, attribute mappings e transformations, e o Provisioning Agent executa o provisionamento no AD DS.

## Decisão

Adotar a arquitetura abaixo para o MVP:

```text
Senior Gestão de Pessoas
        |
        | API
        v
Senior IAM Connector
        |
        | OAuth 2.0 + Microsoft Graph /bulkUpload
        v
Microsoft Entra API-driven inbound provisioning
        |
        v
Microsoft Entra Provisioning Agent
        |
        v
Active Directory Domain Services
        |
        v
Microsoft Entra Connect Sync / Cloud Sync
        |
        v
Microsoft Entra ID
```

## Responsabilidades

### Senior Gestão de Pessoas

- Fonte autoritativa dos dados de vínculo e organização.
- Mantém admissão, situação do vínculo, cargo, departamento, gestor e datas relevantes.
- Não delega ao AD DS a propriedade dos dados de RH.

### Senior IAM Connector

- Autenticar na API do Senior com **Least Privilege**.
- Ler somente os atributos autorizados para IAM.
- Normalizar e validar os registros.
- Enviar registros completos ao inbound provisioning endpoint.
- Manter checkpoint suficiente para coleta e reprocessamento seguro.
- Implementar retry, throttling handling e idempotência.
- Consultar provisioning logs para acompanhar resultado assíncrono.
- Produzir logs técnicos e auditoria sem armazenar dados desnecessários.

O conector **não deve implementar diretamente operações LDAP** no MVP.

### Microsoft Entra Provisioning Service

- Executar matching/correlation conforme configuração aprovada.
- Aplicar scoping, mappings e transformations.
- Determinar create/update/enable/disable a partir do estado recebido e do target.
- Registrar provisioning logs.

### Microsoft Entra Provisioning Agent

- Fornecer a ponte segura entre o Entra Provisioning Service e o AD DS on-premises.
- Executar as operações suportadas no diretório configurado.

### Active Directory Domain Services

- Diretório de destino para identidades híbridas.
- Não é a fonte autoritativa para os atributos controlados pelo Senior.

## Princípios arquiteturais

1. **HR-driven lifecycle:** mudanças relevantes no Senior conduzem o lifecycle da identidade.
2. **Least Privilege:** o conector acessa somente dados e APIs estritamente necessários.
3. **Data minimization:** dados de salário, folha, conta bancária, saúde, benefícios e dependentes ficam fora do contrato IAM.
4. **Stable correlation:** cada pessoa deve ser correlacionada por um identificador único e imutável.
5. **Idempotency:** reprocessar o mesmo registro não pode criar identidades duplicadas.
6. **Event + reconciliation:** eventos podem acelerar alterações, mas uma reconciliação periódica continua obrigatória.
7. **Auditability:** cada processamento deve possuir correlation ID, origem, resultado e timestamps.
8. **Separation of identity and access:** atributos organizacionais e lifecycle são distintos de groups, roles e entitlements.
9. **Fail safe:** dados inválidos ou ambíguos devem interromper o registro afetado em vez de alterar a identidade errada.

## Escopo do MVP

Incluído:

- Leitura de colaboradores do Senior.
- Create no AD DS.
- Update de atributos aprovados.
- Disable conforme regra de Leaver aprovada.
- Correlation/matching.
- Logs, auditoria e tratamento de erros.
- Reconciliação básica.

Fora do MVP:

- RBAC completo.
- Gestão automática de grupos baseada em cargo.
- Licenças Microsoft 365.
- VPN e aplicações SaaS.
- Access packages.
- Access reviews.
- Exclusão de contas pelo conector; o fluxo somente cria, atualiza, habilita e desabilita.

## Consequências positivas

- Reduz lógica específica de AD dentro do conector.
- Usa um mecanismo Microsoft suportado para identidades híbridas.
- Centraliza mappings e transformations no provisioning service.
- Mantém trilha de provisioning no Entra.
- Permite evoluir posteriormente para Lifecycle Workflows e governança de acesso.

## Trade-offs e riscos

- Dependência do Microsoft Entra Provisioning Service e Provisioning Agent.
- O processamento do `/bulkUpload` é assíncrono; resposta HTTP 202 não significa provisionamento concluído.
- O conector precisa acompanhar provisioning logs e tratar falhas por registro.
- Limites e throttling da API precisam ser respeitados.
- Matching incorreto pode afetar uma identidade existente; a regra de correlation precisa ser validada antes de produção.

## Alternativas consideradas

### Escrita direta no AD DS via LDAP/PowerShell

Não escolhida para o MVP porque transfere para o conector responsabilidades de matching, operações de diretório, retry e parte da governança que o Entra Provisioning Service já oferece.

### Criar usuários cloud-only diretamente no Entra ID

Não atende ao cenário em que a identidade precisa nascer no AD DS local e depois sincronizar para o Entra ID.

## Decisões já fechadas para o MVP

- Senior `employeejourney/getEmployee` é a fonte principal de colaborador.
- CPF (`person.cpf`) é a chave funcional de correlação.
- Política de naming de `sAMAccountName` e UPN: `docs/identity/naming-policy.md`.
- A conta é criada/reutilizada assim que o colaborador elegível aparece na Senior.
- A conta é habilitada em D-1 (`hireDate - 1 dia`).
- Gente & Gestão confirmou que não existem múltiplos vínculos simultâneos no escopo.
- Rehire reutiliza e reativa a mesma conta e atualiza os atributos autoritativos.
- O conector nunca exclui contas do AD DS.
- O AD DS será estendido com atributos `seniorIam*` definidos no mapping.
- CPF pode ser armazenado em texto claro no storage interno do IAM na primeira fase, sem propagação para AD/logs/portal.

## Decisões ainda abertas

- Estratégia de delta/polling/eventos disponível na Senior.
- OU padrão e regras de movimentação entre OUs.
- Fonte do timestamp exato de desligamento; `dismissalDate` é apenas data.
- Resolução do gestor a partir de `workstation.hierarchyItem.id`.
- Target físico da matrícula no AD (`employeeNumber`, `employeeID` ou equivalente).
- OIDs/sintaxe final dos atributos customizados `seniorIam*`.
- Fonte e códigos de férias/afastamentos/retorno.
- Processo de initial password / first sign-in.

## Referências

- Microsoft Learn — API-driven inbound provisioning concepts: https://learn.microsoft.com/en-us/entra/identity/app-provisioning/inbound-provisioning-api-concepts
- Microsoft Learn — Configure API-driven inbound provisioning app: https://learn.microsoft.com/en-us/entra/identity/app-provisioning/inbound-provisioning-api-configure-app
- Senior — Papéis e permissões HCM: https://documentacao.senior.com.br/seniorxplatform/manual-do-usuario/hcm/papeis-e-permissoes/
