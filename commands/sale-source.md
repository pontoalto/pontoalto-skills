---
description: "Monta, testa e salva uma fonte de venda customizada (custom CSV) via DSL, usando preview iterativo — para gestores sem acesso ao código do sistema."
argument-hint: "[--local] [nome ou key da fonte]"
---

# Ponto Alto — Fonte de Venda Customizada

Atalho para montar ou ajustar uma **Fonte de Venda** customizada. Use quando o gestor tem um CSV de um sistema ainda não integrado (ex: um ERP vertical, uma planilha exportada) e quer ensinar o PontoAlto a importar aquele layout.

Responda em português. **Todo o passo-a-passo, a forma de apresentar a `mapping_table`, as regras de ouro e a tabela de erros estão na skill `sale-sources`** — este arquivo é só o ponto de entrada. Também consulte `financial-domain` para o contexto de domínio (incluindo a exceção ao modelo de sugestões).

## MCP Server

Argumento recebido: `$ARGUMENTS`

- Sem `--local`: tools com prefixo `mcp__claude_ai_Ponto_Alto__` (produção)
- Com `--local`: tools com prefixo `mcp__pontoalto-local__` (desenvolvimento)

`--local` é flag. Qualquer outra palavra no argumento é tratada como **nome ou key da fonte** que o gestor quer editar (se já existir). Se nada for encontrado, assumir criação de nova fonte.

## Inicialização

1. `list_tenants` — se 2+, perguntar qual usar via `AskUserQuestion`; se 1, usar automaticamente.
2. Confirmar conexão: nome do tenant + organização.
3. Verificar que o usuário é admin — se a primeira tool retornar 403, parar e explicar que esta operação exige perfil `admin`.

## O que fazer agora

Siga o fluxo completo descrito em `sale-sources/SKILL.md`. Ele cobre:

- Dimensionamento via `list_sale_source_definitions` (com `last_used_at` e `sale_imports_count` para decidir entre editar vs. criar).
- Coleta de amostra do CSV, nome, key e separador.
- Carregamento do DSL via `get_sale_source_dsl_reference` (default `section=core` — compacto; passe `section=recipes` quando precisar de exemplos prontos).
- Escolha do template via `get_sale_source_spec_template` (estilos `minimal`, `basic_1to1`, `grouped`). Para perfis mais ricos, use `get_sale_source_definition` em uma fonte já cadastrada como ponto de partida.
- Loop de preview via `preview_sale_source_definition` analisando `mapping_table` (com `severity`), `header_suggestions`, `skipped_rows`, `errors` estruturados, `dedup_stats` e `unmapped_csv_columns`.
- Confirmação dupla + `save_sale_source_definition`.
- Safety net: `revert_sale_source_definition` restaura a versão anterior se algo deu errado.

**Não reescreva o fluxo aqui.** Se for preciso ajustar, edite `sale-sources/SKILL.md` (fonte única).

## Regras de ouro (espelhadas da skill para referência rápida)

- **Preview sempre.** Nunca salvar sem pelo menos uma rodada de `preview_sale_source_definition` com `status=ok`.
- **A `mapping_table` do MCP é autoritativa** — renderize sempre a partir do payload, nunca de cabeça. Atenção redobrada em linhas com `severity=critical`.
- **Nunca invente chaves do DSL.** Se `get_sale_source_dsl_reference` não listar, não existe.
- **Confirmação dupla antes de salvar ou deletar.** `save_sale_source_definition` e `delete_sale_source_definition` são escrita direta.
- **Admin-only.** Sem perfil admin, nenhuma tool desta família funciona.
- **Se algo der errado depois de salvar**, chame `revert_sale_source_definition(key)` — ele lê o activity log e restaura a spec anterior.
