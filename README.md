# trabalho

Boa, vou montar um material completo pra você. Vai ter 4 partes:

1. **O que sua chefe está realmente pedindo** (entendimento profundo do problema)
2. **Como sua análise deve ser interpretada** (o "pra que serve" isso)
3. **Estratégia de execução** (passo a passo do que fazer)
4. **A query e os entregáveis** (o trabalho técnico em si)

---

## Parte 1 — O que sua chefe está realmente pedindo

Antes de SQL, você precisa entender o **porquê** da análise. Senão você entrega o número certo mas a história errada.

### O contexto descoberto

O time já validou uma coisa importante: pedidos **sem a visibilidade** de "saiu pra entrega" cancelam **11,06%** das vezes. Pedidos **com** essa visibilidade cancelam só **1,95%**. Ou seja: quando o cliente não vê o pedido saindo, ele desconfia muito mais e cancela.

Diferença de quase 6x. Isso é enorme.

### A pergunta que ela quer responder

"Beleza, sabemos que falta de visibilidade aumenta cancelamento. Mas **em qual momento exatamente** o cliente desiste?"

Tem duas possibilidades:

- **Hipótese A:** O cliente desiste durante a **preparação** (restaurante demora pra fazer)
- **Hipótese B:** O cliente desiste durante a **entrega** (já saiu, mas demora pra chegar)

Saber qual é o vilão muda completamente o plano de ação:
- Se for **preparação** → problema operacional da loja (cozinha lenta, aceitou pedido demais)
- Se for **entrega** → problema logístico (entregador, distância, rota)

### A camada extra: foi atraso ou não?

Pra cada uma das duas fases, ela quer separar:
- **Delayed:** o restaurante estourou o tempo que ele mesmo prometeu
- **On Time:** cancelou mesmo dentro do prazo prometido (cliente impaciente, ou cancelou por outro motivo)

Isso ajuda a separar dois problemas diferentes:
- "Loja não cumpre o que promete" → ajustar SLA
- "Cliente cancela mesmo dentro do prazo" → ajustar comunicação/expectativa

### Por que isso importa pros 3 OKRs

Essa análise mira direto no **OKR 1 (reduzir B-duty cancellation)**. Mas indiretamente também afeta o OKR 3 (service metrics do marketplace), porque velocidade é um dos componentes.

---

## Parte 2 — Como sua análise deve ser interpretada

### O resultado esperado

Você vai entregar uma tabela tipo essa (números fictícios):

| Phase | Status | Volume | Avg Duration |
|---|---|---|---|
| Preparation | Delayed | 4.500 | 38 min |
| Preparation | On Time | 1.200 | 12 min |
| Delivery | Delayed | 7.800 | 65 min |
| Delivery | On Time | 900 | 42 min |

### Como ler esse resultado

Olhando esse exemplo fictício:

- **Volume total na entrega (8.700) > preparação (5.700)** → o vilão principal é a fase de entrega
- **87% dos cancelamentos na entrega são por atraso** → o problema é majoritariamente loja que não cumpre o SLA
- **Tempo médio de atraso na entrega = 65 min** → é muita coisa, cliente espera mais de 1h antes de desistir
- **Os 1.200 "preparation on time" são interessantes:** cliente cancelou dentro do prazo prometido. Por quê? Provavelmente porque **não viu nada acontecendo** (volta no contexto do 11% sem visibilidade). É o problema de **percepção**, não de tempo real.

Esse tipo de leitura é o que separa "rodei query" de "fiz análise". **Treina isso.**

### O que sua chefe provavelmente vai fazer com o resultado

- Se a maior parte for **preparation delayed:** vai conversar com Regional Ops sobre lojas com cozinha lenta, ou pensar em ajustar SLA de preparo no app
- Se a maior parte for **delivery delayed:** vai escalar pro time de logística do Shop Delivery
- Se tiver muito **on time:** vai pensar em comunicação proativa pro cliente ("seu pedido está sendo preparado", "saiu pra entrega") — exatamente o gap dos 11% vs 1,95%

---

## Parte 3 — Estratégia de execução (passo a passo)

### A estratégia

Você tem incógnitas no problema. Em vez de tentar resolver tudo de cara, faz assim:

1. Roda uma query exploratória curta primeiro pra descobrir os nomes que faltam
2. Valida premissas com o solicitante
3. Roda a análise final
4. Apresenta com as premissas explícitas

Isso é o que analista sênior faz — não sai escrevendo query de 200 linhas no escuro.

### Lista do que você ainda precisa descobrir

**Tabelas já mapeadas:**
- Tabela de pedidos: `[NOME_TABELA_PEDIDOS]` — apelido `d.` na request
- Tabela de shop info: `[NOME_TABELA_SHOP]` — apelido `s.`, onde mora `ka_group_type`
- Tabela de tempo de preparo: `dwd_shop_base_d_whole` ✅
- Tabela de tempo de entrega: `dwd_shop_delivery_d_whole` ✅

**Colunas que você ainda precisa mapear:**
- `[COL_DATA_PEDIDO]` — pra filtrar 01/02 a 12/04/2026
- `[COL_CANCELLED_TIMESTAMP]` — quando o pedido foi cancelado
- `[COL_ACCEPTED_TIMESTAMP]` — quando o restaurante aceitou
- `[COL_SHOP_ID]` — pra fazer o join (provavelmente `shop_id`, mas confirma)

**Unidades de tempo — crítico:**
- `avg_produce_time` está em segundos ou minutos?
- `avg_delivery_eta` está em segundos ou minutos?
- O cálculo `cancelled_timestamp - accepted_timestamp` retorna o quê (segundos, minutos, ou precisa de função tipo `TIMESTAMPDIFF`)?

Se errar a unidade, o resultado fica todo torto.

**Granularidade das tabelas `_d_whole`:**
- A tabela `dwd_shop_base_d_whole` tem 1 linha por shop por dia? Ou 1 linha por shop total? O sufixo `_d_whole` geralmente indica snapshot diário acumulado, mas confirma. Isso afeta como você faz o join.

### IMPORTANTE — premissa que pode quebrar a análise

Os campos que você me passou (`avg_produce_time` e `avg_delivery_eta`) parecem ser **médias por restaurante**, não o ETA específico daquele pedido. Isso muda a interpretação: você está comparando o tempo real até o cancelamento com a **média histórica daquela loja**, não com o ETA prometido naquele pedido.

Pode ser que isso seja exatamente o que sua chefe quer (proxy razoável), ou pode ser que exista uma coluna no próprio pedido com o ETA daquele pedido. **Vale procurar e perguntar antes.**

### Como descobrir o que falta

**Passo 1 — Pergunta direto.** Manda mensagem no canal do time de dados / DS / B-Ops sênior: *"Galera, qual o nome da tabela principal de orders aqui? E onde fica o timestamp de cancelamento e de aceite do restaurante?"* Em 80% dos casos alguém responde rápido.

**Passo 2 — Se ninguém responder, explora o schema:**

```sql
SELECT *
FROM [NOME_TABELA_PEDIDOS]
WHERE [COL_DATA_PEDIDO] >= '2026-04-01'
LIMIT 10
```

Procura colunas com `cancel`, `accept`, `create_time`, `_at`, `_time`. 

Faz o mesmo nas duas `_d_whole`:

```sql
SELECT * FROM dwd_shop_base_d_whole LIMIT 5
SELECT * FROM dwd_shop_delivery_d_whole LIMIT 5
```

Pra ver a granularidade.

**Passo 3 — Valida a unidade.** Pega um shop qualquer, vê o valor de `avg_produce_time`. Se for `1200`, é segundos (= 20 min). Se for `20`, é minutos. Confirma com 3-4 exemplos.

---

## Parte 4 — A query e os entregáveis

### A query (template pronto pra você preencher)

Considerando as regras da sua ferramenta (sem `WITH`, sem `NULLIF`, sem alias no final de campo calculado, colchetes nos campos), montei usando subqueries aninhadas:

```sql
SELECT
  fase,
  status_atraso,
  COUNT(*) AS volume,
  AVG(duracao_real_segundos) / 60.0 AS avg_duracao_minutos
FROM (
  SELECT
    CASE
      WHEN ([COL_CANCELLED_TIMESTAMP] - [COL_ACCEPTED_TIMESTAMP]) <= base.avg_produce_time
        THEN 'Preparation'
      ELSE 'Delivery'
    END AS fase,
    
    CASE
      -- Fase preparação: comparar tempo real até cancelamento com avg_produce_time
      WHEN ([COL_CANCELLED_TIMESTAMP] - [COL_ACCEPTED_TIMESTAMP]) <= base.avg_produce_time
        THEN CASE
          WHEN ([COL_CANCELLED_TIMESTAMP] - [COL_ACCEPTED_TIMESTAMP]) > base.avg_produce_time
            THEN 'Delayed'
          ELSE 'On Time'
        END
      -- Fase entrega: comparar tempo real de entrega com avg_delivery_eta
      ELSE CASE
        WHEN (([COL_CANCELLED_TIMESTAMP] - [COL_ACCEPTED_TIMESTAMP]) - base.avg_produce_time) > deli.avg_delivery_eta
          THEN 'Delayed'
        ELSE 'On Time'
      END
    END AS status_atraso,
    
    ([COL_CANCELLED_TIMESTAMP] - [COL_ACCEPTED_TIMESTAMP]) AS duracao_real_segundos
    
  FROM [NOME_TABELA_PEDIDOS] d
  LEFT JOIN [NOME_TABELA_SHOP] s
    ON d.[COL_SHOP_ID] = s.[COL_SHOP_ID]
  LEFT JOIN dwd_shop_base_d_whole base
    ON d.[COL_SHOP_ID] = base.[COL_SHOP_ID]
  LEFT JOIN dwd_shop_delivery_d_whole deli
    ON d.[COL_SHOP_ID] = deli.[COL_SHOP_ID]
  WHERE d.[COL_DATA_PEDIDO] BETWEEN '2026-02-01' AND '2026-04-12'
    AND d.delivery_type = 2
    AND (CAST(s.ka_group_type AS STRING) != '1' OR s.ka_group_type IS NULL)
    AND (CASE WHEN d.is_td_cancel = 1 AND d.e_dutyinfo_b_responsibility = '100' THEN 1 ELSE 0 END) = 1
) sub
GROUP BY fase, status_atraso
```

### Pontos de atenção no código

**1. As tabelas `_d_whole` provavelmente precisam de filtro de data no join.** Se elas têm 1 linha por shop por dia, o join `ON d.shop_id = base.shop_id` vai duplicar tudo. Você provavelmente precisa de:

```sql
LEFT JOIN dwd_shop_base_d_whole base
  ON d.[COL_SHOP_ID] = base.[COL_SHOP_ID]
  AND base.[COL_DATA] = d.[COL_DATA_PEDIDO]
```

Confirma isso ao explorar a tabela.

**2. Média histórica vs ETA do pedido.** Como falei: `avg_produce_time` e `avg_delivery_eta` são médias da loja. Pode existir uma coluna no próprio pedido com o ETA daquele pedido — vale procurar.

**3. Lógica do Phase 1 — detalhe importante.** Eu interpretei "fase preparação" como: se tempo decorrido ≤ avg_produce_time, foi durante preparação. **Confirma com sua chefe**, porque tem outra leitura possível: "cancelado antes do status mudar pra 'saiu pra entrega'". Se for essa, você precisa de uma coluna de status, não de cálculo de tempo.

**4. Status "On Time" parece estranho na Phase 1.** Se foi cancelado dentro do tempo de preparo médio, é "On Time" mesmo? Pode ser que sim (cliente cancelou por outro motivo). Vale validar.

**5. Unidades.** Garante que `cancelled_timestamp - accepted_timestamp`, `avg_produce_time` e `avg_delivery_eta` estão na mesma unidade. Pode precisar de `TIMESTAMPDIFF` ou similar.

**6. Subquery em vez de CTE.** Sua ferramenta não aceita `WITH`, então fiz aninhado.

**7. Sem alias no campo calculado.** Se for pra rodar como query de extração de dataset, beleza com os `AS`. Se for campo calculado dentro da ferramenta visual, tira os `AS`.

### Análises extras pra você surpreender

A request pediu o básico. Mas se você quiser ir além e impressionar, junto com a tabela principal entrega também:

**Extra 1 — Distribuição de tempo até cancelamento:** Um histograma. Quantos cancelaram em 10-20 min, 20-30 min, etc. Mostra se tem um "ponto de quebra" claro onde cliente desiste.

**Extra 2 — Quebra por categoria de loja:** Pizza, japa, hamburguer cancelam em fases diferentes? Pode ser que pizza cancele mais na preparação (demora pra fazer) e fast food mais na entrega.

**Extra 3 — Os top 20 restaurantes ofensores:** Quem mais aparece em "Delayed"? Lista nominal de lojas que estouram SLA com mais frequência. Isso é literalmente uma lista de ação pro Regional Ops.

**Extra 4 — Quanto tempo o cliente "tolera" em média:** O Avg Duration já mostra isso, mas tirar a mediana e o p90 ajuda. Se o p90 de "delivery delayed" é 80 min, significa que 90% dos clientes esperam até 80 min antes de desistir.

Não precisa entregar todos. Entrega o pedido e oferece: *"Rodei mais 2 análises complementares enquanto fazia essa, queres ver?"*

### Checklist antes de rodar

1. ☐ Descobri o nome real da tabela de pedidos e da tabela de shop
2. ☐ Achei `cancelled_timestamp`, `accepted_timestamp`, `data_pedido`, `shop_id`
3. ☐ Confirmei a unidade de tempo de tudo (segundos vs minutos)
4. ☐ Entendi a granularidade das `_d_whole` (precisa filtrar por data no join?)
5. ☐ Validei com a chefe a lógica de "fase preparação" vs "fase entrega"
6. ☐ Validei se faz sentido usar média da loja em vez de ETA específico do pedido
7. ☐ Rodei a query com `LIMIT 100` primeiro pra ver se o output faz sentido
8. ☐ Bati o número total de cancelamentos com algum report existente

### Como devolver o resultado

**Não entrega só a tabela.** Entrega um pequeno texto junto:

> *"Análise concluída. Antes do resultado, alinhando 3 premissas que assumi:*
> 
> *(1) Usei `avg_produce_time` e `avg_delivery_eta` como proxies de B_SET_ETA e DELIVER_SET_ETA. Note que são médias históricas da loja, não ETA específico do pedido — se preferir o ETA do pedido, posso refazer.*
> 
> *(2) Classifiquei 'fase preparação' como pedidos cuja duração até cancelamento foi ≤ avg_produce_time da loja. Se a definição correta é por status de 'saiu pra entrega', preciso refazer com outra coluna.*
> 
> *(3) Janela: 01/02 a 12/04/2026, Shop Delivery, Non-KA, B-duty cancelled apenas.*
> 
> *Resultado:*
> 
> [tabela]
> 
> *Principais leituras:*
> *- [observação 1]*
> *- [observação 2]*
> *- [observação 3]*
> 
> *Rodei também algumas análises complementares (top lojas ofensoras, distribuição de tempo até cancelamento). Se quiser, te mando junto."*

Isso te diferencia muito. Você não entrega número solto — entrega análise com premissas explícitas, interpretação e oferta de aprofundamento.

---

Quer que eu monte agora a query exploratória pra você descobrir os nomes que faltam, ou prefere que a gente refine a lógica das fases antes de você falar com sua chefe?
