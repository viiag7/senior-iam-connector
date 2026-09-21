# Identity Correlation — Pessoa, Vínculo e Conta AD

- **Status:** Accepted for MVP
- **Objetivo:** definir como uma pessoa da Senior é correlacionada com uma identidade corporativa no AD DS ao longo do ciclo de vida e de vínculos sucessivos.

## Princípio central

A **conta corporativa representa a pessoa, e não o vínculo trabalhista**.

Uma mesma pessoa pode possuir diferentes vínculos ao longo do tempo, por exemplo:

- Estagiário → CLT;
- Temporário → CLT;
- Terceiro → CLT;
- transferência entre empresas/filiais;
- desligamento → recontratação.

Essas mudanças não devem, por si só, resultar na criação de uma nova conta no AD DS.

## Premissa de vínculo único

Gente & Gestão confirmou que não existem múltiplos vínculos simultâneos para a mesma pessoa no escopo desta integração. A validação funcional foi fornecida por **Juliana Croda — Gente & Gestão**.

A integração pode, portanto, assumir **no máximo um vínculo ativo relevante por CPF por vez**. Vínculos históricos e Rehire continuam existindo ao longo do tempo e devem reutilizar a mesma identidade por CPF.

Caso a organização passe a permitir múltiplos vínculos simultâneos no futuro, essa premissa deixa de ser válida e a arquitetura de lifecycle/mapping deve ser revisada antes da mudança entrar em produção.

## Chave de correlação da pessoa

O **CPF é a única chave autorizada para determinar que dois registros ou vínculos da Senior pertencem à mesma pessoa**.

Nenhum outro atributo pode ser utilizado como fallback ou como evidência suficiente para vincular automaticamente um novo vínculo a uma identidade AD existente.

Em especial, não podem ser utilizados para correlação de pessoa:

- `Person ID` da Senior;
- matrícula;
- identificador do vínculo/contrato;
- nome;
- e-mail;
- UPN;
- `sAMAccountName`;
- cargo, departamento ou qualquer atributo organizacional.

Esses campos podem ser armazenados para rastreabilidade, provisioning e contexto operacional, mas não determinam a identidade da pessoa.

## Modelo conceitual

```text
Pessoa
└── CPF: chave de correlação da identidade
    |
    ├── Vínculo histórico A (encerrado)
    │   └── matrícula anterior
    |
    └── Vínculo atual B (único vínculo ativo no escopo)
        └── matrícula atual

                 |
                 v

Identity IAM
├── CPF correlation key
├── AD objectGUID
├── sAMAccountName
└── UPN
```

A matrícula, o identificador de vínculo e outros identificadores da Senior podem mudar. O CPF é o elemento utilizado para reconhecer que se trata da mesma pessoa.

## Regra de matching

A regra é determinística:

```text
Novo vínculo recebido
        |
        v
CPF válido está presente?
        |
        +-- NÃO --> bloquear automação e gerar alerta
        |
        +-- SIM --> existe identidade IAM vinculada a esse CPF?
                        |
                        +-- SIM --> reutilizar a identidade AD existente
                        |
                        +-- NÃO --> criar nova identidade, se elegível
                        |
                        +-- MAIS DE UMA --> bloquear automação e gerar alerta crítico
```

Não existe matching secundário por nome, matrícula, Person ID, e-mail ou qualquer outro atributo.

## Normalização e proteção do CPF

Embora o CPF seja a chave funcional de correlação, ele é um dado pessoal e deve ser tratado com minimização e proteção.

Antes do matching, o valor deve ser normalizado para um formato canônico, removendo pontuação e aplicando validação apropriada.

A implementação deve evitar propagar o CPF para sistemas que não precisam conhecê-lo. Em particular, por padrão o CPF:

- não deve ser armazenado como atributo visível no AD DS;
- não deve ser enviado ao Entra Provisioning quando não for necessário para executar o provisioning;
- não deve aparecer em logs, alertas, correlation IDs ou URLs;
- não deve ser exibido no portal de auditoria por padrão;
- não deve ser usado como identificador apresentado ao operador.

Na primeira fase, o banco do IAM poderá persistir o CPF **em texto claro** para realizar a correlação determinística. Essa decisão foi aceita para simplificar o MVP e deverá ser tratada como risco conhecido.

Mesmo nessa fase, o CPF não deve ser replicado para o AD DS, enviado ao Entra sem necessidade, registrado em logs/alertas ou exposto no portal operacional. Uma fase posterior poderá adotar HMAC, tokenização ou criptografia sem alterar a regra funcional de correlação.

## Person ID da Senior

O `Person ID` ou identificador equivalente da Senior pode ser coletado e armazenado como **referência técnica e de auditoria**, quando útil.

Ele não participa da decisão de matching de pessoa.

Exemplo:

```text
CPF 123...89
   |
   +-- vínculo estágio: Person ID / matrícula A
   |
   +-- vínculo CLT:     Person ID / matrícula B
   |
   +--> mesma Identity IAM
        mesma conta AD
```

Mesmo que os identificadores internos ou matrículas sejam diferentes, o mesmo CPF determina o reaproveitamento da identidade.

## Regra — Estagiário para CLT

Quando um estagiário encerrar o vínculo e iniciar um vínculo CLT com o **mesmo CPF**, o sistema deve reaproveitar a identidade corporativa existente.

Exemplo:

```text
CPF: mesmo valor

Vínculo 1
Tipo: Estagiário
Matrícula: 10234
        |
        | encerramento
        v
Conta AD: joao.silva
        |
        | novo vínculo com mesmo CPF
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

15/03  início do vínculo CLT com mesmo CPF
        -> a mesma conta deve ser reativada conforme regra de lifecycle
```

A decisão de manter, desabilitar ou reativar acesso durante a transição depende das regras de lifecycle e das datas efetivas dos vínculos.

## CPF ausente, inválido ou duplicado

Como não existe fallback de matching, situações envolvendo CPF devem falhar de forma segura.

| Situação | Comportamento |
|---|---|
| CPF ausente | Não provisionar/correlacionar; registrar falha e alertar |
| CPF inválido | Não provisionar/correlacionar; registrar falha e alertar |
| Mesmo CPF associado a uma identidade IAM | Reutilizar a identidade existente |
| Mesmo CPF associado a mais de uma identidade IAM | Bloquear automação e gerar alerta crítico |
| CPF diferente de uma identidade existente aparentemente semelhante | Não correlacionar automaticamente |

Nome, matrícula, e-mail ou similaridade de dados nunca devem substituir a ausência ou divergência de CPF.

## Matching técnico com o AD DS

O CPF define **quem é a pessoa** no domínio IAM. Após a primeira correlação, o sistema deve manter a associação entre essa identidade e o `objectGUID` da conta AD.

Assim, operações posteriores não precisam procurar contas no AD por CPF; elas utilizam a identidade IAM já correlacionada:

```text
CPF
  -> Identity IAM
      -> AD objectGUID
```

A forma de materializar a chave técnica usada pelo Entra inbound provisioning será definida na arquitetura, sem alterar a regra funcional de correlação exclusiva por CPF.

## Auditoria

Toda correlação ou reutilização deve permitir rastrear:

- referência técnica da identidade IAM;
- Person ID da Senior, quando disponível;
- identificador do vínculo/matrícula relevante;
- AD `objectGUID`;
- tipo de operação;
- motivo da reutilização (`same CPF`);
- timestamp;
- correlation ID;
- resultado.

O valor do CPF não deve aparecer em texto aberto na evidência operacional. O sistema pode registrar que a correlação foi realizada por CPF sem registrar o próprio número.

## Critérios de aceite da decisão

Antes de marcar esta decisão como `Accepted`:

- [x] confirmar o campo exato de CPF na API Senior (`person.cpf`);
- [ ] confirmar que todos os vínculos elegíveis possuem CPF disponível para a integração;
- [ ] validar comportamento Estagiário → CLT com mudança de matrícula;
- [ ] validar comportamento técnico de recontratação ponta a ponta;
- [ ] definir normalização e validação do CPF;
- [x] definir persistência inicial do CPF no IAM (texto claro no MVP, com risco aceito);
- [ ] definir a chave técnica utilizada entre IAM, Entra e AD DS;
- [ ] testar CPF ausente, inválido e duplicado;
- [ ] testar que nenhum fallback por nome, matrícula, Person ID ou e-mail é realizado.
