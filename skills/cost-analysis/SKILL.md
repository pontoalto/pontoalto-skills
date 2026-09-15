---
name: cost-analysis
description: "Análise de custos e margem por item vendido no PontoAlto: as sete views do get_cost_analysis (summary, by_service, by_provider, top_costly, missing_costs, execution_coverage, breakdown), base venda vs produção, margem bruta vs margem de contribuição, markup, e a action set_item_cost para cadastrar o custo que falta pela inbox."
version: 0.2.0
---

# Análise de Custos

## O que essa análise mede — e o que ela não mede

A análise de custos **não sai do extrato bancário**. Ela cruza duas coisas:

- **Item faturado** (`SaleItem`) — cada linha da venda importada do sistema de origem: nome do item, quantidade, valor pago, executante
- **Item de custo** (`CostItem` + vigências) — o custo unitário cadastrado para aquele item, opcionalmente por fornecedor e com vigência por data

O casamento é **por nome do item**, e o custo aplicado é o da vigência válida na data — a do executante quando existe, a padrão quando não. Três consequências que mudam a leitura:

- Item vendido sem item de custo de mesmo nome entra com **custo zero** e a margem do período sobe sozinha
- Renomear o item no sistema de origem **quebra o casamento** e o item volta a custar zero
- Custo aqui é do **item vendido**, não da categoria do lançamento. `get_cost_analysis` e o DRE (`get_reports`) respondem perguntas diferentes e divergem legitimamente — não tente casar os dois números

⚠️ **Não confunda com tipo de custo (`cost_types`).** Aquilo é dimensão de obra de incorporadora (Material, Mão de Obra, Imposto), documentada na skill `project-management`. `save_cost_type` **não** cadastra custo de item vendido — para isso existe a action `set_item_cost`, descrita abaixo.

## Comece sempre por missing_costs

Margem sempre parece boa quando falta custo cadastrado. Antes de reportar qualquer margem:

```
get_cost_analysis(view=missing_costs, from=YYYY-MM-DD, to=YYYY-MM-DD)
```

Devolve os itens que entraram no período sem custo algum, ordenados por receita, com `reason`:

| `reason`       | Significa                                                       | Conserto |
|----------------|-----------------------------------------------------------------|----------|
| `sem_cadastro` | Não existe item de custo com esse nome                           | Cadastrar o item de custo na UI, ou importar tabela de preços |
| `custo_zerado` | O item existe (`cost_item_id` preenchido) mas nenhuma vigência devolveu valor | Adicionar a vigência que falta, para a data e o fornecedor certos |

Se `total_revenue_without_cost` for material frente à receita do período, **diga isso antes de qualquer número de margem**: a margem reportada é otimista por construção. O campo `provider_name` de cada item já indica o executante mais frequente — é o candidato natural a receber a vigência.

Cada item vem com as duas âncoras de que a sugestão precisa: `cost_item_id` (quando o item existe) e `sale_item_id` (a linha faturada, quando ele ainda vai ser criado).

## Base de data: venda vs produção

`basis` decide em que mês a linha cai:

- **`venda`** — data do pagamento (caixa). É quando o dinheiro entrou.
- **`producao`** — data do atendimento na agenda. É quando o serviço saiu.

As duas divergem porque o cliente costuma pagar o exame na consulta que o pediu e realizá-lo dias depois. Uma receita paga em 25/03 e executada em 02/04 aparece em março na base venda e em abril na base produção.

**Padrão**: `producao` para clínica, `venda` para negócio de produto (que não tem agenda). Pedir `producao` num negócio de produto não é erro — a resposta volta com `basis: venda` e um `basis_note` explicando. **Sempre informe ao gestor em qual base o número foi medido**; comparar março em produção com abril em venda é comparar coisas diferentes.

**Quando usar cada uma:**
- Rentabilidade da operação, produtividade por profissional, custo de repasse → `producao`
- Conciliar com o caixa, com o DRE ou com o que o gestor vê no banco → `venda`

### execution_coverage — quanto da base produção é confiável

```
get_cost_analysis(view=execution_coverage, basis=producao, from, to)
```

Na base produção, a linha **sem agendamento vinculado** cai na data da venda. Isso é correto para laboratório (ninguém da casa executa, não passa pela agenda), mas **um item em nome de um profissional sem atendimento correspondente é ruído**. É exatamente o que `sale_dated_with_provider_lines` / `sale_dated_with_provider_revenue` contam — o número que merece investigação.

Na base venda a coverage não diz nada (tudo é `executed`): só rode nela para confirmar totais.

## Margem bruta, margem de contribuição e markup

- **Margem bruta** = receita − custo direto do item. É o que `margin` / `gross_margin` mostram.
- **Margem de contribuição** = margem bruta − premissas. As **premissas** são um percentual de estrutura (aluguel, folha administrativa, impostos) configurado pelo gestor na tela de Análise de Custos e aplicado sobre a receita. `summary` devolve `premises_pct` e `premises_total`.
- **Markup** vem como **multiplicador**, não percentual: `2,86x` significa preço 2,86 vezes o custo. Nunca leia markup como porcentagem de margem — são escalas diferentes (`2,86x` equivale a 65% de margem bruta).

Se `premises_pct` estiver em 0%, a margem de contribuição é igual à bruta — avise o gestor que as premissas não estão configuradas em vez de apresentar as duas como se fossem análises distintas.

## As views

| View                 | Responde                                                | Campos-chave |
|----------------------|---------------------------------------------------------|--------------|
| `summary`            | Como foi o período no total                              | `total_revenue`, `total_cost`, `gross_margin_pct`, `contribution_margin_pct`, `premises_pct`, `sale_count` |
| `missing_costs`      | Em que a margem está mentindo                            | `reason`, `revenue`, `avg_revenue`, `provider_name`, `cost_item_id` |
| `by_service`         | Qual tipo de item sustenta o resultado                   | `type`, `revenue`, `margin_pct`, `markup`, `contribution_margin_pct` |
| `by_provider`        | Quanto custa cada fornecedor/executante                  | `total_cost`, `avg_cost`, `cost_pct`, `margin_pct` |
| `top_costly`         | Quais itens têm a pior margem (ordem crescente)          | `item_name`, `cost_item_id`, `unit_cost`, `margin_pct` |
| `execution_coverage` | Quanto do período tem execução comprovada pela agenda    | `executed_lines`, `sale_dated_with_provider_revenue` |
| `breakdown`          | Quais vendas estão por trás de uma linha agregada        | `sale_id`, `customer_name`, `reference_date`, `paid_at`, `executed` |

**Filtros disponíveis em todas as views**: `service_types` (consulta, exame, procedimento, produto) e `provider_ids` (use `list_providers` para os ids). Os filtros voltam ecoados em `filters` na resposta.

**`breakdown`** exige `scope` + `value`:
- `scope=procedure` → `value` no formato `"NOME DO ITEM|Nome do Fornecedor"`, exatamente como `top_costly` devolve
- `scope=type` → `value` é o tipo (`exame`)
- `scope=provider` → `value` é o nome do fornecedor (`Sem fornecedor` para os órfãos)

Use `breakdown` quando um número agregado parecer errado: ele mostra venda a venda, com `reference_date` (execução) ao lado de `paid_at` (caixa) e o flag `executed`.

## Diagnóstico recomendado

Em sequência, uma view por vez, sempre com o mesmo período. A primeira chamada faz a varredura de vendas e agenda; as seguintes do mesmo `from`/`to` reaproveitam o cache do servidor (10 min). Chamar as views em paralelo faz todas varrerem ao mesmo tempo e disputarem os 2 slots de tools pesadas.

```
get_cost_analysis(view=missing_costs, from, to)   → o que está sem custo
get_cost_analysis(view=summary, from, to)         → totais e margens do período
get_cost_analysis(view=by_service, from, to)      → onde o resultado se forma
get_cost_analysis(view=top_costly, from, to)      → itens com pior margem
```

Depois, só se houver dúvida: `execution_coverage` (base produção suspeita) ou `breakdown` (linha agregada suspeita).

Para tendência, rode o mesmo período do mês anterior e compare `margin_pct` por tipo — variação acima de 10 pontos costuma ser custo novo, reajuste de repasse ou item que perdeu o casamento de nome.

## Cadastrar o custo que falta: `set_item_cost`

A lacuna que o `missing_costs` aponta se fecha por sugestão na inbox, como qualquer outra escrita. A action é `set_item_cost` e cobre os dois `reason`:

```json
{
  "type": "set_item_cost",
  "suggestable_type": "sale_item",
  "suggestable_id": 4821,
  "action": "set_item_cost",
  "action_params": {
    "item_name": "RAIO X TORAX",
    "cost": 32.50,
    "provider_id": 17,
    "effective_date": "2026-01-01"
  },
  "confidence": 80,
  "reasoning": "12 exames faturados em março a R$ 120,00 médios, sem custo cadastrado. Repasse de R$ 32,50 confirmado com o laboratório."
}
```

**Suggestable, por caso** — o servidor rejeita a combinação errada:

| `reason`       | `action_params`  | `suggestable_type` | `suggestable_id` |
|----------------|------------------|--------------------|------------------|
| `sem_cadastro` | `item_name`      | `sale_item`        | o `sale_item_id` do item |
| `custo_zerado` | `cost_item_id`   | `cost_item`        | o mesmo `cost_item_id` |

**Regras do payload:**
- `cost` obrigatório e maior que zero — sugerir custo zero é o mesmo estado que já existe
- `provider_id` opcional, mas quase sempre certo: o custo do exame varia por laboratório e o do procedimento por profissional. Use o `provider_name` que o `missing_costs` devolveu, resolvendo o id em `list_providers`; se o fornecedor não existir, encadeie `create_provider` → `set_item_cost` com `create_suggestion_chain`
- `effective_date` opcional; omitida vale o início do ano corrente. Informe quando souber a data do reajuste — a vigência é por data e o custo de março não deve reescrever o de janeiro
- `type` opcional e só usado quando o item é criado; omitido vale o padrão do tipo de negócio (produto para comércio, exame para clínica)
- Vigência já existente para o mesmo item, fornecedor e data é **atualizada**, não duplicada — é assim que se corrige um custo errado

**Confidence:** custo confirmado com o fornecedor (contrato, tabela, nota) → 85+. Custo inferido de itens parecidos ou de média de mercado → 60-75, e diga no `reasoning` de onde veio o número. **Nunca invente um custo plausível**: sem base, reporte o item e deixe o campo para o gestor.

Ao apresentar a lista, entregue **acionável**: nome do item, receita do período, quantidade, executante sugerido (`provider_name`) e a receita média por unidade (`avg_revenue`) — é a referência que o gestor usa para julgar se o custo proposto faz sentido.

O gestor aprova pela inbox, e o undo reverte por inteiro: apaga a vigência criada e também o item, quando foi a sugestão que o criou.

## Regras de ouro

- `missing_costs` antes de qualquer margem — margem sem essa checagem é otimista por construção
- Sempre informe a base (`basis`) junto do número; nunca compare períodos medidos em bases diferentes
- Markup é `x`, margem é `%` — não misture as escalas
- Priorize por receita, não por quantidade: dez itens de R$ 20,00 sem custo importam menos que um de R$ 4.000,00
- Custo sem base não vira sugestão: proponha `set_item_cost` só com número apurado, e diga a origem dele no `reasoning`
