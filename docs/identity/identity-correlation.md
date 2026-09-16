# Identity Correlation — Pessoa, Vínculo e Conta AD

- **Status:** Proposed
- **Objetivo:** definir como uma pessoa da Senior é correlacionada com uma identidade corporativa no AD DS ao longo de múltiplos vínculos.

## Princípio central

A **conta corporativa representa a pessoa, e não o vínculo trabalhista**.

Uma mesma pessoa pode possuir diferentes vínculos ao longo do tempo, por exemplo:

- Estagiário → CLT;
- Temporário → CLT;
- Terceiro → CLT;
- transferência entre empresas/filiais;
- desligamento → recontratação.

Essas mudanças não devem, por si só, resultar na criação de uma nova conta no AD DS.

## Modelo conceitual

```text
Pessoa Senior
├── Person ID: identificador permanente da pessoa
├── CPF: identificador pessoal auxiliar
│
├── Vínculo A
│   ├── Tipo: Estagiário
│   └── Matrícula: 1234
│
└── Vínculo B
    ├── Tipo: CLT
    └── Matrícula: 9876

                 |
                 v

Identity IAM
├── sourcePersonId
├── AD objectGUID
├── sAMAccountName
└── UPN
```

A matrícula e demais identificadores do vínculo podem mudar. A associação entre a pessoa e a identidade corporativa deve permanecer.

## Chave primária de correlação

A chave preferencial deve ser um **identificador interno, único, estável e imutável da Pessoa na Senior** (`Person ID` ou equivalente).

Antes de produção, o Discovery deve comprovar que esse identificador:

- identifica a pessoa, e não apenas um vínculo;
- permanece estável quando a matrícula muda;
- permanece estável em mudança de tipo de vínculo;
- permanece estável em transferência de empresa/filial, quando aplicável;
- permite reconhecer a mesma pessoa em uma recontratação, quando a Senior preservar o cadastro de Pessoa.

Até essa validação, o nome exato do campo permanece **a confirmar**.

## Uso do CPF

O CPF pode ser utilizado como **atributo auxiliar de matching** para reconhecer uma pessoa quando a chave interna ainda não estiver correlacionada.

O CPF **não deve ser a chave técnica principal** se houver um identificador interno estável da Pessoa na Senior.

### Motivos

- CPF é dado pessoal e deve seguir minimização de dados.
- Não há necessidade de propagá-lo para AD DS, Entra, logs, métricas ou URLs do portal para realizar o lifecycle normal.
- A utilização de um identificador técnico interno reduz exposição desnecessária de dados pessoais.

### Restrições

O CPF:

- não deve ser persistido no AD DS pelo conector;
- não deve ser enviado ao Entra Provisioning se não for necessário ao provisioning;
- não deve aparecer em logs, alertas, correlation IDs ou URLs;
- não deve ser exibido por padrão no portal de auditoria;
- deve ser consultado/processado somente quando necessário para matching ou validação de identidade;
- deve possuir acesso restrito e auditável.

Se futuramente houver necessidade de persistir CPF no banco do IAM, isso exige decisão específica de segurança/privacidade e definição de proteção, retenção e finalidade.

## Estratégia de matching

A ordem conceitual de correlação será:

```text
Novo vínculo recebido
        |
        v
Existe sourcePersonId já correlacionado?
        |
        +-- SIM --> reutilizar identidade AD existente
        |
        +-- NÃO --> procurar possível pessoa existente por matching auxiliar
                        |
                        +-- correspondência inequívoca --> correlacionar
                        |
                        +-- nenhuma correspondência --> nova identidade elegível
                        |
                        +-- correspondência ambígua --> bloquear automação e gerar alerta
```

Nome, e-mail, UPN, `sAMAccountName` e matrícula **não podem ser usados isoladamente** como chave primária de pessoa.

## Regra — Estagiário para CLT

Quando uma pessoa encerrar um vínculo de estágio e iniciar um vínculo CLT, o sistema deve reaproveitar a identidade corporativa existente quando a correlação de pessoa for inequívoca.

Exemplo:

```text
Pessoa: João Silva
Person ID: 84572

Vínculo 1
Tipo: Estagiário
Matrícula: 10234
        |
        | encerramento
        v
Conta AD: joao.silva
        |
        | novo vínculo
        v
Vínculo 2
Tipo: CLT
Matrícula: 19873

Conta AD permanece: joao.silva
```

Devem ser preservados, salvo regra explícita em contrário:

- objeto da conta no AD DS;
- `objectGUID`;
- `sAMAccountName`;
- UPN, conforme política de naming;
- histórico da identidade;
- correlação IAM ↔ AD.

Os atributos dependentes do novo vínculo devem ser atualizados conforme o contrato IAM, por exemplo:

- matrícula;
- tipo de vínculo;
- cargo;
- departamento;
- gestor;
- empresa/filial;
- centro de custo;
- situação do vínculo.

## Intervalo entre vínculos

Reutilizar a identidade não significa necessariamente manter o acesso ativo durante o intervalo entre vínculos.

Exemplo:

```text
31/01  fim do estágio
        -> conta pode ser desabilitada

15/03  início do vínculo CLT
        -> a mesma conta pode ser reativada
```

A decisão de manter, desabilitar ou reativar acesso durante a transição depende das regras de lifecycle e das datas efetivas dos vínculos.

## Duplicidade e ambiguidade

Quando houver conflito de identidade, o sistema deve falhar de forma segura.

Exemplos:

- dois registros de Pessoa com mesmo CPF;
- uma chave de Pessoa associada a mais de uma conta AD;
- CPF encontrado em identidade existente, mas `sourcePersonId` incompatível;
- mais de uma conta candidata no AD.

Nesses casos:

1. não criar nova conta automaticamente;
2. não alterar contas candidatas automaticamente;
3. registrar a condição;
4. gerar alerta operacional;
5. exigir investigação antes de continuar o fluxo.

## Auditoria

Toda correlação ou mudança relevante deve permitir rastrear:

- `sourcePersonId`;
- identificador do vínculo/matrícula relevante;
- AD `objectGUID`;
- tipo de operação;
- motivo do matching/reutilização;
- timestamp;
- correlation ID;
- resultado.

O CPF, quando utilizado como matching auxiliar, não deve ser incluído na evidência operacional em texto aberto.

## Critérios de aceite da decisão

Antes de marcar esta decisão como `Accepted`:

- [ ] confirmar o identificador interno da Pessoa na API Senior;
- [ ] comprovar estabilidade em mudança Estagiário → CLT;
- [ ] comprovar comportamento em recontratação;
- [ ] identificar se matrícula pertence ao vínculo e pode mudar;
- [ ] validar se o CPF está disponível à conta técnica IAM sem liberar outros dados indevidos;
- [ ] definir o atributo utilizado pelo Entra para matching;
- [ ] definir como `sourcePersonId` será persistido no IAM e/ou AD;
- [ ] testar duplicidade e matching ambíguo.
