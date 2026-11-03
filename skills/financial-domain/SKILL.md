---
name: financial-domain
description: "Contexto de domínio financeiro brasileiro que complementa as instruções do MCP PontoAlto: escala de confidence, regimes tributários, DRE por competência, DRE Contábil e impostos por regime (PIS/COFINS cumulativo × não cumulativo, IRPJ/CSLL anual ou trimestral, prejuízo fiscal), taxas de cartão (MDR e antecipação), repasses de adquirente de cartão e tipos de reconciliação."
version: 0.7.0
---

# PontoAlto — Domínio Financeiro

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
- **DRE por competência**: por padrão, relatórios usam `competence_date` (fallback: `reference_date`). Transação paga em março referente a fevereiro aparece em fevereiro no DRE.
- **Atenção ao parâmetro `basis`**: ele existe em duas tools com significados diferentes. Em `get_reports` escolhe o regime do DRE (`competencia` / `caixa`); em `get_cost_analysis` escolhe a data que posiciona a linha no período (`venda` / `producao`, ver skill `cost-analysis`). Não são a mesma escolha.
- **DRE por caixa**: `get_reports` aceita `basis=caixa` para gerar o DRE só por `reference_date` (quando o dinheiro entrou/saiu). Default é `basis=competencia`. Usar quando o gestor pedir "DRE por caixa" / "regime de caixa".
- **Competência vs. caixa**: competência = fato gerador (quando o custo ou receita ocorreu); caixa = quando o dinheiro efetivamente entrou/saiu.
- **Repasses de adquirente de cartão**: são **transferências**, não vendas. Nunca categorizar como receita — devem ser liquidados via `create_settlements`.
- **Tipos de reconciliação**:
  - OFX ↔ Transaction: extrato bancário bate com lançamento importado
  - Sale ↔ Transaction: venda do sistema origem (ERP, PDV ou outro) bate com recebimento bancário

## DRE Contábil e Impostos

`get_reports(report=dre_contabil)` é outro relatório que o `dre`: monta o resultado por competência com as notas fiscais classificadas, os lançamentos manuais e os impostos calculados pelo regime — o extrato só entra nas contas com fonte "extrato". Cada grupo traz `managerial_total` (o mesmo grupo no DRE gerencial): diferença entre os dois é esperada, não erro.

- **No contábil vale o imposto calculado, não a guia**: nas contas de imposto, o DARF/DAS/ISS pago no extrato não entra — entra o imposto do mês da nota, ou o valor apurado que o gestor registrou na tela. A guia continua no DRE gerencial e no fluxo de caixa. Nunca some a guia ao imposto do contábil
- **Sem perfil fiscal, sem imposto**: o aviso "Sem perfil fiscal vigente…" em `warnings` quer dizer que o DRE Contábil não calculou imposto nenhum naquele mês. Diga ao gestor que falta cadastrar o perfil fiscal; não estime o imposto pela guia paga
- **Repasse sempre os `warnings`**: notas a classificar, imposto sem categoria marcada, saldo de prejuízo fiscal não registrado, perfil no regime de caixa
- **Memorial**: cada item de `taxes` traz `base`, `rate`, `memo` (a conta passo a passo), `withheld` e `due_date`. `estimated=true` é estimativa (Lucro Real: o valor oficial vem do e-Lalur); `informed_return=true` é o valor apurado registrado pelo gestor
- **Marcações fiscais da categoria vão pela inbox**: `set_category_accounting` (ver abaixo). Valor apurado, saldo de prejuízo fiscal e perfil fiscal continuam só na tela (página DRE Contábil e cadastro do perfil fiscal) — oriente o gestor, não existe action para eles

### Marcar a categoria (`set_category_accounting`)

`list_categories(all=true)` traz em `accounting` as marcações **próprias** de cada categoria (sem `accounting` = tudo herdado ou padrão). Para mudar, uma sugestão por categoria, em `create_suggestion` modo batch:

```json
{
  "type": "category_accounting",
  "suggestable_type": "category",
  "suggestable_id": 67,
  "action": "set_category_accounting",
  "action_params": {"category_id": 67, "generates_pis_cofins_credit": true},
  "confidence": 90,
  "reasoning": "Contador confirmou o não cumulativo; energia elétrica gera crédito (Lei 10.833, art. 3º, III)."
}
```

| Campo | Valores | Para quê |
|---|---|---|
| `pis_cofins_cumulative` | `true` cumulativo, `false` não cumulativo, `null` herdar | Regime do PIS/COFINS da receita no Lucro Real |
| `generates_pis_cofins_credit` | bool | Custo que gera crédito (só compra de pessoa jurídica com documento) |
| `pis_cofins_credit_non_cumulative_only` | bool | Custo só da receita não cumulativa: crédito inteiro, sem rateio |
| `tax_kind` | `das` `iss` `pis` `cofins` `icms` `irpj` `csll` ou `null` | Categoria que recebe o imposto calculado (a guia do extrato deixa de somar) |
| `accounting_source` | `nota` `extrato` `calculado` `manual` `patrimonial` ou `null` | De onde o DRE Contábil tira o valor |
| `accounting_dre_group` | grupo do DRE ou `null` | Outro grupo só no contábil (ex.: MDR em despesas operacionais) |
| `is_payroll`, `generates_icms_credit` | bool | Folha (Fator R) e crédito de ICMS |
| `presumption_group` | `servicos` `hospitalar` ... ou `null` | Presunção do Lucro Presumido para receita sem nota |

- Só os campos informados mudam; `null` volta a herdar da categoria pai
- **A marcação desce para as subcategorias**, e o crédito ligado no pai não desliga na filha. Pai com filhas de natureza diferente (ex.: "Insumos e Materiais" com Medicamentos, que é monofásico e não gera crédito) → marque cada filha. O preview da inbox lista as subcategorias que herdam
- Regime e créditos são decisão do contador: confirme com ele antes de propor; no `reasoning`, cite a confirmação e o fundamento
- Depois de aprovar, confira em `get_reports(report=dre_contabil)` o memorial do PIS/COFINS (débito, créditos, saldo)

### Lucro Real

- **PIS/COFINS cumulativo × não cumulativo** (até 12/2026; a partir de 2027 dão lugar à CBS):
  - **Cumulativo (0,65% + 3%)**: a receita marcada na categoria ("PIS/COFINS no Lucro Real", herdada da categoria pai); sem marcação, nota dos serviços do art. 10, XIII, da Lei 10.833 — subitens 04.01 medicina, 04.02 exames e imagem, 04.03 hospital, clínica e laboratório, 04.08 fisioterapia e fonoaudiologia, 04.12 odontologia e 04.19 banco de sangue — e, em clínica, a receita sem nota
  - **Não cumulativo (1,65% + 7,6%)**: o resto, inclusive enfermagem, nutrição, psicologia, casa de repouso e planos de saúde, menos os créditos
  - **Créditos**: custos de categoria marcada "gera crédito de PIS/COFINS". Custo comum às duas receitas é rateado pela proporção da receita não cumulativa; custo marcado "só da receita não cumulativa" entra inteiro; o crédito que sobra passa para os meses seguintes, inclusive de um ano para o outro. Clínica com toda a receita no cumulativo praticamente não usa crédito — é normal
  - Obstetrícia (04.11), reprodução assistida (04.18) e coleta (04.20) dependem de quem presta: não são cumulativas automáticas. Se a clínica médica ou o laboratório presta, a categoria precisa da marcação de cumulativo
- **IRPJ/CSLL**: anual por estimativa (LAIR acumulado no ano, guia todo mês) ou trimestral, conforme o perfil fiscal (cada trimestre fecha sozinho; a guia vence no último dia útil do mês seguinte ao trimestre). A provisão do mês é o devido no acumulado menos o que já foi provisionado
- **Prejuízo fiscal de anos anteriores**: compensa até 30% do lucro com o saldo do início do ano (Parte B do e-Lalur), registrado pelo gestor na página do DRE Contábil, um por ano. No trimestral, o prejuízo de um trimestre soma ao saldo dos seguintes. O saldo não passa sozinho para o ano seguinte — o aviso "Saldo de prejuízo fiscal de AAAA não registrado" pede o do ano

### Taxas de cartão

- **MDR** (taxa da maquininha): fica nas deduções do DRE gerencial (custo variável de cada venda, entra na margem de contribuição) e vai para despesas operacionais no DRE Contábil (dedução contábil é só imposto sobre a venda, devolução e abatimento)
- **Antecipação de recebíveis**: Resultado Financeiro → Despesas Financeiras nos dois DREs — é decisão de caixa, não custo da venda. Categorize na conta de antecipação de Despesas Financeiras, nunca junto da MDR nas deduções

## Reforço: Exceções ao Modelo de Sugestões

O MCP deixa claro que escrita passa por sugestões. Estas exceções gravam direto, sem passar pelo inbox:

- `create_settlements` — liquida repasses de cartão em lote
- `create_cash_settlements` — cria recebíveis de vendas em dinheiro
- `save_sale_source_definition` — salva fonte de venda customizada (admin-only, com loop de preview antes do save)
- `delete_sale_source_definition` — apaga fonte de venda customizada (admin-only)
- `revert_sale_source_definition` — restaura a versão anterior da spec via activity log (admin-only, safety net)
- `save_project` / `delete_project` — cadastro de obras e etapas de incorporadora (admin-only)
- `save_cost_type` — cadastro de tipos de custo (admin-only)

A linha divisória é **cadastro vs dado financeiro**. Cadastrar uma obra é configuração administrativa e grava direto; **atribuir** essa obra a lançamentos altera o resultado contábil e passa pela inbox, via action `link_project`. Não confunda os dois — não existe tool de escrita direta para atribuir obra.

Nessas exceções, sempre peça confirmação explícita ao gestor antes de gravar. Nas demais operações: sempre via `create_suggestion` / `bulk_create_suggestions` / `create_suggestion_chain`. **A aprovação acontece manualmente pela UI do PontoAlto** — o plugin cria sugestões e para aí. Não chame `approve_suggestion`, `bulk_approve_suggestions` nem `confirm_approval` no CLI; o gestor revisa e aprova pela inbox visual.

## Encadeamento de Actions (Cadeias)

Quando uma operação precisa de N passos onde o step seguinte depende do anterior (ex: criar categoria nova → criar regra usando a categoria → aplicar categoria nas transações), use uma **cadeia de sugestões**. A inbox apresenta a cadeia como **um único card agrupado** com timeline numerada e dois botões ("Aceitar Tudo" / "Rejeitar Tudo") — o gestor aprova ou rejeita a sequência inteira, e a execução é atômica (ou tudo acontece, ou nada).

**Duas formas de criar cadeias:**

1. **Auto-chain (implícito) — `bulk_create_suggestions` com `create_rule: true`** — o par categorize+rule gerado para um mesmo grupo já nasce encadeado (`chain_id` compartilhado, rule executa antes da categorização). **Não precisa fazer nada diferente** — é o comportamento padrão do fluxo automático de categorização e de fornecedores (provider_groups).

2. **Explícito — `create_suggestion_chain(steps=[...])`** — use quando precisar encadear **criação de entidade nova** antes da regra/aplicação. Exemplos:
   - `create_category` (nova) → `create_categorization_rule` → `categorize_transaction`
   - `create_provider` (novo) → `create_provider_linking_rule` → `link_provider`
   - `create_categorization_rule` → `categorize_transaction` (quando categoria já existe mas é um caso específico que quer atomicidade)

**Referenciando IDs entre steps:** dentro de `action_params`, substitua campos que dependem de um ID ainda não criado por `"$chain.N.created_id"`, onde N é o `chain_order` (0-based) do step que cria a entidade. O servidor resolve em runtime, após executar step N.

```json
{
  "steps": [
    {
      "type": "create_rule",
      "suggestable_type": "transaction",
      "suggestable_id": <tx_id_âncora>,
      "action": "create_category",
      "action_params": {"name": "Materiais de Escritório", "type": "expense"},
      "confidence": 90,
      "reasoning": "Nova categoria para agrupar compras recorrentes de material."
    },
    {
      "type": "create_rule",
      "suggestable_type": "transaction",
      "suggestable_id": <tx_id_âncora>,
      "action": "create_categorization_rule",
      "action_params": {
        "pattern": "MATERIAIS ESCRIT",
        "rule_type": "contains",
        "category_id": "$chain.0.created_id"
      },
      "confidence": 90,
      "reasoning": "Auto-categorizar compras futuras deste padrão."
    },
    {
      "type": "categorize",
      "suggestable_type": "transaction",
      "suggestable_id": <tx_id_âncora>,
      "action": "categorize_transaction",
      "action_params": {
        "transaction_ids": [101, 102, 103],
        "category_id": "$chain.0.created_id"
      },
      "confidence": 90,
      "reasoning": "Aplicar a nova categoria nas 3 transações pendentes."
    }
  ]
}
```

**Regras da cadeia:**
- Entre 2 e 5 steps
- Referência `$chain.N.X` deve apontar para step anterior (`N < order atual`) cuja action seja de criação (retorna `created_id`): `create_category`, `create_provider`, `create_categorization_rule`, `create_provider_linking_rule`, `create_competence_rule`
- Cada step precisa de `suggestable_type`/`suggestable_id` válidos — para steps de criação, use uma transação **âncora** representativa (mesma convenção do `bulk_create_suggestions`)
- Undo reverte a cadeia inteira em ordem inversa (desfaz categorização → deleta regra → deleta categoria)

**Quando NÃO usar cadeia:**
- Se a categoria/fornecedor **já existe** e você só precisa criar regra+aplicar, `bulk_create_suggestions` com `create_rule: true` resolve (auto-chain)
- Se for uma operação isolada (só categorizar sem regra), `create_suggestion` modo singular ou batch

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

Ambas seguem o fluxo padrão de sugestões: o plugin cria a sugestão e o gestor aprova manualmente pela UI do PontoAlto (não chame `approve_suggestion`/`confirm_approval` no CLI). Quando propor cada uma e como embasar o reasoning está detalhado nas skills `categorization` § Atualizar/Desativar Regras e `provider-management` § Atualizar/Desativar Regras.

## Tools pesadas e concorrência

`get_cost_analysis`, `analyze_unreconciled_sales`, `get_reports`, `get_budget_comparison` e `get_ltv_analysis` varrem meses de vendas, agenda ou lançamentos. O servidor executa no máximo **2 delas ao mesmo tempo** (entre todos os tenants e usuários) e devolve `Servidor ocupado, tente novamente em alguns segundos` para quem espera mais de 20 s por um slot.

- Chame tools pesadas **em sequência**, nunca várias em paralelo.
- Views de `get_cost_analysis` do mesmo período compartilham cache (10 min): peça a primeira, espere, depois as outras.
- `get_ltv_analysis` não tem cache e varre todo o histórico de vendas a cada chamada: prefira uma chamada com `limit` maior a várias pequenas.
- Ao operar vários tenants (sweep), no máximo **2 tenants por vez**.
- Ao receber `Servidor ocupado`, aguarde 10 s e repita a mesma chamada **uma vez**. Se repetir o erro, siga o fluxo e informe o gestor.

## Erro em tool MCP

Logar o erro e seguir para o próximo item do fluxo. Não bloquear o workflow inteiro por uma tool que falhou em uma transação ou grupo.
