**Tarefa.** Aplicando o framework **T-A-G**, escrever o prompt de IA que, a partir do CSV acima, produza esse relatório alinhado à meta de 15%.
**Entregue.** Prompt, modelo, output e justificativa mostrando como Task, Action e Goal aparecem no prompt.


### 1. Prompt

```

[TASK]
Criar um relatório que mostre as oportunidades de economia com os custos de Cloud sem que o SLA seja afetado. Priorizar cada oportunidade por impacto, quanto cada uma representa em percentual da conta total, o esforço de implementação (baixo, médio, alto) e os riscos ou pré-requisitos envolvidos em cada uma.

Utilize como referência o breakdown de custos AWS do último mês:

CSV
servico,categoria,custo_mensal_usd,uso_medio_pct,observacao 
EC2 reservada,compute,4200,72,contrato de 1 ano 
EC2 on-demand,compute,8200,45,workloads variaveis 
EKS,compute,6700,58,3 clusters 
RDS PostgreSQL,databases,8200,62,multi-AZ 
ElastiCache Redis,databases,2100,40,cluster de producao 
S3 Standard,storage,3100,,5 buckets principais 
EBS gp3,storage,1600,68,volumes de producao 
CloudWatch Logs,observability,2800,,retencao de 90 dias 
CloudWatch Metrics,observability,900,, 
Data Transfer Out,network,1900,,trafego entre regioes 
NAT Gateway,network,1200,,3 gateways ativos 
Lambda,compute,900,30,~12M invocacoes/mes

[ACTION]

1. Calcule o total mensal e o valor de 15%. Mostre a conta.
2. Avalie cada linha do CSV: alavanca AWS aplicável, economia estimada (faixa e valor-base), % sobre a conta TOTAL, esforço (baixo/médio/alto) e riscos/pré-requisitos, com foco em SLA.
3. Ordene por impacto e mostre o % acumulado.
4. Proponha um plano por ondas que atinja os 15% com margem, sem depender das ações de esforço alto.
5. Regras: 
   não invente dados ausentes do CSV; 
   campo vazio = sem dado; 
   "uso_medio" é média (decida capacidade por pico); 
   não remova redundância (multi-AZ, NAT por AZ); 
   explicite premissas; evite dupla contagem entre ações; 
   confira todo cálculo antes de responder.


[GOAL]
Permitir que a Goldie leve à diretoria um plano para reduzir ≥15% do custo mensal recorrente até o fim do trimestre sem degradar SLA.
Sucesso = soma ≥15% com margem, números rastreáveis ao CSV ou a premissas explícitas, riscos de SLA visíveis.


```

### 2. Modelo

```
Opus 5.5
```


### 3. Output

```

# Plano de Otimização de Custos AWS — Redução ≥15% sem impacto em SLA

20 de nov. de 2026 · @Fabricio

## Resumo executivo

O plano base reduz **US$ 6.994,58/mês (16,73% da conta de US$ 41.800)**, acima da meta de US$ 6.270 (15%) com margem de US$ 724,58. Usa só ações de esforço baixo e médio e não toca em Multi-AZ do RDS nem nos 3 NAT Gateways.

- **51% da economia (US$ 3.557,13; 8,51% da conta) vem de compromisso de 1 ano** (Compute Savings Plan e RI do RDS). Isso não muda nada em runtime: risco de SLA nulo, risco financeiro de lock-in.
- **Faixa de incerteza: 11,33% a 22,90%.** No cenário pessimista a meta não é atingida. Por isso há um gate em 04/12, que troca premissas por números reais do Cost Explorer e aciona contingência se preciso.
- **Ações com risco de SLA** (rightsizing de EC2/EKS, power tuning de Lambda, corte de logs) somam 4,13% da conta, ficam nas ondas 2 e 3 e dependem de dados de pico que o CSV não tem.

**Decisões pedidas à diretoria:** aprovar compromissos de 1 ano (Compute SP, RI do RDS, reserva do ElastiCache); aprovar janelas de mudança em dezembro para as ondas 2 e 3; confirmar o requisito de retenção de logs (compliance).

## Total mensal e meta

A conta do último mês é **US$ 41.800** e a meta de 15% é **US$ 6.270/mês**.

- Total: 4.200 + 8.200 + 6.700 + 8.200 + 2.100 + 3.100 + 1.600 + 2.800 + 900 + 1.900 + 1.200 + 900 = **41.800**
- Prova por categoria: compute 20.000 (4.200 + 8.200 + 6.700 + 900) + databases 10.300 (8.200 + 2.100) + storage 4.700 (3.100 + 1.600) + observability 3.700 (2.800 + 900) + network 3.100 (1.900 + 1.200) = **41.800**
- Meta: 41.800 × 0,15 = **6.270**

Todos os percentuais deste documento são sobre a conta total de US$ 41.800, não sobre a linha do serviço.

## Premissas

Tudo que não está no CSV entra como premissa numerada, com faixa e valor-base. Cada uma deve ser substituída por dado real no gate de 04/12.

|   |   |   |   |
|---|---|---|---|
|#|Premissa|Faixa (base)|Como validar|
|P1|Região não consta no CSV; usados preços públicos de us-east-1 e 730 h/mês: EKS control plane US$ 0,10/h por cluster (suporte padrão), NAT US$ 0,045/h e US$ 0,045/GB, Lambda US$ 0,20 por milhão de requisições|—|Tabela de preços da região real|
|P2|Desconto de compromisso de 1 ano sem upfront é referência, não cotação: Compute SP; RI RDS; reserva ElastiCache|20–30% (25%); 25–35% (30%); 25–35% (30%)|Recomendações de compra do Cost Explorer|
|P3|Cobertura do Compute SP sobre EC2 on-demand e nós EKS, dimensionada pelo **piso horário** (mínimo) e não pela média, para o compromisso ficar 100% utilizado|40–60% (50%)|Gasto horário mínimo de 30–60 dias no CUR|
|P4|A linha EKS = control plane (3 × 0,10 × 730 = US$ 219) + nós (US$ 6.481) em on-demand, **fora** da EC2 reservada|—|CUR por usage type. Maior risco ao número do plano|
|P5|Horas de instância do RDS como fração da linha (o resto é storage, IOPS, backup, não coberto por RI)|60–80% (70%)|Cost Explorer por usage type|
|P6|Horas de nó do ElastiCache como fração da linha|90–100% (100%)|Cost Explorer|
|P7|`uso_medio_pct` é média. Nenhum rightsizing é decidido com ela. Ganho de rightsizing condicionado a p95/p99 e máximo de 30 dias|10–20% (15%) da parcela tratada|CloudWatch/Container Insights, pico|
|P8|S3: fração da linha em objetos frios que migram para camadas ~40% mais baratas|20–40% (30%)|S3 Storage Lens / Storage Class Analysis|
|P9|CloudWatch Logs: armazenamento; ingestão; o resto (consultas etc.) não é tratado. Export para S3 Glacier Instant Retrieval a US$ 0,004/GB-mês vs US$ 0,03 no CloudWatch|10–20% (15%); 80–90% (85%)|Cost Explorer por usage type|
|P10|NAT: processamento = 1.200 − 98,55 (3 × 0,045 × 730) = US$ 1.101,45; fração que é tráfego para S3 na mesma região (inclui layers do ECR)|20–40% (30%)|VPC Flow Logs|
|P11|Lambda: duração = 900 − 2,40 (12M × 0,20/M) = US$ 897,60; ganho de power tuning se a memória estiver superdimensionada (uso 30%)|20–40% (30%)|AWS Lambda Power Tuning por função|
|P12|Trimestre = calendário, termina em 31/12/2026. Meta = run-rate mensal recorrente nessa data|—|Confirmar com a Goldie|
|P13|Cenários pessimista/otimista combinam os extremos de todas as premissas ao mesmo tempo (limites, não probabilidades)|—|—|

## Avaliação linha a linha

Das 12 linhas, 9 têm alavanca estimável; EC2 reservada fica em zero no trimestre e EBS e Data Transfer Out não são estimáveis sem dado. Os códigos A1–A11 são as ações do ranking.

|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|Linha (US$/mês)|Uso médio|Alavanca AWS|Economia US$/mês: faixa (base)|% da conta (base)|Esforço|Riscos e pré-requisitos (foco SLA)|
|EC2 reservada (4.200)|72%|Nenhuma no trimestre; na renovação, avaliar trocar por Compute SP|0|0,00%|—|Data de vencimento não está no CSV. Se 72% for utilização da RI, há 28% ocioso: mover workload on-demand para ela reduz o SP necessário (não soma à economia)|
|EC2 on-demand (8.200)|45%|A1 Compute SP 1 ano sobre 50%; A5 rightsizing e autoscaling só na parcela descoberta|A1: 656–1.476 (1.025,00); A5: 410–820 (615,00)|2,45% + 1,47%|Baixo; Médio|A1 não muda runtime; risco é lock-in. A5: subdimensionar no pico. Exige p95/p99, autoscaling, canário e rollback. Spot e Graviton ficam fora (esforço alto)|
|EKS, 3 clusters (6.700)|58%|A1 Compute SP sobre 50% dos nós; A4 consolidação de nós (Karpenter ou Cluster Autoscaler) e requests pelo pico|A1: 518–1.167 (810,13); A4: 324–648 (486,08)|1,94% + 1,16%|Baixo; Médio|A4: evicções durante consolidação. Pré-requisitos: PodDisruptionBudgets, topology spread entre AZs, janela. Checar versão do Kubernetes: suporte estendido custa US$ 0,60/h por cluster (não contado)|
|RDS PostgreSQL Multi-AZ (8.200)|62%|A2 Reserved Instances Multi-AZ 1 ano|1.230–2.296 (1.722,00)|4,12%|Baixo|Mantém Multi-AZ; não muda runtime. Pré-requisito: nenhuma troca de família/engine prevista em 12 meses. Rightsizing não recomendado no trimestre (só há média)|
|ElastiCache Redis (2.100)|40%|A3 nós reservados 1 ano|472–735 (630,00)|1,51%|Baixo|Reserva trava o tamanho do nó: antes, 2 semanas de análise de pico de memória para decidir se há rightsizing. Rightsizing é ganho extra não contado (risco de eviction)|
|S3 Standard, 5 buckets (3.100)|sem dado|A6 Intelligent-Tiering via lifecycle nos buckets elegíveis|248–496 (372,00)|0,89%|Baixo|SLA AWS cai de 99,9% (Standard) para 99,0% (Intelligent-Tiering): aplicar só fora do caminho crítico. Taxa de monitoramento por objeto; objetos < 128 KB não mudam de camada|
|EBS gp3 (1.600)|68%|Investigação: volumes órfãos, IOPS/throughput acima do baseline|Não estimável (sem dado)|—|Médio|EBS não encolhe in-place; 68% de uso médio não justifica reduzir. Não contado|
|CloudWatch Logs, retenção 90 d (2.800)|sem dado|A10 retenção 90 → 30 dias + export para S3 Glacier IR; A8 corte de ingestão (nível de log, health checks, amostragem)|A10: 162–324 (242,67); A8: 224–630 (357,00)|0,58% + 0,85%|Baixo; Médio|A10: confirmar requisito de compliance; logs antigos seguem consultáveis via Athena para RCA. A8: perda de sinal aumenta MTTR. Manter ERROR/WARN e nível de log dinâmico|
|CloudWatch Metrics (900)|sem dado|A11 remover métricas custom sem uso; reduzir alta resolução e cardinalidade|90–180 (135,00)|0,32%|Médio|Métrica usada por alarme removida = ponto cego. Pré-requisito: inventário de alarmes e dashboards|
|Data Transfer Out, entre regiões (1.900)|sem dado|Investigação por usage type (compressão, cache, replicação desnecessária)|Não estimável (sem dado)|—|Médio a alto|Se for replicação de DR, reduzir piora RPO. Não contado|
|NAT Gateway, 3 ativos (1.200)|sem dado|A7 VPC Gateway Endpoint para S3 (sem custo)|220–441 (330,44)|0,79%|Baixo|Mantém 1 NAT por AZ. Bucket policies com aws:SourceIp deixam de casar: revisar antes. Só vale para buckets da mesma região. Rollback = remover a rota|
|Lambda, ~12M invocações (900)|30%|A9 power tuning de memória por função|179–359 (269,28)|0,64%|Médio|Menos memória = menos CPU, duração pode subir. Aceitar só se p99 ficar no orçamento de latência, sem timeouts. Lambda fica fora do SP para não contar duas vezes|

## Ranking por impacto

As duas primeiras ações entregam 8,51% da conta; a meta de 15% é cruzada na 8ª ação (15,19%) e o conjunto chega a 16,73%.

|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|#|Ação|Esforço|Economia base (US$/mês)|% da conta|Acumulado (US$/mês)|% acumulado|
|1|A1 Compute Savings Plan 1 ano (EC2 on-demand + nós EKS)|Baixo|1.835,13|4,39%|1.835,13|4,39%|
|2|A2 RDS Reserved Instances Multi-AZ 1 ano|Baixo|1.722,00|4,12%|3.557,13|8,51%|
|3|A3 ElastiCache nós reservados 1 ano|Baixo|630,00|1,51%|4.187,13|10,02%|
|4|A5 Rightsizing/autoscaling EC2 on-demand (parcela descoberta)|Médio|615,00|1,47%|4.802,13|11,49%|
|5|A4 Consolidação de nós EKS (parcela descoberta)|Médio|486,08|1,16%|5.288,20|12,65%|
|6|A6 S3 Intelligent-Tiering|Baixo|372,00|0,89%|5.660,20|13,54%|
|7|A8 Corte de ingestão no CloudWatch Logs|Médio|357,00|0,85%|6.017,20|14,40%|
|8|A7 VPC Gateway Endpoint S3 (reduz processamento do NAT)|Baixo|330,44|0,79%|6.347,64|15,19%|
|9|A9 Power tuning de Lambda|Médio|269,28|0,64%|6.616,92|15,83%|
|10|A10 Retenção de logs 90 → 30 dias + export S3|Baixo|242,67|0,58%|6.859,58|16,41%|
|11|A11 Limpeza de CloudWatch Metrics|Médio|135,00|0,32%|6.994,58|16,73%|
||**Total base**||**6.994,58**|**16,73%**|||
||Total pessimista (P13)||4.734,62|11,33%|||
||Total otimista (P13)||9.570,86|22,90%|||

Nenhuma ação de esforço alto está no ranking. Acumulados calculados sobre os valores sem arredondamento; a soma das parcelas exibidas difere em US$ 0,02.

## Plano por ondas

Três ondas até 31/12 levam o run-rate a 16,73% de redução (margem de 1,73 p.p. ou US$ 724,58 sobre a meta), sem ações de esforço alto. A Onda 1 só tem ações de esforço baixo e entrega 10,77%.

|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|Onda|Janela|Ações|Economia base (US$/mês)|% da conta|Acumulado|Faixa da onda (US$/mês)|
|1 — compromissos e mudanças reversíveis|23/11 a 04/12|A1, A2, A6, A7, A10|4.502,23|10,77%|10,77%|3.034,50–6.198,70|
|2 — condicionadas a análise|07/12 a 18/12|A3, A8, A11|1.122,00|2,68%|13,46%|786,50–1.545,00|
|3 — capacidade, por pico|07/12 a 31/12|A5, A4, A9|1.370,36|3,28%|16,73%|913,60–1.827,10|

**Sequência e dependências**

1. Semana de 23/11: coletar no CUR o gasto horário mínimo (30–60 dias) e, no CloudWatch, p95/p99 e máximo de CPU e memória de EC2, nós EKS, Redis e Lambda. Essa coleta alimenta o gate e as ondas 2 e 3.
2. Antes de comprar o SP (A1): dimensionar o compromisso pelo piso horário **já descontado o rightsizing previsto** (A4, A5). Assim o SP segue 100% utilizado depois da onda 3.
3. A6 e A7 entram primeiro em buckets e route tables fora do caminho crítico, depois no restante, com rollback documentado.
4. **Gate de 04/12:** trocar as premissas P2–P6 e P10 pelos números reais e recalcular. Se a projeção ficar abaixo de 15%, acionar a contingência (seção seguinte).
5. Ondas 2 e 3 rodam em janelas de mudança, serviço a serviço, com canário e critério de rollback por SLO.

Compromissos (A1–A3) cobram a partir da compra; o mês cheio de economia aparece na fatura de janeiro. A meta é de run-rate (P12). Se houver congelamento de mudanças em dezembro, as ondas 2 e 3 precisam de exceção aprovada.

## Riscos de SLA e contingência

Nenhuma ação remove redundância: Multi-AZ do RDS e 1 NAT Gateway por AZ ficam intactos. O risco de SLA está concentrado nas ações de capacidade e de observabilidade.

|   |   |   |   |
|---|---|---|---|
|Risco|Ações|Efeito no SLA|Mitigação|
|Capacidade abaixo do pico|A4, A5, A9|Latência, erros, timeouts|Decidir pelo p95/p99 e máximo, não pela média; autoscaling; canário; rollback por SLO|
|Evicção de pods na consolidação|A4|Indisponibilidade parcial|PodDisruptionBudgets, topology spread entre AZs, janela de mudança|
|SLA menor no S3|A6|99,9% → 99,0% de disponibilidade contratual|Só em buckets fora do caminho crítico|
|Acesso ao S3 negado após endpoint|A7|Falha de aplicações que leem S3|Revisar bucket policies com aws:SourceIp; aplicar por route table; rollback = remover rota|
|Perda de sinal para diagnóstico|A8, A10, A11|MTTR maior (indireto)|Manter ERROR/WARN; export para S3 consultável via Athena; inventário de alarmes antes de remover métricas|
|Lock-in financeiro de 12 meses|A1, A2, A3|Nenhum (só faturamento)|Compromisso pelo piso horário; sem troca de família/engine planejada|
|Premissa P4 errada (nós EKS já na RI)|A1|Nenhum|Validar no CUR antes do gate; pode tirar até US$ 810,13 do plano|

**Contingência, se o gate de 04/12 projetar menos de 15%.** No cenário pessimista faltam US$ 1.535,38 (6.270 − 4.734,62). Opções sem esforço alto, não contadas no plano:

- Pagamento parcial ou total adiantado nos compromissos: desconto maior, custo de caixa. Valor a calcular com o Cost Explorer.
- Prazo de 3 anos em parte dos compromissos: desconto maior, lock-in maior. Decisão de diretoria.
- Rightsizing do ElastiCache, se a análise de pico de memória permitir (antes de reservar).
- Se a EC2 reservada estiver 28% ociosa, mover workload on-demand para ela.
- Se os 3 clusters EKS estiverem em suporte estendido, o upgrade de versão economiza até (0,60 − 0,10) × 730 × 3 = US$ 1.095/mês. Esforço médio a alto, depende de testes de compatibilidade.

## Conferência dos cálculos

Todos os valores foram recalculados em script a partir do CSV e das premissas; a soma base confere em US$ 6.994,58.

**Fórmulas (valor-base)**

- A1 = 8.200 × 0,50 × 0,25 + 6.481 × 0,50 × 0,25 = 1.025,00 + 810,13 = 1.835,13
- A2 = 8.200 × 0,70 × 0,30 = 1.722,00
- A3 = 2.100 × 1,00 × 0,30 = 630,00
- A4 = 6.481 × 0,50 × 0,15 = 486,08
- A5 = 8.200 × 0,50 × 0,15 = 615,00
- A6 = 3.100 × 0,30 × 0,40 = 372,00
- A7 = 1.101,45 × 0,30 = 330,44
- A8 = 2.800 × 0,85 × 0,15 = 357,00
- A9 = 897,60 × 0,30 = 269,28
- A10 = 2.800 × 0,15 × 2/3 × (1 − 0,004/0,03) = 242,67 (líquido do custo do S3 para os logs exportados)
- A11 = 900 × 0,15 = 135,00

**Regras contra dupla contagem**

- A1 × A4/A5: o SP incide só na parcela coberta (50%); rightsizing só na parcela descoberta (50%).
- A1 × Lambda: Lambda fica fora do SP, porque o power tuning (A9) muda a base.
- A1 × EC2 reservada: o SP não incide sobre horas já cobertas pela RI (premissa P4).
- A2: só horas de instância; storage, IOPS e backup do RDS ficam fora.
- A8 × A10: ingestão e armazenamento são fatias disjuntas da linha. O efeito de menos ingestão sobre o armazenamento não foi somado (conservador).
- A10 × A6: o custo novo no S3 já foi descontado em A10; A6 trata só os objetos que já existem.
- A7 × Data Transfer Out: A7 reduz o processamento do NAT, cobrado à parte do tráfego entre regiões.
- A3: só a reserva conta; rightsizing do Redis é contingência.

**Fora do plano por esforço alto ou falta de dado:** Spot, migração para Graviton, troca de engine do Redis, rightsizing do RDS, EBS e Data Transfer Out.
```

### 4. Justificativa

**Task**: define o entregável (relatório de oportunidades de economia), o que cada oportunidade precisa trazer (impacto, % da conta, esforço e riscos), o insumo (CSV) e a restrição de não afetar o SLA. No relatório, isso aparece na tabela “Avaliação linha a linha”, que tem exatamente essas colunas.

**Action**: transforma a tarefa em passos (calcular total e meta, avaliar cada linha, ordenar por impacto, montar as ondas) e em regras (não inventar dados, campo vazio não é zero, média não decide capacidade, não remover redundância, evitar dupla contagem, conferir as contas). Cada passo virou uma seção do relatório. As regras geraram as premissas P1 a P13, a manutenção do Multi-AZ e dos 3 NAT, e a conferência dos cálculos.

**Goal**: define a meta (≥15% sem degradar SLA) e o critério de sucesso (margem, números rastreáveis, riscos visíveis). Por isso o relatório entrega um plano para decisão da diretoria, com resumo executivo, cenário pessimista, gate em 04/12 com contingência e a premissa de run-rate em 31/12.

**Aprendizado**: na primeira versão, o Action vazio deixou o modelo decidir a estrutura sozinho. Com o Action detalhado, o resultado ficou previsível e auditável.