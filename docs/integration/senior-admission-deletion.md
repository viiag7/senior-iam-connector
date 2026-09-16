# Senior — Exclusão de admissão e cancelamento de pré-provisionamento

- **Status:** Proposed / Discovery required
- **Objetivo:** documentar como o IAM deve tratar a exclusão de uma admissão na Senior quando a identidade já tiver sido pré-provisionada.

## Contexto

No processo de admissão, uma identidade pode ser criada no AD DS antes da data efetiva de início para permitir preparação do onboarding.

Quando o RH cancela a admissão, o registro da pessoa/admissão pode ser excluído da Senior. Nesse cenário, o IAM precisa impedir que uma conta já pré-provisionada seja habilitada na data originalmente planejada.

## Evidência encontrada na documentação da Senior

A documentação pública da Senior mostra que existem mecanismos internos de integração capazes de representar exclusões explicitamente.

No Integrador SST, a exclusão de uma admissão no HCM XT gera uma pendência de integração com operação de **exclusão**. A própria documentação usa como exemplo uma pessoa admitida que não compareceu e desistiu da vaga.

Também existem fluxos da Senior em que exclusões geram pendências específicas de integração, como no processamento de exclusões relacionadas ao eSocial.

Além disso, existem APIs/estruturas da Senior que expõem indicador de registro excluído (`deleted`) em determinados contextos.

Essas evidências demonstram que a Senior possui conceitos técnicos para representar exclusões em integrações. Entretanto, **ainda não está comprovado que o Senior IAM Connector poderá consumir diretamente um evento/webhook genérico de exclusão de admissão**.

## Hipótese de integração preferencial

Durante o Discovery, deve ser priorizada a identificação de um mecanismo explícito para receber ou consultar a exclusão:

```text
Senior
   |
   | exclusão da admissão
   v
Evento / webhook / pendência / API de alterações
   |
   v
Senior IAM Connector
   |
   v
ADMISSION_CANCELLED
```

A ordem de preferência conceitual é:

1. evento/webhook oficial de exclusão, se disponível para o HCM utilizado;
2. pendência/fila de integração que informe operação de exclusão;
3. API incremental/delta que represente registros excluídos;
4. reconciliação periódica como mecanismo de segurança/fallback.

A implementação final dependerá das capacidades efetivamente disponíveis no ambiente Senior da organização.

## Regra funcional definida

Quando uma identidade estiver em `PRE_PROVISIONED` e for confirmada a exclusão/cancelamento da admissão na fonte autoritativa:

```text
PRE_PROVISIONED
      |
      | admissão excluída/cancelada na Senior
      v
ADMISSION_CANCELLED
```

O IAM deve:

- impedir a habilitação da conta na data de admissão anteriormente prevista;
- **garantir que a conta esteja desabilitada no AD DS**;
- registrar no AD DS os metadados de lifecycle/motivo/data definidos no `attribute-mapping.md`;
- não excluir automaticamente o objeto do AD DS no MVP;
- preservar CPF/correlação da pessoa e o vínculo com o `objectGUID` criado;
- cancelar qualquer ação futura agendada de ativação relacionada à admissão excluída;
- registrar o evento e a origem da confirmação de cancelamento;
- manter a identidade disponível para auditoria;
- permitir reutilização da mesma conta caso a pessoa volte a ser cadastrada posteriormente com o mesmo CPF, conforme política de lifecycle e correlação.

Metadados conceituais esperados no AD DS:

```text
iamLifecycleState = ADMISSION_CANCELLED
iamDisableReason  = ADMISSION_CANCELLED
iamDisabledAt     = <data/hora efetiva da desabilitação>
```

Os nomes físicos desses atributos ainda devem ser definidos no POC/mapping.

## Relação com política de retenção

Uma conta em `ADMISSION_CANCELLED` estará desabilitada e poderá futuramente participar de uma política de retenção/limpeza, mas a exclusão automática definitiva **não faz parte do MVP**.

Quando a política de retenção for aprovada, ela deverá considerar explicitamente `iamLifecycleState` e `iamDisableReason`, e não apenas o fato de a conta estar desabilitada há mais de 30 dias.

Isso evita que uma regra genérica de limpeza trate da mesma forma uma admissão cancelada e uma pessoa temporariamente suspensa por férias/afastamento.

## Ausência em consulta não é exclusão confirmada

O fato de um registro não aparecer em uma consulta isolada **não deve ser suficiente, por si só, para concluir que a admissão foi cancelada**.

Uma ausência pode ocorrer por:

- indisponibilidade da API;
- falha de autenticação/autorização;
- filtro incorreto;
- problema de paginação/delta;
- erro temporário da Senior;
- mudança de abrangência/permissão da conta técnica.

Portanto:

```text
Registro não encontrado
        |
        v
Consulta/reconciliação concluída com sucesso e com abrangência válida?
        |
        +-- NÃO --> não alterar lifecycle; registrar falha e retry
        |
        +-- SIM --> existe evidência confiável de exclusão/cancelamento?
                        |
                        +-- SIM --> ADMISSION_CANCELLED + conta desabilitada
                        |
                        +-- NÃO --> manter estado e investigar/reconciliar
```

A regra preferencial continua sendo consumir uma indicação explícita de exclusão quando a Senior disponibilizar esse mecanismo.

## Evento + reconciliação

Mesmo que seja encontrado um evento ou webhook de exclusão, o IAM deve manter reconciliação periódica.

```text
                 Senior
                 /    \
                /      \
       evento/exclusão  reconciliação
              |              |
              +------+-------+
                     |
                     v
              Senior IAM Connector
```

O evento oferece baixa latência. A reconciliação protege contra eventos perdidos, indisponibilidade temporária ou divergências entre fonte e target.

## Auditoria mínima

O cancelamento deve permitir rastrear:

- identidade IAM afetada;
- CPF por representação protegida/chave interna, sem exposição desnecessária;
- matrícula/vínculo pré-provisionado quando aplicável;
- estado anterior (`PRE_PROVISIONED`);
- estado resultante (`ADMISSION_CANCELLED`);
- origem da detecção (`event`, `webhook`, `integration_pending`, `delta`, `reconciliation` ou equivalente);
- data/hora da exclusão quando fornecida pela Senior;
- data/hora do processamento no IAM;
- ação de disable executada no AD DS;
- `iamDisabledAt` e `iamDisableReason` lógicos;
- correlation ID;
- resultado.

## Discovery obrigatório

Antes da implementação, validar no ambiente Senior:

- [ ] se a exclusão de admissão gera evento no Events Hub;
- [ ] se existe webhook consumível para esse evento;
- [ ] se existe endpoint/fila de pendências com operação de exclusão;
- [ ] se alguma API incremental/delta retorna registros excluídos;
- [ ] qual identificador/CPF está disponível no registro de exclusão;
- [ ] se o evento é emitido antes da remoção definitiva do registro consultável;
- [ ] comportamento quando uma admissão excluída é recriada;
- [ ] garantias de entrega, retries e retenção do mecanismo de evento/pendência;
- [ ] permissões mínimas necessárias para consumir a informação sem acesso a dados de RH fora do escopo IAM.

## Referências Senior para validação

- Integrador SST — conceitos da API e operações de exclusão: https://documentacao.senior.com.br/seniorxplatform/manual-do-usuario/hcm/integrador-sst/desenvolvedores/conceitos-da-api/
- HCM / eSocial — eventos de exclusão: https://documentacao.senior.com.br/seniorxplatform/manual-do-usuario/hcm/folha-x/esocial/processos/eventos-de-exclusao/
- HCM — Integrador X: https://documentacao.senior.com.br/seniorxplatform/manual-do-usuario/hcm/implantacao/integrador-x/

> As referências acima comprovam conceitos de exclusão em mecanismos de integração da Senior, mas não devem ser interpretadas como confirmação de um webhook genérico de exclusão de admissão disponível para este projeto. Essa capacidade precisa ser comprovada no Discovery.
