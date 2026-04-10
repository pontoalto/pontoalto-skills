---
name: sale-sources
description: "Guia para montar, testar e salvar definições de fontes de venda customizadas (custom CSV importers) via MCP, sem precisar acessar o código do sistema. Cobre o DSL declarativo, o loop de preview iterativo, a mapping_table devolvida pelo MCP e as exceções ao modelo de sugestões."
version: 0.2.0
---

# Ponto Alto — Fontes de Venda Customizadas

O Ponto Alto importa vendas de múltiplas origens (Feegow, Yzidro, caixa físico, operadoras de cartão). Quando o gestor tem um CSV de um sistema que **ainda não está integrado**, ele pode criar uma **Fonte de Venda customizada**: uma definição declarativa (spec JSON) que ensina o importador a interpretar aquele layout específico — sem precisar alterar o código do PontoAlto.

Esta skill descreve o fluxo de ponta a ponta para o gestor que **não tem acesso ao código** e vai montar a fonte inteiramente conversando com o Claude via MCP.

## Quando usar

- O gestor tem um CSV de vendas de um sistema novo (ex: um ERP vertical, um relatório exportado de planilha) e quer importá-lo no PontoAlto
- Uma fonte existente parou de funcionar porque o fornecedor do sistema-origem mudou o layout do CSV
- O gestor quer entender qual fonte está ativa e como ela mapeia os campos

> **Pré-requisito:** o gestor precisa ter perfil `admin` no tenant. Fontes de venda **não** ficam disponíveis para `financeiro` ou `viewer`.

## Tools MCP envolvidas

Todas no prefixo do servidor escolhido (`mcp__claude_ai_Ponto_Alto__` ou `mcp__pontoalto-local__`):

| Tool | Tipo | Papel no fluxo |
|------|------|----------------|
| `list_sale_source_definitions` | leitura | Ver o que já existe antes de criar nova |
| `get_sale_source_definition` | leitura | Inspecionar uma spec existente (útil como referência) |
| `get_sale_source_dsl_reference` | leitura | **Referência completa do DSL** — resolvers, parsers, modifiers, operators, row filters, enums, recipes + `ui_columns_reference` (9 colunas padrão da UI) |
| `get_sale_source_spec_template` | leitura | Template inicial (`minimal`, `feegow_like`, `yzidro_like`) com placeholders |
| `preview_sale_source_definition` | leitura | Testa a spec contra um CSV amostra **sem persistir nada** — retorna headers, itens parseados, erros, linhas ignoradas **e `mapping_table`** (spec → colunas da UI) |
| `save_sale_source_definition` | **escrita direta** | Cria ou atualiza (upsert por `key`). Não passa pela inbox |
| `delete_sale_source_definition` | **escrita direta** | Remove a fonte. Falha se houver importações vinculadas |

### ⚠️ Exceção ao modelo de sugestões

`save_sale_source_definition` e `delete_sale_source_definition` são **escrita direta, sem sugestão na inbox**. Alinhado com a skill `financial-domain` § Exceções. Isso existe porque o ciclo de iteração (preview → ajustar → preview → salvar) ficaria inviável se cada alteração precisasse de aprovação manual.

**Compensação:** sempre mostrar ao gestor um resumo do que vai salvar (nome, key, número de campos mapeados, se é create ou update) e pedir confirmação via `AskUserQuestion` antes de chamar `save_sale_source_definition`.

## Fluxo recomendado (8 passos)

### 1. Dimensionar o que já existe

```
list_sale_source_definitions()
```

Mostrar ao gestor quais fontes estão ativas/desativadas. Se a fonte que ele quer já existe e só está desabilitada, talvez o caminho seja **editar**, não criar nova.

### 2. Coletar amostra do CSV

Pedir ao gestor para **colar o conteúdo do CSV** na conversa (primeiras 10-30 linhas já bastam para montar e testar). Evitar anexar o arquivo inteiro de produção — amostra enxuta basta para validar o parse.

Perguntar via `AskUserQuestion`:
- **Nome** (ex: "Vendas Sistema XPTO") e **key** (slug, ex: `xpto_vendas`)
- **Separador** do CSV (`,`, `;`, `\t`)
- **Linha do header** (geralmente 0, mas alguns exports têm metadados nas primeiras linhas)

### 3. Aprender o DSL antes de escrever

```
get_sale_source_dsl_reference()
```

Essa é a **fonte única da verdade** do DSL. Ela lista tudo que o interpretador entende: resolvers (`column`, `literal`, `concat`), parsers (`decimal_br`, `date_br`, `date_iso`), modifiers (`trim`, `upper`, `default`), operators para `detection` e `row_filters`, enums válidos para `payment_method`, e "recipes" com exemplos prontos.

**Nunca invente chaves.** Se a ref não lista, não existe.

### 4. Partir de um template

```
get_sale_source_spec_template(style="feegow_like" | "yzidro_like" | "minimal")
```

Escolher o template mais próximo do CSV do gestor:
- `feegow_like` — 1 linha = 1 item de venda, cliente nomeado, valor total explícito
- `yzidro_like` — múltiplas linhas por venda (agrupadas por `grouping_key`), cada linha um item
- `minimal` — esqueleto vazio, quando nenhum dos outros encaixa

O template vem com placeholders (`<HEADER_DESCRICAO>`, `<HEADER_DATA>`, etc.) — substituir pelos headers reais do CSV do gestor.

### 5. Primeiro preview + apresentar a `mapping_table`

**Obrigatório** — antes de qualquer discussão sobre ajustes, fazer o primeiro `preview_sale_source_definition` e apresentar ao gestor a `mapping_table` devolvida pelo MCP. Esse campo é **autoridade**: listado pelo servidor, ele descreve exatamente o que a UI de importação do PontoAlto vai exibir quando essa spec rodar de verdade. Nunca inventar a tabela de cabeça — sempre renderizar a partir do que o MCP retornou.

```
preview_sale_source_definition(csv_content=<amostra>, spec=<spec_atual>)
```

O retorno tem um campo `mapping_table` — array com uma entrada por coluna padrão da UI, na ordem canônica:

```jsonc
[
  {"ui_column": "DESCRIÇÃO",  "spec_field": "fields.description",    "required": true,  "mapped": true,  "csv_source": "Produto",     "parser": null,        "kind": "column"},
  {"ui_column": "FORNECEDOR", "spec_field": "fields.provider_name",  "required": false, "mapped": false, "csv_source": null,         "parser": null,        "kind": null},
  {"ui_column": "CLIENTE",    "spec_field": "customer.name",         "required": true,  "mapped": true,  "csv_source": "Cliente",     "parser": null,        "kind": "column"},
  {"ui_column": "DATA REF.",  "spec_field": "fields.reference_date", "required": true,  "mapped": true,  "csv_source": "Emissão",     "parser": "date_br",   "kind": "column"},
  {"ui_column": "QTD",        "spec_field": "fields.quantity",       "required": false, "mapped": false, "csv_source": null,         "parser": null,        "kind": null},
  {"ui_column": "UNITÁRIO",   "spec_field": "fields.unit_price",     "required": false, "mapped": false, "csv_source": null,         "parser": null,        "kind": null},
  {"ui_column": "TOTAL",      "spec_field": "fields.total_paid",     "required": true,  "mapped": true,  "csv_source": "Valor Total", "parser": "money_br",  "kind": "column"},
  {"ui_column": "PAGAMENTO",  "spec_field": "fields.payment_method", "required": false, "mapped": true,  "csv_source": "Forma Pagto", "parser": null,        "kind": "column"},
  {"ui_column": "CANCELADA",  "spec_field": "fields.is_cancelled",   "required": false, "mapped": false, "csv_source": null,         "parser": null,        "kind": null}
]
```

Semântica dos campos:

- `mapped` — `false` significa que a coluna vai aparecer **vazia (`—`)** na UI de importação
- `csv_source` — quando `kind=column`, o header exato do CSV de onde o valor vem
- `parser` — parser aplicado (`decimal_br`, `money_br`, `date_br`, etc.) — `null` quando não há parse
- `kind` — `column` (from CSV), `literal` (valor fixo), `computed` (concat / if / operator / setting) ou `null` quando não mapeado
- `required` — se `SpecValidator` rejeita spec sem esse campo mapeado

Renderizar a `mapping_table` para o gestor como uma tabela legível e, em seguida, perguntar via `AskUserQuestion`:

- Para cada linha com `mapped=false`, é de propósito? Ou existe a coluna no CSV e eu deveria mapear?
- Colunas com `kind=computed` (ex: `CANCELADA` via `any` operator) — o gestor entende que o valor é derivado de uma regra, não de uma coluna direta?
- Algum campo não-padrão que ele quer preservar vai pra `raw_data`?

**Nunca pular essa etapa.** Mesmo que o template escolhido no passo 4 pareça "completo", a `mapping_table` torna explícito o que a UI vai exibir — incluindo campos que o template deixou de fora.

### 6. Loop de preview (iterar até limpar)

```
preview_sale_source_definition(csv_content=<amostra>, spec=<spec_atual>)
```

Analisar o retorno:
- `headers` — o que o parser enxergou
- `missing_headers` — obrigatórios que não bateram (maioria dos erros vem daqui)
- `items_count` / `items` — quantidade e amostra de itens extraídos
- `skipped_count` — linhas puladas por `row_filters`
- `errors` — erros de parse em campos (ex: decimal inválido)
- `mapping_table` — recompute-se a cada chamada; se o gestor alterou a spec em resposta ao passo 5, revisar a tabela antes de seguir

**Critério de sucesso:** `missing_headers == []`, `errors == []`, `items_count` bate com o esperado e a `mapping_table` reflete decisões conscientes (nenhum `mapped=false` inesperado). Se `skipped_count > 0`, confirmar com o gestor que as linhas ignoradas são mesmo para ignorar (ex: totais no rodapé, linhas vazias).

Iterar: ajustar a spec → re-chamar `preview_sale_source_definition` → repetir. Sem limite — esse é o único jeito de validar sem tocar em produção.

### 7. Confirmar e salvar

Antes de chamar `save_sale_source_definition`, apresentar um resumo via `AskUserQuestion` **reincluindo a `mapping_table` do último preview** (a que o gestor validou). Renderizar diretamente a partir do payload do MCP — não reescrever de cabeça:

```
Vou salvar esta fonte:
  • Nome: Vendas Sistema XPTO
  • Key: xpto_vendas
  • Operação: criar nova
  • Último preview: 18 itens OK, 0 erros, 0 linhas ignoradas
  • Mapeamento (da mapping_table do último preview):
      - DESCRIÇÃO   ← Produto          (column)
      - FORNECEDOR  —  não mapeado
      - CLIENTE     ← Cliente          (column)
      - DATA REF.   ← Emissão          (column, parse: date_br)
      - QTD         —  não mapeado
      - UNITÁRIO    —  não mapeado
      - TOTAL       ← Valor Total      (column, parse: money_br)
      - PAGAMENTO   ← Forma Pagto      (column)
      - CANCELADA   —  computado por row_filter (confirmado no passo 5)

[Salvar] [Revisar mais uma vez] [Cancelar]
```

Se confirmado:

```
save_sale_source_definition(key="xpto_vendas", name="Vendas Sistema XPTO", spec=<spec_final>)
```

### 8. Orientar o gestor para o próximo passo

Depois de salva, a fonte **não importa automaticamente** — ela fica disponível na tela `Configurações > Fontes de Venda` do PontoAlto para o gestor usar no fluxo normal de importação de CSV. Explicitar isso: "Fonte criada. Agora vá em **Configurações > Fontes de Venda > Testar com CSV** para um teste final com o arquivo completo, ou use o importador normal de vendas selecionando esta fonte."

## Regras de ouro

- **A `mapping_table` do MCP é a fonte autoritativa.** Sempre renderizar a tabela apresentada ao gestor diretamente do payload de `preview_sale_source_definition` — nunca reescrever de cabeça ou inferir das chaves da spec. Isso elimina drift entre o que a UI realmente exibe e o que o gestor viu na conversa
- **Apresentar a `mapping_table` ao gestor no primeiro preview E antes de cada save.** Linhas com `mapped=false` precisam ser decisão consciente — nunca pular "porque parece óbvio". Se o gestor corrige um `mapped=false` inesperado, re-rodar o preview
- **Nunca editar uma fonte de produção sem preview antes.** `save_sale_source_definition` é upsert direto — sobrescreve sem histórico via MCP
- **Nunca inventar chaves do DSL.** Se `get_sale_source_dsl_reference` não listou, não existe. A seção `ui_columns_reference` desse mesmo retorno lista as 9 colunas padrão da UI
- **Preview é barato, preview sempre.** O loop é: preview → ajustar → preview. Não tentar adivinhar a spec de primeira
- **Mostrar a spec final ao gestor antes de salvar** (mesmo que ele não entenda JSON — ele precisa ver nome, key, e a `mapping_table` renderizada)
- **Deleção só com confirmação dupla.** `delete_sale_source_definition` falha se há imports vinculados — se falhar, **não** tentar forçar; explicar que há histórico preso e sugerir apenas desabilitar (`enabled=false` via save)
- **Admin-only.** Se o gestor não for admin, a tool vai falhar com 403 — explicar o porquê ao invés de tentar contornar

## Erros comuns e como resolver

| Sintoma | Causa provável | Ação |
|---------|---------------|------|
| `missing_headers: ["total"]` | Header do CSV real é diferente do placeholder (ex: "Valor Total" vs. "total") | Ajustar o `resolver` do campo `fields.total_paid` para apontar ao header real |
| `errors: ["decimal_br: '1,234.56'"]` | CSV usa formato americano, parser espera BR | Trocar parser de `decimal_br` para `decimal_en` |
| `skipped_count` alto, `skipped_reasons: ["row_filter"]` | `detection` ou `row_filters` muito restritivos | Relaxar filtro ou confirmar com gestor que linhas são para ignorar mesmo |
| `save_sale_source_definition` retorna 422 | Spec falhou validação (chave desconhecida, obrigatório faltando) | Ler a mensagem de erro, revisitar `get_sale_source_dsl_reference`, corrigir |
| `delete_sale_source_definition` retorna 409 | Existem `sale_imports` vinculados a essa fonte | **Não deletar** — sugerir desabilitar via `save_sale_source_definition(..., enabled=false)` |

## Quando não usar esta skill

- Importar um CSV de fonte **já integrada** (Feegow, Yzidro, SIPAG, caixa): usar o fluxo normal de importação, não criar fonte nova
- Consertar categorização ou reconciliação de vendas já importadas: ver skills `categorization` e `reconciliation`
- Alterar mapeamento de **uma única importação específica**: não dá — o mapeamento é por fonte, não por import. Corrigir a fonte e reimportar
