---
name: reconciliation
description: "Liquidação de repasses de cartão e vendas em dinheiro e conciliação de vendas com transações bancárias no PontoAlto."
version: 0.3.0
---

# Liquidações e Conciliação

## Liquidar Repasses de Cartão e Vendas em Dinheiro

Após importar, criar os lançamentos de liquidação:
- **Cartão**: `list_settlements` para ver pendentes → `create_settlements` para criar em lote
- **Dinheiro**: `list_cash_sales` para ver vendas em dinheiro pendentes → `create_cash_settlements` para criar recebíveis

## Conciliar Vendas

Vendas do sistema origem (ERP, PDV ou outro) precisam ser conciliadas com transações bancárias para confirmar recebimento.

**Como verificar:** `list_sales(status=unreconciled)` + `list_sales(status=partially_reconciled)` → vendas sem match.

**Como agir:**
1. `analyze_unreconciled_sales` → encontra matches por scoring (valor, data, nome, método de pagamento)
2. Matches com score alto (≥80): criar sugestões `reconcile_sale` (ver § Criar as sugestões)
3. Matches com score médio (50-79): apresentar ao gestor para decisão
4. Sem match: pode indicar importação faltante ou divergência

### Criar as sugestões

`create_suggestion` em modo batch, uma sugestão por par venda × lançamento. `bulk_create_suggestions` **não** serve — ele só cria categorização.

```json
{
  "suggestions": [
    {
      "type": "reconcile_sale",
      "suggestable_type": "sale",
      "suggestable_id": <sale_id>,
      "action": "reconcile_sale",
      "action_params": {"sale_id": <sale_id>, "transaction_id": <transaction_id>, "type": "revenue", "amount": <valor da venda coberto pelo lançamento>},
      "confidence": <score>,
      "reasoning": "PIX R$ 728,00 em 20/10, mesmo valor e mesma data da venda"
    }
  ]
}
```

- `suggestable_type` é sempre `sale`, com `suggestable_id` igual ao `sale_id`. O servidor aceita **uma** `reconcile_sale` pendente por venda: em venda paga em duas formas (PIX + cartão), o segundo par entra como `skipped_duplicates` — crie depois que o primeiro for aprovado
- `type: "revenue"` para recebimento da venda; `"cost"` só para pagamento de custo ligado à venda
- `amount` é quanto daquele lançamento cobre a venda — num PIX que paga duas vendas, cada sugestão leva só a parte da sua venda

### Depois de aprovar

Aprovada a sugestão, a venda ganha o vínculo e o status é recalculado:

- **Totalmente Reconciliado** — o que foi vinculado cobre o valor recebido da venda
- **Parcialmente Reconciliado** — falta parte. Típico de venda paga em duas formas (PIX + cartão) com só uma vinculada: procure o restante em `analyze_unreconciled_sales` antes de dar a venda por resolvida

Reaprovar um par que já estava vinculado não duplica o vínculo, só acerta o status.

### Tenant multi-conta

Se o tenant tem mais de uma conta bancária, rode `list_bank_accounts` no diagnóstico inicial. É importante para:

- Saber em qual conta cair o recebimento esperado (PIX pode chegar na conta A, boleto em B)
- Interpretar por que um match falhou — venda bate em valor mas a transação candidata está em outra conta
- Filtrar `list_sales`/`list_transactions` por `bank_account_id` quando o volume de ambas as contas polui o scoring

Em tenant de conta única, pode pular.

### Drill-down (`get_transaction`)

Quando um match ≥80 do `analyze_unreconciled_sales` parecer suspeito (ex: valor igual mas nome divergente), use `get_transaction(id)` para ver o payload completo da transação candidata: `raw_description`, histórico de enriquecimento PIX (nome completo do pagador, CPF mascarado) e label atual. Confirma ou descarta o match antes de criar a sugestão.

## Custos e Margem

Fora do escopo desta skill. Custo do item vendido, margem, markup e itens sem custo cadastrado estão na skill `cost-analysis` — inclusive a base de data (venda vs produção), que não é a mesma coisa que conciliação.
