---
description: "LTV (lifetime value) no PontoAlto: valor e margem de cada paciente/cliente ao longo do tempo, por médico, por médico × especialidade (clínica) ou por cliente (produto) — onde investir em marketing, recompra e clientes recorrentes que pararam de comprar. Só consulta."
argument-hint: "[--local] [medico|especialidade|cliente] [maior|primeiro|separado] [YYYY|YYYY-MM]"
---

# PontoAlto — LTV

Atalho para a análise de LTV. Use quando o gestor quer saber quais médicos trazem os pacientes que mais deixam margem (onde investir em marketing), em qual especialidade cada médico rende mais, quem são os melhores clientes, quanto da base volta a comprar ou quais clientes recorrentes sumiram. Apenas consulta — não cria sugestões.

Responda em português. Use a skill `ltv-analysis` para a leitura detalhada e `financial-domain` para o contexto de domínio.

## MCP Server

Argumento recebido: `$ARGUMENTS`

- Sem `--local`: tools com prefixo `mcp__claude_ai_Ponto_Alto__` (produção)
- Com `--local`: tools com prefixo `mcp__pontoalto-local__` (desenvolvimento)

**Dimensão**: `medico`, `especialidade` (→ `group_by=medico_especialidade`) ou `cliente` nos argumentos força o `group_by`. Sem isso, deixe a tool escolher (médico em clínica, cliente em negócio de produto).

**Visão** (só por médico): `maior` (→ `attribution=maior_gasto`), `primeiro` (→ `attribution=primeiro_medico`) ou `separado` (→ `attribution=separado`) nos argumentos força a atribuição. Sem isso, a tool usa `separado`, como a tela — que além disso ordena por MC/paciente e esconde médicos com menos de 5 pacientes; se o gestor estiver comparando com a tela, use `sort_by=contribution_margin_per_patient` e `min_patients=5`. As regras de cada visão estão na skill `ltv-analysis` § Visões de atribuição.

**Escopo**: sem período nos argumentos → histórico completo (lifetime, não passar `from`/`to`). `YYYY` → o ano inteiro (`YYYY-01-01` a `YYYY-12-31`). `YYYY-MM` → o mês, avisando que em um mês a frequência fica perto de 1 e o LTV se aproxima do ticket.

## Inicialização

1. `list_tenants` — escolher via `AskUserQuestion` se 2+, automático se 1
2. Confirmar conexão: nome do tenant + organização
3. Ir direto ao diagnóstico

## Diagnóstico

Uma única chamada, já dimensionada para responder as perguntas seguintes sem refazer a varredura:

```
get_ltv_analysis(tenant_id, [group_by], [attribution], [from, to], limit=100)
```

A tool é pesada e **não tem cache**: cada chamada varre e precifica todo o histórico de vendas. Não chame em paralelo com outra tool pesada e não pagine à toa. Se vier `Servidor ocupado, tente novamente em alguns segundos`, aguarde 10 s e repita a mesma chamada uma vez.

Apresentar o resumo (receita **e** margem) e as 10 primeiras linhas, e em seguida as observações da skill `ltv-analysis` § Leitura crítica que se aplicarem (amostra pequena, `revenue_without_cost_pct` alto, `last_visit` antiga, `frequency` colada em 1,0, `(sem médico)` grande). Se `summary.premises_pct` vier 0%, avisar que o LTV está sem premissas e a margem de contribuição é igual à bruta — configuram-se na tela do LTV (Editar premissas, ou Copiar da Análise de Custos).

## Escolha do subfluxo

Apresentar via `AskUserQuestion`, conforme o `group_by` devolvido.

**Por médico (ou médico × especialidade):**

1. **Onde investir (Recomendado)** — em `attribution=primeiro_medico` (quem trouxe o paciente; nova chamada se o diagnóstico veio em outra visão), ranking por `contribution_margin_per_patient`, só com base razoável (10+ pacientes) e médicos ativos (`last_visit` nos últimos 90 dias, contados a partir da maior `last_visit` devolvida — a importação pode estar atrasada). Mostrar ao lado a margem por visita e `revenue_without_cost_pct`, e avisar quando este passar de ~10%
2. **Por especialidade** — `group_by=medico_especialidade`: médico que atende duas especialidades aparece separado; comparar a margem por paciente entre as linhas do mesmo médico
3. **Quem fideliza** — em `attribution=separado` (retornos ao próprio médico), reordenar por `frequency`, só entre médicos com base razoável
4. **Tendência** — este ano contra o ano anterior, duas chamadas com `from`/`to`, em sequência

Se o gestor perguntar quanto cada médico produz, ou quanto gera com os exames que pede, use `attribution=separado` e avise que ali o paciente conta em cada médico que o atendeu (a soma de pacientes das linhas passa do resumo). Se um grupo de exames ou um ultrassonografista aparecer no topo em `maior_gasto`, mostre a mesma pergunta em `primeiro_medico`.

Se o gestor citar um médico, `search="nome"` (em `medico_especialidade`, o nome da especialidade também serve).

**Por cliente:**

1. **Reativação (Recomendado)** — `get_ltv_analysis(group_by=cliente, min_orders=2, sort_by=last_purchase, direction=asc)`: clientes recorrentes que estão há mais tempo sem comprar
2. **Melhores clientes** — o top por margem de contribuição (`sort_by=contribution_margin`), com LTV, pedidos, margem por pedido e última compra
3. **Um cliente específico** — `search="nome"`

Reordenar, recortar o top ou filtrar por base mínima e por `last_visit` pode ser feito sobre as linhas que já vieram quando `has_more=false`. Só chame a tool de novo quando precisar de outra dimensão, de outra visão, de outro filtro no servidor (`min_patients`, `active_since`, `min_orders`, `search`, período) ou de linhas além das recebidas.

## Formato de Saída

```
💎 LTV — [por Médico | por Médico × Especialidade | por Cliente]
Tenant: [nome] · Escopo: [histórico completo | DD/MM/YYYY a DD/MM/YYYY] · Visão: [Maior gasto | Primeiro médico | Separado] (só por médico)

Receita          R$ ...
Custo            R$ ...
Margem bruta     R$ ...  (..%)
Premissas        ..%
Margem contrib.  R$ ...  (..%)
Sem custo        R$ ...  (..% da receita)
Pacientes        ...          (clientes: ... · recorrentes: ... (..%))
Visitas          ...          (pedidos: ...)
LTV médio        R$ ...
MC por paciente  R$ ...       (por cliente)
MC por visita    R$ ...       (por pedido)
```

Depois, tabela com o top, sempre com a base ao lado:

- Por médico: Médico · [Especialidade] · Pacientes · Visitas/pac. · LTV · MC/paciente · MC/visita · MC % · Sem custo % · Última visita (DD/MM/YYYY)
- Por cliente: Cliente · Pedidos · LTV · Margem contrib. · MC/pedido · Última compra (DD/MM/YYYY)

Valores em R$ formato brasileiro.

## Regras de Ouro

- **Só consulta** — nada deste command vira sugestão na inbox
- Informar sempre dimensão, visão (por médico) e escopo junto do número; nunca comparar lifetime com período nem visões diferentes como se fossem o mesmo número
- Para decidir investimento, comparar margem de contribuição por paciente, não LTV — e nunca multiplicar o LTV pela margem de `get_cost_analysis(view=by_provider)`
- Mostrar `patient_count` / `orders` e `revenue_without_cost_pct` ao lado da margem; amostra pequena e margem inflada não entram em ranking sem aviso
- Não recomendar médico com `last_visit` antiga
- Item sem custo se resolve em `/pontoalto:costs` (sugestão `set_item_cost`), não aqui
- Nomes de clientes são dado pessoal: mostrar ao gestor, não copiar para fora do PontoAlto sem ele pedir
