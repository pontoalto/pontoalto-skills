---
description: "Atalho para gestão de obras de incorporadora no PontoAlto: cadastro de obras/etapas/tipos de custo, atribuição de lançamentos e resultado por empreendimento."
argument-hint: "[--local] [nome da obra ou contexto livre]"
---

# PontoAlto — Obras (Incorporadora)

Atalho para acompanhar custo e receita por empreendimento. Use em tenants do ramo incorporadora — cadastro de obras, atribuição de lançamentos e leitura do resultado por obra.

Responda em português. Use a skill `project-management` para o fluxo detalhado e `financial-domain` para o contexto de domínio.

## MCP Server

Argumento recebido: `$ARGUMENTS`

- Sem `--local`: tools com prefixo `mcp__claude_ai_Ponto_Alto__` (produção)
- Com `--local`: tools com prefixo `mcp__pontoalto-local__` (desenvolvimento)

## Inicialização

1. `list_tenants` — escolher via `AskUserQuestion` se 2+, automático se 1
2. Confirmar conexão: nome do tenant + organização
3. `list_projects` — se vazio, o tenant pode não ser incorporadora: **confirme com o gestor** antes de cadastrar

## Diagnóstico

```
list_projects                        → obras existentes + etapas + contagem de lançamentos
list_cost_types                      → tipos de custo cadastrados
get_project_results                  → receita/custo/margem acumulados + bloco unassigned
list_suggestions(status=pending)     → sugestões pendentes (não duplicar)
```

Resumir: obras cadastradas e situação, custo acumulado por obra, e **quantos lançamentos estão sem obra** — essa é a fila de trabalho.

## Escolha do subfluxo

Apresentar via `AskUserQuestion`:

1. **Atribuir lançamentos a obras (Recomendado)** — quando há itens no `unassigned`
2. **Cadastrar/editar obra** — nova obra, ajustar etapas, encerrar obra entregue
3. **Configurar tipos de custo** — criar os padrões e mapear categorias para herança automática
4. **Só relatório** — resultado por obra, sem criar nada

Se `unassigned.transaction_count` for 0 e não houver obra cadastrada, comece pelo cadastro (opção 2).

## Atribuir Lançamentos — Detalhe

1. `get_project_results` → tamanho da fila `unassigned`
2. `list_transactions(type=debit, date_from, date_to)` → lançamentos sem obra
3. `analyze_provider_payments` → agrupar por fornecedor; empreiteiro costuma atuar numa obra por vez
4. `list_projects` → IDs de obra e etapa
5. `bulk_create_suggestions` com action `link_project`, agrupando por fornecedor ou pattern

**Contas são compartilhadas entre obras** — o extrato chega misturado e a conta bancária não indica a obra. Infira por fornecedor, descrição, categoria e data. Sem sinal claro, **pergunte** em vez de chutar.

Na dúvida sobre a etapa, atribua só a obra (omita `project_phase_id`). Etapa errada é pior que etapa vazia.

## Cadastrar Obra — Detalhe

1. `list_projects` → confirmar que a obra não existe (upsert é por nome)
2. `save_project(name, status, start_date?, expected_end_date?, seed_default_phases: true)`
3. Ajustar etapas se o gestor tiver cronograma próprio: `save_project(id, phases: [...])`

Status: `planejamento` → `em_obra` → `entregue` → `encerrada`.

Para tirar uma obra dos seletores sem perder histórico: `save_project(id, status: "encerrada")`. `delete_project` só funciona em obra sem nenhum lançamento.

## Configurar Tipos de Custo — Detalhe

1. `save_cost_type(seed_defaults: true)` → cria os padrões de uma vez
2. `list_categories` → achar as categorias de custo de obra
3. `save_cost_type(name: "Material", category_ids: [...])` → categorias passam a **herdar** o tipo

O passo 3 é o que evita escolher o tipo em cada lançamento na mão. É o mais esquecido — não pule.

Se `list_categories` não mostrar categorias de obra ("Aquisição de Terrenos", "Cimento, Areia e Brita", "Empreiteiros e Construtoras"), o tenant está com o DRE genérico. Avise que é preciso rodar **Reorganizar DRE → Incorporadora** na tela de Categorias — não há tool MCP para isso.

## Relatório Final

Reportar:
- Obras cadastradas e situação de cada uma
- Custo acumulado, receita e margem por obra (maior custo primeiro)
- Quebra por etapa e por tipo de custo das obras relevantes
- Lançamentos atribuídos nesta sessão (quantos, para quais obras)
- **Quanto ainda está sem obra** — em valor e em quantidade
- Sugestões pendentes restantes

Margem negativa em obra `em_obra` é **normal** — a obra gasta antes de vender. Só sinalize como problema em obra `entregue` ou `encerrada`.

## Regras de Ouro

- Cadastro de obra/tipo é escrita direta (admin); **atribuição a lançamento é sempre sugestão**
- `list_projects` antes de qualquer `link_project` — IDs mudam por tenant
- Etapa precisa pertencer à obra informada, senão a action recusa
- Omita `cost_type_id` para herdar da categoria — só informe para sobrescrever
- Custo sem obra distorce o resultado de todas as obras: trate o `unassigned` como prioridade
