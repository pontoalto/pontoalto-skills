---
name: sale-sources
description: "Guia canônico para montar, testar e salvar definições de fontes de venda customizadas (custom CSV importers) via MCP, sem precisar acessar o código do sistema. Cobre o DSL declarativo, o loop de preview iterativo, a mapping_table devolvida pelo MCP (agrupada por Sale vs SaleItem, com severity por linha), os novos sinais de debug (skipped_rows, errors estruturados, header_suggestions, dedup_stats, unmapped_csv_columns), o safety net de revert e as exceções ao modelo de sugestões."
version: 0.4.0
---

# Ponto Alto — Fontes de Venda Customizadas

O Ponto Alto importa vendas de múltiplas origens (Feegow, Yzidro, caixa físico, operadoras de cartão). Quando o gestor tem um CSV de um sistema que **ainda não está integrado**, ele pode criar uma **Fonte de Venda customizada**: uma definição declarativa (spec JSON) que ensina o importador a interpretar aquele layout específico — sem alterar o código do PontoAlto.

Esta skill é a **fonte única da verdade** do fluxo de ponta a ponta para quem monta fontes conversando com o Claude via MCP. O comando `/sale-source` é um atalho enxuto que delega aqui.

## Quando usar

- O gestor tem um CSV de vendas de um sistema novo (ex: um ERP vertical, um relatório exportado de planilha).
- Uma fonte existente parou de funcionar porque o fornecedor mudou o layout do CSV.
- O gestor quer entender qual fonte está ativa e como ela mapeia os campos.

> **Pré-requisito:** o gestor precisa ter perfil `admin` no tenant. Fontes de venda **não** ficam disponíveis para `financeiro` ou `viewer`.

## Tools MCP envolvidas

Todas no prefixo do servidor escolhido (`mcp__claude_ai_Ponto_Alto__` ou `mcp__pontoalto-local__`):

| Tool | Tipo | Papel no fluxo |
|------|------|----------------|
| `list_sale_source_definitions` | leitura | Ver o que já existe (com `last_used_at` e `sale_imports_count`) |
| `get_sale_source_definition` | leitura | Inspecionar uma spec existente (key ou id) |
| `get_sale_source_dsl_reference` | leitura | **Referência canônica do DSL.** Aceita `section=core` (default — resolvers, parsers, modifiers, operadores, row_filters, enums, ui_columns_reference), `section=recipes` (recipes + workflow), `section=full` (tudo). **Comece sempre por `core`** — é o suficiente para escrever a spec e é substancialmente menor |
| `get_sale_source_spec_template` | leitura | Template inicial. Estilos em ordem de complexidade: `minimal` (skeleton) → `basic_1to1` (1 linha = 1 venda) → `grouped` (N linhas = 1 venda) → `feegow_like` (seed Feegow real) → `yzidro_like` (seed Yzidro real) |
| `preview_sale_source_definition` | leitura | Testa a spec contra um CSV amostra **sem persistir nada**. Retorna `items`, `sales`, `mapping_table`, `skipped_rows`, `errors`, `header_suggestions`, `unmapped_csv_columns`, `dedup_stats`. Aceita `sample_mode`: `head` (default, primeiras N linhas), `stratified` (amostra balanceada início/meio/fim), `full` (até 500 linhas, para validação final) |
| `save_sale_source_definition` | **escrita direta** | Cria ou atualiza (upsert por `key`). Não passa pela inbox |
| `delete_sale_source_definition` | **escrita direta** | Remove a fonte. Falha se houver importações vinculadas |
| `revert_sale_source_definition` | **escrita direta** | Restaura a spec para a versão imediatamente anterior (lê o activity log). Safety net quando um save deu errado |

### ⚠️ Exceção ao modelo de sugestões

`save_sale_source_definition`, `delete_sale_source_definition` e `revert_sale_source_definition` são **escrita direta, sem sugestão na inbox** — alinhado com `financial-domain` § Exceções. O motivo: o ciclo preview → ajustar → preview → salvar ficaria inviável passando por aprovação manual a cada passo.

**Compensação:** sempre mostre ao gestor um resumo do que vai salvar (nome, key, contagem de campos mapeados, se é create ou update) e peça confirmação via `AskUserQuestion` antes de chamar `save_sale_source_definition`. O `revert_sale_source_definition` também precisa de confirmação antes — só use quando o gestor pedir explicitamente ou quando o save imediatamente anterior ficou claramente errado.

## Fluxo recomendado (8 passos)

### 1. Dimensionar o que já existe

```
list_sale_source_definitions()
```

Mostre ao gestor as fontes ativas/desativadas, **incluindo `last_used_at` e `sale_imports_count`** — isso dá contexto para decidir entre editar uma fonte já usada (alto impacto se quebrar) ou criar uma nova. Se a fonte que ele quer já existe e só está desabilitada, talvez o caminho seja editar, não criar.

### 2. Coletar amostra do CSV

Peça para **colar o conteúdo do CSV** na conversa (primeiras 10-30 linhas bastam). Evite anexar o arquivo inteiro — amostra enxuta basta para validar o parse.

Pergunte via `AskUserQuestion`:
- **Nome** (ex: "Vendas Sistema XPTO") e **key** (slug, ex: `xpto_vendas`)
- **Separador** do CSV (`,`, `;`, `\t`)
- **Linha do header** (geralmente 0, mas alguns exports têm metadados nas primeiras linhas)

### 3. Aprender o DSL antes de escrever

```
get_sale_source_dsl_reference()          → section=core (default)
```

Esta é a **fonte única da verdade** do DSL. Por padrão o retorno vem em `section=core` — lista resolvers (`from`, `literal`, `setting`, `concat`, `if/then/else`, `lookup_category_by_name`), parsers (`decimal_br`, `date_br`, `money_br`, etc.), modifiers (`default`, `abs`, `cast`, `min`, `max`, `map`), operadores para `detection` e `row_filters`, enums válidos, e `ui_columns_reference` (3 colunas Sale + 8 colunas SaleItem).

Se precisar de exemplos prontos para casos comuns (cancelamento multi-campo, concat condicional, filtro por setting, quantidade negativa, CSV sem header, header deslocado), chame com `section=recipes` — **só quando for necessário**, porque o payload é pesado. Use `section=full` apenas em situações de treino ou depuração profunda.

**Nunca invente chaves.** Se a referência não lista, não existe.

### 4. Partir de um template

```
get_sale_source_spec_template(style=<style>)
```

Escolha o template mais próximo do CSV do gestor. Escada de complexidade:

- **`minimal`** — esqueleto vazio com placeholders `REPLACE_*`. Use só se nenhum outro encaixa.
- **`basic_1to1`** — 1 linha do CSV = 1 venda completa, sem agrupamento. Ideal para exports flat: e-commerce simples, Stripe/Asaas line items, planilhas de vendas em que cada linha já tem cliente + produto + valor.
- **`grouped`** — N linhas colapsam em 1 venda via coluna "Pedido/Ordem". Ideal para PDV/ERP que emite uma linha por item dentro de um mesmo pedido.
- **`feegow_like`** — baseado no seed Feegow real (clínica): filtro de unidade via Setting, detecção de cancelamento multi-campo, inferência de categoria, grouping por paciente+data.
- **`yzidro_like`** — baseado no seed Yzidro real (varejo): parse de data BR, grouping por pedido, concat descrição+código, quantidade com abs+cast.

Cada template vem com `description` e `next_steps` explicando o que substituir. Os placeholders (`REPLACE_*`) precisam ser trocados pelos headers reais do CSV do gestor.

### 5. Primeiro preview + apresentar a `mapping_table`

**Obrigatório** — antes de qualquer discussão sobre ajustes, faça o primeiro `preview_sale_source_definition` e apresente a `mapping_table` devolvida. Esse campo é **autoritativo**: descreve exatamente o que a UI de importação do PontoAlto vai exibir quando essa spec rodar de verdade. **Sempre renderize a partir do payload do MCP** — nunca reescreva de cabeça.

```
preview_sale_source_definition(csv_content=<amostra>, spec=<spec_atual>)
```

O retorno tem um campo `mapping_table` agrupado por tabela alvo — **`sale`** (3 colunas que populam `sales`) e **`sale_item`** (8 colunas que populam `sale_items`). Cada linha carrega agora um campo `severity`:

```jsonc
{
  "sale": [
    {"ui_column": "Ref. Cliente", "spec_field": "customer.reference",   "required": true,  "mapped": true,  "csv_source": "Pedido",      "parser": null,       "kind": "column",  "severity": "ok"},
    {"ui_column": "Cliente",      "spec_field": "customer.name",        "required": true,  "mapped": true,  "csv_source": "Cliente",     "parser": null,       "kind": "column",  "severity": "ok"},
    {"ui_column": "CPF",          "spec_field": "customer.cpf",         "required": false, "mapped": false, "csv_source": null,          "parser": null,       "kind": null,      "severity": "info"}
  ],
  "sale_item": [
    {"ui_column": "Item",               "spec_field": "fields.description",    "required": true,  "mapped": true,  "csv_source": "Produto",     "parser": null,       "kind": "column",  "severity": "ok"},
    {"ui_column": "Data de Referência", "spec_field": "fields.reference_date", "required": true,  "mapped": true,  "csv_source": "Emissão",     "parser": "date_br",  "kind": "column",  "severity": "ok"},
    {"ui_column": "Valor Cobrado",      "spec_field": "fields.total_paid",     "required": true,  "mapped": true,  "csv_source": "Valor Total", "parser": "money_br", "kind": "column",  "severity": "ok"}
    // ... demais colunas
  ]
}
```

**Semântica de `severity`:**

- **`ok`** — campo em dia (required+mapped OU optional+mapped). Não precisa ação.
- **`critical`** — **required+não mapeado**. Bloqueia o save. Spec será rejeitada se persistir. Destaque em vermelho na apresentação ao gestor.
- **`info`** — optional+não mapeado. A UI mostrará "—". Confirme com o gestor se a decisão foi consciente.

**Regra crítica:** `customer.reference` com `severity=critical` é o bug mais comum — todos os itens caem num mesmo Sale. Se aparecer, pergunte imediatamente qual coluna do CSV contém o identificador único do comprador/pedido.

**Além da `mapping_table`, o preview agora devolve cinco sinais de debug** que o gestor/agente pode usar sem voltar ao CSV:

- **`header_suggestions`** — para cada header ausente em `missing_headers`, sugere o header mais parecido do CSV (Levenshtein case+trim insensitive). Se você vê `{expected: "Total Pago", closest_csv_header: "total_pago", distance: 2}`, é quase certo que o gestor só precisa corrigir o nome na spec.
- **`unmapped_csv_columns`** — lista os headers do CSV que nenhuma expressão da spec referencia. Candidatos naturais a entrar em `raw_data` para auditoria. Apresente como "colunas no CSV que a spec ignora" e pergunte se deveriam ser preservadas.
- **`skipped_rows`** — array de até 20 exemplos `{line, reason, filter_index, filter_type, raw_line}` explicando POR QUÊ cada linha foi pulada. Se `skipped_count > 0`, sempre mostre pelo menos 1-2 exemplos para o gestor confirmar que é de propósito.
- **`errors`** — agora é um array de objetos `{line, message, raw_line}` (não mais strings flat). Quando um row explode, o gestor vê a linha exata do CSV que causou o problema.
- **`dedup_stats`** — `{items_before, items_after, duplicates}`. Quando > 0 duplicatas, mostre ao gestor: pode ser dedup legítimo OU dedup_key muito restritivo (ex: apenas `description`+`total_paid` sem `reference_date` colapsa duas vendas legítimas em dias diferentes).

Também use `sales` (array agrupado) para mostrar **quantas vendas vão ser criadas** (não só itens) — se o gestor espera 30 vendas e `sales_count` = 1, `customer.reference` ou `grouping_key` está errado.

**Nunca pule a apresentação da `mapping_table` + sinais de debug no primeiro preview.** É aqui que a maioria dos erros aparece.

### 6. Loop de preview (iterar até limpar)

```
preview_sale_source_definition(csv_content=<amostra>, spec=<spec_atual>)
```

**Critério de sucesso:**
- `status == "ok"` (tool computa automaticamente a partir de `missing_headers == []` e `errored_count == 0`)
- `sales_count` bate com o esperado pelo gestor
- Nenhuma linha com `severity=critical` na `mapping_table`
- `header_suggestions` vazio (ou todas aceitas e aplicadas na spec)
- `skipped_rows` só contém linhas que o gestor confirmou como "pra ignorar"
- `dedup_stats.duplicates` = 0 ou explicado

**Quando usar `sample_mode`:**
- **`head` (default)** — iteração normal. Primeiras N linhas do arquivo (max 100).
- **`stratified`** — quando suspeita que erros aparecem só no meio/fim do CSV. Amostra balanceada de início + meio + fim.
- **`full`** — **última rodada antes de salvar**. Processa até 500 linhas para pegar erros que a amostra ignorou.

Iterar: ajustar spec → re-chamar `preview_sale_source_definition` → repetir. Sem limite — esse é o único jeito de validar sem tocar em produção. Reporte brevemente a cada iteração: "Preview X — 12 vendas / 18 itens OK, 0 erros, 2 linhas ignoradas (totais no rodapé), mapping_table OK".

### 7. Confirmar e salvar

Antes de chamar `save_sale_source_definition`, apresente um resumo via `AskUserQuestion` **reincluindo a `mapping_table` do último preview** (a que o gestor validou). Renderize diretamente a partir do payload do MCP — não reescreva de cabeça:

```
Vou salvar esta fonte:
  • Nome:        Vendas Sistema XPTO
  • Key:         xpto_vendas
  • Operação:    criar nova | atualizar existente
  • Último preview (sample_mode=full, 250 linhas): 12 vendas, 18 itens OK, 0 erros, 0 skipped, dedup_stats 18→18
  • Mapeamento (mapping_table):

    Sale (sales):
      - Ref. Cliente       ← Pedido                        (severity=ok)
      - Cliente            ← Cliente                       (severity=ok)
      - CPF                —  não mapeado                  (severity=info)

    SaleItem (sale_items):
      - Item               ← Produto                       (severity=ok)
      - Fornecedor         —  não mapeado                  (severity=info)
      - Data de Referência ← Emissão       (parse date_br) (severity=ok)
      - Qtd                —  não mapeado                  (severity=info)
      - Valor Unit.        —  não mapeado                  (severity=info)
      - Valor Cobrado      ← Valor Total   (parse money_br)(severity=ok)
      - Forma Pgto         ← Forma Pagto                   (severity=ok)
      - Cancelado          —  computado via row_filter     (severity=ok)

[Salvar] [Revisar mais uma vez] [Cancelar]
```

Se confirmado:

```
save_sale_source_definition(key="xpto_vendas", name="Vendas Sistema XPTO", spec=<spec_final>)
```

A tool valida via `SpecValidator::validateDetailed`. Se a spec for inválida, o erro retornado agora carrega `[code] field: message — Dica: fix_hint` por erro, separado por ` | `. Exemplo: `[missing_required_field] fields.description: Campo obrigatório 'description' ausente. Dica: Declare fields.description: {'from': '<header>'}`. Use essa mensagem para corrigir sem precisar perguntar ao gestor.

### 8. Orientar o gestor e manter a rede de segurança

Depois de salva, a fonte **não importa automaticamente** — ela fica disponível em `Configurações > Fontes de Venda` para o gestor usar no fluxo normal. Diga: *"Fonte criada. Vá em Configurações > Fontes de Venda > Testar com CSV para um teste final com o arquivo completo, ou use o importador normal selecionando esta fonte."*

Se **logo após salvar** o gestor perceber que algo ficou errado (ex: mapeou a coluna errada de valor):

```
revert_sale_source_definition(key="xpto_vendas")
```

Essa tool lê o activity log e restaura a spec imediatamente anterior. Só reverte UMA versão por chamada — cada revert gera um novo entry no log, então é possível andar para trás passo a passo. Falha se não houver histórico de update (fonte recém-criada). **Nunca chame sem confirmação dupla com o gestor** — é escrita direta e sobrescreve a spec atual.

## Regras de ouro

- **A `mapping_table` do MCP é autoritativa.** Sempre renderize a partir do payload de `preview_sale_source_definition` — nunca reescreva de cabeça ou infira das chaves da spec. Isso elimina drift entre o que a UI realmente exibe e o que o gestor viu na conversa.
- **Destaque `severity=critical` e trate como bug.** Nenhum save com critical deve passar — o `SpecValidator` rejeita, mas o gestor precisa ver ANTES para corrigir a lógica da spec.
- **Apresentar a `mapping_table` + sinais de debug no primeiro preview E antes de cada save.** Não "pule porque parece óbvio".
- **Use `header_suggestions` antes de perguntar ao gestor.** Se o MCP já sugeriu o header certo, aplique a correção na spec e rode de novo — economiza uma rodada de conversa.
- **`sample_mode=full` na rodada final.** Head é suficiente para iterar, mas o último preview antes de salvar deveria passar `full` para pegar erros de linhas mais distantes.
- **Nunca invente chaves do DSL.** Se `get_sale_source_dsl_reference` não listou, não existe. `section=core` é suficiente para 90% dos casos — só peça `recipes` quando precisar de exemplos.
- **Preview é barato, preview sempre.** O loop é: preview → ajustar → preview. Não tente adivinhar a spec de primeira.
- **Mostre a spec final ao gestor antes de salvar** (mesmo que ele não entenda JSON — ele precisa ver nome, key e mapping_table renderizada).
- **Deleção só com confirmação dupla.** `delete_sale_source_definition` falha se há imports vinculados — se falhar, **não** tente forçar; explique que há histórico preso e sugira apenas desabilitar (`enabled=false` via save).
- **Revert é rede de segurança, não atalho.** Só chame `revert_sale_source_definition` quando o save imediatamente anterior ficou claramente errado e o gestor confirmou.
- **Admin-only.** Se o gestor não for admin, a tool vai falhar com 403 — explique o porquê ao invés de tentar contornar.

## Erros comuns e como resolver

| Sintoma no preview | Causa provável | Ação |
|---|---|---|
| `missing_headers: ["Total Pago"]` + `header_suggestions` com distância baixa | Header do CSV tem whitespace/case diferente (ex: `"Total Pago "` com espaço) | Aplicar a sugestão diretamente no `resolver` do campo correspondente |
| `status: "errors"` com items_count > 0 e errored_count > 0 | Alguns rows explodem por parse incorreto (decimal_br em CSV formato US) | Ler `errors[].raw_line` para identificar o row problemático, trocar `decimal_br` por `decimal_en` ou ajustar o default |
| `skipped_count` alto + `skipped_rows[].reason` mostra filter correto | `detection` ou `row_filters` muito restritivos | Relaxar filtro ou confirmar com gestor que linhas são para ignorar |
| `sales_count == 1` mas `items_count >> 1` | `grouping_key` ou `customer.reference` apontam para coluna constante | Investigar `mapping_table.sale` — se `customer.reference` tem `severity=critical`, resolver primeiro |
| `dedup_stats.duplicates > 0` inesperado | `dedup_key` muito frouxo (ex: só `description`+`total_paid` sem data) | Adicionar campo discriminador ao `dedup_key` |
| `unmapped_csv_columns` lista colunas importantes | Spec esquece de referenciar colunas úteis | Adicionar a `raw_data` para auditoria ou a `fields` se relevante |
| `save_sale_source_definition` retorna erro `[code]` estruturado | Spec falhou validação | Ler `code`, `field` e `fix_hint` na mensagem — já indica o conserto |
| `delete_sale_source_definition` retorna erro com contagem de imports | Existem `sale_imports` vinculados | **Não deletar** — sugerir desabilitar via `save_sale_source_definition(..., enabled=false)` |
| Save aceito mas gestor percebeu que ficou errado | Spec anterior perdida via update | `revert_sale_source_definition(key)` restaura imediatamente |

## Quando não usar esta skill

- Importar um CSV de fonte **já integrada** (Feegow, Yzidro, SIPAG, caixa): usar o fluxo normal de importação.
- Consertar categorização ou reconciliação de vendas já importadas: ver skills `categorization` e `reconciliation`.
- Alterar mapeamento de **uma única importação específica**: não dá — o mapeamento é por fonte, não por import. Corrigir a fonte e reimportar.
