# Requisitos Funcionais

- **Status:** Draft
- **Escopo:** Senior IAM Connector — MVP
- **Objetivo:** definir o comportamento funcional esperado do sistema antes das decisões de implementação.

## Princípios funcionais

- O Senior Gestão de Pessoas é a fonte autoritativa do estado do vínculo do colaborador.
- O sistema deve interpretar estados e datas de RH e refletir o lifecycle esperado da identidade no AD DS.
- O comportamento deve ser determinístico, auditável e idempotente.
- O MVP não deve depender de abertura manual de tickets para operações normais de lifecycle.
- Dados fora do escopo IAM, como remuneração e folha, não fazem parte do contrato funcional.

## Requisitos

| ID | Requisito funcional |
|---|---|
| RF-001 | O sistema deve identificar novos colaboradores elegíveis para provisionamento a partir dos dados autoritativos da Senior. |
| RF-002 | O sistema deve criar automaticamente a identidade correspondente no AD DS para colaboradores elegíveis. |
| RF-003 | O sistema deve atualizar atributos da identidade no AD DS quando ocorrer alteração relevante e autorizada na Senior. |
| RF-004 | O sistema deve desabilitar a conta no AD DS quando o vínculo atingir um estado de desligamento conforme as regras de negócio definidas. |
| RF-005 | O sistema deve tratar recontratações sem gerar identidades duplicadas e conforme política de reativação/reutilização definida. |
| RF-006 | O sistema deve correlacionar de forma inequívoca um colaborador da Senior com sua identidade correspondente no AD DS. |
| RF-007 | O sistema deve impedir a criação duplicada da mesma identidade mesmo em caso de reprocessamento, eventos repetidos ou retries. |
| RF-008 | O sistema deve registrar cada tentativa de provisionamento e seu resultado de forma auditável. |
| RF-009 | O sistema deve registrar falhas de processamento de forma rastreável, incluindo a identidade afetada, operação e motivo da falha. |
| RF-010 | O sistema deve permitir reprocessamento seguro de operações com falha sem gerar duplicidades ou efeitos colaterais indevidos. |
| RF-011 | O sistema deve executar reconciliação periódica entre o estado autoritativo da Senior e o estado esperado da identidade para detectar divergências ou operações perdidas. |
| RF-012 | O sistema deve ignorar ou rejeitar, de forma controlada e auditável, colaboradores que não atendam aos critérios de elegibilidade para provisionamento. |
| RF-013 | O sistema deve suportar pré-provisionamento da identidade antes da data de admissão, dentro de uma janela configurável definida pelas regras de negócio, para permitir preparação prévia de conta, equipamentos, aplicações e demais processos de onboarding. |
| RF-014 | O sistema deve respeitar a data e o horário efetivos de desligamento ao decidir quando desabilitar a identidade, conforme política aprovada. |
| RF-015 | O sistema deve atualizar os atributos organizacionais aprovados, incluindo gestor, cargo, departamento, empresa/filial, centro de custo e demais campos definidos no contrato de identidade. |
| RF-016 | O sistema deve processar múltiplas alterações do mesmo colaborador preservando consistência e ordem lógica do estado final. |
| RF-017 | O sistema deve permitir identificar o estado atual do processamento/provisionamento de cada colaborador. |
| RF-018 | O sistema deve distinguir e auditar operações de criação, alteração, suspensão temporária, reativação, desligamento e recontratação. |
| RF-019 | O sistema não deve utilizar informações salariais, de folha de pagamento ou outros dados de RH fora do escopo IAM para executar o provisionamento. |
| RF-020 | O sistema deve manter um vínculo persistente entre a identidade de origem na Senior e a identidade correspondente no AD DS. |
| RF-021 | O sistema deve desabilitar temporariamente a conta no AD DS quando o colaborador entrar em uma situação de suspensão de acesso definida pela organização, como férias, determinados tipos de afastamento ou outras situações configuradas. |
| RF-022 | O sistema deve reativar automaticamente a mesma conta do AD DS quando o colaborador sair de uma situação de suspensão temporária e voltar a um estado elegível para acesso, desde que o vínculo permaneça válido. |
| RF-023 | A suspensão temporária não deve ser tratada como desligamento e não deve causar exclusão, recriação ou perda da correlação da identidade. |
| RF-024 | O sistema deve considerar datas de início e término de férias, afastamentos e outras suspensões temporárias quando disponíveis, permitindo programar a desativação e reativação de acordo com a vigência efetiva do evento. |
| RF-025 | O sistema deve impedir reativação automática quando existir outro estado de maior precedência que determine bloqueio da conta, como desligamento efetivo ou outra condição impeditiva definida nas regras de negócio. |

## Estados funcionais iniciais

O modelo funcional deve distinguir, no mínimo, os seguintes estados da identidade:

```text
PRE_PROVISIONED
ACTIVE
TEMPORARILY_SUSPENDED
DISABLED
ERROR
```

O mapeamento entre esses estados e os códigos/situações reais da Senior será definido nas regras de negócio.

### Transições esperadas

```text
Novo colaborador elegível
        |
        v
PRE_PROVISIONED
        |
        | data/condição de início do acesso
        v
ACTIVE
        |
        | férias / afastamento / suspensão configurada
        v
TEMPORARILY_SUSPENDED
        |
        | retorno ao trabalho e vínculo ainda elegível
        v
ACTIVE
        |
        | desligamento
        v
DISABLED
```

Uma recontratação poderá resultar em `DISABLED -> ACTIVE` ou em novo provisionamento, dependendo da política de Rehire e da existência da identidade anterior.

## Pontos que ainda exigem regra de negócio

Os requisitos acima definem **o que** o sistema deve suportar. Ainda precisam ser definidos separadamente:

- quantos dias antes da admissão inicia o pré-provisionamento;
- se a conta pré-provisionada nasce habilitada ou desabilitada;
- quando credenciais podem ser utilizadas;
- quais códigos de férias/afastamento causam suspensão de acesso;
- se todas as férias desabilitam a conta ou apenas situações específicas;
- horário efetivo de início e fim de suspensão;
- precedência entre férias, afastamento, desligamento e outras situações simultâneas;
- comportamento quando a data de retorno é alterada;
- comportamento quando um afastamento não possui data final;
- política de exceções aprovada por RH/Segurança.

## Fora do escopo funcional inicial

Salvo decisão posterior explícita, o MVP não inclui:

- concessão de privilégios administrativos baseada em cargo;
- RBAC completo;
- provisionamento de grupos e entitlements de aplicações;
- licenciamento Microsoft 365;
- exclusão automática definitiva de contas;
- gestão de senha de usuário final;
- dados de remuneração, folha, benefícios, saúde, dependentes ou dados bancários.
