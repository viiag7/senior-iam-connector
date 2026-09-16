# Senior IAM Connector

Integração de **HR-driven identity provisioning** entre o Senior Gestão de Pessoas, Microsoft Entra ID e Active Directory Domain Services (AD DS).

## Objetivo

Automatizar o ciclo de vida de identidades de colaboradores usando o Senior como **authoritative source** dos dados de RH e o Microsoft Entra API-driven inbound provisioning como mecanismo de provisionamento para o AD DS.

Fluxo alvo inicial:

```text
Senior Gestão de Pessoas
        |
        v
Senior IAM Connector
        |
        v
Microsoft Entra API-driven inbound provisioning
        |
        v
Microsoft Entra Provisioning Agent
        |
        v
Active Directory Domain Services
```

## MVP

O primeiro MVP deve comprovar o fluxo ponta a ponta para um colaborador fictício:

1. Ler apenas os atributos IAM autorizados no Senior.
2. Normalizar e validar os dados.
3. Enviar o registro ao endpoint de inbound provisioning do Microsoft Entra.
4. Criar a identidade correspondente no AD DS.
5. Registrar correlação, resultado e auditoria do processamento.

O MVP inicial cobre **Create / Update / Disable**. Gestão avançada de grupos, roles, entitlements, licenciamento e provisionamento de aplicações ficam fora do primeiro incremento.

## Princípios

- Senior é a fonte autoritativa para dados de vínculo e organização.
- Least Privilege no acesso à API do Senior.
- Dados de remuneração, folha, dados bancários, saúde, benefícios e dependentes não fazem parte do contrato IAM.
- Identidades devem ser correlacionadas por um identificador único, estável e imutável definido no Senior.
- O conector deve ser idempotente.
- Provisionamento deve ser auditável e observável.
- Webhook/eventos, quando disponíveis, não substituem reconciliação periódica.
- Regras de acesso devem ser separadas dos atributos de identidade.

## Documentação

As decisões e contratos do projeto ficam em `docs/`.

Estrutura planejada:

```text
docs/
├── architecture/
├── identity/
├── integration/
└── operations/
```

## Status

Projeto em fase de **Discovery / Foundation**.
