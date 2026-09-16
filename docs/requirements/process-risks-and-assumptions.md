# Riscos e Premissas do Processo Atual

- **Status:** Draft para discussão
- **Escopo:** Senior IAM Connector — onboarding e pré-provisionamento
- **Objetivo:** registrar particularidades do processo atual que podem limitar a automação, gerar riscos operacionais ou exigir mudanças de processo fora do código da integração.

> Este documento não redefine o escopo técnico do conector. Ele registra dependências do processo de RH que precisam ser consideradas para que o pré-provisionamento e o onboarding funcionem dentro do prazo esperado.

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

## Limitação técnica importante

Sem integração com o Quickin ou outra fonte anterior à Senior, o Senior IAM Connector **não possui como detectar tecnicamente um candidato que ainda não foi cadastrado na Senior**.

Portanto, nenhuma lógica interna do conector consegue compensar integralmente um cadastro feito tarde demais na fonte autoritativa.

O sistema pode medir o tempo disponível após o registro aparecer na Senior, mas não consegue inferir que existe uma futura admissão ainda ausente da Senior.

## Impactos potenciais

Um cadastro tardio pode causar:

- criação da conta AD muito próxima da data de admissão;
- redução do tempo disponível para preparação de ativos;
- atraso em processos que dependam da identidade corporativa;
- onboarding incompleto no primeiro dia;
- aumento de tratativas manuais e urgentes entre RH e TI;
- falsa percepção de falha da integração quando a causa real é a antecedência insuficiente do dado de origem.

## Escopo atual

Neste momento:

- Quickin permanece fora do escopo da integração;
- não será utilizado Quickin como fonte autoritativa para criação de identidade;
- não será criado fluxo Quickin -> IAM no MVP;
- o provisionamento continuará sendo iniciado a partir da Senior;
- o risco de antecedência deve ser tratado inicialmente por processo, governança e monitoramento.

## Alternativas para discussão

As seguintes alternativas devem ser avaliadas com RH, Recrutamento, TI/IAM e responsáveis pelo onboarding antes de definir uma solução final.

### 1. SLA para cadastro antecipado na Senior

Definir um prazo mínimo para que uma admissão confirmada seja cadastrada na Senior.

Exemplo conceitual:

```text
Data de admissão: D
Cadastro obrigatório na Senior: até D-5
```

O número de dias ainda deve ser definido conforme o tempo real necessário para preparar acessos e ativos.

### 2. Tornar o cadastro na Senior um marco formal do onboarding

A confirmação do processo de contratação pode incluir explicitamente a obrigação de cadastrar a admissão na Senior com antecedência suficiente para iniciar o onboarding técnico.

A automação depende desse marco para começar.

### 3. Indicador de antecedência

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

### 4. Alerta de cadastro tardio

Quando um novo Joiner aparecer na Senior com antecedência menor que a janela definida, o sistema pode gerar um aviso operacional informando que o tempo disponível para onboarding está abaixo do esperado.

Esse alerta não corrige o cadastro tardio, mas dá visibilidade imediata ao risco.

### 5. Revisão manual de Quickin x Senior como controle de processo

Enquanto não houver integração com o Quickin, RH/Recrutamento pode adotar um controle operacional para confirmar que candidatos já aprovados e com admissão prevista foram efetivamente cadastrados na Senior dentro do SLA.

Esse controle é externo ao Senior IAM Connector.

## Alternativa não recomendada no MVP

Criar contas no AD com base em informações manuais, planilhas, e-mails ou dados não autoritativos antes do cadastro na Senior adicionaria uma segunda origem de identidade e aumentaria o risco de:

- duplicidade;
- criação para contratação posteriormente cancelada;
- inconsistência de CPF/nome/vínculo;
- dificuldade de auditoria;
- reconciliação ambígua quando a pessoa aparecer posteriormente na Senior.

Por isso, essa alternativa não deve ser adotada sem uma decisão arquitetural específica.

## Decisões necessárias com o time

- [ ] Qual antecedência mínima é necessária para o onboarding técnico?
- [ ] Em qual momento do processo o RH já possui informação suficiente para cadastrar a admissão na Senior?
- [ ] É viável estabelecer SLA de cadastro na Senior em relação à data de admissão?
- [ ] Quem será responsável por acompanhar violações desse SLA?
- [ ] O IAM deve gerar alerta quando um Joiner chegar abaixo da antecedência mínima?
- [ ] O portal de auditoria deve mostrar o indicador de lead time do onboarding?
- [ ] Como tratar admissões urgentes legitimamente cadastradas fora do prazo?
- [ ] Existe algum processo operacional atual que possa comparar Quickin e Senior sem integrar tecnicamente os dois sistemas?

## Diretriz provisória

Até nova decisão:

> O Senior IAM Connector inicia o processo somente quando a admissão estiver disponível na Senior. A automação não garante a preparação antecipada do onboarding quando o cadastro na fonte autoritativa ocorrer fora da janela mínima necessária. O risco deve ser medido, auditado e tratado inicialmente por melhoria de processo, sem ampliar o escopo do MVP para integração com o Quickin.

## Referências internas

- `docs/requirements/functional-requirements.md`
- `docs/identity/joiner-mover-leaver.md`
- `docs/integration/senior-admission-deletion.md`
