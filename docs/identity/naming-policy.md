# Account Naming Policy — sAMAccountName e UPN

- **Status:** Accepted for MVP
- **Objetivo:** definir uma política determinística, única e estável para criação e eventual alteração de `sAMAccountName` e `userPrincipalName` no AD DS.

> Esta política foi adotada como regra de naming do MVP. Os pontos marcados como **A definir com o time** são parâmetros operacionais que ainda precisam ser fechados antes da produção (por exemplo sufixo UPN e limites), mas não alteram o algoritmo-base `primeiro_nome.ultimo_nome`, a verificação de unicidade nem a estabilidade do naming após criação.

## Princípios

1. O nome de conta deve ser previsível e derivado do nome autoritativo recebido da Senior.
2. O sistema deve verificar unicidade no AD DS antes de criar ou renomear uma conta.
3. O naming deve ser determinístico: para a mesma entrada e o mesmo estado do diretório, a regra deve produzir o mesmo resultado.
4. Após a criação da conta, `sAMAccountName` e UPN são considerados **estáveis** e não devem ser recalculados por alterações de cargo, departamento, gestor, matrícula, vínculo ou outras mudanças organizacionais.
5. Uma mudança do nome de usuário/nome autoritativo na Senior pode iniciar uma avaliação de rename conforme política aprovada.
6. Uma alteração de naming nunca deve criar uma nova identidade; a mesma conta AD e a mesma correlação IAM devem ser preservadas.
7. Antes de qualquer rename, o novo `sAMAccountName` e UPN devem passar novamente pela validação de unicidade.
8. Rehire ou mudança de vínculo da mesma pessoa não deve, por si só, recalcular ou alterar o nome de conta existente.
9. **Atualização direta no AD DS não faz parte do fluxo normal de naming.** Ela somente poderá ser utilizada como exceção quando o nome gerado automaticamente não atender às regras aprovadas e for necessário definir um nome diferente.
10. Toda exceção de naming aplicada diretamente no AD DS deve ser registrada pelo IAM para que reconciliações futuras preservem o valor aprovado e não o sobrescrevam automaticamente.

## Padrão inicial

O padrão preferencial para novas contas será:

```text
primeiro_nome.ultimo_nome
```

Exemplo:

```text
João Pedro de Almeida da Silva

Candidato inicial:
joao.silva
```

O UPN será formado a partir do mesmo identificador lógico:

```text
joao.silva@<upn-suffix>
```

O sufixo real de UPN será definido com o time responsável pelo AD/Entra.

## Normalização do nome

Antes de gerar candidatos, o sistema deverá normalizar o nome de forma determinística.

Baseline proposta:

- converter para minúsculas;
- remover acentos/diacríticos;
- remover caracteres não permitidos pela política de naming;
- tratar espaços e separadores;
- ignorar conectores nominais na formação principal, por exemplo: `da`, `de`, `do`, `das`, `dos` e `e`;
- utilizar o primeiro nome útil e o último sobrenome útil para o primeiro candidato.

Exemplo:

```text
João Pedro de Almeida da Silva
        |
        v
joao.silva
```

**A definir com o time:** regras para nomes estrangeiros, nomes com apóstrofo/hífen, nomes muito curtos, nomes sociais/preferidos e demais exceções linguísticas.

## Verificação de unicidade

Antes da criação, o sistema deve verificar se o candidato já está em uso no AD DS.

A verificação deve considerar, no mínimo:

- `sAMAccountName`;
- `userPrincipalName`.

Uma conta só pode ser criada quando os identificadores finais estiverem livres ou quando o sistema comprovar que pertencem à própria identidade em um cenário de rename/reprocessamento.

## Tratamento de colisão

Quando `primeiro.ultimo` já existir, o sistema deve tentar combinações adicionais utilizando nomes intermediários do colaborador.

Sequência conceitual proposta:

```text
Nome: João Pedro de Almeida da Silva

1. joao.silva
2. joao.pedro.silva
3. joao.pedro.almeida.silva
```

Os conectores nominais ignorados não participam da combinação.

Caso todas as combinações naturais definidas pela política estejam ocupadas, deverá existir um fallback determinístico.

Exemplo possível:

```text
joao.silva2
joao.silva3
```

**A definir com o time:**

- ordem exata das combinações intermediárias;
- limite máximo de tentativas;
- formato do fallback numérico ou alternativo;
- comportamento quando o limite técnico de tamanho for atingido;
- se a verificação de colisão deve considerar aliases/proxy addresses além de `sAMAccountName` e UPN.

## Estabilidade após criação

Depois que uma conta for criada, o sistema deve persistir e reutilizar os identificadores atribuídos.

Exemplo:

```text
Identity IAM
├── CPF / chave protegida de correlação
├── AD objectGUID
├── sAMAccountName = joao.pedro.silva
├── UPN = joao.pedro.silva@empresa.com
└── namingSourceName = "João Pedro da Silva"
```

Uma reconciliação normal não deve recalcular o username com base na disponibilidade atual do diretório.

Mudanças como estas não devem alterar o naming:

- cargo;
- departamento;
- gestor;
- centro de custo;
- empresa/filial;
- matrícula;
- mudança Estagiário → CLT;
- desligamento e posterior Rehire da mesma pessoa, salvo decisão explícita de naming.

## Nome que originou a conta

O IAM deve manter referência ao nome autoritativo utilizado para gerar o naming atual, por exemplo `namingSourceName`.

Isso permite diferenciar:

```text
Senior continua: João Pedro da Silva
→ nenhuma ação de rename
```

De:

```text
Senior mudou: João Pedro Oliveira
→ mudança real de nome detectada
→ avaliar política de rename
```

Esse valor é metadado de controle do IAM e não substitui o nome autoritativo da Senior.

## Alteração de nome na Senior

Uma mudança do nome autoritativo/nome de usuário na Senior é o único gatilho funcional atualmente previsto para avaliar alteração de `sAMAccountName` e UPN após a criação.

Fluxo conceitual:

```text
Nome autoritativo mudou na Senior
        |
        v
Política determina rename?
        |
        +-- NÃO --> manter naming atual
        |
        +-- SIM --> gerar novo candidato
                         |
                         v
                  verificar unicidade
                         |
                         v
                  atualizar a MESMA identidade
```

A alteração deve preservar:

- correlação pelo CPF;
- identidade IAM;
- AD `objectGUID`;
- histórico e audit trail;
- demais atributos não relacionados ao rename.

O rename nunca deve ser interpretado como criação de uma nova pessoa.

### Colisão durante rename

Se o novo candidato já existir, o mesmo algoritmo de colisão utilizado na criação deve ser aplicado.

## sAMAccountName e UPN

A proposta inicial é manter ambos alinhados no identificador lógico sempre que possível:

```text
sAMAccountName: joao.silva
UPN:            joao.silva@empresa.com
```

Contudo, são atributos distintos e suas restrições técnicas devem ser respeitadas independentemente.

**A definir com o time:**

- sufixo oficial de UPN;
- limite e regra final para `sAMAccountName`;
- necessidade de manter UPN alinhado ao e-mail principal;
- comportamento de aliases/e-mail quando ocorrer rename;
- necessidade de preservar aliases antigos após mudança de nome.

## Exceção — atualização direta no AD DS

A atualização direta de `sAMAccountName`/UPN no AD DS será tratada como **exceção**, e não como mecanismo normal do processo de provisionamento.

Ela poderá ser utilizada quando:

- o nome calculado automaticamente pelo algoritmo não atender às regras ou necessidades aprovadas;
- for necessário atribuir um nome diferente do resultado gerado automaticamente;
- a exceção estiver devidamente autorizada conforme processo a ser definido com o time.

Exemplo conceitual:

```text
Nome gerado automaticamente
        |
        v
nao atende a regra/caso excepcional
        |
        v
nome alternativo aprovado
        |
        v
atualização direta no AD DS
        |
        v
IAM registra o naming efetivo
```

Após essa alteração, o valor efetivamente aprovado passa a ser tratado como o naming atual da identidade e **não deve ser sobrescrito por reconciliação normal**.

A atualização direta no AD DS não deve:

- criar outro objeto AD;
- alterar a correlação por CPF;
- ser usada para mudanças rotineiras de cargo, departamento, vínculo ou matrícula;
- provocar recálculo automático posterior do username;
- ser utilizada como mecanismo de correção informal fora do processo auditável.

O audit trail deve registrar, no mínimo:

- identidade afetada;
- `objectGUID` ou chave técnica equivalente;
- valor calculado originalmente;
- valor efetivamente aplicado;
- responsável/autorização da exceção, quando disponível;
- justificativa;
- timestamp;
- correlation ID;
- resultado da operação.

## Falhas e segurança

Se o sistema não conseguir garantir unicidade ou realizar uma alteração com segurança:

1. não criar uma segunda conta;
2. não aplicar alteração parcial insegura;
3. manter a identidade existente em estado conhecido;
4. registrar a falha;
5. gerar alerta conforme política de provisioning;
6. permitir investigação e reconciliação posterior.

## Cenários mínimos de teste

Antes de aprovar a política, devem ser testados:

- criação sem colisão;
- criação com colisão em `primeiro.ultimo`;
- colisão em múltiplas combinações;
- acentos e caracteres especiais;
- nomes com conectores (`da`, `de`, `dos` etc.);
- limite de tamanho de `sAMAccountName`;
- Rehire mantendo naming existente;
- Estagiário → CLT mantendo naming existente;
- alteração de nome na Senior sem colisão;
- alteração de nome na Senior com colisão;
- exceção em que o nome gerado não atende à regra e é aplicado nome diferente diretamente no AD DS;
- reconciliação preservando o naming definido por exceção;
- retry de criação sem gerar outro username;
- retry de rename sem gerar resultados diferentes;
- falha entre atualização de `sAMAccountName` e UPN;
- reconciliação após rename.

## Decisões pendentes para aprovação do time

- [ ] Campo/nome autoritativo da Senior usado como fonte de naming.
- [ ] Uso ou não de nome social/preferido.
- [ ] Lista final de conectores ignorados.
- [ ] Ordem final das combinações em caso de colisão.
- [ ] Fallback final após esgotar combinações naturais.
- [ ] Limites de tamanho e truncamento.
- [ ] Sufixo oficial de UPN.
- [ ] Relação entre UPN e e-mail principal.
- [ ] Política de aliases após rename.
- [ ] Se toda mudança de nome autoritativo gera rename automático ou se haverá um indicador específico na Senior/política IAM.
- [ ] Comportamento em alterações apenas ortográficas/correções pequenas.
- [ ] Quem pode autorizar uma exceção de naming aplicada diretamente no AD DS.
- [ ] Como o IAM registrará e preservará um naming definido por exceção.

## Referências internas

- [`functional-requirements.md`](../requirements/functional-requirements.md)
- [`attribute-mapping.md`](./attribute-mapping.md)
- [`identity-correlation.md`](./identity-correlation.md)
