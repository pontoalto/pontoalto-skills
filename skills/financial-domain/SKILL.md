---
name: financial-domain
description: "Contexto de domínio financeiro brasileiro que complementa as instruções do MCP Ponto Alto: escala de confidence, regimes tributários, DRE por competência, repasses de adquirente de cartão e tipos de reconciliação."
version: 0.2.0
---

# Ponto Alto — Domínio Financeiro

Complemento de domínio ao que o MCP server já define em `instructions` (convenções de output, modelo de escrita via sugestões, actions disponíveis). Este documento cobre o que o MCP **não** diz e o Claude precisa saber para operar bem.

## Escala de Confidence (0-100)

Usada para decidir se cria sugestão, apresenta ao gestor ou descarta.

| Faixa  | Ação padrão                                    | Exemplo |
|--------|-----------------------------------------------|---------|
| 95+    | Criar sugestão, aprovar em lote                | Valor idêntico + mesma data + mesmo nome |
| 85-94  | Criar sugestão, aprovar individual             | Valor exato + ±1 dia, contains claro de provider |
| 70-84  | Criar sugestão, apresentar ao gestor           | Padrão parcial, abreviação, output de `suggest_category` |
| 55-69  | Advisory/alerta — **não criar** sugestão       | Anomalias, possíveis duplicatas, matches fracos |
| <55    | Ignorar                                       | Incerteza alta demais para propor ação |

## Contexto Financeiro Brasileiro

- **Regimes tributários** (por tenant): Lucro Real, Lucro Presumido ou Simples Nacional — adaptar terminologia e análise ao regime configurado.
- **Segmentos atendidos**: clínicas, comércio, incorporadoras, serviços. A mesma tool pode precisar de framing diferente por segmento.
- **DRE por competência**: relatórios usam `competence_date` (fallback: `reference_date`). Transação paga em março referente a fevereiro aparece em fevereiro no DRE.
- **Competência vs. caixa**: competência = fato gerador (quando o custo ou receita ocorreu); caixa = quando o dinheiro efetivamente entrou/saiu.
- **Repasses de adquirente de cartão**: são **transferências**, não vendas. Nunca categorizar como receita — devem ser liquidados via `create_settlements`.
- **Tipos de reconciliação**:
  - OFX ↔ Transaction: extrato bancário bate com lançamento importado
  - Sale ↔ Transaction: venda do sistema origem (ERP, PDV ou outro) bate com recebimento bancário

## Reforço: Exceções ao Modelo de Sugestões

O MCP deixa claro que escrita passa por sugestões. **Duas exceções** gravam direto, sem passar pelo inbox:

- `create_settlements` — liquida repasses de cartão em lote
- `create_cash_settlements` — cria recebíveis de vendas em dinheiro

Nas demais operações: sempre via `create_suggestion` / `bulk_create_suggestions`. **A aprovação acontece manualmente pela UI do PontoAlto** — o plugin cria sugestões e para aí. Não chame `approve_suggestion`, `bulk_approve_suggestions` nem `confirm_approval` no CLI; o gestor revisa e aprova pela inbox visual.

## Higiene de Inbox (reject / delete)

Antes de rodar um sweep grande (categorização, fornecedores, conciliação), verifique o estado da inbox com `list_suggestions(status=pending)`. Duas ferramentas para limpar pendentes obsoletos:

- **`reject_suggestion(id)`** — marca uma sugestão como rejeitada mantendo histórico. Use quando a sugestão **foi analisada e está errada** (ex: match de fornecedor que o gestor confirmou como incorreto). O registro fica na inbox com `status=rejected` e serve de sinal para não recriar o mesmo pattern.
- **`delete_suggestions(ids=[...])`** ou **`delete_suggestions(all_pending=true)`** — remove fisicamente da inbox. Use para **limpeza em massa** de sugestões obsoletas (ex: sweep anterior criou 80 sugestões de uma área e o gestor pediu para refazer do zero).

**Regra prática:** se a sugestão estava certa mas virou obsoleta por novo sweep → `delete_suggestions`. Se estava errada e o gestor quer histórico do erro → `reject_suggestion`. Nunca delete sugestões `status=accepted` — isso apaga rastro de ações executadas.

**Quando aplicar:** sempre rode `list_suggestions(status=pending)` no diagnóstico inicial de qualquer command. Se o total de pendentes for alto (> 50) e você está prestes a criar mais em massa, ofereça ao gestor via `AskUserQuestion` limpar as antigas antes (deletar `all_pending` da área específica) ou acumular.

## Manutenção de Regras (update_rule / delete_rule)

Além de `create_categorization_rule`, `create_provider_linking_rule` e `create_competence_rule`, o MCP expõe duas actions para **manter** regras existentes:

- **`update_rule`** — altera pattern, match style, categoria, fornecedor, prioridade, scope (`transaction_type`), ou status (`is_active`). Uso principal: corrigir regras antigas quando o histórico recente mostra padrão diferente.
- **`delete_rule`** — **soft-delete**: marca `is_active=false` e mantém histórico. Para reativar, chamar `update_rule` com `is_active: true`. Nunca há perda permanente de regra via MCP.

**Payload compartilhado:**
- `rule_type` (discriminador): `"categorization"` | `"competence"` | `"provider"`
- `rule_id`: id da regra (use `list_rules` para obter)
- Campos opcionais: `pattern`, `rule_match_type` (≠ do discriminador! é a match style `exact`/`starts_with`/`contains`/`regex`), `category_id`, `provider_id`, `transaction_type`, `priority`, `is_active`, `offset_months`, `offset_days`, `day_of_month`

**Suggestable obrigatório:** para `update_rule`/`delete_rule`, a sugestão deve apontar para a **própria regra** como suggestable — não uma transação/venda âncora. Use `suggestable_type: "rule_categorization"` | `"rule_provider"` | `"rule_competence"` (casando com `action_params.rule_type`) e `suggestable_id` igual ao `rule_id`. Isso dá dedup natural (uma `update_rule` pendente por regra) e descrição correta no inbox. O servidor rejeita mismatch entre morph e `action_params`.

**Validações aplicadas automaticamente pelo MCP no update:**
- Regex compila (se `rule_match_type=regex` ou a regra já é regex e o pattern mudou)
- Duplicata: rejeita se pattern+match+scope bate com outra regra do mesmo tipo (self-exclusion)
- Compatibilidade de categoria: `transaction_type=debit` exige `category.type ∈ {expense, both}`; `credit` exige `{revenue, both}`
- Categoria ativa: rejeita categoria com `is_active=false`

Ambas seguem o fluxo padrão de sugestões (inbox → `approve_suggestion` → `confirm_approval`). Quando propor cada uma e como embasar o reasoning está detalhado nas skills `categorization` § Atualizar/Desativar Regras e `supplier-management` § Atualizar/Desativar Regras.

## Erro em tool MCP

Logar o erro e seguir para o próximo item do fluxo. Não bloquear o workflow inteiro por uma tool que falhou em uma transação ou grupo.
