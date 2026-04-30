---
description: "Atalho para vincular fornecedores a transações e ajustar datas de competência (DRE) no PontoAlto."
argument-hint: "[--local] [contexto livre]"
---

# PontoAlto — Fornecedores e Competência

Atalho para as etapas de fornecedores e competência. Use quando há despesas sem fornecedor vinculado ou lançamentos com competência incorreta afetando o DRE.

Responda em português. Use a skill `provider-management` para o fluxo detalhado e `financial-domain` para o contexto de domínio.

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
analyze_provider_payments(mês atual) → pagamentos agrupados por empresa/pessoa
list_providers                       → fornecedores já cadastrados
list_suggestions(status=pending)     → sugestões pendentes (não duplicar)
list_rules                           → regras de vinculação e competência existentes
```

Resumir: grupos sem fornecedor + grupos com match sugerido + transações com competência suspeita.

## Escolha do subfluxo

Apresentar via `AskUserQuestion`:

1. **Vincular fornecedores (Recomendado)** — ação rápida, matches de `analyze_provider_payments`
2. **Ajustar competência** — focar em lançamentos recorrentes (aluguel, seguros, parcelas)
3. **Ambos em sequência** — fornecedores primeiro, depois competência

## Vincular Fornecedores — Detalhe

1. `analyze_provider_payments` → grupos com/sem match
2. **Match com fornecedor existente (confidence ≥ 85)** — criar sugestões `link_provider` em lote via `bulk_create_suggestions` com `create_rule: true` (tipo `contains`)
3. **Empresa sem fornecedor cadastrado** — sugerir `create_provider` + `link_provider` no mesmo lote
4. **Padrão recorrente confirmado** — incluir sugestão `create_provider_linking_rule` para automatizar próximos meses

## Ajustar Competência — Detalhe

1. `list_transactions` filtrando por padrões recorrentes (aluguel, condomínio, seguros, parcelas de empréstimo). Use `amount_min` para focar em lançamentos de maior impacto e `exclude_internal_transfers=true` para remover ruído de transferências entre contas.
2. Identificar transações onde `competence_date` está ausente ou diferente do mês de competência real
3. Criar sugestões `set_competence_date` agrupadas por pattern (`transaction_ids: [...]`)
4. Para recorrências confirmadas, sugerir `create_competence_rule`

## Relatório Final

Reportar:
- Fornecedores vinculados (quantos e quais categorias)
- Fornecedores novos propostos (aguardando criação)
- Transações com competência ajustada
- Regras criadas (linking + competência)
- Sugestões pendentes restantes

## Regras de Ouro

- Sempre `list_rules` antes de criar sugestões de vinculação — evitar duplicata de pattern
- Priorizar por impacto monetário
- Competência é crítica para DRE — erros aqui distorcem o resultado do mês
