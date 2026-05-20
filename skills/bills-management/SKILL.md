---
name: bills-management
description: "Gestão de Contas a Pagar/Receber (Bills) no PontoAlto: agendar (com recorrência), marcar como pago (via extrato ou Caixa) e cancelar agendadas/séries — sempre via sugestões na inbox."
version: 0.1.0
---

# Contas a Pagar/Receber (Bills)

`Bill` é uma conta agendada (intenção). Quando efetivada vira `Transaction` (fato contábil). O assistente IA pode operar a agenda inteira via sugestões — o gestor aprova na inbox.

**Como verificar:** `get_bills_aging` → totais por bucket. Se `overdue.count > 0` há contas vencidas (prioridade máxima).

## Princípios

1. **Sugestões, nunca escrita direta** — `create_bill`, `mark_bill_as_paid` e `cancel_bill` SEMPRE viram sugestão na inbox. O gestor aprova.
2. **Suggestable correto**:
   - `create_bill` → `suggestable_type="transaction"` (transação âncora, ex.: última fatura paga do mesmo fornecedor) — alinha com o padrão de outras "create" actions.
   - `mark_bill_as_paid` e `cancel_bill` → `suggestable_type="bill"` e `suggestable_id=bill_id`. Isso dá dedup natural e descrição correta no Inbox.
3. **Priorizar vencidas** — Bills com `due_date < hoje` e `status=scheduled` são prioridade. Use `list_bills(overdue=true)` ou `list_bills(aging_bucket="overdue")`.
4. **Match com extrato é fonte da verdade** — Para marcar como pago, prefira `transaction_id` (linkar a Transaction do OFX/operadora). `payment_date` (criar nova Transaction) só é permitido para conta **Caixa**.
5. **Recorrência cuidadosa** — `recurrence_mode="always"` cria bills até dez/ano corrente; `installments` cria N parcelas. Antes de criar série, verifique `list_bills(recurring_only=true)` para evitar duplicar.

## Fluxo 1: Diagnóstico Inicial

```
get_bills_aging                                    → totais por bucket
list_bills(overdue=true, limit=50)                 → vencidas (se aging mostrar)
list_bills(aging_bucket="today")                   → vencendo hoje
list_suggestions(status=pending, suggestable_type=bill) → pendentes não duplicar
```

Reportar resumo conciso (formato tabela ASCII). Priorizar conta com maior impacto monetário.

## Fluxo 2: Marcar Bills como Pagas

Para cada bill vencida com alta probabilidade de já estar paga:

1. `list_transactions(date_from=due_date-15, date_to=due_date+15, amount_min=valor-0.01, amount_max=valor+0.01, type=debit/credit)` — buscar match no extrato
2. Avaliar candidatos por proximidade de data e similaridade textual com `bill.description`
3. **Match único e claro (≥1 candidato com data próxima)**: criar sugestão `mark_bill_as_paid`:
   ```json
   {
     "type": "general",
     "suggestable_type": "bill",
     "suggestable_id": <bill_id>,
     "action": "mark_bill_as_paid",
     "action_params": {
       "bill_id": <bill_id>,
       "bank_account_id": <conta da transação>,
       "transaction_id": <transaction_id>
     },
     "confidence": 90,
     "reasoning": "Bill #X vence em DD/MM, R$ N. Transação #Y de mesma conta, valor exato, data ±N dias."
   }
   ```
4. **Sem match no extrato + conta Caixa**: criar sugestão usando `payment_date` (em vez de `transaction_id`). Confidence baixa (50-70).
5. **Sem match e conta bancária**: não propor. Avisar o gestor — pode ser bill obsoleta ou pagamento ainda não importado.

## Fluxo 3: Agendar Bills Novas

Gestor traz uma lista (whatsapp, planilha) com novas contas. Para cada item:

1. Confirmar tipo (`payable`/`receivable`), `amount`, `description`, `due_date`
2. Se categoria não foi informada: sugerir via `suggest_category(description)`
3. `list_bills(recurring_only=true)` — verificar se já há série similar (evitar duplicar)
4. Criar sugestão `create_bill`:
   ```json
   {
     "type": "general",
     "suggestable_type": "transaction",
     "suggestable_id": <transaction âncora>,
     "action": "create_bill",
     "action_params": {
       "type": "payable",
       "amount": 1500.00,
       "description": "Aluguel sede",
       "due_date": "2026-06-10",
       "category_id": 42,
       "provider_id": 7,
       "has_recurrence": true,
       "recurrence_mode": "always"
     },
     "confidence": 95,
     "reasoning": "Gestor confirmou aluguel mensal sempre dia 10. Provider Imobiliária X já cadastrado."
   }
   ```

> **Âncora**: a transação âncora não é o alvo da ação — é só um suggestable necessário pelo modelo de sugestões. Use a última Transaction paga do mesmo fornecedor/categoria. O card do Inbox usa `action_detail` (verde) — a âncora é ignorada visualmente.

## Fluxo 4: Cancelar Bills/Séries

1. `list_bills(status=scheduled, recurring_only=true, search=...)` — identificar série obsoleta
2. Para vencimento isolado (gestor confirma que essa conta específica não acontece mais):
   ```json
   {
     "action": "cancel_bill",
     "action_params": {"bill_id": <id>, "scope": "single"}
   }
   ```
3. Para encerrar série inteira (ex.: contrato encerrado):
   ```json
   {
     "action": "cancel_bill",
     "action_params": {"bill_id": <bill âncora>, "scope": "series"}
   }
   ```
   `scope="series"` cancela todas as bills agendadas com mesma `recurrence_series_id` — preserva pagas.

## Casos Especiais

- **Bill paga em Caixa físico não importada**: `mark_bill_as_paid` com `payment_date` e `bank_account_id` apontando para conta tipo Caixa. Em outra conta o sistema rejeita.
- **Bill com `competence_date` errada**: depois de marcada como paga, ajuste competência na Transaction resultante via `set_competence_date` (sugestão sobre `suggestable_type=transaction`).
- **Categoria incompatível**: bill `payable` exige categoria `expense` ou `both`. Bill `receivable` exige `revenue` ou `both`. O servidor rejeita.

## Quando NÃO Usar Bills

- Lançamento já no extrato e categorizado: opera direto em `Transaction` via `categorize_transaction`/`link_provider`. Não crie bill retroativa.
- Repasse de cartão / liquidação de dinheiro: use `create_settlements`/`create_cash_settlements` — fluxo separado de bills.

## Regras de Ouro

- Sempre `get_bills_aging` antes de listar — dimensiona o problema
- Para marcar como pago, prefira `transaction_id` (extrato é fonte da verdade); `payment_date` só em Caixa
- Cancelar série é irreversível — confirme com o gestor
- `suggestable_type` certo: `bill` para mark/cancel, `transaction` para create_bill
- Bills agendadas com `competence_date` mudam o DRE no mês da competência (não no de vencimento)
