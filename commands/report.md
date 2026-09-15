---
description: "Gera relatório mensal consolidado no PontoAlto: DRE, orçado vs realizado e análise de custos. Só consulta — não cria sugestões."
argument-hint: "[--local] [YYYY-MM]"
---

# PontoAlto — Relatório Mensal

Atalho para gerar o relatório mensal consolidado. Apenas consulta — não cria nem aprova sugestões.

Responda em português. Use `financial-domain` para o contexto de domínio (DRE por competência ou caixa, regimes tributários) e `cost-analysis` para a leitura da margem.

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

Chamar as tools nesta ordem, uma de cada vez (são todas tools pesadas; o servidor executa no máximo 2 ao mesmo tempo):

```
get_reports(report=dre, from=YYYY-MM-DD, to=YYYY-MM-DD)
get_budget_comparison(from=YYYY-MM-DD, to=YYYY-MM-DD)
get_cost_analysis(view=by_service, from=YYYY-MM-DD, to=YYYY-MM-DD)
get_cost_analysis(view=missing_costs, from=YYYY-MM-DD, to=YYYY-MM-DD)
```

As duas views de `get_cost_analysis` do mesmo período compartilham cache no servidor: a segunda é barata se vier depois da primeira. Se vier `Servidor ocupado, tente novamente em alguns segundos`, aguarde 10 s e repita a mesma chamada uma vez.

**Regime do DRE**: o padrão é competência (`basis=competencia`). Se o gestor pedir "DRE por caixa" / "regime de caixa", passar `basis=caixa` em `get_reports` e indicar o regime no cabeçalho do relatório.

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

### 4. Análise de Custos por Item

Informar a base medida (`basis`: venda ou produção — vem na resposta) e, antes da margem, quanto de receita entrou sem custo cadastrado (`missing_costs.total_revenue_without_cost`): a margem abaixo é otimista na proporção disso.

Top 10 tipos de item por margem:
- Itens com maior margem bruta
- Itens com margem < 20% (alerta)
- Itens sem custo cadastrado (advisory — entram com custo zero e inflam a margem)

Markup vem como multiplicador (`2,86x`), não percentual. Detalhe na skill `cost-analysis`.

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
