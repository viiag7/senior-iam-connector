# Requisitos Funcionais

- **Status:** Draft em revisão
- **Escopo:** Senior IAM Connector — MVP
- **Objetivo:** definir o comportamento funcional esperado do sistema antes das decisões de implementação.

## Princípios funcionais

- O Senior Gestão de Pessoas é a fonte autoritativa do estado do vínculo e dos eventos de RH do colaborador.
- O sistema deve interpretar estados e datas de RH e refletir o lifecycle esperado da identidade no AD DS.
- O comportamento deve ser determinístico, auditável e idempotente.
- O MVP não deve depender de abertura manual de tickets para operações normais de lifecycle.
- Dados fora do escopo IAM, como remuneração e folha, não fazem parte do contrato funcional.
- Exceções de acesso configuradas no IAM não alteram o dado autoritativo no Senior; elas apenas modificam a ação de acesso permitida pela política IAM.
- Estados de maior precedência, como desligamento, não podem ser anulados por uma exceção de férias ou afastamento.

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
| RF-012 | O sistema não deve provisionar colaboradores que não atendam aos critérios de elegibilidade definidos pelas regras de negócio, registrando a decisão quando necessário para auditoria. |
| RF-013 | O sistema deve permitir criar a identidade antes da data de admissão, dentro de uma janela configurável, possibilitando que processos dependentes de onboarding sejam executados antes do início efetivo do colaborador. |
| RF-014 | O sistema deve respeitar a data e o horário efetivos de desligamento ao decidir quando desabilitar a identidade, conforme política aprovada. |
| RF-015 | O sistema deve atualizar os atributos organizacionais aprovados, incluindo gestor, cargo, departamento, empresa/filial, centro de custo e demais campos definidos no contrato de identidade. |
| RF-016 | O sistema deve processar múltiplas alterações do mesmo colaborador preservando consistência e ordem lógica do estado final. |
| RF-017 | O sistema deve permitir identificar separadamente o estado de lifecycle esperado da identidade e o estado do processamento/provisionamento. |
| RF-018 | O sistema deve distinguir e auditar operações de criação, alteração, suspensão temporária, reativação, desligamento e recontratação. |
| RF-019 | O sistema não deve utilizar informações salariais, de folha de pagamento ou outros dados de RH fora do escopo IAM para executar o provisionamento. |
| RF-020 | O sistema deve manter um vínculo persistente entre a identidade de origem na Senior e a identidade correspondente no AD DS. |
| RF-021 | O sistema deve suportar suspensão temporária de acesso no AD DS quando o colaborador entrar em uma situação configurada de suspensão, como férias, determinados tipos de afastamento ou outras situações definidas pela organização. |
| RF-022 | O sistema deve reativar automaticamente a mesma conta do AD DS quando o colaborador sair de uma situação de suspensão temporária e voltar a um estado elegível para acesso, desde que não exista outro estado impeditivo. |
| RF-023 | A suspensão temporária não deve ser tratada como desligamento e não deve causar exclusão, recriação ou perda da correlação da identidade. |
| RF-024 | O sistema deve considerar datas de início e término de férias, afastamentos e outras suspensões temporárias quando disponíveis, executando as ações de acesso conforme a vigência efetiva do evento. |
| RF-025 | O sistema deve impedir reativação automática quando existir outro estado de maior precedência que determine bloqueio da conta, como desligamento efetivo ou outra condição impeditiva definida nas regras de negócio. |
| RF-026 | O sistema deve tratar separadamente a criação da identidade e a habilitação do acesso, permitindo que uma identidade pré-provisionada permaneça sem acesso até a condição autorizada de ativação. |
| RF-027 | O sistema deve tratar cancelamento de admissão de uma identidade pré-provisionada, impedindo que a conta seja habilitada na data originalmente prevista. |
| RF-028 | O sistema deve considerar alterações ou cancelamentos de eventos futuros de lifecycle antes de executar ações previamente previstas, incluindo admissão, férias, afastamento e desligamento. |
| RF-029 | O sistema deve rejeitar de forma controlada e auditável registros elegíveis que não possuam os dados obrigatórios necessários ao provisionamento. |
| RF-030 | O sistema deve suportar uma política padrão de acesso durante férias e permitir configuração de exceção por colaborador, determinando se a conta deve permanecer habilitada ou ser temporariamente suspensa durante o período. |
| RF-031 | Toda exceção individual à política de férias ou suspensão deve possuir trilha de auditoria contendo, no mínimo, colaborador afetado, decisão aplicada, responsável pela alteração, data/hora, justificativa e vigência quando aplicável. |
| RF-032 | Uma exceção individual de férias ou suspensão não deve permitir reativar ou manter habilitada uma conta quando houver estado de maior precedência que exija bloqueio, especialmente desligamento. |
| RF-033 | O sistema deve gerar alerta operacional quando uma operação de provisionamento não atingir o estado esperado após a política de tentativas configurada ou quando ocorrer falha classificada como não recuperável. |
| RF-034 | O sistema deve permitir classificar alertas por severidade conforme o impacto da operação de lifecycle, permitindo tratamento prioritário de falhas de desabilitação, reativação e criação de identidade. |
| RF-035 | O sistema deve manter rastreabilidade entre uma falha de provisionamento, suas tentativas, o alerta gerado, o colaborador afetado e a resolução ou reprocessamento correspondente. |
| RF-036 | O sistema deve permitir detectar e alertar divergências persistentes encontradas pela reconciliação quando o estado real do AD DS não convergir para o estado esperado definido pelo lifecycle. |

## Modelo de estados

O sistema deve separar o **estado funcional da identidade** do **estado técnico do processamento**.

### Identity Lifecycle State

```text
PRE_PROVISIONED
ACTIVE
TEMPORARILY_SUSPENDED
TERMINATED
```

Esses estados representam o que a identidade **deveria ser** segundo os dados autoritativos e as regras IAM.

### Provisioning State

```text
PENDING
PROCESSING
SUCCEEDED
FAILED
```

Esses estados representam se o sistema conseguiu ou não materializar o estado esperado no target.

Exemplo:

```text
Identity Lifecycle State: ACTIVE
Provisioning State: FAILED
```

Significa que o colaborador deveria estar ativo, porém o provisionamento necessário para atingir esse estado falhou.

## Transições funcionais iniciais

```text
Novo colaborador elegível
        |
        v
PRE_PROVISIONED
        |
        | condição autorizada de início do acesso
        v
ACTIVE
        |
        | férias / afastamento / suspensão cuja política exija bloqueio
        v
TEMPORARILY_SUSPENDED
        |
        | retorno + política permite acesso + nenhum estado impeditivo
        v
ACTIVE
        |
        | desligamento
        v
TERMINATED
```

Uma recontratação poderá resultar em `TERMINATED -> ACTIVE`, passar novamente por `PRE_PROVISIONED` ou exigir novo provisionamento, conforme a política de Rehire e a existência da identidade anterior.

## Política de férias e exceções individuais

O sistema deve permitir um modelo em camadas:

```text
Política padrão da organização
           |
           v
Regra aplicável ao evento de férias
           |
           +--> sem override: aplicar política padrão
           |
           +--> override autorizado para colaborador: aplicar exceção
```

Exemplo conceitual:

| Colaborador | Evento | Política efetiva | Resultado esperado |
|---|---|---|---|
| Colaborador A | Férias | Suspender acesso | `TEMPORARILY_SUSPENDED` |
| Colaborador B | Férias | Exceção: manter acesso | `ACTIVE` |
| Colaborador C | Férias + desligamento efetivo | Desligamento tem precedência | `TERMINATED` |

A origem, autorização, vigência e governança desses overrides serão definidas nas regras de negócio e controles de segurança.

## Pontos que ainda exigem regra de negócio

Os requisitos acima definem **o que** o sistema deve suportar. Ainda precisam ser definidos separadamente:

- quantos dias antes da admissão inicia o pré-provisionamento;
- se a conta pré-provisionada nasce tecnicamente habilitada ou desabilitada;
- quando credenciais podem ser utilizadas;
- critérios de elegibilidade por empresa, tipo de vínculo ou outras condições;
- política padrão da organização para férias;
- quem pode autorizar um override individual de férias/suspensão;
- duração e expiração de overrides individuais;
- quais códigos/tipos de afastamento causam suspensão de acesso;
- horário efetivo de início e fim de suspensão;
- precedência entre férias, afastamento, desligamento e outras situações simultâneas;
- comportamento quando a data de retorno é alterada;
- comportamento quando um afastamento não possui data final;
- comportamento da conta após cancelamento de admissão;
- política de tentativas e backoff antes da emissão de alerta;
- classificação de severidade por operação de lifecycle;
- canais e responsáveis pelo recebimento dos alertas;
- política de exceções aprovada por RH/IAM/Segurança.

## Fora do escopo funcional inicial

Salvo decisão posterior explícita, o MVP não inclui:

- concessão de privilégios administrativos baseada em cargo;
- RBAC completo;
- provisionamento de grupos e entitlements de aplicações;
- licenciamento Microsoft 365;
- exclusão automática definitiva de contas;
- gestão de senha de usuário final;
- dados de remuneração, folha, benefícios, saúde, dependentes ou dados bancários.
