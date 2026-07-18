# adapta-data — Agent Skill

Skill para agentes de IA consumirem os **dados de negócio da Adapta** (ScoreCard
BUs) pelo servidor MCP `adapta-data`: receita, vendas, reembolso, renovação/NRR,
marketing, COGS de IA e placar semanal, por BU — com a metodologia oficial da casa.

## Instalar

Compatível com o padrão [Agent Skills](https://skills.sh) (Claude Code, Cursor,
Codex, Gemini CLI e outros):

```bash
npx skills add JBonfim99/adapta-data-skill
```

Depois de instalar a skill, conecte o MCP (precisa de uma chave pessoal, gerada
em https://scorecard-bus-ae3ea.goskip.app/mcp):

```bash
claude mcp add adapta-data --transport http https://scorecard-bus-ae3ea.shrd00.internal.goskip.dev/backend/v1/mcp --header "Authorization: Bearer SUA_CHAVE"
```

## O que a skill ensina

Três pilares:

1. **Captura** — como o dado nasce (Guru/Ads/Trackings → Nekt → ScoreCard,
   sync 4x/dia) e as melhores práticas que decorrem disso: allowlist de
   produtos, atribuição por keyword, cobertura, competência capada, coortes
   que maturam.
2. **Definições oficiais** — o glossário da casa com as fórmulas exatas:
   receita bruta/líquida, taxa de reembolso madura, Taxa de Renovação
   canônica, NRR/GRR padrão, churn voluntário×involuntário, ROAS/CPA/cobertura,
   margem do placar, COGS de IA.
3. **Uso correto do MCP** — qual das 12 tools usar, semântica
   OFICIAL × EXPLORATÓRIO (campo `confianca`), e receitas prontas.

## Estrutura

```
skills/
└── adapta-data/
    └── SKILL.md
```

Documentação viva do servidor MCP (markdown, sem auth):
`https://scorecard-bus-ae3ea.shrd00.internal.goskip.dev/backend/v1/mcp-docs`
