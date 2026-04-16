---
name: supplier-management
description: "Ajuste de datas de competência e vinculação de fornecedores a transações no PontoAlto: analyze_supplier_payments, link_provider, create_provider, set_competence_date."
version: 0.2.0
---

# Competência e Fornecedores

## Ajustar Datas de Competência

A data de competência determina em qual mês o lançamento aparece no DRE. Muitas transações têm competência diferente da data do extrato (ex: aluguel pago dia 5 refere ao mês anterior).

**Como verificar:** `list_transactions` com filtros de período — procurar transações onde `competence_date` está ausente ou parece incorreta.

**Como agir:**
1. Identificar transações que precisam de ajuste (recorrentes como aluguel, seguros, parcelas)
2. Criar sugestões via `create_suggestion` (type=set_competence_date)
3. Se há padrões recorrentes: sugerir `create_competence_rule`

## Vincular Fornecedores

Transações de despesa precisam estar vinculadas ao fornecedor correto para análise de custos.

**Como verificar:** `analyze_supplier_payments` → mostra pagamentos agrupados por empresa/pessoa, com matches a fornecedores conhecidos.

**Drill-down raw:** quando o agrupamento do `analyze_supplier_payments` não bate com o esperado (grupos grandes demais, pattern fragmentado) ou quando precisa focar em valores altos: `list_transactions(type=debit, has_provider=false, amount_min=<valor>, date_from=..., date_to=...)` lista as despesas órfãs sem agrupar.

**Como agir:**
1. Para matches com fornecedor existente: `bulk_create_suggestions` com `supplier_groups=[...]`. Cria automaticamente **cadeia** (regra de vinculação + link) para cada grupo com `create_rule: true` — o par aparece na inbox como card único e é aceito/rejeitado atomicamente.
2. Para empresas sem fornecedor cadastrado: usar `create_suggestion_chain` encadeando `create_provider` → `create_provider_linking_rule` → `link_provider` (ver § Criar Fornecedor Novo em Cadeia abaixo).
3. Se há padrões recorrentes com fornecedor existente: o `create_rule: true` do `bulk_create_suggestions` já resolve.

## Criar Fornecedor Novo em Cadeia

Quando `analyze_supplier_payments` mostra um grupo de pagamentos sem fornecedor cadastrado, e você quer cadastrar + vincular + criar regra em uma operação atômica: use `create_suggestion_chain`.

**Fluxo:**
1. `list_providers` → confirmar que o fornecedor não existe (case-insensitive)
2. Montar cadeia de 3 steps:
   - `create_provider` (step 0) — `name` (nome da empresa como aparece no extrato)
   - `create_provider_linking_rule` (step 1) — pattern do grupo, `rule_type: "contains"`, `provider_id: "$chain.0.created_id"`
   - `link_provider` (step 2) — `transaction_ids: [...]` do grupo, `provider_id: "$chain.0.created_id"`
3. Chamar `create_suggestion_chain(steps=[...])`
4. Reportar: "Cadeia criada — gestor aprova no inbox e os 3 passos executam atomicamente (cadastro + regra + vinculação)."

**suggestable âncora:** use a primeira transação do grupo como `suggestable_id` nos 3 steps (convenção do `bulk_create_suggestions`).

**Exemplo completo:** ver skill `financial-domain` § Encadeamento de Actions.

## Criar Regra Isolada (Fornecedor Já Existe e Já Vinculado)

Caso específico: o fornecedor já está cadastrado, transações anteriores já estão vinculadas manualmente, mas **não há regra** que automatize vínculos futuros. Aqui não cabe `bulk_create_suggestions` (sem transações pendentes para linkar) nem cadeia (nada a criar antes). Use `create_suggestion` singular com `action: "create_provider_linking_rule"`.

**Quando aplicar:**
- `list_rules(type=provider)` mostra que não existe regra para o provider
- `list_transactions` mostra vínculos manuais recorrentes com o mesmo provider (pattern estável na description)
- Próxima importação tende a trazer o mesmo padrão e você quer automatizar

**Payload:**
```json
{
  "suggestable_type": "transaction",
  "suggestable_id": <tx_id_âncora representativa do padrão>,
  "action": "create_provider_linking_rule",
  "action_params": {
    "pattern": "NOME DO FORNECEDOR",
    "rule_type": "contains",
    "provider_id": <id>,
    "category_id": <id opcional — preenche label + categoria>,
    "priority": 0
  },
  "confidence": 85,
  "reasoning": "3 pagamentos manuais nos últimos 2 meses com mesmo pattern — automatizar próximos."
}
```

**Dedup do servidor:** rejeita se já existe regra com mesmo `(pattern, rule_type, provider_id)` — checar `list_rules(type=provider)` antes para não gerar sugestão que vai falhar na execução.

## Atualizar/Desativar Regras

Mesmo princípio da skill `categorization` § Atualizar/Desativar Regras: quando `list_rules(type=provider)` ou `list_rules(type=competence)` mostrar uma regra obsoleta ou errada, propor `update_rule`/`delete_rule` via `create_suggestion` em vez de ignorar.

**Casos típicos:**
- Fornecedor renomeado ou fundido com outro → `update_rule` mudando `provider_id`
- Regra de vinculação casando com transações de outro fornecedor → `update_rule` (pattern/rule_match_type) ou `delete_rule`
- Regra de competência de contrato encerrado → `delete_rule`

**action_params** usam os mesmos campos que `categorization`, trocando apenas o discriminador: `rule_type: "provider"` ou `rule_type: "competence"`. Ver exemplo completo na skill `categorization`.

**Suggestable** deve apontar para a **própria regra**: use `suggestable_type: "rule_provider"` (para `rule_type: "provider"`) ou `suggestable_type: "rule_competence"` (para `rule_type: "competence"`), com `suggestable_id` igual ao `rule_id`. O servidor rejeita mismatch entre morph e `action_params`.

**`delete_rule` é soft-delete** (marca `is_active=false`). Para reativar, `update_rule` com `is_active: true`.
