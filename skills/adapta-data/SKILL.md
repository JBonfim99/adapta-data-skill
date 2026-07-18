---
name: adapta-data
description: Use when answering questions about Adapta's business data — receita, vendas, reembolso, renovação/NRR, marketing (ROAS/CPA/ads), LTV, placar semanal, por BU (ONE/LABS/SKIP/SUMMIT). Explains how Adapta captures its data (Guru → Nekt → ScoreCard), the official metric definitions (house methodology) and the correct use of the adapta-data MCP server. Trigger on "dados da Adapta", "scorecard", "NRR", "receita da BU", "quanto vendemos", "taxa de reembolso", "renovação", "ROAS", "placar", "LTV".
---

# adapta-data — captura, definições e uso correto do MCP

Manual de domínio para responder sobre os dados de negócio da Adapta.
Três partes: **como o dado é capturado** (e o que isso implica), **o que cada
métrica significa** (definições oficiais da casa) e **como usar o MCP** sem
produzir número errado.

---

## Parte 1 — Como a Adapta captura os dados (e as melhores práticas)

### O pipeline

```
Guru (vendas/assinaturas) ─┐
Meta Ads / Google Ads ─────┤→ Nekt (warehouse Athena: Bronze → Silver → Gold)
Trackings (UTMs, ad_id) ───┘        │  gold = star schema "fonte da verdade"
AI Gateway (custo de IA) ───────────┤
                                    ▼
                    ScoreCard BUs (Skip Cloud) — sync 4x/dia
                    (~06h/12h/18h/00h Brasília, full-replace)
                    cubos agregados → dashboard + MCP adapta-data
```

### O que a captura implica (melhores práticas ao interpretar)

1. **Dado não é tempo real.** Sincroniza 4x/dia; o dia corrente é sempre
   parcial. Sempre diga a janela do último sync ao falar de "hoje".
2. **BU não vem do produto "naturalmente" — vem da ALLOWLIST.** Cada produto é
   linkado manualmente a uma BU no mapa `produtos_bu`. Produto NÃO linkado sai
   de TODAS as métricas (decisão deliberada). O catálogo drifta: produto novo
   nasce fora até alguém linkar. Se um número parecer baixo, cheque a allowlist
   (`list_produtos_bu`, `incluir_drift=true`).
3. **Competência tem receita FUTURA.** Renovações agendadas entram em
   `dt_competencia` à frente. "Realizado" = sempre capado em hoje.
4. **Atribuição de marketing é por KEYWORD, com resolução de ad_id.** A BU de
   uma campanha vem de regras de keyword (campanha → adset → anúncio, cascata
   por prioridade). A receita atribuída resolve o ad_id nos trackings — isso
   recupera parte do "Sem Rastreio". Consequências: existe "Não atribuído"
   (campanha sem keyword) e a **cobertura de atribuição é < 100%** — cite a
   cobertura junto do ROAS quando relevante.
5. **Vazamento existe e é medido**: campanha de uma BU pode vender produto de
   outra. Upsell e créditos extras NÃO contam como vazamento (venda
   complementar).
6. **Coortes maturam.** Compra tem garantia de 30d (reembolso); renovação
   cobrada também tem 30d de estorno; matching de expansão usa 45d;
   recuperação de cobrança fecha em 60d. Métrica citada sem maturidade é
   métrica errada — ver Parte 2.
7. **Nem tudo é automático**: o placar aceita override manual (ex.: taxa de
   erro da aplicação) — linhas manuais vêm marcadas (`manualAtual`).
8. **Custo de IA** vem do AI Gateway (Vercel), em USD, líquido de desconto e
   excluindo usuários dev.

---

## Parte 2 — Definições oficiais (glossário da casa)

Use EXATAMENTE estas definições. Não invente variações.

### Receita e vendas

- **Receita bruta (gerencial)** — receita rateada por BU, ANTES de reembolso,
  por competência. É o "faturamento".
- **Receita líquida** = bruta − reembolsada.
- **Ticket médio** = receita bruta ÷ vendas.
- **Vendas novas × renovação** — aquisição × retenção; "Nova" = tudo que não é
  renovação (Self-Service, Inside Sales…).
- **BUs**: ONE, LABS, SKIP, SUMMIT (+ Consolidado = soma exata das 4; a
  invariante soma-das-BUs = Consolidado é validada).

### Reembolso (coorte por DIA DE COMPRA)

- **Taxa de reembolso** = reembolsos ÷ vendas (qtd) ou receita reembolsada ÷
  receita (R$), da MESMA coorte de compra.
- **Madura** = compra + 30d ≤ hoje (garantia encerrada). Taxa citável = madura.
  Período em curso SEMPRE subestima a taxa (coorte-defasada).
- **Chargeback/disputa** contam como reembolso (`is_reembolso` =
  refunded/chargeback/dispute).
- **Motivos**: classificação data-driven; sem ticket = "Não informado".
- Cohorts 1/7/15/30d = leading indicator (imaturo, tendência apenas).

### Renovação e NRR (coorte por VENCIMENTO — dt_encerramento)

- **Elegíveis / receita a vencer** — contratos cujo vencimento cai na janela.
- **Taxa de Renovação canônica** = receita renovada ÷ receita a vencer, da
  mesma safra de vencimento, **coortes maduras** (venc + 30d ≤ hoje).
  **Bruta** usa `is_renovado`; **líquida** usa `is_renovado_net` (renovou E não
  estornou).
- **NRR padrão da casa** = (retida líquida + expansão) ÷ receita a vencer
  madura. Sem termo de contração. Expansão = ticket novo > velho (matching 45d,
  estimativa). **GRR** = mesmo sem expansão.
- **Realização líquida** = dos que renovaram, quanto sobreviveu ao estorno.
- **Churn voluntário** = cliente cancelou antes da cobrança; **involuntário** =
  pagamento recusado; **Outro** = administrativo/sem info.
- **Recuperação de cobrança** — coorte por mês da 1ª recusa; recuperado =
  aprovou renovação do mesmo grupo em até 60d (por retentativa automática ou
  campanha).
- **Tendência semanal** = MM4 (média móvel de 4 semanas) de coortes fechadas.
- **NRR é métrica de coorte/período longo** — "NRR da semana" não é recorte
  usual; ofereça o anual + tendência semanal da taxa.
- SUMMIT não tem recorrência (sem renovação/NRR).

### Marketing

- **Atribuição de BU** = keyword (campanha→adset→anúncio, prioridade). Fonte
  única de BU de mídia.
- **Receita atribuída** = por ad_id resolvido nos trackings.
- **ROAS** = receita confirmada ÷ spend. **CPA** = spend ÷ vendas atribuídas.
- **Cobertura** = receita atribuída ÷ receita total de aquisição — cite junto.
- **Vazamento** = campanha da BU vendendo produto de outra BU (excl.
  upsell/créditos extras).
- **Qualidade por canal** (reembolso/renovação do tráfego) usa as MESMAS
  maturidades canônicas acima.

### Placar semanal

- **Semana** = FECHADA, segunda→domingo. Nunca a semana em curso.
- **Margem (fórmula da casa)** = (ativos × R$99 × 7/30) − (custo_gateway_usd ×
  R$5,00 × 1,0636 VAT), sobre a receita pro-rata.
- Cada linha do placar carrega a própria nota de metodologia — repita-a ao citar.

### Produto

- **LTV** — médio e mediano por cliente, com recorte quem renova × quem não.
- **Ciclos** — distribuição de contratos por nº de renovações (totais × ativos).
- **COGS de IA** — custo USD do AI Gateway (líquido de desconto, sem devs).

---

## Parte 3 — Uso correto do MCP adapta-data

### Conexão (se o MCP não estiver disponível)

Chave pessoal em https://scorecard-bus-ae3ea.goskip.app/mcp (aparece 1x):

```bash
claude mcp add adapta-data --transport http https://scorecard-bus-ae3ea.shrd00.internal.goskip.dev/backend/v1/mcp --header "Authorization: Bearer SUA_CHAVE"
```

Clientes com config JSON (Cursor, `.mcp.json`): `type:"http"`, mesma URL,
header `Authorization: Bearer SUA_CHAVE`. Docs vivas (markdown, sem auth):
`GET https://scorecard-bus-ae3ea.shrd00.internal.goskip.dev/backend/v1/mcp-docs`
— em dúvida sobre parâmetro/campo, leia antes de chutar.

### Regra de ouro: OFICIAL × EXPLORATÓRIO

Toda resposta traz `confianca`:

- **OFICIAL** — a tool executa AS MESMAS rotas da dashboard (allowlist +
  atribuição aplicadas). É o número da empresa; citável em reporte.
- **EXPLORATÓRIO** (`explore_nekt_sql`) — warehouse cru, SEM as regras da
  dash. Só quando nenhuma tool oficial cobre o cruzamento. Para métricas com
  definição oficial (NRR, taxas), NUNCA recalcule por conta própria no SQL.
  **Se divergir do oficial, declare: metodologias de coleta diferentes; a
  fonte oficial é a dashboard.**

### Qual tool usar

| Pergunta sobre… | Tool |
|---|---|
| Receita, vendas, ticket, funil, status, mix de pagamento | `get_scorecard` |
| Comparar BUs | `compare_bus` |
| Gasto, ROAS, CPA, canal/funil/influencer, vazamento (resumo), cobertura | `marketing_performance` |
| CPM/CPC/CTR e qualidade da venda por canal | `marketing_canal` |
| Anúncios/criativos (escala, profit por ad) | `marketing_ads` |
| Vazamento com nome de campanha | `vazamento_detalhe` (LENTO ~30-90s — avise) |
| Renovação, NRR, GRR, churn, recuperação | `renovacao_metrics` (ONE/LABS/SKIP) |
| Reembolso: taxa, motivos, prazo, chargeback, uso de IA | `reembolso_metrics` |
| Placar semanal (semana fechada, metas) | `placar` |
| LTV, ciclos, COGS de IA | `produto_metrics` |
| Allowlist produto→BU, produto fora das métricas | `list_produtos_bu` |
| Cruzamento não coberto | `explore_nekt_sql` (metodologia + star schema na descrição da tool) |

Datas `YYYY-MM-DD`. BUs: `Consolidado`, `ONE`, `LABS`, `SKIP`, `SUMMIT`.

### Receitas prontas

- **"Como foi a semana?"** → `get_scorecard` (from/to + prevFrom/prevTo) com
  ressalva de semana parcial; para reporte executivo, `placar` (semana fechada).
- **"Qual o NRR?"** → `renovacao_metrics {bu, periodo:"2026"}` → cite
  `nrr.valor` + `grr`, com a metodologia (coortes maduras por vencimento).
- **"Por que o reembolso subiu?"** → `reembolso_metrics` → compare campos
  `*Mad` entre períodos; olhe `motivosReemb`, `buckets` (uso de IA), `quebraSku`.
- **"ROAS de ontem?"** → `marketing_performance` from=to=ontem + cobertura.
- **Número estranho?** → 1º `list_produtos_bu` (produto deslinkado some das
  métricas); 2º horário do sync; 3º maturidade da coorte.

### Limites

- 1ª chamada pode demorar (cold start); durante o sync (~5min, 4x/dia) fica
  lento.
- `explore_nekt_sql`: só SELECT/WITH, 1 statement, só schema `nekt_gold`,
  500 linhas (agregue no SQL), cada consulta custa scan no Athena.
- Chave pessoal por usuário; revogável na tela API / MCP.
