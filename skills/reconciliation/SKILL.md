---
name: reconciliation
description: "Liquidação de repasses de cartão e vendas em dinheiro, conciliação de vendas com transações bancárias, e análise de custos de serviços no PontoAlto."
version: 0.1.0
---

# Liquidações e Conciliação

## Liquidar Repasses de Cartão e Vendas em Dinheiro

Após importar, criar os lançamentos de liquidação:
- **Cartão**: `list_settlements` para ver pendentes → `create_settlements` para criar em lote
- **Dinheiro**: `list_cash_sales` para ver vendas em dinheiro pendentes → `create_cash_settlements` para criar recebíveis

## Conciliar Vendas

Vendas do sistema (Feegow/outro) precisam ser conciliadas com transações bancárias para confirmar recebimento.

**Como verificar:** `list_sales(status=unreconciled)` + `list_sales(status=partially_reconciled)` → vendas sem match.

**Como agir:**
1. `analyze_unreconciled_sales` → encontra matches por scoring (valor, data, nome, método de pagamento)
2. Matches com score alto (≥80): criar sugestões `reconcile_sale`
3. Matches com score médio (50-79): apresentar ao gestor para decisão
4. Sem match: pode indicar importação faltante ou divergência

### Tenant multi-conta

Se o tenant tem mais de uma conta bancária, rode `list_bank_accounts` no diagnóstico inicial. É importante para:

- Saber em qual conta cair o recebimento esperado (Feegow PIX pode chegar na conta A, boleto em B)
- Interpretar por que um match falhou — venda bate em valor mas a transação candidata está em outra conta
- Filtrar `list_sales`/`list_transactions` por `bank_account_id` quando o volume de ambas as contas polui o scoring

Em tenant de conta única, pode pular.

### Drill-down (`get_transaction`)

Quando um match ≥80 do `analyze_unreconciled_sales` parecer suspeito (ex: valor igual mas nome divergente), use `get_transaction(id)` para ver o payload completo da transação candidata: `raw_description`, histórico de enriquecimento PIX (nome completo do pagador, CPF mascarado) e label atual. Confirma ou descarta o match antes de criar a sugestão.

## Cadastrar/Atualizar Custos de Serviços

Serviços e procedimentos precisam ter custo cadastrado para análise de rentabilidade.

**Como verificar:** `get_cost_analysis(view=by_service)` → serviços com margem zerada ou sem custo indicam cadastro incompleto.

**Como agir:**
1. Listar serviços sem custo atribuído
2. Informar o gestor quais procedimentos/produtos precisam de atualização de custo
3. Se há custos desatualizados: comparar com períodos anteriores para identificar variações
