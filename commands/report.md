---
description: "Gera relatório mensal consolidado no PontoAlto: DRE, orçado vs realizado e análise de custos. Só consulta — não cria sugestões."
argument-hint: "[--local] [YYYY-MM]"
---

# PontoAlto — Relatório Mensal

Atalho para gerar o relatório mensal consolidado. Apenas consulta — não cria nem aprova sugestões.

Responda em português. Use `financial-domain` para o contexto de domínio (DRE por competência, regimes tributários).

## MCP Server

Argumento recebido: `$ARGUMENTS`

- Sem `--local`: tools com prefixo `mcp__claude_ai_Ponto_Alto__` (produção)
- Com `--local`: tools com prefixo `mcp__pontoalto-local__` (desenvolvimento)

**Período**: se o argumento contiver `YYYY-MM`, usar esse mês. Caso contrário, usar o mês atual fechado (se hoje é dia ≤ 10, oferecer o mês anterior; senão o atual).

## Inicialização

1. `list_tenants` — escolher via `AskUserQuestion` se 2+, automático se 1
2. Confirmar conexão: nome do tenant + organização + regime tributário
3. Confirmar o período com o gestor via `AskUserQuestion` se houver ambiguidade (2 opções: mês atual / mês anterior)

## Pré-checagem de Qualidade

Antes de gerar o relatório, avisar o gestor se o mês **não está pronto** para fechamento. Usar uma única chamada consolidada:

```
get_workflow_status(tenant_id, period=YYYY-MM) → status de categorização, fornecedores, reconciliação, competência, custos
```

Verificar os campos `categorization.status`, `providers.status`, `reconciliation.status`, `competence.status` e `costs.status`. Se algum estiver em `warning`, apresentar o impacto. Só aprofunde em `transaction_stats` / `analyze_provider_payments` / `list_sales` se o gestor quiser detalhar as pendências.

Se há pendências significativas, **apresentar o impacto** antes de gerar o relatório:

```
⚠️  Relatório do mês MM/YYYY pode estar incompleto:
   - 12 transações sem categoria (R$ 8.450,00)
   - 5 pagamentos sem fornecedor (R$ 3.200,00)
   - 3 vendas sem conciliação (R$ 1.890,00)

Deseja gerar assim mesmo ou resolver pendências primeiro?
```

Usar `AskUserQuestion` com 2 opções: **Gerar assim mesmo** / **Resolver pendências primeiro** (redireciona para `/pontoalto:manager`).

## Geração do Relatório

Chamar as 3 tools em paralelo:

```
get_reports(type=dre, period=YYYY-MM)
get_budget_comparison(period=YYYY-MM)
get_cost_analysis(view=by_service, period=YYYY-MM)
```

## Formato de Saída

Apresentar em **markdown estruturado**:

### 1. Cabeçalho

```
📊 Relatório Financeiro — MM/YYYY
Tenant: [nome]
Regime: [Lucro Real / Presumido / Simples]
Gerado em: DD/MM/YYYY
```

### 2. DRE Resumida

Tabela com: Receita Bruta, Deduções, Receita Líquida, Custos, Lucro Bruto, Despesas Operacionais, Resultado. Valores em R$ formato brasileiro.

### 3. Orçado vs. Realizado

Top 10 categorias com maior variação (% e R$). Destacar:
- 🟢 Realizado abaixo do orçado (economia)
- 🔴 Realizado acima do orçado (estouro > 10%)

### 4. Análise de Custos por Serviço

Top 10 serviços por margem:
- Serviços com maior margem bruta
- Serviços com margem < 20% (alerta)
- Serviços sem custo cadastrado (advisory — não entra no cálculo)

### 5. Alertas e Observações

- Variações anômalas vs. mês anterior (> 30% em qualquer linha)
- Pendências que afetam a DRE (se houver, do diagnóstico inicial)
- Ações sugeridas (ex: "revisar categoria X — valor aumentou 45% vs. mês anterior")

## Regras de Ouro

- **Nunca criar sugestões** neste command — só consulta
- Sempre informar o período no cabeçalho
- Sempre mostrar o regime tributário (afeta interpretação)
- Priorizar linhas do DRE por impacto monetário
- Se o gestor pedir detalhe de uma linha, usar `list_transactions` com filtro de categoria
