---
description: "Atalho para Contas a Pagar/Receber no PontoAlto: agendar novas contas, marcar como pago e cancelar agendadas (via sugestões)."
argument-hint: "[--local] [contexto livre]"
---

# PontoAlto — Contas a Pagar/Receber

Atalho para a gestão da agenda de contas a pagar/receber (Bills). Use quando o gestor quer revisar vencimentos, agendar contas novas (com ou sem recorrência), marcar contas pagas e limpar série recorrente obsoleta.

Responda em português. Use a skill `bills-management` para o fluxo detalhado e `financial-domain` para contexto de domínio.

## MCP Server

Argumento recebido: `$ARGUMENTS`

- Sem `--local`: tools com prefixo `mcp__claude_ai_Ponto_Alto__` (produção)
- Com `--local`: tools com prefixo `mcp__pontoalto-local__` (desenvolvimento)

## Inicialização

1. `list_tenants` — escolher via `AskUserQuestion` se 2+, automático se 1
2. Confirmar conexão: nome do tenant + organização
3. Ir direto ao diagnóstico

## Diagnóstico

```
get_bills_aging                       → totais por bucket (overdue/today/next7/30/60 + total)
list_bills(overdue=true)              → contas vencidas (prioridade máxima)
list_bills(aging_bucket=today)        → vencendo hoje
list_bills(aging_bucket=next7days)    → próximos 7 dias
list_suggestions(status=pending)      → sugestões pendentes (não duplicar)
```

Reportar resumo:

```
📋 Contas a Pagar/Receber — [Tenant]
┌────────────────────┬──────────┬────────────────┐
│ Bucket             │ Quantos  │ Valor          │
├────────────────────┼──────────┼────────────────┤
│ Vencidas           │ N        │ R$ X,XX        │
│ Hoje               │ N        │ R$ X,XX        │
│ Próximos 7 dias    │ N        │ R$ X,XX        │
│ Próximos 30 dias   │ N        │ R$ X,XX        │
│ Próximos 60 dias   │ N        │ R$ X,XX        │
└────────────────────┴──────────┴────────────────┘
```

## Escolha do subfluxo

Apresentar via `AskUserQuestion`:

1. **Revisar vencidas (Recomendado)** — focar em `aging_bucket=overdue` e propor `mark_bill_as_paid` para as que já foram quitadas via extrato
2. **Agendar contas novas** — receber lista do gestor e criar sugestões `create_bill` (single ou em série recorrente)
3. **Limpar série recorrente** — identificar contas duplicadas/canceladas via `recurring_only=true` e propor `cancel_bill(scope=series)`
4. **Ambos em sequência** — vencidas primeiro, depois cadastro de novas

## Marcar como Pago — Detalhe

Para cada bill vencida:

1. `list_transactions(date_from=due_date-15, date_to=due_date+15, amount_min=valor, amount_max=valor, type=debit|credit conforme bill type)` — buscar candidatos do extrato
2. Se houver match único e claro: criar sugestão `mark_bill_as_paid` com `bill_id`, `bank_account_id`, `transaction_id` (suggestable_type=bill, suggestable_id=bill_id)
3. Se a bill foi paga em Caixa físico e não tem extrato: criar sugestão com `payment_date` no lugar de `transaction_id` (apenas se `bank_account_id` aponta para conta tipo Caixa)
4. Se não há match plausível: deixar para o gestor decidir

> Nunca proponha `payment_date` em conta bancária — só Caixa. Em conta corrente/poupança/operadora, a Transaction precisa vir do extrato.

## Agendar Contas Novas — Detalhe

1. Pedir ao gestor a lista de contas (tipo, valor, descrição, vencimento, opcional: categoria, fornecedor, recorrência)
2. `list_bills(recurring_only=true)` — verificar se já existe série semelhante (evitar duplicar)
3. `list_rules` + `list_categorization_rules` — sugerir `category_id` quando houver regra ativa que case com a descrição
4. Para cada nova conta: criar sugestão `create_bill` com `suggestable_type="transaction"` apontando para uma transação representativa (ex.: última fatura paga do mesmo fornecedor)
5. **Recorrência**: se o gestor confirmar série recorrente, incluir `has_recurrence: true`, `recurrence_mode: "installments"|"always"`, `recurrence_frequency`, `recurrence_count`

## Cancelar Bill — Detalhe

1. Listar via `list_bills(status=scheduled, recurring_only=true, search=...)` para identificar série obsoleta
2. Propor `cancel_bill` com:
   - `scope: "single"` para cancelar apenas um vencimento isolado
   - `scope: "series"` para encerrar toda a série recorrente (preserva pagas)
3. Confirmar com o gestor antes de criar a sugestão — cancelar série é irreversível

## Relatório Final

Reportar:
- Contas marcadas como pagas (quantas, soma)
- Contas novas agendadas (single + parcelado + sempre-recorrente)
- Séries canceladas
- Sugestões pendentes restantes

## Regras de Ouro

- Sempre `get_bills_aging` antes de listar — dimensiona o problema
- Priorize por impacto monetário e vencimento (vencidas primeiro, depois `today`, depois `next7days`)
- Para marcar como pago, priorize match com Transaction do extrato — Caixa só quando bill é dinheiro físico
- Bills agendadas com `competence_date` afetam o DRE quando efetivadas — verificar se faz sentido fiscalmente
- Cancelar série é irreversível — confirmar antes
