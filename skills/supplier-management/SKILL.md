---
name: supplier-management
description: "Ajuste de datas de competência e vinculação de fornecedores a transações no PontoAlto: analyze_supplier_payments, link_provider, create_provider, set_competence_date."
version: 0.1.0
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

**Como agir:**
1. Para matches com fornecedor existente: criar sugestões `link_provider`
2. Para empresas sem fornecedor cadastrado: sugerir `create_provider` + `link_provider`
3. Se há padrões recorrentes: sugerir `create_provider_linking_rule`

## Atualizar/Desativar Regras

Mesmo princípio da skill `categorization` § Atualizar/Desativar Regras: quando `list_rules(type=provider)` ou `list_rules(type=competence)` mostrar uma regra obsoleta ou errada, propor `update_rule`/`delete_rule` via `create_suggestion` em vez de ignorar.

**Casos típicos:**
- Fornecedor renomeado ou fundido com outro → `update_rule` mudando `provider_id`
- Regra de vinculação casando com transações de outro fornecedor → `update_rule` (pattern/rule_match_type) ou `delete_rule`
- Regra de competência de contrato encerrado → `delete_rule`

**action_params** usam os mesmos campos que `categorization`, trocando apenas o discriminador: `rule_type: "provider"` ou `rule_type: "competence"`. Ver exemplo completo na skill `categorization`.

**Suggestable** deve apontar para a **própria regra**: use `suggestable_type: "rule_provider"` (para `rule_type: "provider"`) ou `suggestable_type: "rule_competence"` (para `rule_type: "competence"`), com `suggestable_id` igual ao `rule_id`. O servidor rejeita mismatch entre morph e `action_params`.

**`delete_rule` é soft-delete** (marca `is_active=false`). Para reativar, `update_rule` com `is_active: true`.
