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
