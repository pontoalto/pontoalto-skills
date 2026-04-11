---
description: "Atalho para categorizar transações pendentes no Ponto Alto (automático, consulta ou manual), sem passar pelo fluxo completo de fechamento."
argument-hint: "[--local] [contexto livre]"
---

# Ponto Alto — Categorização

Atalho para a etapa de categorização. Use quando há lançamentos sem categoria e o gestor quer resolver esse ponto específico.

Responda em português. Use a skill `categorization` para o fluxo detalhado e `financial-domain` para o contexto de domínio.

## MCP Server

Argumento recebido: `$ARGUMENTS`

- Sem `--local`: tools com prefixo `mcp__claude_ai_Ponto_Alto__` (produção)
- Com `--local`: tools com prefixo `mcp__pontoalto-local__` (desenvolvimento)

`--local` é flag. Qualquer outra palavra nos argumentos é contexto livre (ex: período específico, segmento de transações).

## Inicialização

1. `list_tenants` — se 2+, perguntar qual usar via `AskUserQuestion`; se 1, usar automaticamente
2. Confirmar conexão: nome do tenant + organização
3. Ir **direto** ao dimensionamento — não mostrar checklist do fluxo completo

## Dimensionamento (sempre executar primeiro)

```
transaction_stats(mês atual)         → count_uncategorized, valor total sem categoria
list_rules                           → regras existentes para não duplicar
list_suggestions(status=pending)     → sugestões pendentes para não duplicar
```

Se `count_uncategorized == 0`: reportar "Nenhuma transação pendente de categorização este mês" e encerrar.

## Escolha do fluxo

Apresentar ao gestor via `AskUserQuestion` (3 opções):

1. **Automático (Recomendado)** — `analyze_uncategorized` + `suggest_category` + `bulk_create_suggestions` com `create_rule: true` em todos os grupos
2. **Consulta WhatsApp** — listar transações em texto plano pronto para copiar/colar (pagamentos PIX para pessoas físicas, despesas sem pattern claro)
3. **Manual guiado** — gestor dita a categoria de cada grupo, Claude cria as sugestões via `create_suggestion` modo batch (rodar `list_categories` primeiro para mapear nome → `category_id`)

Executar o fluxo escolhido conforme a skill `categorization`. Ao final, reportar:
- Total categorizado por categoria
- Quantidade de regras criadas
- Patterns ambíguos onde a regra **não** foi criada (com motivo)
- Sugestões pendentes restantes

## Regras de Ouro

- Uma sugestão por grupo/pattern (`transaction_ids: [...]`)
- Sempre `create_rule: true` + `rule_type: "contains"`, exceto nos casos de ambiguidade documentados em `categorization` § Princípio 2
- Priorizar por impacto monetário (maiores valores primeiro)
- Se o gestor não responder em uma consulta WhatsApp: não bloquear — reportar como "aguardando resposta" e encerrar
