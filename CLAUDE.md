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
│   ├── report.md          # /pontoalto:report — relatório mensal
│   └── sale-source.md     # /pontoalto:sale-source — fonte de venda customizada (DSL + preview loop)
├── skills/
│   ├── financial-domain/
│   │   └── SKILL.md       # Confidence scale, contexto financeiro brasileiro, exceções
│   ├── categorization/
│   │   └── SKILL.md       # Fluxo automático, consulta e manual de categorização
│   ├── reconciliation/
│   │   └── SKILL.md       # Liquidações, conciliação de vendas, custos de serviços
│   ├── provider-management/
│   │   └── SKILL.md       # Competência e vinculação de fornecedores
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
- `categorization` — fluxos específicos de categorização (automático, consulta WhatsApp, manual)
- `reconciliation` — liquidações e conciliação detalhada
- `provider-management` — fornecedores e competência
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
