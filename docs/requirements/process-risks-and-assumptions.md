# Riscos e Premissas do Processo Atual

- **Status:** Draft para discussão
- **Escopo:** Senior IAM Connector — onboarding, pré-provisionamento e lifecycle
- **Objetivo:** registrar particularidades do processo atual que podem limitar a automação, gerar riscos operacionais ou exigir mudanças de processo ou regras de negócio fora do código da integração.

> Este documento não redefine o escopo técnico do conector. Ele registra dependências e particularidades do processo de RH que precisam ser consideradas para que o lifecycle funcione corretamente.

## Processo atual — recrutamento até cadastro no Senior

Fluxo informado atualmente:

```text
Recrutamento seleciona candidato
        |
        v
Candidato escolhido é registrado no Quickin
        |
        v
Processo segue internamente
        |
        v
RH realiza posteriormente o cadastro/admissão na Senior
        |
        v
Somente a partir deste ponto o Senior IAM Connector
pode conhecer a pessoa e iniciar o pré-provisionamento
```

O Quickin ocorre antes da Senior no processo atual, porém **a integração com o Quickin não faz parte do escopo atual deste projeto**.

A Senior permanece como fonte autoritativa para a integração IAM.

## Risco identificado — cadastro tardio na Senior

Em alguns casos, o cadastro da pessoa na Senior é realizado somente próximo da data de entrega dos ativos ou da própria admissão.

Isso pode reduzir ou eliminar a janela planejada de pré-provisionamento.

Exemplo:

```text
Admissão: dia 20
Janela desejada de pré-provisionamento: D-5

Cenário esperado
15 -> cadastro já disponível na Senior
15 -> IAM cria identidade
16-19 -> TI executa processos dependentes de onboarding
20 -> colaborador inicia com ambiente preparado

Cenário de risco
19 -> RH cadastra a pessoa na Senior
19 -> IAM cria identidade
20 -> admissão

Resultado: a integração funcionou corretamente,
mas o processo não forneceu antecedência suficiente
para o onboarding.
```

### Limitação técnica importante

Sem integração com o Quickin ou outra fonte anterior à Senior, o Senior IAM Connector **não possui como detectar tecnicamente um candidato que ainda não foi cadastrado na Senior**.

Portanto, nenhuma lógica interna do conector consegue compensar integralmente um cadastro feito tarde demais na fonte autoritativa.

O sistema pode medir o tempo disponível após o registro aparecer na Senior, mas não consegue inferir que existe uma futura admissão ainda ausente da Senior.

### Impactos potenciais

Um cadastro tardio pode causar:

- criação da conta AD muito próxima da data de admissão;
- redução do tempo disponível para preparação de ativos;
- atraso em processos que dependam da identidade corporativa;
- onboarding incompleto no primeiro dia;
- aumento de tratativas manuais e urgentes entre RH e TI;
- falsa percepção de falha da integração quando a causa real é a antecedência insuficiente do dado de origem.

### Escopo atual

Neste momento:

- Quickin permanece fora do escopo da integração;
- não será utilizado Quickin como fonte autoritativa para criação de identidade;
- não será criado fluxo Quickin -> IAM no MVP;
- o provisionamento continuará sendo iniciado a partir da Senior;
- o risco de antecedência deve ser tratado inicialmente por processo, governança e monitoramento.

### Alternativas para discussão

#### 1. SLA para cadastro antecipado na Senior

Definir um prazo mínimo para que uma admissão confirmada seja cadastrada na Senior.

Exemplo conceitual:

```text
Data de admissão: D
Cadastro obrigatório na Senior: até D-5
```

O número de dias ainda deve ser definido conforme o tempo real necessário para preparar acessos e ativos.

#### 2. Tornar o cadastro na Senior um marco formal do onboarding

A confirmação do processo de contratação pode incluir explicitamente a obrigação de cadastrar a admissão na Senior com antecedência suficiente para iniciar o onboarding técnico.

A automação depende desse marco para começar.

#### 3. Indicador de antecedência

O IAM pode registrar, para cada Joiner:

- data/hora em que a admissão ficou disponível na Senior;
- data efetiva de admissão;
- quantidade de dias/horas disponíveis para pré-provisionamento;
- se a entrada ficou dentro ou fora da janela esperada.

Isso permite distinguir problema técnico de problema de processo e produzir evidência para melhoria contínua.

Exemplo:

```text
AdmissionDate: 20/09
FirstSeenInSenior: 19/09 14:32
LeadTime: < 1 dia
ExpectedLeadTime: 5 dias
OnboardingReadiness: LATE_SOURCE_REGISTRATION
```

O nome final do indicador e sua classificação ainda devem ser definidos.

#### 4. Alerta de cadastro tardio

Quando um novo Joiner aparecer na Senior com antecedência menor que a janela definida, o sistema pode gerar um aviso operacional informando que o tempo disponível para onboarding está abaixo do esperado.

Esse alerta não corrige o cadastro tardio, mas dá visibilidade imediata ao risco.

#### 5. Revisão manual de Quickin x Senior como controle de processo

Enquanto não houver integração com o Quickin, RH/Recrutamento pode adotar um controle operacional para confirmar que candidatos já aprovados e com admissão prevista foram efetivamente cadastrados na Senior dentro do SLA.

Esse controle é externo ao Senior IAM Connector.

### Alternativa não recomendada no MVP

Criar contas no AD com base em informações manuais, planilhas, e-mails ou dados não autoritativos antes do cadastro na Senior adicionaria uma segunda origem de identidade e aumentaria o risco de:

- duplicidade;
- criação para contratação posteriormente cancelada;
- inconsistência de CPF/nome/vínculo;
- dificuldade de auditoria;
- reconciliação ambígua quando a pessoa aparecer posteriormente na Senior.

Por isso, essa alternativa não deve ser adotada sem uma decisão arquitetural específica.

### Decisões necessárias com o time

- [ ] Qual antecedência mínima é necessária para o onboarding técnico?
- [ ] Em qual momento do processo o RH já possui informação suficiente para cadastrar a admissão na Senior?
- [ ] É viável estabelecer SLA de cadastro na Senior em relação à data de admissão?
- [ ] Quem será responsável por acompanhar violações desse SLA?
- [ ] O IAM deve gerar alerta quando um Joiner chegar abaixo da antecedência mínima?
- [ ] O portal de auditoria deve mostrar o indicador de lead time do onboarding?
- [ ] Como tratar admissões urgentes legitimamente cadastradas fora do prazo?
- [ ] Existe algum processo operacional atual que possa comparar Quickin e Senior sem integrar tecnicamente os dois sistemas?

### Diretriz provisória

> O Senior IAM Connector inicia o processo somente quando a admissão estiver disponível na Senior. A automação não garante a preparação antecipada do onboarding quando o cadastro na fonte autoritativa ocorrer fora da janela mínima necessária. O risco deve ser medido, auditado e tratado inicialmente por melhoria de processo, sem ampliar o escopo do MVP para integração com o Quickin.

---

## Risco identificado — múltiplos vínculos simultâneos para a mesma pessoa

Uma mesma pessoa pode possuir **mais de um vínculo ativo simultaneamente** na organização, por exemplo:

```text
Pessoa / mesmo CPF
├── Vínculo A: CLT — ACTIVE
└── Vínculo B: PJ  — ACTIVE

              |
              v
       uma única Identity IAM
              |
              v
        uma única conta AD
```

Esse cenário é diferente de uma simples mudança de vínculo, como Estagiário -> CLT, porque os vínculos podem coexistir durante o mesmo período.

### Risco funcional

Se o IAM tratar cada vínculo isoladamente, uma alteração em apenas um deles pode produzir uma ação incorreta sobre a identidade da pessoa.

Exemplo crítico:

```text
CPF: mesma pessoa

CLT: TERMINATED
PJ:  ACTIVE

Implementação incorreta:
CLT terminou -> desabilitar conta AD

Resultado:
a pessoa perde acesso apesar de ainda possuir
outro vínculo ativo e elegível.
```

Portanto, **o lifecycle da conta não pode ser derivado cegamente do estado de um único vínculo** quando a pessoa possuir múltiplos vínculos.

### Risco específico — desligamento parcial

Quando uma pessoa possui dois ou mais vínculos e apenas um deles é encerrado, o evento deve ser tratado como **desligamento de vínculo**, e não automaticamente como **desligamento da pessoa**.

Exemplo:

```text
Estado inicial

Pessoa / CPF 123
├── CLT = ACTIVE
└── PJ  = ACTIVE

Identity = ACTIVE
AD account = ENABLED

          |
          | desligamento somente do CLT
          v

Estado resultante

Pessoa / CPF 123
├── CLT = TERMINATED
└── PJ  = ACTIVE

Identity = ACTIVE
AD account = ENABLED
```

Antes de executar qualquer `disable` motivado por desligamento, o IAM deve avaliar **todos os vínculos conhecidos da pessoa** e determinar se ainda existe pelo menos um vínculo ativo e elegível para manter acesso.

A conta somente poderá assumir `TERMINATED` por desligamento quando a regra agregada concluir que **não existe nenhum outro vínculo elegível que justifique a manutenção do acesso**.

#### Risco de atributos residuais

Mesmo quando a conta deve permanecer habilitada, o encerramento de um dos vínculos pode exigir alteração de atributos no AD DS.

Exemplo:

```text
Antes

CLT = ACTIVE
  cargo       = Gerente
  gestor      = Gestor A
  empresa     = Empresa A
  matrícula   = 1001

PJ = ACTIVE
  cargo       = Consultor
  gestor      = Gestor B
  empresa     = Empresa B
  matrícula   = 9001

AD estava projetando atributos do CLT.

Depois

CLT = TERMINATED
PJ  = ACTIVE
```

Se o vínculo CLT era a origem dos atributos projetados, o IAM não pode simplesmente manter os valores antigos. Deve recalcular a origem efetiva dos atributos conforme a política de precedência e atualizar o AD de forma determinística.

Sem essa regra, a conta permaneceria habilitada corretamente, porém com informações organizacionais de um vínculo já encerrado.

#### Risco de ordem de eventos

O IAM não deve assumir que eventos de vínculos diferentes chegarão sempre na ordem ideal.

Exemplo:

```text
Evento 1 recebido: CLT -> TERMINATED
Evento 2 ainda não processado: PJ continua ACTIVE
```

Uma implementação que execute imediatamente o disable com base apenas no primeiro evento pode causar indisponibilidade temporária indevida.

Por isso, a decisão de lifecycle deve ser baseada no **estado consolidado mais recente da pessoa**, preferencialmente relendo/reconciliando seus vínculos antes de uma operação destrutiva de acesso como `disable` por desligamento.

### Impactos potenciais

Sem uma regra explícita para múltiplos vínculos, podem ocorrer:

- desabilitação indevida da conta quando apenas um dos vínculos é encerrado;
- manutenção de atributos de um vínculo já encerrado;
- reativação indevida por um vínculo quando outro vínculo deveria bloquear acesso;
- duplicidade de conta caso dois vínculos do mesmo CPF sejam processados como pessoas diferentes;
- conflito de atributos como matrícula, cargo, departamento, gestor, empresa/filial, centro de custo e tipo de vínculo;
- oscilações de atributos no AD conforme vínculos diferentes sejam processados em momentos distintos;
- comportamento não determinístico na reconciliação;
- indisponibilidade temporária causada por processamento de eventos fora de ordem;
- auditoria difícil de explicar, pois uma ação sobre a identidade pode ter sido causada por apenas um dos vários vínculos existentes.

### Modelo conceitual necessário

A identidade deve representar a pessoa, enquanto os vínculos devem ser tratados como entidades relacionadas à mesma identidade:

```text
Pessoa / CPF
      |
      v
Identity IAM
      |
      +--> Employment A / CLT
      |
      +--> Employment B / PJ
      |
      +--> Employment C / outro vínculo, quando aplicável
      |
      v
Estado efetivo da identidade
calculado a partir do conjunto de vínculos
```

A existência de múltiplos vínculos **não altera a regra de correlação por CPF**: o mesmo CPF continua representando uma única pessoa e uma única identidade corporativa.

### Casos que precisam de regra de negócio

#### Encerramento de apenas um vínculo

Exemplo:

```text
Antes:
CLT = ACTIVE
PJ  = ACTIVE
Identity = ACTIVE

Depois:
CLT = TERMINATED
PJ  = ACTIVE
```

A conta **não deve ser automaticamente tratada como `TERMINATED`** apenas porque o vínculo CLT terminou. A regra final deve considerar se ainda existe outro vínculo ativo e elegível que justifique manutenção do acesso.

Além disso, deve ser avaliado se os atributos projetados no AD precisam migrar do vínculo encerrado para o vínculo remanescente.

#### Suspensão de apenas um vínculo

Exemplo:

```text
CLT = TEMPORARILY_SUSPENDED
PJ  = ACTIVE
```

Ainda precisa ser decidido se a identidade permanece `ACTIVE` pelo vínculo PJ ou se alguma situação de suspensão possui precedência global sobre todos os vínculos da pessoa.

Essa decisão depende da natureza da suspensão e das políticas de acesso da organização.

#### Atributos conflitantes

Com múltiplos vínculos ativos, dois registros podem apresentar valores diferentes:

```text
CLT
Cargo: Analista
Gestor: Gestor A
Departamento: Tecnologia
Empresa: Empresa A

PJ
Cargo: Consultor
Gestor: Gestor B
Departamento: Projetos
Empresa: Empresa B
```

É necessário definir qual vínculo será considerado fonte dos atributos corporativos gravados no AD, ou se determinados atributos não podem ser representados por um único valor.

Possíveis abordagens a discutir:

- vínculo principal explicitamente indicado pela Senior/processo;
- precedência por tipo de vínculo;
- precedência por empresa/filial;
- regra específica por atributo;
- armazenamento de todos os vínculos no IAM, mas projeção de apenas um conjunto de atributos para o AD.

Nenhuma dessas abordagens está aprovada neste momento.

### Diretriz provisória

Até a regra ser aprovada:

> O IAM deve assumir que uma pessoa pode possuir zero, um ou vários vínculos simultâneos. A decisão de habilitar, suspender ou desligar a identidade deve considerar o conjunto de vínculos relevantes da pessoa e não apenas o registro que disparou o processamento. O encerramento de um vínculo isolado não deve causar automaticamente a desabilitação da conta quando existir outro vínculo ativo e elegível para a mesma pessoa. Antes de um disable por desligamento, o estado consolidado da pessoa deve ser validado. Se o vínculo encerrado fornecia atributos ao AD, esses atributos devem ser recalculados a partir dos vínculos remanescentes conforme política determinística.

### Decisões necessárias com o time

- [ ] Quais tipos de vínculo podem coexistir para a mesma pessoa?
- [ ] Quais tipos de vínculo são elegíveis para manter acesso corporativo?
- [ ] Como calcular o estado efetivo da identidade quando existem vários vínculos?
- [ ] O encerramento de um vínculo pode desabilitar a conta se outro vínculo continuar ativo?
- [ ] Antes de um disable por desligamento, quais dados devem ser relidos/reconciliados para garantir que não existe outro vínculo elegível?
- [ ] Como tratar eventos de vínculos recebidos fora de ordem?
- [ ] Como férias/afastamento/suspensão de um vínculo interagem com outro vínculo ativo?
- [ ] Existe na Senior algum indicador de vínculo principal?
- [ ] Caso não exista, qual regra determina o vínculo principal?
- [ ] Qual vínculo fornece matrícula, cargo, gestor, empresa, departamento e centro de custo ao AD?
- [ ] Quando o vínculo principal é encerrado, como os atributos migram para um vínculo remanescente?
- [ ] A precedência deve ser definida por vínculo ou por atributo?
- [ ] Como auditar qual conjunto de vínculos levou ao estado final aplicado na identidade?
- [ ] Como o portal de auditoria exibirá múltiplos vínculos para uma mesma identidade?

## Referências internas

- `docs/requirements/functional-requirements.md`
- `docs/identity/identity-correlation.md`
- `docs/identity/joiner-mover-leaver.md`
- `docs/integration/senior-admission-deletion.md`
