---
name: categorization
description: "Fluxo completo de categorização de transações no PontoAlto: automático (analyze_uncategorized + suggest_category + bulk_create_suggestions), consulta (listar para WhatsApp) e manual (gestor informa categoria)."
version: 0.1.0
---

# Categorização de Lançamentos

Transações importadas chegam sem categoria. O gestor precisa categorizar todas.

**Como verificar:** `transaction_stats` → `count_uncategorized`. Se > 0, há trabalho.

**OBRIGATÓRIO — Formato WhatsApp:** quando listar transações para o gestor consultar a equipe, use **texto simples** — formato `N. DD/MM — Nome — R$ Valor`, sem tabelas markdown, pipes (`|`) ou negrito. Ver § Fluxo consulta.

**OBRIGATÓRIO — Regras de categorização:** ao criar sugestões via `bulk_create_suggestions`, SEMPRE incluir `create_rule: true` e `rule_type: "contains"` em cada grupo/supplier_group. Exceções (ver Princípio 2 abaixo): pattern já existe como regra (`list_rules`) ou pattern é ambíguo (usado em categorias diferentes).

**Se o gestor não responder:** marcar etapa como "aguardando resposta" e avançar para a próxima etapa. Retomar quando o gestor fornecer as respostas.

## Princípios

1. **Uma sugestão por grupo** — transações do mesmo destinatário/pattern vão em UMA sugestão com `transaction_ids: [id1, id2, ...]`
2. **Sempre criar regra junto** — ao categorizar, incluir `create_rule` (`rule_type: "contains"`). Exceções:
   - **Duplicata**: `list_rules` já contém regra com mesmo pattern e categoria
   - **Ambíguo**: pattern já existe em regra de OUTRA categoria (verificar via `list_rules(search=pattern)`). Neste caso: categorizar sem regra, informar o gestor
3. **Verificar duplicatas** — `list_suggestions(status=pending)` antes de criar, para não duplicar

## Fluxo automático (analyze_uncategorized)

1. `transaction_stats` → dimensionar o problema (quantas sem categoria)
2. `list_rules` → guardar regras existentes
3. `list_suggestions(status=pending)` → guardar IDs com sugestão pendente
4. `analyze_uncategorized(min_group_size=1)` → grupos por padrão de descrição
5. Para grupos sem `suggested_category` ou confiança baixa: `suggest_category`
6. Sempre criar regra junto com a categorização (ver Princípio 2 para exceções)
7. `bulk_create_suggestions` uma única vez com todos os grupos no array `groups`
8. Reportar: total por categoria + patterns ambíguos onde regra NÃO foi criada

> **Tool choice:** `bulk_create_suggestions` aceita `groups` (output de `analyze_uncategorized`) — usar no fluxo automático. `create_suggestion` modo batch (param `suggestions`) — usar no fluxo manual quando os grupos não vieram de `analyze_uncategorized`.

## Fluxo consulta (listar para perguntar ao responsável)

Quando há transações sem categoria que não têm sugestão automática (ex: PIX para pessoas físicas com nomes variados), o gestor precisa perguntar ao pagador/responsável o que é cada uma.

1. `analyze_uncategorized(min_group_size=1)` → coletar todos os IDs
2. `list_transactions` com os IDs → buscar detalhes completos (até 30 transações por vez)
3. Apresentar em **texto simples** (sem tabela markdown) — pronto para copiar e colar no WhatsApp:

```
*[Tenant] — Pix sem categoria (lista N)*
Faltam estes pra fechar [mês]. Me diga a que se refere:

1. *JOAO DA SILVA* — R$ 49,90 (25/03)

2. *Pedro Alves* — R$ 814,00 total
   • R$ 148,00 (24/03) + R$ 666,00 (28/03)

3. *MARIA SOUZA / MARIA DE SOUZA* — R$ 3.150,00 total
   • R$ 1.500,00 (15/03) + R$ 1.650,00 (27/03)
```

**Formatação WhatsApp (asterisco = negrito nativo do WhatsApp, não markdown):**
- Cabeçalho: `*[Tenant] — Pix sem categoria (lista N)*`
- Linha de contexto: `Faltam estes pra fechar [mês]. Me diga a que se refere:`
- Transação única: `N. *NOME* — R$ Valor (DD/MM)`
- Múltiplas tx do mesmo destinatário: `N. *NOME* — R$ total total` + sub-item `   • R$ X (DD/MM) + R$ Y (DD/MM)`
- Variação de nome: separar com `/` dentro do mesmo item — ex: `*MARIA SOUZA / MARIA DE SOUZA*`
- NÃO usar: tabelas markdown, pipes (`|`), negrito markdown (`**`), links

4. **Sempre buscar detalhes de TODAS as transações** — não listar IDs sem nome/valor
5. Ordenar por data (mais recente primeiro) ou valor (maior primeiro, conforme contexto)
6. Após resposta do gestor, seguir o **Fluxo manual** abaixo
7. **Se o gestor não responder:** não bloquear o workflow — avançar para a próxima etapa e retomar a categorização quando houver resposta

## Fluxo manual (gestor informa a categoria)

Quando o gestor responde com a categoria (ex: via WhatsApp):

1. `list_rules` + `list_suggestions(status=pending)` → preparar contexto
2. Agrupar por destinatário/pattern
3. `create_suggestion` modo batch — cada item com `transaction_ids: [...]` agrupados
4. Sempre incluir sugestão `create_rule` junto com a categorização, exceto se o pattern é ambíguo (aparece em categorias diferentes no histórico)
5. Reportar: total por categoria + patterns ambíguos onde regra NÃO foi criada
