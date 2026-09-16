# Requisitos Funcionais

- **Status:** Draft em revisão
- **Escopo:** Senior IAM Connector — MVP
- **Objetivo:** definir o comportamento funcional esperado do sistema antes das decisões de implementação.

## Princípios funcionais

- O Senior Gestão de Pessoas é a fonte autoritativa do estado do vínculo e dos eventos de RH do colaborador.
- O sistema deve interpretar estados e datas de RH e refletir o lifecycle esperado da identidade no AD DS.
- A conta corporativa representa a **pessoa**, e não um vínculo trabalhista específico.
- O **CPF é a única chave autorizada para determinar que vínculos pertencem à mesma pessoa**.
- Mudanças de vínculo não devem gerar nova identidade quando o CPF já estiver associado de forma inequívoca a uma conta corporativa existente.
- Nenhum outro atributo pode ser utilizado como fallback de correlação de pessoa.
- O comportamento deve ser determinístico, auditável e idempotente.
- Uma falha no processamento de uma identidade não pode interromper o processamento das demais identidades.
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
| RF-004 | O sistema deve desabilitar a conta no AD DS quando o estado efetivo da pessoa atingir desligamento conforme as regras de negócio, considerando todos os vínculos elegíveis existentes. |
| RF-005 | O sistema deve tratar recontratações sem gerar identidades duplicadas e conforme política de reativação/reutilização definida. |
| RF-006 | O sistema deve correlacionar a pessoa exclusivamente pelo CPF e, a partir dessa correlação, localizar sua identidade correspondente no IAM/AD DS. |
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
| RF-021 | A política padrão para férias é suspender temporariamente o acesso, mantendo a identidade existente e desabilitando a conta no AD DS durante a vigência do evento, salvo override individual válido. |
| RF-022 | O sistema deve reativar automaticamente a mesma conta do AD DS quando o colaborador sair de uma situação de suspensão temporária e voltar a um estado elegível para acesso, desde que não exista outro estado impeditivo. |
| RF-023 | A suspensão temporária não deve ser tratada como desligamento e não deve causar exclusão, recriação ou perda da correlação da identidade. |
| RF-024 | O sistema deve considerar as datas vigentes de início e término de férias, afastamentos e outras suspensões temporárias. Quando a data de retorno for alterada na Senior, a ação de reativação deve acompanhar a nova data. |
| RF-025 | O sistema deve impedir reativação automática quando existir outro estado de maior precedência que determine bloqueio da conta, como desligamento efetivo ou outra condição impeditiva definida nas regras de negócio. |
| RF-026 | O sistema deve tratar separadamente a criação da identidade e a habilitação do acesso, permitindo que uma identidade pré-provisionada permaneça sem acesso até a condição autorizada de ativação. |
| RF-027 | Quando uma admissão pré-provisionada for confirmadamente cancelada/excluída na Senior, o sistema deve impedir a ativação e garantir que a conta permaneça desabilitada no AD DS, preservando a identidade para auditoria e eventual reutilização pelo mesmo CPF. |
| RF-028 | O sistema deve considerar alterações ou cancelamentos de eventos futuros de lifecycle antes de executar ações previamente previstas, incluindo admissão, férias, afastamento e desligamento. |
| RF-029 | O sistema deve rejeitar de forma controlada e auditável registros elegíveis que não possuam os dados obrigatórios necessários ao provisionamento. |
| RF-030 | O sistema deve permitir override individual da política de férias/suspensão somente quando autorizado pelo time de **Gente e Gestão**. Para férias, o override deve vigorar durante todo o período vigente de férias e acompanhar alterações de datas registradas na Senior. |
| RF-031 | Toda exceção individual de férias ou suspensão deve possuir trilha de auditoria contendo, no mínimo, colaborador afetado, decisão aplicada, responsável/autorizador, data/hora, justificativa e vigência. |
| RF-032 | Uma exceção individual de férias ou suspensão não deve permitir reativar ou manter habilitada uma conta quando houver estado de maior precedência que exija bloqueio, especialmente desligamento. |
| RF-033 | Cada operação de provisioning com falha transitória deve possuir no máximo **3 tentativas por ciclo de processamento, incluindo a tentativa inicial**. Após a terceira falha deve ser gerado alerta. A falha de uma identidade deve ser isolada e não pode impedir o processamento de outros colaboradores. |
| RF-034 | Alertas de falha de desligamento devem ser classificados como **CRITICAL**. As demais operações devem seguir a matriz de severidade definida em `audit-observability.md`. |
| RF-035 | O sistema deve manter rastreabilidade entre uma falha de provisionamento, suas tentativas, o alerta gerado, o colaborador afetado e a resolução ou reprocessamento correspondente. |
| RF-036 | O sistema deve permitir detectar e alertar divergências persistentes encontradas pela reconciliação quando o estado real do AD DS não convergir para o estado esperado definido pelo lifecycle. |
| RF-037 | O sistema deve associar a identidade corporativa à pessoa, utilizando o CPF como identificador funcional exclusivo dessa pessoa. |
| RF-038 | Quando um novo vínculo possuir o mesmo CPF de uma identidade já existente, incluindo transições como Estagiário → CLT, o sistema deve reutilizar a mesma conta AD, preservando a identidade e atualizando os atributos do novo vínculo. |
| RF-039 | O sistema não deve utilizar Person ID, matrícula, nome, e-mail, UPN, `sAMAccountName` ou qualquer outro atributo como fallback para correlacionar automaticamente uma pessoa quando o CPF estiver ausente, inválido ou divergente. |
| RF-040 | Quando o CPF estiver ausente, inválido ou associado de forma conflitante a mais de uma identidade, o sistema não deve criar, reativar ou alterar automaticamente uma conta candidata; deve registrar a condição e gerar alerta para investigação. |
| RF-041 | Quando um afastamento que exige bloqueio não possuir data final, a conta deve permanecer desabilitada até que a Senior forneça um novo estado autoritativo que permita reativação. |
| RF-042 | O AD DS deve possuir metadados de lifecycle suficientes para identificar **quando** e **por que** uma conta foi desabilitada. Esses metadados devem permitir que rotinas de retenção diferenciem desligamento/cancelamento de admissão de suspensão temporária; uma conta `TEMPORARILY_SUSPENDED` não pode ser excluída apenas por estar desabilitada há mais de 30 dias. |

## Modelo de estados

O sistema deve separar o **estado funcional da identidade** do **estado técnico do processamento**.

### Identity Lifecycle State

```text
PRE_PROVISIONED
ACTIVE
TEMPORARILY_SUSPENDED
ADMISSION_CANCELLED
TERMINATED
```

Esses estados representam o que a identidade **deveria ser** segundo os dados autoritativos e as regras IAM.

`ADMISSION_CANCELLED` representa uma identidade criada antes da admissão cuja admissão foi posteriormente cancelada/excluída na Senior. Esse estado exige conta desabilitada e não deve ser confundido com um colaborador que efetivamente iniciou e depois foi desligado.

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
        +---- admissão cancelada/excluída ----> ADMISSION_CANCELLED
        |
        | condição autorizada de início do acesso
        v
ACTIVE
        |
        | férias / afastamento / suspensão cuja política exija bloqueio
        v
TEMPORARILY_SUSPENDED
        |
        | retorno vigente + nenhum estado impeditivo
        v
ACTIVE
        |
        | desligamento efetivo da pessoa
        v
TERMINATED
```

Uma recontratação ou mudança de vínculo deve reutilizar a identidade existente quando o CPF já estiver associado a uma identidade IAM. Dependendo das datas e regras de lifecycle, a conta poderá permanecer ativa, ser temporariamente desabilitada entre vínculos ou ser reativada no início do novo vínculo.

## Identidade da pessoa e múltiplos vínculos

O sistema deve distinguir conceitualmente:

```text
Pessoa
  ├── CPF: chave exclusiva de correlação
  └── identidade corporativa persistente

Vínculos
  ├── vínculo A
  ├── vínculo B
  └── ...
```

O estado da conta deve ser calculado a partir do conjunto dos vínculos elegíveis da pessoa. O encerramento isolado de um vínculo não pode desabilitar a conta quando outro vínculo elegível permanecer ativo.

Exemplo:

```text
Mesmo CPF
├── CLT = TERMINATED
└── PJ  = ACTIVE

Resultado: identidade continua ACTIVE
```

A especificação detalhada de correlação está em [`../identity/identity-correlation.md`](../identity/identity-correlation.md). As regras de múltiplos vínculos ainda estão em discussão na Issue #8.

## Política de férias e exceções individuais

A política padrão da organização é **suspender o acesso durante todo o período de férias**, desabilitando a conta no AD DS e preservando a identidade.

A Microsoft documenta o uso do estado desabilitado da conta (`ACCOUNTDISABLE`) no AD DS para impedir novos logons. Em ambiente híbrido, a revogação imediata de sessões/tokens de nuvem deve ser avaliada separadamente, pois desabilitar a conta local não garante encerramento instantâneo de sessões já emitidas.

O sistema deve permitir exceção individual autorizada pelo time de **Gente e Gestão**:

```text
Política padrão: férias => suspender acesso
           |
           +--> sem override válido: TEMPORARILY_SUSPENDED
           |
           +--> override Gente e Gestão: aplicar exceção durante toda a vigência das férias
```

Se a Senior alterar as datas das férias, tanto a suspensão padrão quanto a vigência do override devem acompanhar as novas datas.

## Metadados de desabilitação no AD DS

Para permitir auditoria e futura rotina de retenção/limpeza, o AD DS deve receber atributos controlados pelo IAM que representem, conceitualmente:

```text
iamLifecycleState
iamDisabledAt
iamDisableReason
```

Os nomes/atributos físicos ainda serão definidos no mapping do AD DS.

Exemplos de `iamDisableReason`:

```text
VACATION
LEAVE
LEAVE_NO_END_DATE
ADMISSION_CANCELLED
TERMINATION
```

Uma rotina que futuramente exclua contas desabilitadas há mais de 30 dias **não pode usar somente a idade da desabilitação**. Ela deve avaliar o lifecycle/motivo e excluir suspensões temporárias, evitando apagar uma conta de colaborador que permaneça afastado por período longo.

A exclusão automática definitiva continua fora do MVP até que a política de retenção seja aprovada.

## Política de tentativas e isolamento de falhas

Baseline funcional:

```text
Tentativa 1 -> imediata
Tentativa 2 -> após backoff
Tentativa 3 -> após backoff
Falhou novamente -> gerar alerta
```

São **3 tentativas no total por ciclo**, incluindo a inicial. O backoff deve ser progressivo e configurável; quando a dependência informar `Retry-After` ou mecanismo equivalente, esse valor deve ser respeitado. Os intervalos padrão serão definidos na configuração operacional.

Uma falha deve ficar isolada à identidade/operação afetada. O processamento dos demais colaboradores continua normalmente.

## Pontos que ainda exigem regra de negócio

Os requisitos acima definem **o que** o sistema deve suportar. Ainda precisam ser definidos separadamente:

- quantos dias antes da admissão inicia o pré-provisionamento;
- se a conta pré-provisionada nasce tecnicamente habilitada ou desabilitada;
- quando credenciais podem ser utilizadas;
- critérios de elegibilidade por empresa, tipo de vínculo ou outras condições;
- quais códigos/tipos de afastamento causam suspensão de acesso;
- horário efetivo de início e fim de suspensão;
- precedência entre férias, afastamento, desligamento e outras situações simultâneas;
- comportamento de acesso quando existe intervalo entre dois vínculos da mesma pessoa;
- regra de agregação para múltiplos vínculos simultâneos;
- precedência de atributos quando múltiplos vínculos possuem valores conflitantes;
- normalização e validação do CPF;
- estratégia de proteção/persistência da chave CPF no IAM;
- intervalos padrão do backoff quando a dependência não fornecer orientação;
- responsáveis operacionais por cada nível de severidade;
- detalhes técnicos de integração com WhatsApp, e-mail e Teams;
- política de retenção/exclusão definitiva após período de conta desabilitada.

## Fora do escopo funcional inicial

Salvo decisão posterior explícita, o MVP não inclui:

- concessão de privilégios administrativos baseada em cargo;
- RBAC completo;
- provisionamento de grupos e entitlements de aplicações;
- licenciamento Microsoft 365;
- exclusão automática definitiva de contas;
- gestão de senha de usuário final;
- dados de remuneração, folha, benefícios, saúde, dependentes ou dados bancários.

## Referências

- `docs/identity/joiner-mover-leaver.md`
- `docs/identity/attribute-mapping.md`
- `docs/requirements/audit-observability.md`
- `docs/integration/senior-admission-deletion.md`
- Microsoft Learn — UserAccountControl property flags: https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties
- Microsoft Learn — Revoke user access in Microsoft Entra ID: https://learn.microsoft.com/en-us/entra/identity/users/users-revoke-access
