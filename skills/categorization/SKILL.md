---
name: categorization
description: "Fluxo completo de categorização de transações no PontoAlto: automático (analyze_uncategorized + suggest_category + bulk_create_suggestions), consulta (listar para WhatsApp) e manual (gestor informa categoria)."
version: 0.2.0
---

# Categorização de Lançamentos

Transações importadas chegam sem categoria. O gestor precisa categorizar todas.

**Como verificar:** `transaction_stats` → `count_uncategorized`. Se > 0, há trabalho.

**OBRIGATÓRIO — Formato WhatsApp:** quando listar transações para o gestor consultar a equipe, use **texto simples** — formato `N. DD/MM — Nome — R$ Valor`, sem tabelas markdown, pipes (`|`) ou negrito. Ver § Fluxo consulta.

**OBRIGATÓRIO — Regras de categorização:** ao criar sugestões via `bulk_create_suggestions`, SEMPRE incluir `create_rule: true` e `rule_type: "contains"` em cada grupo/supplier_group. Exceções (ver Princípio 2 abaixo): pattern já existe como regra (`list_rules`) ou pattern é ambíguo (usado em categorias diferentes).

**Se o gestor não responder:** marcar etapa como "aguardando resposta" e avançar para a próxima etapa. Retomar quando o gestor fornecer as respostas.

## Princípios

1. **Uma sugestão por grupo** — transações do mesmo destinatário/pattern vão em UMA sugestão com `transaction_ids: [id1, id2, ...]`
2. **Sempre criar regra junto** — ao categorizar, incluir `create_rule` (`rule_type: "contains"`). O par (regra + categorização) é criado automaticamente como **cadeia** pelo `bulk_create_suggestions`, aparecendo na inbox como um único card agrupado que é aceito ou rejeitado atomicamente. Exceções:
   - **Duplicata exata**: `list_rules` já contém regra com mesmo pattern **e** mesma categoria — não cria.
   - **Conflito resolvível**: pattern já existe em regra de OUTRA categoria, **mas o histórico recente mostra que a categoria correta agora é a nova**. Propor `update_rule` na regra antiga (mudando `category_id`) ou `delete_rule` (soft-delete, marca `is_active=false`) via `create_suggestion` — o gestor aprova pela inbox. Ver § Atualizar/Desativar Regras abaixo.
   - **Ambíguo não resolvível**: pattern casa múltiplas categorias no histórico sem padrão claro. Categorizar sem regra e avisar o gestor.
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

> **Tool choice:** `bulk_create_suggestions` aceita `groups` (output de `analyze_uncategorized`) — usar no fluxo automático. Cria automaticamente uma **cadeia** (regra + categorização) para cada grupo com `create_rule: true`. `create_suggestion` modo batch (param `suggestions`) — usar no fluxo manual quando os grupos não vieram de `analyze_uncategorized`. `create_suggestion_chain` — usar quando precisar criar **categoria nova** como parte do fluxo (ver § Criar Categoria Nova em Cadeia abaixo).

## Criar Categoria Nova em Cadeia

Quando o gestor pede pra categorizar transações em uma categoria que **ainda não existe** no sistema (ou quando você identifica que um grupo precisa de categoria nova), use `create_suggestion_chain` para encadear atomicamente: criar categoria → criar regra → categorizar.

**Quando usar:**
- Grupo com padrão claro (ex: 15 transações "PROVISAO GASTO CART CRED") que não cabe em nenhuma categoria existente → nova categoria "Provisão Cartão Crédito"
- Gestor explicitamente pede nova categoria na resposta do fluxo manual

**Quando NÃO usar:**
- Categoria já existe → `bulk_create_suggestions` resolve (auto-chain)
- Gestor pede "só categorizar isso, sem regra" → `create_suggestion` singular

**Fluxo:**
1. `list_categories` → confirmar que a categoria não existe (evitar duplicata)
2. Montar cadeia de 3 steps:
   - `create_category` (step 0) — nome, `type: expense|revenue|both`, opcional `parent_id`, `dre_group`
   - `create_categorization_rule` (step 1) — pattern do grupo, `rule_type: "contains"`, `category_id: "$chain.0.created_id"`
   - `categorize_transaction` (step 2) — `transaction_ids: [...]`, `category_id: "$chain.0.created_id"`
3. Chamar `create_suggestion_chain(steps=[...])`
4. Reportar: "Cadeia criada — gestor aprova no inbox e os 3 passos executam atomicamente."

**suggestable âncora:** para os 3 steps, use `suggestable_type: "transaction"` + `suggestable_id` = primeiro ID do grupo (mesma convenção do `bulk_create_suggestions`). A transação âncora não precisa ser alvo da ação; é só identificador.

**Exemplo completo:** ver skill `financial-domain` § Encadeamento de Actions.

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
2. `list_categories` → **obter `category_id` a partir do nome** que o gestor forneceu. O gestor fala por nome ("Marketing", "Aluguel"), mas a action exige ID. Faça match case-insensitive e, se ambíguo (ex: "Aluguel" vs "Aluguel Equipamentos"), confirme via `AskUserQuestion` antes de criar a sugestão
3. Agrupar por destinatário/pattern
4. `create_suggestion` modo batch — cada item com `transaction_ids: [...]` agrupados
5. Sempre incluir sugestão `create_rule` junto com a categorização, exceto se o pattern é ambíguo (aparece em categorias diferentes no histórico)
6. Reportar: total por categoria + patterns ambíguos onde regra NÃO foi criada

## Drill-down em transação individual (`get_transaction`)

Quando uma transação está difícil de categorizar mesmo depois de `suggest_category` e `analyze_uncategorized`, use `get_transaction(id)` para inspecionar o payload completo: `raw_description` (muitas vezes mais informativo que `description`), metadata do OFX, histórico de enriquecimento PIX (nome completo + CPF mascarado do pagador), tentativas anteriores de regra e `label` atual. É a tool certa para casos isolados — não use para listar em massa (use `list_transactions`).

## Atualizar/Desativar Regras (update_rule / delete_rule)

Quando `list_rules` mostrar uma regra que **não deveria mais estar criando efeito** — categoria errada detectada pelo histórico recente, fornecedor encerrado, pattern obsoleto — propor a correção via sugestão. Não ignora nem cria regra nova por cima.

**Quando propor `update_rule`:**
- Regra existente com **categoria errada** e o histórico de transações dos últimos 60 dias aponta consistentemente para outra categoria — propor mudança de `category_id`.
- Regra com prioridade baixa demais sendo suplantada por outra — propor `priority` maior.
- Regra regex quebrada — propor `pattern` corrigido.

**Quando propor `delete_rule`:**
- Regra ativa que **não casou nenhuma transação** no período analisado E o fornecedor/pattern deixou de aparecer nos extratos (fornecedor encerrado, conta trocada).
- Regra duplicada por acidente (mesmo pattern em categorias diferentes) — desativar a errada, manter a certa.

**Como propor (via inbox, como qualquer outra ação):**

```
create_suggestion({
  type: "general",
  // A sugestão aponta para a PRÓPRIA REGRA como suggestable — não uma transação "âncora".
  // Isso garante dedup natural (uma update_rule pendente por regra) e descrição correta no inbox.
  suggestable_type: "rule_categorization",  // rule_categorization | rule_provider | rule_competence
  suggestable_id: <id_da_regra>,            // DEVE ser igual a action_params.rule_id
  action: "update_rule",             // ou "delete_rule"
  action_params: {
    rule_type: "categorization",     // discriminador: categorization | competence | provider
                                      // DEVE casar com suggestable_type (rule_<rule_type>)
    rule_id: <id_da_regra>,           // DEVE ser igual a suggestable_id
    // Apenas os campos a alterar:
    category_id: 42,                  // opcional
    rule_match_type: "starts_with",   // opcional — nova match style (exact/starts_with/contains/regex)
    priority: 100,                    // opcional
    is_active: true                   // opcional — para reativar uma regra desativada
  },
  confidence: 85,
  reasoning: "Regra atual aponta para 'Transporte' mas histórico de 2026-02/03 mostra 8 transações categorizadas como 'Marketing'. Corrigindo."
})
```

**Mapeamento rule_type → suggestable_type:**
- `rule_type: "categorization"` → `suggestable_type: "rule_categorization"` (regras de `CategoryLearningRule`)
- `rule_type: "provider"` → `suggestable_type: "rule_provider"` (regras de `ProviderLinkingRule`)
- `rule_type: "competence"` → `suggestable_type: "rule_competence"` (regras de `CompetenceDateRule`)

O servidor rejeita a sugestão se `suggestable_type`/`suggestable_id` não casarem com `action_params.rule_type`/`rule_id`. Para `create_categorization_rule` / `create_provider_linking_rule` / `create_competence_rule` (criação de regra nova), continua usando `suggestable_type: "transaction"` apontando para uma transação representativa do padrão — a restrição de morph só vale para `update_rule`/`delete_rule`.

**Atenção — semântica de `delete_rule`:** é **soft-delete** (marca `is_active=false`, mantém histórico). Para reativar, chame `update_rule` passando `is_active: true`. O gestor nunca perde a regra — só para de aplicá-la.

**Antes de propor:** rode `list_rules(search=<pattern>)` para inspecionar a regra atual e confirmar que você não está pedindo ao gestor uma mudança sem base. O reasoning deve citar **dados concretos** (ex: "8 transações em 2026-03 / pattern parou de casar desde 2026-01").
