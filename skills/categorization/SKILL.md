---
name: categorization
description: "Fluxo completo de categorização de transações no PontoAlto: automático (analyze_uncategorized + suggest_category + bulk_create_suggestions), consulta (listar para WhatsApp) e manual (gestor informa categoria)."
version: 0.1.0
---

# Categorização de Lançamentos

Transações importadas chegam sem categoria. O gestor precisa categorizar todas.

**Como verificar:** `transaction_stats` → `count_uncategorized`. Se > 0, há trabalho.

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
Transações sem categoria — 03/2026

1. 28/03 — Edson Francisco Filho — R$ 666,00
2. 27/03 — Wallas Lima — R$ 1.650,00
3. 25/03 — José Aparecido dos Santos — R$ 49,90

Total: 3 transações — R$ 2.365,90
```

**Formatação WhatsApp:**
- Uma linha por transação: `N. DD/MM — Nome — R$ Valor`
- Total no final
- Separar grupos distintos com linha em branco e subtítulo
- NÃO usar: tabelas markdown, pipes (|), negrito (**), links

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
