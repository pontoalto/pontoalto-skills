# pontoalto

Plugin do assistente financeiro PontoAlto para uso com Claude Code.

## Estrutura

```
pontoalto-skills/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   ├── manager.md         # /pontoalto:manager — fluxo completo
│   ├── categorize.md      # /pontoalto:categorize — só categorização
│   ├── reconcile.md       # /pontoalto:reconcile — liquidações + conciliação
│   ├── providers.md       # /pontoalto:providers — fornecedores + competência
│   ├── bills.md           # /pontoalto:bills — Contas a Pagar/Receber (agenda, marcar pago, cancelar)
│   ├── obras.md           # /pontoalto:obras — obras de incorporadora (cadastro, atribuição, resultado)
│   ├── costs.md           # /pontoalto:costs — custos e margem por item (+ set_item_cost)
│   ├── report.md          # /pontoalto:report — relatório mensal
│   └── sale-source.md     # /pontoalto:sale-source — fonte de venda customizada (DSL + preview loop)
├── skills/
│   ├── financial-domain/
│   │   └── SKILL.md       # Confidence scale, contexto financeiro brasileiro, exceções
│   ├── categorization/
│   │   └── SKILL.md       # Fluxo automático, consulta e manual de categorização + split
│   ├── reconciliation/
│   │   └── SKILL.md       # Liquidações e conciliação de vendas
│   ├── cost-analysis/
│   │   └── SKILL.md       # Custo do item vendido, margem, markup, base venda vs produção, set_item_cost
│   ├── provider-management/
│   │   └── SKILL.md       # Competência e vinculação de fornecedores
│   ├── bills-management/
│   │   └── SKILL.md       # Bills: agendamento, marcação como pago, cancelamento (via sugestões)
│   ├── project-management/
│   │   └── SKILL.md       # Obras: cadastro (escrita direta) + atribuição via link_project
│   └── sale-sources/
│       └── SKILL.md       # DSL de fontes customizadas, preview iterativo, exceção à inbox
├── README.md              # Orientado ao gestor final
└── CLAUDE.md              # Este arquivo — orientado ao desenvolvedor
```

## Commands

Namespaceados automaticamente pelo `name` do plugin (`pontoalto`):

- `/pontoalto:manager [--local]` — fluxo completo do mês com checklist de status
- `/pontoalto:categorize [--local]` — só categorização (automático, consulta WhatsApp ou manual)
- `/pontoalto:reconcile [--local]` — liquidações (cartão/dinheiro) + conciliação de vendas
- `/pontoalto:providers [--local]` — vinculação de fornecedores + ajuste de competência
- `/pontoalto:bills [--local]` — Contas a Pagar/Receber (agenda, marcar como pago, cancelar série)
- `/pontoalto:obras [--local] [obra]` — obras de incorporadora: cadastro, atribuição de lançamentos, resultado por empreendimento
- `/pontoalto:costs [--local] [YYYY-MM]` — análise de custos: itens sem custo cadastrado (e a sugestão que fecha a lacuna), margem por tipo e por fornecedor, itens com pior margem
- `/pontoalto:report [--local] [YYYY-MM]` — relatório mensal (DRE, orçado vs realizado, custos)
- `/pontoalto:sale-source [--local] [nome|key]` — monta/ajusta fonte de venda customizada via DSL + preview iterativo (admin-only)

> **Naming**: os arquivos em `commands/` **não** devem ser prefixados com o nome do plugin. O Claude Code faz o namespacing automaticamente via `{plugin-name}:{command-name}`. Prefixar o arquivo causa redundância (ex: `/pontoalto:pontoalto-manager`).

## MCP Servers

Os commands dependem dos MCP servers do PontoAlto:

| Flag        | MCP Server               | Ambiente              |
|-------------|--------------------------|-----------------------|
| `--local`   | `pontoalto-local`        | Desenvolvimento local |
| _(sem flag)_| `claude_ai_Ponto_Alto`   | Produção (pontoalto.app) |

As instruções dos MCP servers (convenções de R$, datas, modelo de escrita via sugestões, actions disponíveis) vêm diretamente do próprio servidor. As skills do plugin complementam com o que o MCP não fornece:

- `financial-domain` — escala de confidence, contexto financeiro brasileiro (regimes, DRE, adquirente de cartão), exceções ao modelo de sugestões
- `categorization` — fluxos específicos de categorização (automático, consulta WhatsApp, manual) e divisão de lançamentos (`split_transaction`) quando um pagamento cobre naturezas diferentes
- `reconciliation` — liquidações e conciliação detalhada
- `cost-analysis` — custo do item vendido (`SaleItem` × `CostItem`, casamento por nome), as sete views do `get_cost_analysis`, base de data venda vs produção, margem bruta vs de contribuição, markup como multiplicador, e a action `set_item_cost` (cadastra item + vigência pela inbox, com âncora `cost_item` ou `sale_item` conforme o item já exista)
- `provider-management` — fornecedores e competência
- `bills-management` — Contas a Pagar/Receber: agendamento (single/recorrente), marcação como pago (via extrato ou Caixa) e cancelamento de séries — sempre via sugestões na inbox
- `project-management` — obras de incorporadora. Obra→Etapa é hierarquia; Tipo de Custo é dimensão global (não é nível da árvore). Cadastro é escrita direta admin-only (`save_project`, `save_cost_type`, `delete_project`), mas **atribuir** obra a lançamento passa pela inbox via action `link_project`
- `sale-sources` — DSL de fontes customizadas, loop de preview, exceção à inbox (escrita direta em `save_sale_source_definition` / `delete_sale_source_definition`)

## Desenvolvimento local

Para iterar no plugin com MCP real, crie um `.mcp.json` no root do repo (gitignored) apontando para o repo Laravel do PontoAlto:

```json
{
  "mcpServers": {
    "pontoalto-local": {
      "command": "php",
      "args": ["artisan", "mcp:start", "financial"],
      "cwd": "/caminho/absoluto/para/cdc-dashboard",
      "env": { "MCP_USER": "1" }
    }
  }
}
```

Dev loop: `cd pontoalto-skills && claude --plugin-dir .` → edita SKILL.md ou command → `/reload-plugins` → testa `/pontoalto:manager --local`.

> A flag `--plugin-dir .` é necessária para o Claude Code carregar este diretório como plugin durante desenvolvimento. Sem ela, os commands `/pontoalto:*` não ficam disponíveis.

## Convenções

- Valores em R$ formato brasileiro (R$ 1.234,56)
- Datas DD/MM/YYYY, períodos YYYY-MM-DD nos parâmetros das tools
- Respostas sempre em português
- Priorizar por impacto monetário (maiores valores primeiro)
