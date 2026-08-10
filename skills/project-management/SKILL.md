---
name: project-management
description: "Gestão de obras (incorporadora) no PontoAlto: cadastro de obras/etapas/tipos de custo via escrita direta, atribuição de lançamentos via sugestões (link_project) e leitura do resultado por obra."
version: 0.1.0
---

# Obras (Incorporadora)

Aplica-se apenas a tenants do ramo **incorporadora** — empresas que compram terreno, constroem e vendem. O objetivo é acompanhar custo e receita **por empreendimento**, não só da empresa como um todo.

Se `list_projects` retornar vazio e o gestor não falar em obras, o tenant provavelmente não é do ramo — confirme antes de cadastrar qualquer coisa.

## O Modelo: duas hierarquias, não três

- **Obra** (`projects`) — o empreendimento. Ex.: "Residencial Aurora"
- **Etapa** (`project_phases`) — pertence a **uma** obra. Ex.: Terreno, Fundação, Estrutura, Acabamento
- **Tipo de Custo** (`cost_types`) — dimensão **global**, transversal. Ex.: Material, Serviço, Mão de Obra, Imposto

Obra→Etapa é hierarquia de verdade. Tipo de Custo **não** é filho da etapa: ele se repete igual em toda etapa de toda obra. Por isso é tabela própria, aplicada por lançamento.

O cruzamento que o gestor quer — *"na Fundação da Obra A, quanto foi Material vs Serviço?"* — sai do group by das duas colunas do mesmo lançamento. Não tente modelar Tipo de Custo como nível da árvore.

## Modelo de escrita: dois regimes

Esta é a distinção que mais confunde. Preste atenção nela:

| Operação | Regime | Por quê |
|---|---|---|
| Cadastrar/editar obra, etapa, tipo de custo | **Escrita direta** (admin-only) | Configuração administrativa, como `save_sale_source_definition` |
| **Atribuir** obra a lançamentos | **Sugestão na inbox** (`link_project`) | Altera dado financeiro — o gestor decide |

Nunca atribua obra a lançamento por escrita direta. Não existe tool para isso — só a action `link_project` via sugestão.

## Tools

```
list_projects(only_assignable?, search?)   → obras + etapas + contagem de lançamentos
save_project(...)                          → cria/atualiza obra e etapas (upsert por nome)
delete_project(id)                         → só obra sem lançamentos
list_cost_types(only_active?)              → tipos de custo
save_cost_type(...)                        → cria/atualiza; seed_defaults cria os padrões
get_project_results(from?, to?)            → receita, custo, margem, quebras
```

## Cadastro Inicial de um Tenant Novo

Ordem importa:

1. **`save_cost_type(seed_defaults: true)`** — cria Material, Serviço, Mão de Obra, Imposto, Documentação, Terreno, Outro. Os que já existem são preservados.
2. **`save_project(name, status, seed_default_phases: true)`** — cria a obra já com as etapas usuais (Terreno, Projetos e Aprovações, Fundação, Estrutura, Alvenaria e Instalações, Acabamento, Entrega).
3. **Vincular categorias aos tipos de custo** — `save_cost_type(name: "Material", category_ids: [...])`. Isso faz o lançamento **herdar** o tipo da categoria automaticamente. Sem isso, o gestor teria que escolher o tipo em cada lançamento na mão.

O passo 3 é o que mais economiza trabalho e é o mais esquecido. Use `list_categories` para achar as categorias de custo de obra (Materiais de Construção, Mão de Obra de Construção, Projetos e Engenharia, Terrenos) e mapeie cada uma ao seu tipo.

### Se o plano de contas não for de incorporadora

Tenants novos nascem com o **DRE genérico**, não o de incorporadora — isso é comportamento intencional do sistema. Se `list_categories` não mostrar "Aquisição de Terrenos", "Cimento, Areia e Brita", "Empreiteiros e Construtoras", avise o gestor que ele precisa rodar **Reorganizar DRE → Incorporadora** na tela de Categorias antes. Não há tool MCP para isso.

## Atribuir Lançamentos a Obras

**Contexto que muda tudo:** as contas bancárias costumam ser compartilhadas entre obras. O extrato chega misturado e **não dá para deduzir a obra pela conta**. A atribuição é lançamento a lançamento.

Fluxo:

1. `get_project_results()` → olhar o bloco **`unassigned`**: é a fila de trabalho
2. `list_transactions(type=debit, date_from, date_to)` → os lançamentos sem obra
3. `list_projects()` → IDs de obra e etapa
4. `bulk_create_suggestions` com action `link_project`

Sinais para inferir a obra:

- **Fornecedor** — empreiteiro costuma trabalhar numa obra por vez. `analyze_provider_payments` ajuda. Mas cuidado: fornecedor migra de obra ao longo do tempo, então **a data importa** — não assuma que o vínculo de março vale para agosto.
- **Descrição** — nota fiscal às vezes traz o nome da obra ou o endereço
- **Categoria** — "Aquisição de Terrenos" na fase inicial de uma obra específica
- **Valor e data** — medições de empreitada seguem cronograma da obra

Quando não houver sinal claro, **pergunte ao gestor**. Atribuir errado polui o custo de duas obras de uma vez e o gestor pode não perceber.

## Payload do link_project

```json
{
  "suggestable_type": "transaction",
  "suggestable_id": 1234,
  "action": "link_project",
  "action_params": {
    "transaction_ids": [1234, 1235, 1236],
    "project_id": 2,
    "project_phase_id": 7,
    "cost_type_id": 3
  },
  "confidence": 85,
  "reasoning": "Empreiteiro X atua na Obra Aurora desde março; medição mensal da fundação"
}
```

- `project_phase_id` é **opcional** — omita quando não souber a etapa. É melhor atribuir só a obra do que chutar a etapa.
- `project_phase_id` **precisa pertencer** à obra informada, senão a action recusa.
- `cost_type_id` é opcional — **omita para herdar da categoria**. Só informe quando quiser sobrescrever.

## Rateio entre Obras

Nota única que atende duas obras (ex.: caminhão de areia dividido) usa o split que já existe no sistema: o lançamento pai é dividido em filhos, e **cada filho recebe sua própria obra**.

Sugira `split_transaction` primeiro; depois, `link_project` em cada filho. O relatório conta só os filhos — nunca o pai — então não há duplicação.

## Ler o Resultado

`get_project_results()` retorna **acumulado da obra** (inception-to-date) por padrão. Isso é deliberado: para incorporadora o que importa é o custo total do empreendimento, não o do mês. Só passe `from`/`to` quando o gestor pedir um recorte.

Ao reportar:

- Priorize por **impacto monetário** — obra com maior custo primeiro
- Margem negativa em obra `em_obra` é **normal** (gasta antes de vender) — não sinalize como problema. Só é alerta em obra `entregue` ou `encerrada`.
- Sempre reporte o `unassigned`: custo sem obra é resultado distorcido em todas as obras
- Quebra por etapa mostra onde o dinheiro está indo; por tipo de custo mostra a natureza do gasto

## Encerrar uma Obra

Obra com lançamentos **não pode ser excluída** (histórico contábil). Para tirar dos seletores sem perder o histórico:

```
save_project(id: X, status: "encerrada")   → some dos seletores, fica nos relatórios
save_project(id: X, is_active: false)      → mesmo efeito
```

`delete_project` só funciona em obra sem nenhum lançamento — tipicamente um cadastro feito por engano.

## Regras de Ouro

- `list_projects` **antes** de qualquer sugestão de `link_project` — os IDs mudam por tenant
- `list_suggestions(status=pending)` antes de criar em lote — não duplicar
- Etapa errada é pior que etapa vazia: na dúvida, atribua só a obra
- Nunca atribua obra por escrita direta — sempre via sugestão
- Margem negativa em obra em andamento não é anomalia
