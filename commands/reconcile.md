---
description: "Atalho para liquidações de repasses (cartão/dinheiro) e conciliação de vendas com transações bancárias no Ponto Alto."
argument-hint: "[--local] [contexto livre]"
---

# Ponto Alto — Liquidações e Conciliação

Atalho para as etapas de liquidação e conciliação de vendas. Use para recebimentos de cartão, vendas em dinheiro e para casar vendas do sistema origem (ERP, PDV ou outro) com entradas no extrato.

Responda em português. Use a skill `reconciliation` para o fluxo detalhado e `financial-domain` para o contexto de domínio.

## MCP Server

Argumento recebido: `$ARGUMENTS`

- Sem `--local`: tools com prefixo `mcp__claude_ai_Ponto_Alto__` (produção)
- Com `--local`: tools com prefixo `mcp__pontoalto-local__` (desenvolvimento)

## Inicialização

1. `list_tenants` — escolher via `AskUserQuestion` se 2+, automático se 1
2. Confirmar conexão: nome do tenant + organização
3. Ir direto ao diagnóstico

## Diagnóstico (sempre executar primeiro)

```
list_bank_accounts                       → (só se o tenant for multi-conta) mapa das contas disponíveis
list_settlements(status=pending)         → repasses de cartão pendentes
list_cash_sales(unreconciled=true)       → vendas em dinheiro pendentes
list_sales(status=unreconciled)          → vendas sem conciliação
list_sales(status=partially_reconciled)  → vendas com conciliação parcial
```

Resumir: quantas pendências em cada categoria e valor total.

## Escolha do subfluxo

Apresentar via `AskUserQuestion` (até 3 opções, baseado no que tem pendente):

1. **Liquidar repasses de cartão** — se `list_settlements` > 0 → `create_settlements` (escrita direta, sem sugestão)
2. **Liquidar vendas em dinheiro** — se `list_cash_sales` > 0 → `create_cash_settlements` (escrita direta, sem sugestão)
3. **Conciliar vendas** — se `list_sales(unreconciled)` > 0 → `analyze_unreconciled_sales` + sugestões `reconcile_sale`

Executar um subfluxo por vez, reportar resultado, perguntar se quer continuar para o próximo.

## Conciliar Vendas — Detalhe

1. `analyze_unreconciled_sales` → retorna matches com score (valor + data + nome + método)
2. **Score ≥ 80** — criar sugestões `reconcile_sale` em lote via `bulk_create_suggestions`
3. **Score 50-79** — apresentar ao gestor (pode ser texto WhatsApp ou tabela) para decisão
4. **Sem match** — listar vendas sem candidato, sugerir investigar imports faltantes ou divergências de valor

Reportar ao final:
- Vendas conciliadas automaticamente (score ≥ 80)
- Vendas apresentadas ao gestor (50-79)
- Vendas sem match (< 50) — com motivo provável

## Regras de Ouro

- Liquidações de cartão/dinheiro são **escrita direta**, não criam sugestão
- Conciliação de vendas passa por sugestão (`reconcile_sale`)
- Nunca categorizar repasse de adquirente de cartão como receita — é transferência
- Priorizar por valor (maiores primeiro)
