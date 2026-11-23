**Tarefa.** Aplicando o framework **T-A-G**, escrever o prompt de IA que produza essa query SQL.
**Entregue.** Prompt, modelo, output e justificativa mostrando como Task, Action e Goal aparecem no prompt.


### 1. Prompt

```

[TASK]

Escreva uma query SQL para PostgreSQL (banco Ledger) que consolide, por mês e por categoria, as transações concluídas dos últimos 6 meses, para medir o crescimento
de transações por categoria.

Considere a estrutura das tabelas abaixo:

CREATE TABLE transactions ( id BIGSERIAL PRIMARY KEY, customer_id BIGINT NOT NULL REFERENCES customers(id), category VARCHAR(32) NOT NULL, amount_cents BIGINT NOT NULL, status VARCHAR(16) NOT NULL, payment_method VARCHAR(16), created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(), completed_at TIMESTAMPTZ ); CREATE INDEX idx_transactions_created_at ON transactions(created_at); CREATE INDEX idx_transactions_status ON transactions(status); CREATE INDEX idx_transactions_category ON transactions(category);

CREATE TABLE customers ( id BIGSERIAL PRIMARY KEY, segment VARCHAR(16) NOT NULL, country CHAR(2) NOT NULL, signup_at TIMESTAMPTZ NOT NULL DEFAULT NOW() );


[ACTION]

1. Filtre status = 'completed'.
2. Inclua só as categorias subscription, one_time, refund e credit_adjustment.
3. Use created_at como data da transação.
4. Recorte: últimos 6 meses corridos a partir de 2026-04-24.
5. Agrupe por mês (YYYY-MM) e categoria.
6. Métricas: quantidade de transações e volume total em reais (Converter o campo `amount_cents` de centavos para reais com duas casas decimais).
7. Ordene por mês crescente e depois por categoria crescente.
8. Retorne apenas a query, com um comentário em cada regra acima. No topo, em comentário, liste as premissas que afetam a leitura dos números (meses parciais nas pontas, sinal de valor em refund).


[GOAL]

A Jennifer, que não escreve SQL, vai usar o resultado na apresentação sobre crescimento de transações que leva à Goldie na semana que vem. Sucesso = a query roda sem ajuste e nenhuma premissa que altere a leitura do crescimento fica escondida.



```

### 2. Modelo

```
Opus 5.5
```


### 3. Output

```SQL
/* =====================================================================
   PREMISSAS QUE AFETAM A LEITURA DOS NÚMEROS
   ---------------------------------------------------------------------
   1) MESES PARCIAIS NAS PONTAS: o recorte vai de 24/10/2025 a 24/04/2026.
      - 2025-10 contém só ~8 dias (24 a 31/10).
      - 2026-04 contém só 24 dias (01 a 24/04).
      Esses dois meses aparecerão "baixos" e NÃO devem ser comparados
      diretamente com os meses cheios (nov/2025 a mar/2026) como se fossem
      queda ou crescimento. Para comparar crescimento mês a mês, use apenas
      os meses cheios, ou compare as pontas com o mesmo período dos outros meses.

   2) SINAL DO VALOR EM REFUND (e credit_adjustment): a query soma
      amount_cents exatamente como está gravado. Não sabemos pela estrutura
      da tabela se reembolsos são gravados como valor negativo ou positivo.
      - Se forem negativos: o volume de refund aparecerá negativo.
      - Se forem positivos: o volume de refund aparecerá positivo e
        representa dinheiro devolvido, não receita.
      Em ambos os casos, refund NÃO deve ser somado às outras categorias
      como "crescimento de receita". O mesmo vale para credit_adjustment,
      que pode conter valores positivos e negativos no mesmo mês.
      Confirme com o time de dados antes da apresentação.

   3) DATA DA TRANSAÇÃO = created_at (criação), não completed_at (conclusão).
      Uma transação criada em março e concluída em abril conta em março.
      Transações criadas dentro do recorte e concluídas depois da extração
      ainda não aparecem como 'completed', então o mês mais recente pode
      crescer um pouco se a query for rodada de novo mais tarde.

   4) FUSO HORÁRIO: meses e limites do recorte calculados no horário de
      Brasília (America/Sao_Paulo).

   5) DATA DE REFERÊNCIA FIXA: "hoje" = 2026-04-24, conforme solicitado.
      Rodar a query em outra data devolve o mesmo resultado.
   ===================================================================== */

SELECT
    -- Regra 5: agrupamento por mês no formato YYYY-MM (horário de Brasília)
    TO_CHAR(t.created_at AT TIME ZONE 'America/Sao_Paulo', 'YYYY-MM') AS mes,

    -- Regra 5: agrupamento por categoria
    t.category                                                         AS categoria,

    -- Regra 6: métrica 1, quantidade de transações
    COUNT(*)                                                           AS qtd_transacoes,

    -- Regra 6: métrica 2, volume total em reais com 2 casas decimais
    -- (amount_cents / 100; cast para NUMERIC evita divisão inteira;
    --  sinal mantido como gravado, ver premissa 2)
    ROUND(SUM(t.amount_cents)::NUMERIC / 100, 2)                       AS volume_total_reais

FROM transactions t

WHERE
    -- Regra 1: somente transações concluídas
    t.status = 'completed'

    -- Regra 2: somente as categorias solicitadas
    AND t.category IN ('subscription', 'one_time', 'refund', 'credit_adjustment')

    -- Regra 3: created_at é a data da transação (usada no filtro e no agrupamento)
    -- Regra 4: últimos 6 meses corridos a partir de 2026-04-24,
    --          de 24/10/2025 00:00 até o fim de 24/04/2026 (inclusive), horário de Brasília.
    --          Filtro direto na coluna para aproveitar idx_transactions_created_at.
    AND t.created_at >= (DATE '2026-04-24' - INTERVAL '6 months')::TIMESTAMP AT TIME ZONE 'America/Sao_Paulo'
    AND t.created_at <  (DATE '2026-04-24' + INTERVAL '1 day')::TIMESTAMP   AT TIME ZONE 'America/Sao_Paulo'

-- Regra 5: agrupa por mês e categoria
GROUP BY 1, 2

-- Regra 7: ordena por mês crescente e depois por categoria crescente
ORDER BY mes ASC, categoria ASC;

```

### 4. Justificativa

**Task:** define o entregável (uma query PostgreSQL), o banco (Ledger), o insumo (as duas tabelas) e o objetivo analítico (crescimento por categoria em 6 meses). É o que faz o modelo produzir SQL, e não uma explicação ou um relatório.

**Action:** transforma cada regra do enunciado em uma instrução que dá para conferir na saída: status, categorias, coluna de data e fuso, janela com limites explícitos, conversão para reais, agrupamento, métricas, ordenação, comentários e premissas. Cada item aparece como um bloco comentado na query. Explicitar a coluna e os limites tira do modelo decisões que antes ficavam por conta dele.

**Goal:** diz para quem é e para que serve: a Jennifer não escreve SQL e vai apresentar para a Goldie. Daí vem o critério de sucesso: rodar sem ajuste e premissas visíveis. Isso faz o modelo avisar dos meses parciais e do sinal de `refund`, que afetam a leitura do crescimento.