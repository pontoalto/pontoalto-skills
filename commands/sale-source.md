---
description: "Monta, testa e salva uma fonte de venda customizada (custom CSV) via DSL, usando preview iterativo — para gestores sem acesso ao código do sistema."
argument-hint: "[--local] [nome ou key da fonte]"
---

# Ponto Alto — Fonte de Venda Customizada

Atalho para montar ou ajustar uma **Fonte de Venda** customizada. Use quando o gestor tem um CSV de um sistema ainda não integrado (ex: um ERP vertical, uma planilha exportada) e quer ensinar o PontoAlto a importar aquele layout.

Responda em português. Use a skill `sale-sources` para o fluxo detalhado e `financial-domain` para o contexto de domínio (incluindo a exceção ao modelo de sugestões).

## MCP Server

Argumento recebido: `$ARGUMENTS`

- Sem `--local`: tools com prefixo `mcp__claude_ai_Ponto_Alto__` (produção)
- Com `--local`: tools com prefixo `mcp__pontoalto-local__` (desenvolvimento)

`--local` é flag. Qualquer outra palavra no argumento é tratada como **nome ou key da fonte** que o gestor quer editar (se já existir) — se nada for encontrado, assumir criação de nova fonte.

## Inicialização

1. `list_tenants` — se 2+, perguntar qual usar via `AskUserQuestion`; se 1, usar automaticamente
2. Confirmar conexão: nome do tenant + organização
3. Verificar que o usuário é admin — se a primeira tool retornar 403, parar e explicar que esta operação exige perfil `admin`

## Dimensionamento (sempre executar primeiro)

```
list_sale_source_definitions()    → ver fontes existentes
```

Apresentar a lista ao gestor. Se o argumento livre bate com uma fonte existente (match parcial por nome ou key), oferecer via `AskUserQuestion`:

1. **Editar a fonte existente** — carrega spec atual via `get_sale_source_definition`
2. **Criar uma nova fonte** — parte de um template

Se não há match, ir direto para criação.

## Coleta de contexto

Antes de escrever qualquer spec, pedir ao gestor:

- **Conteúdo de amostra do CSV** (colar na conversa — 10 a 30 linhas bastam)
- **Nome** da fonte (ex: "Vendas Sistema XPTO")
- **Key** (slug único, ex: `xpto_vendas`) — sugerir a partir do nome
- **Separador** do CSV (`AskUserQuestion`: `,` / `;` / tab / outro)
- **Linha do header** (default 0; perguntar apenas se a amostra sugerir metadados no topo)

Se o gestor mencionar um formato conhecido ("parecido com Feegow", "parecido com Yzidro"), registrar para usar o template adequado no próximo passo.

## Montagem da spec

```
get_sale_source_dsl_reference()                    → aprender o DSL (fonte única da verdade)
get_sale_source_spec_template(style=...)           → baseline: feegow_like, yzidro_like ou minimal
```

Substituir placeholders pelos headers reais do CSV do gestor. **Não inventar chaves** — se a referência do DSL não lista, não existe.

## Loop de preview (iterar)

```
preview_sale_source_definition(csv_content=<amostra>, spec=<spec_atual>)
```

Analisar: `headers`, `missing_headers`, `items_count`, `skipped_count`, `errors`, **`mapping_table`**.

**Primeira chamada:** apresentar a `mapping_table` ao gestor — ela lista as 9 colunas padrão da UI (DESCRIÇÃO, FORNECEDOR, CLIENTE, DATA REF., QTD, UNITÁRIO, TOTAL, PAGAMENTO, CANCELADA) com `mapped`, `csv_source`, `parser` e `kind`. Colunas com `mapped=false` aparecem vazias na UI de importação — confirmar com o gestor se é decisão consciente.

**Critério de sucesso:** `missing_headers == []`, `errors == []`, `items_count` bate com o esperado, e nenhum `mapped=false` inesperado na `mapping_table`.

Iterar sem limite. Reportar brevemente a cada iteração: "Preview X — 18 itens OK, 0 erros, 2 linhas ignoradas (totais no rodapé), mapping_table OK".

Se o gestor indicar que está tudo certo, avançar para salvar. Se pedir ajustes, iterar mais — lembrando que cada preview recalcula a `mapping_table`.

> Detalhe: a skill `sale-sources` tem o formato completo da apresentação da `mapping_table` e o que perguntar ao gestor em cada iteração.

## Confirmar e salvar

Antes de `save_sale_source_definition`, apresentar um **resumo de commit** via `AskUserQuestion`, **reincluindo a `mapping_table`** do último preview (renderizada a partir do payload do MCP — não de cabeça):

```
Vou salvar:
  • Nome:       Vendas Sistema XPTO
  • Key:        xpto_vendas
  • Operação:   criar nova | atualizar existente
  • Preview OK: 18 itens, 0 erros, 0 linhas ignoradas
  • Mapeamento (mapping_table):
      - DESCRIÇÃO   ← Produto
      - FORNECEDOR  —  não mapeado
      - CLIENTE     ← Cliente
      - DATA REF.   ← Emissão (date_br)
      - QTD         —  não mapeado
      - UNITÁRIO    —  não mapeado
      - TOTAL       ← Valor Total (money_br)
      - PAGAMENTO   ← Forma Pagto
      - CANCELADA   —  computado via row_filter

[Salvar] [Revisar mais uma vez] [Cancelar]
```

Só chamar `save_sale_source_definition` após confirmação explícita. Essa tool é **escrita direta, sem passar pela inbox** — documentado na skill `sale-sources`.

## Encerramento

Após salvar, explicar ao gestor:

1. A fonte já está disponível na tela **Configurações > Fontes de Venda** do PontoAlto
2. Recomendado: fazer um teste final com o CSV completo via "Testar com CSV" no próprio PontoAlto
3. Depois disso, usar o importador normal de vendas e selecionar esta fonte

## Regras de Ouro

- **Preview sempre.** Nunca salvar sem pelo menos uma rodada de `preview_sale_source_definition` limpa
- **Confirmação dupla antes de salvar ou deletar.** `save_sale_source_definition` e `delete_sale_source_definition` são escrita direta
- **Nunca forçar deleção.** Se `delete_sale_source_definition` falhar por imports vinculados, sugerir apenas desabilitar (`enabled=false` via save)
- **Admin-only.** Sem perfil admin, nenhuma tool desta família funciona — parar e explicar, não tentar contornar
- **Priorizar por impacto:** se editando uma fonte em produção, alertar o gestor sobre o volume de imports históricos afetados antes de alterar
