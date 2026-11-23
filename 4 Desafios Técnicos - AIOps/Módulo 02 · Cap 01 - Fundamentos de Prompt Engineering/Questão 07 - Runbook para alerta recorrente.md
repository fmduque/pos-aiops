**Tarefa.** Aplicando o framework **R-I-S-E**, escrever o prompt de IA que produza esse runbook procedural completo.
**Entregue.** Prompt, modelo, output e justificativa mostrando como Role, Input, Steps e Expectation aparecem no prompt.


## 1. Prompt

```

# Role
Você é um SRE sênior com experiência em Kubernetes (EKS), Argo CD, observabilidade com Prometheus e Grafana, e criação de runbooks operacionais que qualquer plantonista consiga seguir sem depender de quem conhece o sistema.

# Input
- Serviço: Chronos API
- Ambiente: cluster EKS, namespace `production`, 6 réplicas com HPA (min 4, max 12, target de CPU 70%)
- Deploy: Argo CD a partir do repositório `hvt/chronos-api`
- Dependências diretas: Ledger (PostgreSQL) e Reactor (filas SQS)
- Observabilidade: métricas expostas em `/metrics`, logs centralizados no Beacon, dashboards no Grafana
- Ferramentas disponíveis para o plantonista: apenas kubectl, aws cli e argocd cli.
- Alerta recorrente: `[CRITICAL] High memory usage on Chronos API pods (>85% for 10min)`, disparado em média 4 vezes por semana no canal `#oncall-chronos`
- Situação atual: não existe procedimento documentado; cada incidente leva de 30 a 40 minutos e o tempo varia muito
- Escalação: `@chronos-core` (SLA de resposta de 15 minutos em horário comercial e 30 minutos fora dele)
- Restrição: ajustes de réplicas ou limites não devem ser feitos com `kubectl edit` ou `kubectl scale`, porque o Argo CD reverte mudanças fora do Git

# Steps
1. Confirmar que o alerta é real e persistente utilizando as ferramentas disponiveis para os plantonistas.
2. Verificar o estado do HPA e das réplicas, identificando se o número já está no máximo (12).
3. Identificar a distribuição do problema entre os pods: se é um ou dois pods (possível vazamento ou requisição pesada) ou todos (possível carga ou mudança de comportamento).
4. Verificar reinícios e OOMKill nos pods, lendo `Last State` e `Restart Count`.
5. Verificar mudanças recentes: último sync e histórico no Argo CD e commits recentes em `hvt/chronos-api`.
6. Comparar a taxa de requisições com a linha de base para separar consumo legítimo de vazamento.
7. Analisar os logs do pod e no Beacon, filtrando por erros, exceções, `OutOfMemory` e retentativas.
8. Verificar a saúde das dependências: pool de conexões e consultas lentas no Ledger, e profundidade da fila no Reactor (`ApproximateNumberOfMessagesVisible`).
9. Aplicar a mitigação correspondente à causa identificada: reinício de pod isolado, rollback via `argocd app rollback` quando a causa for um deploy, ou ajuste de réplicas ou limites pelo repositório.
10. Verificar se a memória voltou abaixo de 85% e se não há novos OOMKill após a mitigação.
11. Escalar para `@chronos-core` quando ocorrer qualquer um destes casos: memória acima de 85% após a mitigação, OOMKill recorrente, dependência indisponível ou degradada, ou necessidade de rollback que o runbook não cubra.
12. Encerrar o incidente quando a memória estiver abaixo de 85% por pelo menos 10 minutos, sem novos OOMKill, com a causa registrada.

# Expectation
Espero um runbook operacional completo e pronto para uso por qualquer plantonista, com:
- Para cada etapa: objetivo, comando exato a executar (`kubectl`, `argocd`, `aws cli` ou PromQL), a verificação esperada no resultado e o tempo máximo da etapa
- Tempo máximo total de 40 minutos
- Uma árvore de decisão que separe vazamento, carga legítima, problema de deploy e problema de dependência, com a ação correspondente a cada caminho
- Critérios objetivos e numéricos para escalar para `@chronos-core`
- Critério claro de rollback, com o comando `argocd app rollback` e a condição que o justifica
- Critério de encerramento do incidente
- Linguagem direta, sem ambiguidade, em português, com placeholders explícitos (ex.: `<pod>`) onde o plantonista precisar substituir valores



```


## 2. Modelo

```
Haiku 5.5
```


## 3. Output
---
### Runbook: Alerta de memória alta no Chronos API

**Alerta:** `[CRITICAL] High memory usage on Chronos API pods (>85% for 10min)`  
**Serviço:** Chronos API | **Namespace:** `production` | **Cluster:** EKS  
**Escalação:** `@chronos-core` (SLA: 15 min em horário comercial, 30 min fora dele)  
**Tempo máximo total:** 40 minutos

> **Regras deste runbook**
> 
> - Não use `kubectl edit` nem `kubectl scale` para réplicas ou limites. O Argo CD reverte essas mudanças. Ajustes de réplicas, HPA ou limites são feitos por commit no `hvt/chronos-api` e sincronizados pelo Argo CD.
> - O HPA escala por **CPU (70%)**, não por memória. Adicionar réplicas não resolve vazamento de memória.
> - Substitua os valores entre `< >` antes de executar. Os nomes `chronos-api`, `app=chronos-api` e as métricas PromQL devem ser confirmados no primeiro uso (ver nota na etapa 1).

---

#### Pré-requisitos (antes de começar, 1 min)

bash

```bash
kubectl config current-context        # confirme que está no cluster EKS de produção
argocd account get-user-info          # confirme que está logado no Argo CD
aws sts get-caller-identity           # confirme a conta AWS
```

**Esperado:** contexto do cluster de produção, usuário logado e conta AWS correta. Se algo estiver errado, pare e faça login antes de continuar.

---

#### Etapa 1 — Confirmar que o alerta é real e persistente

**Objetivo:** descartar falso positivo e confirmar que a memória está acima de 85% agora.  
**Tempo máximo:** 2 min

bash

```bash
kubectl -n production top pods -l app=chronos-api --containers
```

Para obter o limite de memória de cada pod:

bash

```bash
kubectl -n production get pods -l app=chronos-api \
  -o custom-columns=NAME:.metadata.name,MEM_LIMIT:.spec.containers[0].resources.limits.memory
```

Calcule o percentual de cada pod: `uso / limite × 100`.

**Verificação esperada:** pelo menos um pod acima de 85% do limite. Se nenhum pod estiver acima de 85%, aguarde 2 minutos e repita. Se continuar abaixo, o alerta pode ter sido transitório. Registre no canal e encerre.

> **Nota:** `kubectl top` depende do metrics-server. Se não retornar dados, use o dashboard do Chronos API no Grafana, painel de memória por pod, com a query:
> 
> promql
> 
> ```promql
> max by (pod) (container_memory_working_set_bytes{namespace="production", container="chronos-api"})
>   / max by (pod) (kube_pod_container_resource_limits{namespace="production", container="chronos-api", resource="memory"}) * 100
> ```

---

#### Etapa 2 — Verificar o estado do HPA e das réplicas

**Objetivo:** saber se o HPA já está no teto (12 réplicas).  
**Tempo máximo:** 3 min

bash

```bash
kubectl -n production get hpa chronos-api
kubectl -n production describe hpa chronos-api
kubectl -n production get deploy chronos-api
```

**Verificação esperada:**

- `MINPODS` = 4 e `MAXPODS` = 12
- Campo `REPLICAS` atual

**Decisão:**

- Se as réplicas atuais **= 12** → o HPA está no teto. Vá para o ramo **Carga legítima** da árvore de decisão.
- Se estiver abaixo de 12 e a memória alta persiste → o HPA não escala por memória. Continue o diagnóstico normalmente.
- Em `describe hpa`, se aparecer `ScalingLimited: True` com `TooManyReplicas`, confirme que o teto foi atingido.

---

#### Etapa 3 — Identificar a distribuição do problema entre os pods

**Objetivo:** separar um problema localizado de um problema geral.  
**Tempo máximo:** 2 min

Use o resultado da etapa 1 e classifique:

|Padrão|Interpretação|
|---|---|
|1 ou 2 pods acima de 85%, os demais normais|Vazamento ou requisição pesada específica|
|Todos os pods acima de 75%|Carga geral ou mudança de comportamento|

**Verificação esperada:** padrão identificado e registrado.

---

#### Etapa 4 — Verificar reinícios e OOMKill

**Objetivo:** confirmar se os pods estão sendo finalizados pelo kernel por falta de memória.  
**Tempo máximo:** 2 min

bash

```bash
kubectl -n production get pods -l app=chronos-api \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[0].restartCount}{"\t"}{.status.containerStatuses[0].lastState.terminated.reason}{"\t"}{.status.containerStatuses[0].lastState.terminated.finishedAt}{"\n"}{end}'
```

Para o pod específico:

bash

```bash
kubectl -n production describe pod <pod> | grep -A 6 "Last State"
```

**Verificação esperada:**

- `Restart Count` e `Last State`
- Se `Reason: OOMKilled` aparecer, o pod foi morto por limite de memória.

**Decisão:**

- `OOMKilled` com `Restart Count` crescendo → registre; isso é gatilho de escalação se recorrente (ver etapa 11).
- Sem reinícios, mas memória alta → o pod está próximo do limite sem ser morto. Continue.

---

#### Etapa 5 — Verificar mudanças recentes (deploy)

**Objetivo:** verificar se o problema começou logo após um sync ou commit.  
**Tempo máximo:** 3 min

bash

```bash
argocd app get chronos-api
argocd app history chronos-api
```

**Verificação esperada:**

- `Sync Status` = `Synced` e `Health Status` = `Healthy`
- Data do último sync em `argocd app history`
- Revisão atual (SHA do commit)

Compare a data do último sync com o início do alerta. Veja também os commits recentes no GitHub: `hvt/chronos-api`, branch de produção, últimos 24h.

**Decisão:**

- Sync feito **até 30 min antes** do início da memória alta → suspeita de **problema de deploy**. Vá para o ramo **Problema de deploy**.
- Sem sync recente → siga com o diagnóstico.

---

#### Etapa 6 — Comparar a taxa de requisições com a linha de base

**Objetivo:** separar consumo legítimo de vazamento. Carga maior com memória proporcional sugere carga legítima; memória subindo com carga estável sugere vazamento.  
**Tempo máximo:** 3 min

No Grafana, dashboard do Chronos API, execute no Explore:

promql

```promql
sum(rate(http_requests_total{namespace="production", service="chronos-api"}[5m]))
```

Compare com a mesma métrica 7 dias atrás (mesmo dia da semana e horário):

promql

```promql
sum(rate(http_requests_total{namespace="production", service="chronos-api"}[5m] offset 7d))
```

> Se `http_requests_total` não existir com esse nome, use o nome exposto no `/metrics` do serviço. Confirme no dashboard existente.

**Verificação esperada:** razão atual / linha de base.

**Decisão:**

- Razão **≥ 1,5** e memória proporcional à carga → **carga legítima**.
- Razão **< 1,3** e memória subindo continuamente → **vazamento**.

---

#### Etapa 7 — Analisar logs do pod e do Beacon

**Objetivo:** encontrar exceções, `OutOfMemory` e retentativas.  
**Tempo máximo:** 3 min

Logs do pod mais afetado nos últimos 15 minutos:

bash

```bash
kubectl -n production logs <pod> --since=15m | grep -Ei "OutOfMemory|exception|error|retry|retrying"
```

Se o pod reiniciou, veja também os logs da instância anterior:

bash

```bash
kubectl -n production logs <pod> --previous --tail=200
```

No Beacon, filtre por `service=chronos-api`, `namespace=production` e a janela de tempo do alerta. Busque por `OutOfMemory`, `exception` e `retry`.

**Verificação esperada:**

- Contagem de erros por minuto
- Padrão de exceções repetidas
- Mensagens de retentativa relacionadas a Ledger ou Reactor

**Decisão:**

- `OutOfMemory` ou exceções em cache/buffer que crescem → aponta para **vazamento**.
- Erros de conexão ou timeout com Ledger ou Reactor → vá para a etapa 8 e o ramo **Problema de dependência**.

---

#### Etapa 8 — Verificar a saúde das dependências

**Objetivo:** identificar se Ledger ou Reactor estão causando acúmulo de memória no Chronos (conexões presas, respostas acumuladas, mensagens retidas).  
**Tempo máximo:** 2 min

**Reactor (SQS):**

bash

```bash
aws sqs get-queue-attributes \
  --queue-url <url-da-fila-do-chronos> \
  --attribute-names ApproximateNumberOfMessagesVisible ApproximateNumberOfMessagesNotVisible
```

**Verificação esperada:** profundidade da fila estável ou caindo. Crescimento contínuo por mais de 10 minutos indica problema.

**Ledger (PostgreSQL):**

bash

```bash
aws rds describe-db-instances \
  --db-instance-identifier <id-do-ledger> \
  --query 'DBInstances[0].[DBInstanceStatus,Endpoint.Address]'
```

**Verificação esperada:** status `available`.

Para pool de conexões e consultas lentas, use o dashboard do Ledger no Grafana. Verifique:

- Conexões ativas próximas do máximo do pool
- Consultas com duração acima de 1s

**Decisão:**

- Ledger indisponível ou status diferente de `available` → **escalar imediatamente** (critério 11.3).
- Fila crescendo ou pool esgotado → **problema de dependência** (ver árvore de decisão).

---

#### Árvore de decisão

Use os resultados das etapas 2 a 8 para escolher **um** ramo.

```
Memória > 85% confirmada (etapa 1)
│
├─ Etapa 5: sync ou commit até 30 min antes do alerta?
│   └─ SIM → PROBLEMA DE DEPLOY
│       Ação: rollback (ver seção "Critério de rollback")
│
├─ Etapa 8: Ledger indisponível ou fila do Reactor crescendo > 10 min?
│   └─ SIM → PROBLEMA DE DEPENDÊNCIA
│       Ação: não reinicie pods em loop. Escalar para @chronos-core (critério 11.3).
│
├─ Etapa 2: réplicas = 12 e razão de carga (etapa 6) ≥ 1,5?
│   └─ SIM → CARGA LEGÍTIMA
│       Ação: ajuste de capacidade pelo repositório (ver "Ajuste pelo Git").
│       Se a memória continuar alta após o ajuste → escalar.
│
├─ Etapa 3: 1 ou 2 pods concentram o problema, carga estável (etapa 6 < 1,3),
│   memória subindo continuamente?
│   └─ SIM → VAZAMENTO
│       Ação: reinício isolado de pod (ver "Mitigação: reinício de pod").
│       Se voltar a subir em menos de 60 min → escalar (critério 11.2).
│
└─ Nenhum dos casos acima, ou padrão ambíguo
    └─ Escalar para @chronos-core com o registro coletado.
```

---

### Mitigações

#### Mitigação A — Reinício de pod isolado (vazamento)

**Quando usar:** ramo **Vazamento**.  
**Tempo máximo:** 3 min

Reinicie **um pod por vez**, sem apagar vários de uma vez:

bash

```bash
kubectl -n production delete pod <pod>
kubectl -n production get pods -l app=chronos-api -w
```

O Deployment recria o pod automaticamente. Aguarde o novo pod estar `Running` e `Ready` antes de reiniciar outro.

**Verificação esperada:** pod novo em `Running` e `1/1 Ready` em até 2 minutos.

#### Mitigação B — Rollback via Argo CD (problema de deploy)

**Quando usar:** ramo **Problema de deploy** ou critério de rollback atendido.  
**Tempo máximo:** 5 min

Liste o histórico e identifique o ID da revisão anterior estável:

bash

```bash
argocd app history chronos-api
```

Execute o rollback para a revisão anterior ao deploy suspeito:

bash

```bash
argocd app rollback chronos-api <id-da-revisao-anterior>
argocd app wait chronos-api --health --timeout 300
```

**Verificação esperada:**

- `Health Status` = `Healthy`
- `Sync Status` = `Synced`
- Nova versão em execução

Após o rollback, o Argo CD mantém a revisão anterior. Abra um incidente para o time de desenvolvimento corrigir o commit problemático no Git.

#### Ajuste pelo Git (carga legítima ou ajuste de limites)

**Quando usar:** ramo **Carga legítima**. Não execute `kubectl scale` nem `kubectl edit`.

1. Peça ao `@chronos-core` ou abra um PR no `hvt/chronos-api` alterando, conforme o caso:
    - `maxReplicas` do HPA (hoje 12) ou CPU target, se a carga for estruturalmente maior
    - `resources.limits.memory` do container, se o consumo normal legítimo estiver próximo do limite atual
2. Após o merge, sincronize:

bash

```bash
argocd app sync chronos-api
argocd app wait chronos-api --health --timeout 300
```

> Mudanças de limite de memória causam rollout dos pods. Faça isso em janela controlada, com o `@chronos-core` ciente, e nunca em dois pods ao mesmo tempo.

---

#### Etapa 9 — Aplicar a mitigação

**Objetivo:** executar a ação do ramo escolhido.  
**Tempo máximo:** 5 min

- Vazamento → Mitigação A
- Problema de deploy → Mitigação B
- Carga legítima → Ajuste pelo Git (e escalar se não resolver)
- Problema de dependência → não reinicie. Escale.

**Verificação esperada:** ação executada e registrada no canal `#oncall-chronos` com horário e comando.

---

#### Etapa 10 — Verificar a memória após a mitigação

**Objetivo:** confirmar que a memória voltou abaixo de 85% e se manteve.  
**Tempo máximo:** 10 min (janela de observação)

Repita a cada 2 minutos:

bash

```bash
kubectl -n production top pods -l app=chronos-api --containers
kubectl -n production get pods -l app=chronos-api
```

Verifique também novos OOMKill:

bash

```bash
kubectl -n production get events -n production \
  --field-selector reason=OOMKilling --sort-by=.lastTimestamp | tail -20
```

**Verificação esperada:**

- Todos os pods abaixo de 85% do limite
- Nenhum novo evento `OOMKilling` após a mitigação

Se a memória não voltar abaixo de 85% em 5 minutos, vá para a etapa 11.

---

#### Etapa 11 — Critérios de escalação para @chronos-core

Escale **imediatamente**, com o registro das etapas executadas, se ocorrer **qualquer um** destes casos:

|#|Critério (objetivo e numérico)|
|---|---|
|11.1|Memória **> 85%** em qualquer pod **após** a mitigação, por **5 minutos** consecutivos|
|11.2|**2 ou mais** eventos `OOMKilling` em **30 minutos**|
|11.3|Ledger com status diferente de `available`, ou fila do Reactor crescendo por **10 minutos** seguidos|
|11.4|Rollback necessário que este runbook não cobre (ex.: migração de banco, mudança de schema, múltiplas revisões)|
|11.5|Incidente sem mitigação em **25 minutos** desde o início, independentemente do diagnóstico|

**Mensagem de escalação (modelo):**

```
@chronos-core — alerta High memory usage Chronos API (production)
Início: <hora>
Pods afetados: <lista>
Memória atual: <valores>
Réplicas HPA: <atual>/12
Restarts/OOMKill: <contagem>
Última sync Argo CD: <revisão e hora>
Dependências: Ledger <status>, Reactor <profundidade da fila>
Ações executadas: <lista com horários>
Critério de escalação atendido: <número>
```

**SLA:** 15 min em horário comercial, 30 min fora dele. Continue a investigação enquanto aguarda.

---

#### Etapa 12 — Encerramento do incidente

O incidente só pode ser encerrado quando **todas** as condições forem verdadeiras:

- [ ] Memória **< 85%** em todos os pods por **pelo menos 10 minutos** consecutivos
- [ ] **Nenhum** novo `OOMKilling` após a mitigação
- [ ] Causa registrada (vazamento, carga legítima, deploy, dependência)
- [ ] Ação de correção definitiva registrada: PR, rollback ou issue para o time de desenvolvimento
- [ ] Mensagem de encerramento no `#oncall-chronos`

**Modelo de encerramento:**

```
RESOLVIDO — High memory usage Chronos API
Início: <hora> | Fim: <hora> | Duração: <min>
Causa: <vazamento | carga legítima | deploy <revisão> | dependência <nome>>
Mitigação: <ação executada>
Correção definitiva: <PR/issue/rollback>
```

---

### Critério de rollback (resumo)

Execute `argocd app rollback chronos-api <revisão-anterior>` **somente** quando **todas** as condições forem verdadeiras:

1. Houve sync **até 30 min** antes do início do alerta
2. A memória subiu **após** a troca de revisão, com carga estável (razão < 1,3)
3. Não há problema de dependência (etapa 8 normal)

Se a condição 3 não for atendida, o rollback não é a ação correta. Escale.

---

### Resumo rápido para o plantonista

1. Confirme memória > 85% (`kubectl top`)
2. HPA no teto? → carga legítima
3. Sync recente? → rollback
4. Dependência fora? → escale
5. Pod isolado com vazamento? → reinicie um por vez
6. Memória < 85% por 10 min, sem OOMKill → encerre
7. Qualquer critério da etapa 11 → escale


## 4. Justificativa

Alguns ajustes foram necessários no prompt. Incialmente pedi para avaliar a saúde dos pods e ele colocou no runbook verificar no grafana, mas o grafana não está disponível para os plantonistas. Após um ajuste ele indicou os comandos corretos com o kubectl, argocd e aws-cli. Este framework e este exemplo vão ser muito uteis no meu dia a dia.