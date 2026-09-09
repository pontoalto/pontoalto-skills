---
description: "Análise de custos e margem por item vendido no PontoAlto: itens sem custo cadastrado, margem por tipo e por fornecedor, itens com pior margem. Só consulta — não cria sugestões."
argument-hint: "[--local] [YYYY-MM]"
---

# PontoAlto — Análise de Custos

Atalho para a análise de custos e margem. Use quando o gestor quer saber onde o resultado se forma, quais itens dão prejuízo, ou por que a margem do mês mudou.

Responda em português. Use a skill `cost-analysis` para o fluxo detalhado e `financial-domain` para o contexto de domínio.

## MCP Server

Argumento recebido: `$ARGUMENTS`

- Sem `--local`: tools com prefixo `mcp__claude_ai_Ponto_Alto__` (produção)
- Com `--local`: tools com prefixo `mcp__pontoalto-local__` (desenvolvimento)

**Período**: se o argumento contiver `YYYY-MM`, usar esse mês. Caso contrário, o mês atual. `get_cost_analysis` recebe `from` e `to` em `YYYY-MM-DD` — não existe parâmetro `period` nesta tool.

## Inicialização

1. `list_tenants` — escolher via `AskUserQuestion` se 2+, automático se 1
2. Confirmar conexão: nome do tenant + organização
3. Ir direto ao diagnóstico

## Diagnóstico

Chamar em paralelo:

```
get_cost_analysis(view=missing_costs, from, to)  → itens faturados sem custo
get_cost_analysis(view=summary, from, to)        → totais, margem bruta e de contribuição
get_cost_analysis(view=by_service, from, to)     → margem por tipo de item
get_cost_analysis(view=top_costly, from, to)     → itens com pior margem
```

**Abrir o relatório pelo `missing_costs`**, não pelo summary: se há receita relevante sem custo cadastrado, toda margem abaixo está inflada e o gestor precisa saber disso antes de ler o número.

Sempre informar em qual `basis` os números foram medidos (a resposta devolve o campo). Se vier `basis_note`, repassar o aviso.

## Escolha do subfluxo

Apresentar via `AskUserQuestion`:

1. **Fechar as lacunas de custo (Recomendado)** — detalhar `missing_costs` com receita, quantidade e executante sugerido, para o gestor cadastrar na tela
2. **Entender a margem** — `by_service` + `by_provider`, comparando com o mês anterior
3. **Investigar um item específico** — `top_costly` → `breakdown` do item escolhido

## Fechar Lacunas de Custo — Detalhe

1. `get_cost_analysis(view=missing_costs)` → ordenar por receita
2. Separar por `reason`: `sem_cadastro` (falta o item) vs `custo_zerado` (falta a vigência)
3. Para cada item do topo, entregar a linha pronta para digitar: nome do item, receita do período, quantidade, `provider_name` sugerido e receita média por unidade
4. Se o executante não estiver cadastrado como fornecedor, checar em `list_providers` e avisar que o cadastro do fornecedor vem antes
5. **Não prometer cadastrar** — não existe tool de escrita para custo de item; o cadastro é do gestor, na tela de Custos de Serviços Faltantes

## Entender a Margem — Detalhe

1. `by_service` do período e do mês anterior → variação de `margin_pct` por tipo
2. `by_provider` → concentração de custo (`cost_pct`) e fornecedores com margem baixa
3. Variação acima de 10 pontos: investigar custo novo, reajuste de repasse ou item que perdeu o casamento de nome
4. Filtrar com `provider_ids` ou `service_types` quando o gestor quiser recortar

## Investigar um Item — Detalhe

1. `top_costly` → escolher o item (a lista já vem da pior margem para a melhor)
2. `get_cost_analysis(view=breakdown, scope=procedure, value="NOME DO ITEM|Nome do Fornecedor")` → vendas por trás da linha
3. Conferir `reference_date` (execução) vs `paid_at` (caixa) e o flag `executed`
4. Se muitas linhas vierem com `executed=false` e com profissional, rodar `execution_coverage` e reportar o ruído da agenda

## Formato de Saída

```
💰 Análise de Custos — MM/YYYY
Tenant: [nome] · Base: [venda|produção]

⚠️  R$ X sem custo cadastrado (N itens) — margem abaixo está otimista

Receita        R$ ...
Custo direto   R$ ...  (..%)
Margem bruta   R$ ...  (..%)
Premissas      R$ ...  (..%)
Margem contrib R$ ...  (..%)
```

Depois: tabela de margem por tipo, top itens com pior margem, e a lista acionável de itens sem custo. Valores em R$ formato brasileiro, markup em `x`.

## Regras de Ouro

- **Nunca criar sugestões** neste command — só consulta
- `missing_costs` primeiro, sempre
- Informar a base (`basis`) junto de qualquer número; nunca comparar períodos em bases diferentes
- Markup é multiplicador (`2,86x`), margem é percentual — não misturar
- Priorizar por receita, não por quantidade
- Não confundir custo de item vendido com `cost_types` (dimensão de obra, skill `project-management`)
