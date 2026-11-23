**Tarefa.** Escolher entre os cinco frameworks do capítulo (R-T-F, T-A-G, B-A-B, C-A-R-E ou R-I-S-E) aquele que se aplica melhor a esse cenário e escrever o prompt de IA que produza o postmortem técnico que o Doc pediu.

Nesta questão a **justificativa é o coração da entrega**. Além de explicar o framework escolhido e como seus componentes aparecem no prompt, comparar explicitamente com pelo menos **2 outros frameworks candidatos**, apontando o que se ganharia e o que se perderia em cada um.

**Entregue.** Prompt, modelo, output e justificativa estendida com comparação entre frameworks.

## 1. Prompt

```

# Role
Você é um SRE sênior com experiência em Kubernetes (EKS), Argo CD, observabilidade com Prometheus e Grafana, e precisa criar um postmortem técnico em 20 minutos para decidir entre **rollback do deploy v2.48.0** (que subiu ontem) e **scaling emergencial** (aumento de limits do RDS e do pool de conexões)

# Input
Os artefatos disponíveis para análise são os seguintes.

Evento do deploy anterior (ontem, 18:42 UTC):
Deploy chronos-api: v2.47.0 -> v2.48.0
Argo CD sync: 2026-04-23 18:42:11 UTC
Changelog:
- Adicionado endpoint POST /v2/transactions/batch
- Refatorado cliente do Ledger (pool de conexoes movido para nova biblioteca interna)
- Bump de psycopg 3.1.18 -> 3.2.0
- Reduzido timeout do Ledger de 5s para 2s

Métricas do Beacon nos últimos 30 minutos:
timestamp                p99_latency_ms   req_rate_s   err_rate_pct
2026-04-24 13:30 UTC     420              1200         0.2
2026-04-24 13:45 UTC     510              1450         0.3
2026-04-24 14:00 UTC     780              1780         0.8
2026-04-24 14:10 UTC     2400             2100         4.5
2026-04-24 14:15 UTC     5200             2400         8.2
2026-04-24 14:20 UTC     8100             2650         11.7

Trecho do log do pod `chronos-api-79c4d8b9-xk2jp`:
2026-04-24 14:19:48 [ERROR] [ledger-client] connection pool exhausted (max=20, active=20, waiting=147)
2026-04-24 14:19:49 [WARN]  [ledger-client] query timeout after 2000ms: SELECT ... FROM transactions WHERE ...
2026-04-24 14:19:49 [ERROR] [handler] POST /v2/transactions/batch failed: context deadline exceeded
2026-04-24 14:19:50 [ERROR] [ledger-client] connection reset by peer
2026-04-24 14:19:51 [WARN]  [circuit-breaker] ledger-client OPEN (threshold 50%, current 87%)
2026-04-24 14:19:52 [ERROR] [reactor] failed to publish message: chronos-api upstream error

Estado do Reactor (fila `chronos-transactions`):
- 50.127 mensagens acumuladas, crescendo a ~800/min.
- Consumer lag atual: 18 minutos e aumentando.

Estado do cluster:
- Chronos: 12/12 pods running (HPA no máximo).
- CPU médio dos pods: 62%.
- Memória média dos pods: 71%.
- Conexões ativas ao Ledger: 240/250 (limite do RDS).

# Steps
1. Monte a linha do tempo em UTC, ligando o deploy às métricas.
2. Compare a degradação com a variação do tráfego: a latência cresceu proporcionalmente ou mais rápido?
3. Calcule as conexões máximas do pool (pods × pool por pod) e compare com o limite do RDS.
4. Liste as hipóteses de causa. Para cada uma, cite a evidência do input e o que a confirmaria ou refutaria.
5. Avalie rollback da v2.48.0 e scaling emergencial quanto a: tempo para efeito, risco,reversibilidade e se atacam a hipótese mais forte.
6. Estime o backlog do Reactor, o tempo de drenagem e o risco de perda ou duplicidade de mensagens.
7. Liste o que os artefatos não mostram e que impede uma conclusão.


# Expectation
Formato de postmortem, nesta ordem: 
Resumo (até 3 linhas) · Impacto · Linha do tempo · Causa provável (com nível de confiança) · Fatores contribuintes · Opções para o CTO
(com recomendação e justificativa) · Lacunas de dados · Ações de follow-up.
Regras: use apenas números dos artefatos ou calculados a partir deles, mostrando a conta.
Não invente métricas, nomes ou horários. Marque como hipótese o que não estiver comprovado.
Máximo de uma página.




```


## 2. Modelo

```
Haiku 5.5
```


## 3. Output
---

### Postmortem: degradação do chronos-api após o deploy v2.48.0

**Resumo:** Após o deploy da v2.48.0, o pool de conexões ao Ledger saturou (240 de 240 em uso) e o p99 subiu de 420 ms para 8.100 ms, com 11,7% de erro. A saturação é um fato comprovado pelos logs e pelas contagens. O gatilho (regressão da v2.48 ou aumento de carga) não está comprovado. Recomendo rollback para v2.47.0 agora.

**Impacto**

- Erro: 0,2% → 11,7%. p99: 420 ms → 8.100 ms (19,3×).
- Circuit breaker do ledger-client aberto com 87% de erro (limiar: 50%).
- Reactor: 50.127 mensagens acumuladas, +800/min, lag de 18 min e crescendo.
- Os 12 pods estão no máximo do HPA.

**Linha do tempo (UTC)**

- 23/04 18:42:11: sync do Argo CD, v2.47.0 → v2.48.0.
- 24/04 13:30: p99 420 ms, 1.200 req/s, erro 0,2%.
- 13:45: 510 ms, 1.450 req/s, 0,3%.
- 14:00: 780 ms, 1.780 req/s, 0,8%. Primeiro sinal claro.
- 14:10: 2.400 ms, 2.100 req/s, 4,5%.
- 14:15: 5.200 ms, 2.400 req/s, 8,2%.
- 14:19:48: pool esgotado no pod xk2jp (max=20, active=20, waiting=147).
- 14:19:51: circuit breaker abre.
- 14:20: 8.100 ms, 2.650 req/s, 11,7%.

O deploy e o início da degradação estão separados por cerca de 19 h (18:42 → ~13:45 do dia seguinte). Isso sugere efeito lento ou gatilho de carga, não falha imediata.

**Degradação vs. tráfego**  
Tráfego: 1.200 → 2.650 req/s (+121%). p99: 420 → 8.100 ms (+1.829%, 19,3×). A latência cresce muito mais rápido que a carga, o que é assinatura de fila no pool, não de escala linear. Entre 14:00 e 14:10, o tráfego subiu 18% e o p99 subiu 208%.

**Conexões**

- Pool máximo: 12 pods × 20 = 240.
- Limite do RDS: 250. Folga: 250 − 240 = 10 conexões (4%) para outros clientes.
- Uso atual: 240/250 (96%). Os pools estão 100% cheios.
- Se o pool subir para 25 por pod: 12 × 25 = 300, que excede 250 em 50. Qualquer ajuste de pool ou de pods exige antes aumentar o limite do RDS.

**Causa provável**

- **Esgotamento do pool de conexões ao Ledger** (confiança alta). Logs e contagem 240/240 são consistentes.
- **Gatilho:** carga acima da capacidade efetiva, possivelmente reduzida pela v2.48 (pool movido para nova biblioteca, psycopg 3.1.18 → 3.2.0, timeout 5 s → 2 s) (confiança baixa a média, **hipótese**).

**Hipóteses**

|Hipótese|Evidência no input|Confirma|Refuta|
|---|---|---|---|
|H1: regressão da v2.48 no pool/driver/timeout|Mudanças no caminho exato do pool. Log "after 2000ms" bate com o novo timeout|v2.47 sob a mesma carga mantém o pool abaixo de 20 por pod|v2.47 também satura em ~2.650 req/s|
|H2: carga acima da capacidade|Tráfego +121%; HPA no máximo|v2.47 satura com a mesma carga|v2.47 estável nessa carga|
|H3: Ledger/RDS lento|Timeout em SELECT em `transactions`; "connection reset by peer"|CPU, IOPS, locks ou slow queries altos no RDS|RDS saudável e queries rápidas|
|H4: endpoint /batch (novo)|"context deadline exceeded" no handler do batch|Fatia grande do tráfego no /batch com duração longa|Tráfego no /batch pequeno|

**Fatores contribuintes**

- HPA no máximo, sem elasticidade de pods.
- Folga de 10 conexões entre a soma dos pools e o limite do RDS.
- Timeout reduzido de 5 s para 2 s. Pode aumentar erros e retries (**hipótese**; o input não mostra retries).
- Falha em cascata: o circuit breaker abriu e o Reactor passou a falhar ao publicar.

**Opções para o CTO**

1. **Rollback para v2.47.0 (Argo CD).**
    - Tempo para efeito: minutos (estimativa a validar no cluster).
    - Risco: o endpoint /batch deixa de existir; verificar se algum cliente já o usa. Não há migração de banco no input.
    - Reversível: sim, por novo sync da v2.48.
    - Ataca H1 diretamente. Ataca H2 só se a v2.47 aguentar 2.650 req/s, o que é desconhecido.
2. **Escala emergencial (aumento do limite do RDS, da classe da instância e do pool).**
    - Tempo para efeito: de minutos a dezenas de minutos. Alterar `max_connections` costuma exigir reinício, a verificar.
    - Risco: possível indisponibilidade, custo, e folga real só se CPU e IOPS do RDS aguentarem.
    - Reversível: o pool, sim. A troca de classe, com custo e tempo.
    - Ataca H2 e o sintoma. Se H1 for um vazamento, só adia o problema.

**Recomendação:** rollback agora. É a única mudança no caminho do recurso saturado, é reversível e não exige alterar o banco. Em paralelo, preparar a opção 2 e levantar as métricas do RDS. Se o p99 não voltar perto de 420 ms (patamar das 13:30) após o rollback, H2 ganha peso e a opção 2 entra.

**Backlog do Reactor**

- O tempo de drenagem não é calculável, porque falta a vazão do consumidor. Fórmula: tempo = 50.127 ÷ (vazão de consumo − vazão de chegada).
- Ilustração: com 1.000 msg/min de capacidade líquida, a fila drena em ~50 min. Hoje a fila cresce 800/min, então a capacidade líquida é negativa.
- Risco de duplicidade: "failed to publish" e "deadline exceeded" podem gerar reenvio. Há duplicidade se publish e ack não forem idempotentes.
- Risco de perda: possível se o ack ocorrer antes do processamento. Nenhum dos dois está no input.

**Lacunas de dados**

- Baseline da v2.47 com a mesma carga.
- Métricas de pool por pod (só o pod xk2jp tem log).
- RDS: CPU, IOPS, locks, slow queries, `max_connections` atual.
- Latência do Ledger e duração por endpoint (/batch vs. demais).
- Política de retry dos clientes e idempotência do /batch.
- Vazão e semântica de ack do Reactor.
- Consumo dos 10 conexões de folga por outros clientes.
- Inconsistências: o título fala em "últimos 30 min", mas a tabela vai de 13:30 a 14:20 (50 min). Os artefatos são de 23 e 24/04/2026, e a data de hoje é 22/11/2026. Confirmar que se trata do mesmo evento.

**Ações de follow-up** (dono a definir)

- Executar o rollback e registrar o horário de efeito.
- Testar a v2.47 e a v2.48 com 2.650 req/s em staging.
- Coletar métricas de RDS e de pool por pod.
- Revisar o timeout de 2 s e a política de retry.
- Garantir idempotência no /batch e no publish; revisar DLQ e reprocessamento do Reactor.
- Criar alertas para pool acima de 80% e folga do RDS abaixo de um limite definido.
- Definir plano de capacidade: pods × pool deve caber no limite do RDS com margem.

## 4. Justificativa

Escolhi o RISE porque a análise tem etapas que se correlacionam. Primeiro vem a linha do tempo, depois o último deploy é confrontado com o aumento de conexões e do tempo de resposta, e só então vem a decisão. O Steps garante que essa sequência seja seguida. O Role define o papel do modelo e o Expectation reduz o risco de invenção de dados e define o formato do postmortem. No output, isso apareceu na linha do tempo, na conta 12 × 20 = 240 contra o limite de 250 e no formato de saída.

Com o RTF, o prompt seria mais curto e a conclusão poderia até estar certa, mas não haveria garantia de que as etapas foram feitas nem como auditar o raciocínio. Com o CARE, o Example ajudaria a padronizar o postmortem com o modelo da empresa, o que seria um ganho, mas eu perderia a sequência, porque o Action é uma instrução única e a ordem da análise ficaria a critério do modelo. O custo do RISE é um prompt mais longo e uma ordem rígida, que pode gerar análises desnecessárias em partes simples.