# Requisitos Funcionais

- **Status:** Draft consolidado em revisão
- **Escopo:** Senior IAM Connector — MVP
- **Objetivo:** definir o comportamento funcional esperado do sistema de forma explícita, testável e sem depender de regras subentendidas.

> Esta revisão consolida requisitos anteriormente fragmentados. Cada RF representa uma capacidade funcional única, mas suas condições, exceções e efeitos obrigatórios permanecem escritos de forma explícita.

## Princípios funcionais

- O Senior Gestão de Pessoas é a fonte autoritativa do estado dos vínculos e dos eventos de RH utilizados pelo IAM.
- A conta corporativa representa a **pessoa**, e não um vínculo trabalhista específico.
- O **CPF é a única chave funcional autorizada para determinar que diferentes vínculos pertencem à mesma pessoa**.
- Nenhum outro atributo pode ser utilizado como fallback de correlação de pessoa.
- Uma pessoa pode possuir zero, um ou vários vínculos simultâneos e continuar representada por uma única Identity IAM e uma única conta AD DS.
- O estado efetivo da identidade deve ser derivado do conjunto de vínculos relevantes da pessoa, e não apenas do registro que disparou o processamento.
- Criação da identidade e habilitação do acesso são decisões separadas.
- O comportamento deve ser determinístico, auditável e idempotente.
- Uma falha no processamento de uma identidade não pode interromper o processamento das demais identidades independentes.
- Dados fora do escopo IAM, como remuneração e folha, não fazem parte do contrato funcional.
- Exceções IAM não alteram os dados autoritativos da Senior; alteram apenas a ação de acesso permitida pela política IAM.
- Estados impeditivos de maior precedência não podem ser anulados por exceções de férias ou suspensão.

---

# Requisitos consolidados

## RF-001 — Elegibilidade e leitura da fonte autoritativa

O sistema deve identificar, a partir dos dados autoritativos da Senior, pessoas e vínculos elegíveis para processamento IAM.

O sistema deve:

1. aplicar os critérios de elegibilidade aprovados para empresa, tipo de vínculo, situação e demais regras de escopo;
2. não provisionar registros que não atendam aos critérios de elegibilidade;
3. registrar a decisão de não provisionar quando necessário para auditoria;
4. interpretar estados e datas da Senior para determinar o lifecycle esperado da identidade.

Os códigos/situações exatos da Senior continuam sujeitos ao Discovery.

## RF-002 — Criação automática da identidade

Para uma pessoa elegível sem identidade corporativa existente, o sistema deve criar automaticamente uma única identidade correspondente no AD DS.

A criação deve:

1. ocorrer apenas após correlação e validação bem-sucedidas;
2. utilizar os atributos obrigatórios aprovados;
3. utilizar naming determinístico conforme a política de naming;
4. não criar uma segunda conta em reprocessamentos, retries ou eventos duplicados;
5. produzir trilha de auditoria ponta a ponta.

## RF-003 — Pré-provisionamento e eventos futuros

O sistema deve permitir criar a identidade antes da data efetiva de admissão, dentro de uma janela configurável de pré-provisionamento.

A criação antecipada não implica habilitação do acesso.

Antes de qualquer ativação futura, o sistema deve considerar alterações, adiamentos ou cancelamentos registrados na Senior.

Quando uma admissão já pré-provisionada for confirmadamente cancelada/excluída na Senior, o sistema deve:

1. cancelar qualquer ativação futura relacionada à admissão original;
2. classificar a identidade como `ADMISSION_CANCELLED`;
3. garantir que a conta permaneça ou seja colocada em estado desabilitado no AD DS;
4. preservar a identidade IAM, a correlação por CPF e o objeto AD para auditoria e possível reutilização futura;
5. registrar de forma auditável como o cancelamento foi confirmado;
6. não interpretar uma simples falha de consulta ou ausência isolada em uma consulta como cancelamento confirmado.

## RF-004 — Correlação exclusiva da pessoa por CPF

O sistema deve utilizar exclusivamente o **CPF válido** como chave funcional para determinar que diferentes vínculos pertencem à mesma pessoa.

A partir dessa correlação, o IAM deve localizar a Identity IAM e a conta AD DS já associadas à pessoa.

É proibido utilizar como fallback de correlação automática de pessoa:

- Person ID da Senior;
- matrícula;
- identificador de vínculo/contrato;
- nome;
- e-mail;
- UPN;
- `sAMAccountName`;
- cargo, departamento, gestor ou qualquer outro atributo.

Esses campos podem ser utilizados para contexto, provisioning e auditoria, mas não para decidir que dois registros pertencem à mesma pessoa.

## RF-005 — Falha ou conflito de correlação

Quando o CPF estiver ausente, inválido ou associado de forma conflitante a mais de uma identidade, o sistema deve:

1. bloquear criação, alteração, reativação ou reutilização automática da identidade candidata;
2. não tentar correlação por qualquer outro atributo;
3. registrar a condição de forma auditável;
4. gerar alerta para investigação conforme severidade aplicável;
5. isolar a falha à identidade afetada, sem bloquear o processamento das demais pessoas.

## RF-006 — Validação de dados obrigatórios

O sistema deve rejeitar de forma controlada registros elegíveis que não possuam os dados obrigatórios necessários à operação solicitada.

A rejeição deve:

1. não produzir alteração parcial insegura no AD DS;
2. registrar quais requisitos de dados não foram atendidos sem expor dados proibidos;
3. gerar alerta quando a ausência impedir uma operação relevante de lifecycle;
4. permitir reprocessamento posterior quando o dado autoritativo for corrigido.

## RF-007 — Idempotência, ordenação lógica e reprocessamento

O sistema deve processar eventos, alterações e retries de forma idempotente.

O sistema deve:

1. impedir duplicidade de identidade em eventos repetidos ou reprocessamentos;
2. permitir reprocessar com segurança uma operação com falha;
3. preservar o estado final correto mesmo quando múltiplas alterações da mesma pessoa forem recebidas em momentos diferentes;
4. evitar que a ordem técnica de chegada de eventos produza estado funcional incorreto;
5. recalcular o estado efetivo da pessoa quando necessário antes de executar ações destrutivas ou de bloqueio.

## RF-008 — Vínculo persistente entre IAM e AD DS

O sistema deve manter associação técnica persistente entre a Identity IAM e o objeto correspondente no AD DS.

Depois que a pessoa estiver correlacionada, as operações técnicas devem utilizar essa associação para atuar sobre a mesma conta, preservando, quando aplicável:

- AD `objectGUID` ou chave técnica equivalente;
- `sAMAccountName`;
- UPN;
- histórico de lifecycle;
- histórico de provisioning.

A associação técnica não substitui o CPF como chave funcional exclusiva para determinar a pessoa.

## RF-009 — Múltiplos vínculos simultâneos e estado efetivo da identidade

Uma Identity IAM pode estar associada a múltiplos vínculos simultâneos da mesma pessoa, incluindo cenários como CLT + PJ.

Antes de habilitar, suspender, reativar ou desabilitar a conta, o sistema deve considerar o conjunto de vínculos conhecidos e elegíveis da pessoa.

O sistema deve obedecer às seguintes regras:

1. um segundo vínculo com o mesmo CPF não cria uma segunda conta AD;
2. o encerramento de um vínculo isolado não deve desabilitar a conta quando existir outro vínculo ativo e elegível;
3. a identidade somente pode assumir `TERMINATED` quando não existir outro vínculo elegível que justifique manutenção do acesso;
4. antes de um `disable` de Leaver, deve ser validado o estado consolidado mais recente da pessoa;
5. quando o vínculo encerrado era a origem de atributos projetados no AD DS, esses atributos devem ser recalculados conforme a política de precedência aplicável aos vínculos remanescentes;
6. o processamento em ordens diferentes deve convergir para o mesmo estado final.

A regra final de precedência entre atributos de vínculos simultâneos permanece em definição na Issue #8.

## RF-010 — Rehire e mudança de vínculo

Quando um novo vínculo possuir o mesmo CPF de uma identidade já existente, incluindo Rehire, Estagiário → CLT, temporário → CLT, PJ → CLT ou outras transições elegíveis, o sistema deve reutilizar a mesma identidade corporativa.

O sistema deve:

1. reutilizar a mesma conta AD DS quando ela ainda existir;
2. preservar a correlação da pessoa;
3. preservar `objectGUID`, username e UPN por padrão, conforme política de naming;
4. atualizar os atributos relacionados ao vínculo vigente;
5. reativar a conta quando o lifecycle permitir;
6. nunca criar uma nova identidade apenas porque matrícula, tipo de vínculo ou referência interna da Senior mudou.

## RF-011 — Mover e atualização de atributos aprovados

Quando ocorrer alteração relevante e autorizada na Senior, o sistema deve atualizar na mesma conta AD DS apenas os atributos aprovados no contrato de identidade.

Podem fazer parte desse contrato, conforme mapping aprovado:

- gestor;
- cargo;
- departamento;
- empresa/filial;
- centro de custo;
- localidade;
- tipo de vínculo;
- nome e demais atributos de identidade aprovados.

A lista técnica, o campo Senior e o atributo AD correspondente são definidos em `attribute-mapping.md`.

Mudanças de cargo, departamento ou vínculo não concedem automaticamente privilégios administrativos, grupos ou entitlements fora do escopo aprovado.

## RF-012 — Naming, unicidade e estabilidade da conta

A criação de `sAMAccountName` e UPN deve seguir a política determinística definida em `docs/identity/naming-policy.md`.

O sistema deve:

1. gerar candidatos de naming conforme a política aprovada;
2. validar unicidade antes da criação ou rename;
3. persistir e reutilizar o naming atribuído;
4. não recalcular username/UPN por alterações normais de cargo, departamento, gestor, empresa, matrícula, Rehire ou mudança de vínculo;
5. preservar a mesma identidade quando houver mudança de nome;
6. somente avaliar rename quando existir gatilho autorizado pela política de naming;
7. quando o nome gerado automaticamente não atender às regras e for necessário um nome diferente, permitir tratamento excepcional conforme processo aprovado, preservando a mesma conta e registrando o naming efetivamente aplicado;
8. impedir que reconciliações futuras sobrescrevam um naming excepcional aprovado por novo cálculo automático.

## RF-013 — Separação entre Lifecycle State e Provisioning State

O sistema deve manter separadamente:

- o estado funcional esperado da identidade (`Identity Lifecycle State`);
- o estado técnico do processamento (`Provisioning State`).

Uma falha de provisioning não deve alterar indevidamente o significado do lifecycle.

Exemplo:

```text
Identity Lifecycle State: ACTIVE
Provisioning State: FAILED
```

Isso significa que a pessoa deveria estar ativa, mas a operação técnica necessária não convergiu.

## RF-014 — Suspensão temporária

O sistema deve suportar suspensão temporária de acesso sem tratar a pessoa como desligada.

Quando a política efetiva exigir bloqueio, o sistema deve:

1. utilizar a mesma conta AD DS;
2. colocar a conta em estado desabilitado;
3. classificar a identidade como `TEMPORARILY_SUSPENDED`;
4. preservar a correlação, o objeto AD e o histórico;
5. não excluir nem recriar a conta;
6. respeitar a vigência do evento autoritativo;
7. registrar o motivo da suspensão para auditoria e retenção.

## RF-015 — Política de férias e override individual

A política padrão da organização para férias é **desabilitar a conta no AD DS durante todo o período de férias**, preservando a identidade.

O sistema deve permitir override individual somente quando autorizado pelo time de **Gente e Gestão**.

Todo override deve:

1. registrar colaborador afetado;
2. registrar decisão aplicada;
3. registrar responsável/autorizador;
4. registrar data/hora e justificativa;
5. possuir vigência correspondente ao período inteiro das férias;
6. acompanhar eventual alteração das datas de férias registrada na Senior;
7. não prevalecer sobre desligamento ou outro estado impeditivo de maior precedência.

A eventual revogação imediata de sessões/tokens em Microsoft Entra/M365 é controle técnico adicional a ser validado separadamente; o baseline funcional no AD DS é a conta desabilitada.

## RF-016 — Retorno, alteração de data e afastamento sem data final

O sistema deve reativar a mesma conta quando a pessoa sair de uma suspensão e voltar a um estado elegível, desde que não exista outra condição impeditiva.

O sistema deve:

1. utilizar a data vigente da Senior para determinar o retorno;
2. quando a data de retorno for alterada, substituir a programação anterior pela nova data;
3. impedir reativação automática quando houver estado de maior precedência que exija bloqueio;
4. quando um afastamento que exige bloqueio não possuir data final, manter a conta desabilitada por tempo indeterminado;
5. reativar uma conta em afastamento sem data final somente após novo estado autoritativo da Senior permitir o retorno.

## RF-017 — Metadados de lifecycle no AD DS e proteção de retenção

O AD DS deve possuir metadados controlados pelo IAM suficientes para identificar **quando** e **por que** uma conta foi desabilitada.

Conceitualmente, devem existir informações equivalentes a:

```text
iamLifecycleState
iamDisabledAt
iamDisableReason
```

Os atributos físicos serão definidos no mapping técnico.

Esses metadados devem permitir diferenciar, no mínimo:

- férias;
- afastamento;
- afastamento sem data final;
- cancelamento de admissão;
- desligamento.

Uma futura rotina de limpeza de contas desabilitadas há mais de 30 dias não poderá utilizar somente a idade da desabilitação. Contas em `TEMPORARILY_SUSPENDED`, inclusive afastamentos prolongados sem data final, não podem ser excluídas apenas por ultrapassarem 30 dias desabilitadas.

A exclusão automática definitiva permanece fora do MVP até aprovação de política específica de retenção.

## RF-018 — Leaver e desabilitação por desligamento

Quando o **estado efetivo da pessoa** atingir desligamento, o sistema deve desabilitar a conta no AD DS conforme a data e o horário efetivos definidos pela regra de negócio.

Antes do disable, o sistema deve verificar o estado consolidado dos vínculos da pessoa.

O sistema não deve desabilitar a conta por encerramento isolado de um vínculo quando outro vínculo ativo e elegível permanecer vigente.

Quando o desligamento efetivo da pessoa estiver confirmado:

1. a identidade assume `TERMINATED`;
2. a conta deve ser desabilitada;
3. férias, afastamentos ou overrides não podem impedir o bloqueio;
4. a falha em atingir o estado desabilitado deve gerar alerta `CRITICAL`;
5. o MVP não deve excluir automaticamente a conta.

## RF-019 — Auditoria das operações de lifecycle

O sistema deve manter trilha auditável das operações de lifecycle e provisioning.

A auditoria deve cobrir, no mínimo:

- criação;
- alteração;
- pré-provisionamento;
- cancelamento de admissão;
- suspensão temporária;
- override;
- reativação;
- desligamento;
- Rehire;
- reconciliação;
- retries e falhas.

Cada fluxo deve permitir identificar, quando aplicável:

1. identidade afetada;
2. operação;
3. estado esperado antes e depois;
4. origem da decisão;
5. timestamp;
6. correlation ID;
7. tentativa;
8. resultado;
9. erro;
10. alerta relacionado;
11. responsável/autorizador quando houver intervenção ou override.

## RF-020 — Retry e isolamento de falhas

Cada operação de provisioning com falha transitória deve possuir no máximo **3 tentativas por ciclo**, incluindo a tentativa inicial.

O sistema deve:

1. utilizar backoff progressivo e configurável entre tentativas;
2. respeitar `Retry-After` ou mecanismo equivalente quando fornecido pela dependência;
3. após a terceira falha, manter a operação identificada como `FAILED` e gerar alerta conforme severidade;
4. preservar o histórico das tentativas;
5. permitir reprocessamento posterior seguro;
6. garantir que uma identidade em falha/retry não bloqueie, interrompa ou impeça o processamento das demais identidades independentes.

## RF-021 — Severidade e canais de alerta

O sistema deve classificar alertas conforme impacto de lifecycle.

Baseline funcional:

| Situação | Severidade | Canais mínimos |
|---|---|---|
| Pessoa deveria estar `TERMINATED`, mas a conta permanece habilitada | `CRITICAL` | WhatsApp + e-mail + Teams |
| Suspensão obrigatória não aplicada | `HIGH` | e-mail + Teams |
| `ADMISSION_CANCELLED` com conta ainda habilitada | `HIGH` | e-mail + Teams |
| Falha persistente de reativação de pessoa que deveria estar `ACTIVE` | `HIGH` | e-mail + Teams |
| Falha de criação com impacto imediato no onboarding | `HIGH` | e-mail + Teams |
| Falha persistente em atributo organizacional não crítico | `MEDIUM` | Teams |
| Cadastro tardio/processo fora do SLA, quando essa política for aprovada | `MEDIUM` | Teams |
| Retry que convergiu automaticamente sem impacto persistente | `LOW/INFO` | portal/audit log |

Os grupos, responsáveis e integrações técnicas exatas de WhatsApp, e-mail e Teams ainda devem ser definidos operacionalmente.

## RF-022 — Reconciliação e detecção de divergência

O sistema deve executar reconciliação periódica entre:

- estado autoritativo da Senior;
- estado esperado calculado pelo IAM;
- estado materializado no AD DS/provisioning.

A reconciliação deve detectar, no mínimo:

1. eventos perdidos;
2. alterações não processadas;
3. divergências persistentes;
4. operações assíncronas que não convergiram;
5. contas que deveriam estar bloqueadas ou reativadas;
6. inconsistências provocadas por múltiplos vínculos ou eventos fora de ordem.

Divergências persistentes devem gerar alerta conforme impacto.

Mesmo que a Senior disponibilize eventos/webhooks, a reconciliação continua obrigatória como mecanismo de segurança.

## RF-023 — Minimização de dados e exclusão de dados de folha

O sistema deve utilizar apenas os dados necessários para identidade e lifecycle.

O conector não deve utilizar, solicitar para finalidade IAM, persistir desnecessariamente ou registrar em logs:

- salário/remuneração;
- eventos e valores de folha;
- dados bancários;
- benefícios fora do escopo IAM;
- dados de saúde/médicos;
- dependentes;
- outros dados pessoais sem necessidade técnica comprovada.

O CPF é a exceção necessária para correlação funcional da pessoa e deve possuir controles específicos de proteção e minimização de exposição.

---

# Modelo de estados

O sistema deve separar o **estado funcional da identidade** do **estado técnico do processamento**.

## Identity Lifecycle State

```text
PRE_PROVISIONED
ACTIVE
TEMPORARILY_SUSPENDED
ADMISSION_CANCELLED
TERMINATED
```

### Significado

- `PRE_PROVISIONED`: identidade criada antes da liberação do acesso.
- `ACTIVE`: pessoa possui condição efetiva que permite acesso.
- `TEMPORARILY_SUSPENDED`: identidade continua válida, mas o acesso deve permanecer temporariamente bloqueado.
- `ADMISSION_CANCELLED`: identidade foi criada antes da admissão, mas a admissão foi cancelada/excluída antes do início efetivo.
- `TERMINATED`: o estado consolidado da pessoa não possui vínculo elegível que justifique manutenção do acesso e a regra de desligamento foi atingida.

## Provisioning State

```text
PENDING
PROCESSING
SUCCEEDED
FAILED
```

Esses estados representam se o sistema conseguiu materializar o estado esperado no target.

---

# Transições funcionais iniciais

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
        | férias / afastamento / suspensão cuja política exige bloqueio
        v
TEMPORARILY_SUSPENDED
        |
        | retorno vigente + nenhum estado impeditivo
        v
ACTIVE
        |
        | nenhum vínculo elegível permanece ativo
        | + regra efetiva de desligamento atingida
        v
TERMINATED
```

Com múltiplos vínculos, eventos ocorrem no vínculo, mas a decisão de acesso deve ser tomada sobre o **estado consolidado da identidade da pessoa**.

Exemplo:

```text
CLT = TERMINATED
PJ  = ACTIVE

Identity = ACTIVE
AD = ENABLED
```

---

# Política de precedência

A precedência final precisa considerar também múltiplos vínculos. Como baseline de lifecycle da identidade:

```text
TERMINATED
        >
TEMPORARILY_SUSPENDED
        >
ACTIVE
        >
PRE_PROVISIONED
```

`ADMISSION_CANCELLED` é um estado terminal específico de pré-admissão e não deve ser tratado como `TERMINATED` de uma pessoa que efetivamente iniciou o vínculo.

A aplicação dessa precedência depende primeiro do cálculo consolidado dos vínculos. Um `TERMINATED` isolado em um vínculo não significa `TERMINATED` da identidade quando outro vínculo elegível permanece ativo.

---

# Decisões ainda abertas

Os requisitos consolidados acima registram as regras já definidas. Permanecem pendentes:

- número de dias de antecedência do pré-provisionamento;
- estado técnico inicial da conta pré-provisionada;
- momento exato de liberação inicial de credenciais;
- critérios finais de elegibilidade por empresa e tipo de vínculo;
- códigos/situações da Senior que representam férias, afastamentos, retorno e desligamento;
- horário efetivo de início/fim de suspensão e desligamento;
- comportamento detalhado quando eventos de suspensão se sobrepõem;
- política de lifecycle quando um vínculo está suspenso e outro vínculo simultâneo está ativo;
- regra de precedência de atributos entre múltiplos vínculos;
- comportamento quando a conta de um Rehire já foi removida por futura política de retenção;
- normalização/validação técnica do CPF;
- estratégia de proteção da representação persistida do CPF;
- atributos físicos do AD DS usados para `iamLifecycleState`, `iamDisabledAt` e `iamDisableReason`;
- intervalos padrão de backoff;
- grupos/responsáveis exatos por alertas;
- mecanismo técnico de WhatsApp;
- eventual revogação imediata de sessões/tokens de Microsoft Entra/M365 durante férias/suspensões;
- SLA de cadastro antecipado na Senior para onboarding, tratado na Issue #7;
- política final de retenção/exclusão de contas;
- regras finais de múltiplos vínculos, tratadas na Issue #8.

---

# Fora do escopo funcional inicial

Salvo decisão posterior explícita, o MVP não inclui:

- integração Quickin → IAM;
- criação de identidade a partir do Quickin;
- concessão automática de privilégios administrativos baseada em cargo;
- RBAC completo;
- provisionamento de grupos e entitlements de aplicações;
- licenciamento Microsoft 365;
- exclusão automática definitiva de contas;
- gestão de senha de usuário final;
- dados de remuneração, folha, benefícios, saúde, dependentes ou dados bancários.

---

# Rastreabilidade da consolidação

A numeração anterior possuía RF-001 a RF-042. A tabela abaixo registra onde cada requisito anterior passou a ser representado.

| RF anterior | RF consolidado | Observação |
|---|---|---|
| RF-001 | RF-001 | Elegibilidade |
| RF-002 | RF-002 | Criação automática |
| RF-003 | RF-011 | Atualização de atributos |
| RF-004 | RF-018 | Leaver considerando estado efetivo da pessoa |
| RF-005 | RF-010 | Rehire/reutilização |
| RF-006 | RF-004 | Correlação exclusiva por CPF |
| RF-007 | RF-007 | Idempotência/duplicidade |
| RF-008 | RF-019 | Auditoria de tentativas |
| RF-009 | RF-019 | Auditoria de falhas |
| RF-010 | RF-007 / RF-020 | Reprocessamento seguro e retry |
| RF-011 | RF-022 | Reconciliação |
| RF-012 | RF-001 / RF-006 | Elegibilidade e validação |
| RF-013 | RF-003 | Pré-provisionamento |
| RF-014 | RF-018 | Data/horário de desligamento |
| RF-015 | RF-011 | Atributos organizacionais |
| RF-016 | RF-007 | Ordenação e consistência |
| RF-017 | RF-013 | Lifecycle State x Provisioning State |
| RF-018 | RF-019 | Auditoria de operações de lifecycle |
| RF-019 | RF-023 | Minimização de dados |
| RF-020 | RF-008 | Vínculo persistente IAM ↔ AD |
| RF-021 | RF-014 / RF-015 | Suspensão e férias |
| RF-022 | RF-016 | Reativação |
| RF-023 | RF-014 | Suspensão não é desligamento |
| RF-024 | RF-014 / RF-016 | Vigência e alteração de datas |
| RF-025 | RF-016 | Bloqueio por precedência |
| RF-026 | RF-003 | Criação separada de habilitação |
| RF-027 | RF-003 | Cancelamento de admissão |
| RF-028 | RF-003 / RF-007 | Alteração/cancelamento de eventos futuros |
| RF-029 | RF-006 | Dados obrigatórios |
| RF-030 | RF-015 | Override autorizado por Gente e Gestão |
| RF-031 | RF-015 / RF-019 | Auditoria de override |
| RF-032 | RF-015 / RF-016 | Precedência sobre override |
| RF-033 | RF-020 / RF-021 | 3 tentativas + alerta + isolamento |
| RF-034 | RF-021 | Severidade de alertas |
| RF-035 | RF-019 | Rastreabilidade de falha/alerta/resolução |
| RF-036 | RF-022 | Divergência persistente |
| RF-037 | RF-004 | Identidade associada à pessoa por CPF |
| RF-038 | RF-010 | Reutilização em novo vínculo |
| RF-039 | RF-004 | Proibição de fallback |
| RF-040 | RF-005 | Falha/conflito de CPF |
| RF-041 | RF-016 | Afastamento sem data final |
| RF-042 | RF-017 | Metadados no AD e proteção contra exclusão indevida |

## Novos requisitos explícitos gerados pela consolidação

Algumas regras já documentadas em outros arquivos passaram a aparecer explicitamente neste documento:

- **RF-009:** múltiplos vínculos simultâneos e desligamento parcial;
- **RF-012:** naming, unicidade, estabilidade e exceção de naming;
- **RF-021:** matriz funcional de severidade e canais de alerta.

---

# Referências internas

- `docs/identity/identity-correlation.md`
- `docs/identity/naming-policy.md`
- `docs/identity/attribute-mapping.md`
- `docs/identity/joiner-mover-leaver.md`
- `docs/requirements/audit-observability.md`
- `docs/requirements/process-risks-and-assumptions.md`
- `docs/integration/senior-admission-deletion.md`
- Issue #7 — SLA de cadastro na Senior para onboarding
- Issue #8 — múltiplos vínculos simultâneos

## Referências Microsoft para validação técnica

- Microsoft Learn — UserAccountControl property flags: https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties
- Microsoft Learn — Revoke user access in Microsoft Entra ID: https://learn.microsoft.com/en-us/entra/identity/users/users-revoke-access
