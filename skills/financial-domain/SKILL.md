---
name: financial-domain
description: "Contexto de domínio financeiro brasileiro que complementa as instruções do MCP Ponto Alto: escala de confidence, regimes tributários, DRE por competência, repasses SIPAG e tipos de reconciliação."
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
- **Repasses SIPAG (cartão)**: são **transferências**, não vendas. Nunca categorizar como receita — devem ser liquidados via `create_settlements`.
- **Tipos de reconciliação**:
  - OFX ↔ Transaction: extrato bancário bate com lançamento importado
  - Sale ↔ Transaction: venda do sistema origem (Feegow, etc) bate com recebimento bancário

## Reforço: Exceções ao Modelo de Sugestões

O MCP deixa claro que escrita passa por sugestões. **Duas exceções** gravam direto, sem passar pelo inbox:

- `create_settlements` — liquida repasses de cartão em lote
- `create_cash_settlements` — cria recebíveis de vendas em dinheiro

Nas demais operações: sempre via `create_suggestion` / `bulk_create_suggestions` → aprovação → `confirm_approval(token)`.

## Erro em tool MCP

Logar o erro e seguir para o próximo item do fluxo. Não bloquear o workflow inteiro por uma tool que falhou em uma transação ou grupo.
