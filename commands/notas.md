---
description: "Notas fiscais do DRE Contábil no PontoAlto: classificar itens de notas emitidas e recebidas e vincular notas recebidas aos pagamentos do extrato, via sugestões na inbox."
argument-hint: "[--local] [YYYY-MM] [classificar|pagamentos]"
---

# PontoAlto — Notas Fiscais

Atalho para as notas fiscais que alimentam o DRE Contábil. Use quando o gestor quer resolver itens de nota sem classificação ou notas recebidas sem pagamento vinculado.

Responda em português. Use a skill `fiscal-classification` para o fluxo detalhado e `financial-domain` para o contexto de domínio.

## MCP Server

Argumento recebido: `$ARGUMENTS`

- Sem `--local`: tools com prefixo `mcp__claude_ai_Ponto_Alto__` (produção)
- Com `--local`: tools com prefixo `mcp__pontoalto-local__` (desenvolvimento)

**Período**: se o argumento contiver `YYYY-MM`, usar a competência desse mês (`date_from`/`date_to`). Sem período, trabalhar com todas as pendentes — classificação pendente não é presa a um mês.

**Subfluxo**: `classificar` ou `pagamentos` nos argumentos pula a escolha.

## Inicialização

1. `list_tenants` — escolher via `AskUserQuestion` se 2+, automático se 1
2. Confirmar conexão: nome do tenant + organização
3. Ir direto ao dimensionamento

## Dimensionamento

```
analyze_fiscal_classification(date_from?, date_to?)                    → itens pendentes (summary + groups)
list_fiscal_documents(role=destinatario, payment=sem_pagamento, limit=50) → recebidas sem pagamento (total)
list_suggestions(status=pending)                                        → não duplicar
```

Se não houver itens pendentes nem notas recebidas sem pagamento: reportar "Notas fiscais em dia" e encerrar.

Reportar antes de agir:

```
🧾 Notas Fiscais — [Tenant] [· MM/YYYY]
Itens a classificar   N (R$ ...) em G grupos
Recebidas sem pagamento  N (R$ ...)
Sugestões de nota já na inbox  N
```

## Escolha do subfluxo

Via `AskUserQuestion`:

1. **Classificar itens (Recomendado quando há pendentes)** — grupos do `analyze_fiscal_classification` → `classify_fiscal_items` com regra para contrapartes recorrentes
2. **Vincular pagamentos** — `list_fiscal_documents(payment=sem_pagamento)` → `link_fiscal_payment` para candidatos confirmados ou únicos
3. **Os dois, nessa ordem**

## Classificar — Detalhe

1. `list_categories` → nome → `category_id`
2. Para cada grupo (maior valor primeiro): usar `suggested_categories` conforme a skill; sem pista, juntar os grupos numa pergunta ao gestor (`AskUserQuestion`, até 4 por vez) em vez de chutar
3. Confirmar com o gestor o lote antes de criar (quantas sugestões, valor total, regras que serão criadas)
4. `create_suggestion` em modo batch, `suggestable_type="fiscal_document"`, `suggestable_id = fiscal_document_id`: uma sugestão com `create_rule: true` por grupo recorrente (na nota de maior valor); uma por nota nos casos pontuais

## Vincular Pagamentos — Detalhe

1. `list_fiscal_documents(role=destinatario, payment=sem_pagamento)` → por nota, olhar `payment_candidates`
2. Candidato com `confirmed_by` ou único candidato → sugestão `link_fiscal_payment`
3. Vários candidatos sem confirmação → listar para o gestor (data, valor, descrição) e propor só o escolhido
4. Sem candidato → reportar como "sem pagamento no extrato" (cartão de crédito, boleto agrupado, ainda não pago)

## Saída final

- Sugestões criadas por tipo (Classificar Nota / Nota × Pagamento) e valor coberto
- Regras que serão criadas ao aprovar
- Grupos que ficaram sem decisão (com o motivo)
- Lembrete: aprovar na inbox; o efeito aparece em `get_reports(report=dre_contabil)`

## Regras de Ouro

- Só sugestões — o plugin não aprova
- Não mexer no que o sistema já classificou (`source` regra/manual/pagamento) nem nas notas já vinculadas
- Nota de pessoa física (paciente) é "Pessoa física": nunca pedir ou repassar nome/CPF
- Priorizar por valor
