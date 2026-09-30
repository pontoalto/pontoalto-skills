---
name: ltv-analysis
description: "LTV (lifetime value) no PontoAlto: get_ltv_analysis por médico, por médico × especialidade (clínica) ou por cliente (negócio de produto), com a margem da cesta inteira do paciente (custo, margem bruta, premissas e margem de contribuição por paciente e por visita). Três visões de atribuição ao médico (maior gasto, primeiro médico, separado), identidade do cliente, lifetime vs período, recompra, onde investir em marketing e clientes recorrentes que pararam de comprar. Só leitura."
version: 0.5.0
---

# LTV — Lifetime Value

## O que o LTV mede — e o que ele não mede

O LTV **não sai do extrato bancário**. Ele lê as **vendas importadas** (`SaleItem` dos itens não cancelados) e soma o `total_paid` de cada paciente ou cliente ao longo do tempo. Consequências:

- **LTV é receita; a margem vem ao lado.** Toda linha traz também o custo de tudo o que os pacientes dela compraram — laboratório, imagem e exames de outros executantes inclusos — e daí a margem bruta e a de contribuição, em valor, em % e por paciente/visita. É essa margem, não o LTV, que diz quanto o paciente deixa na clínica (veja § Margem)
- **Não depende de categorização nem de conciliação** — só das importações de vendas. Mês sem venda importada é mês sem LTV
- **O histórico começa na primeira importação.** "Lifetime" é desde que o tenant começou a importar vendas no PontoAlto: paciente antigo aparece como novo e o LTV dele fica subestimado. Se o gestor estranhar números baixos, confira com ele desde quando as vendas estão no sistema

## As dimensões

`get_ltv_analysis` agrega por **médico**, por **médico × especialidade** ou por **cliente**. O padrão acompanha a tela: `medico` para clínica, `cliente` para negócio de produto. A resposta devolve o `group_by` usado e, por médico, o `attribution` — sempre informe os dois ao gestor.

### Por médico (`group_by=medico`)

Responde "qual médico traz pacientes que voltam, gastam mais e deixam mais margem ao longo da vida?" — ou, na visão separada, "quanto cada médico produz".

- **Atribuição**: o parâmetro `attribution` decide a qual médico cada item vai (§ Visões de atribuição). Sem ele, `maior_gasto`
- **A tela abre diferente da tool**: em "Por médico" + `separado`, ordenada por MC/paciente e escondendo linhas com menos de 5 pacientes. Para bater com o que o gestor está vendo, chame com `attribution=separado`, `sort_by=contribution_margin_per_patient` e `min_patients=5`. A tela também tem a visão "Por especialidade" (uma linha por especialidade, somando os médicos); na tool, use `group_by=medico_especialidade` e some as linhas da mesma especialidade — em `separado`, sem somar pacientes, que podem se repetir entre médicos
- **Identidade do paciente**: `customer_reference` (prontuário). Venda sem prontuário conta como **paciente novo a cada venda** — a frequência dele é sempre 1,0 e o LTV vira o ticket. Se um médico tem muitos pacientes e `frequency` colada em 1,0, desconfie de prontuário faltando na importação antes de concluir que ele não fideliza
- **`last_visit`**: data da venda mais recente com item **do próprio médico** — não a última compra dos pacientes atribuídos a ele. É o que diz se o médico ainda atende na clínica, e é a mesma nas três visões

### Visões de atribuição (`attribution`, só por médico)

| Visão | Regra | Responde |
|-------|-------|----------|
| `maior_gasto` (padrão) | Paciente **inteiro** no profissional com quem mais gastou no recorte (empate → ordem alfabética) | "De quem é o paciente?" |
| `primeiro_medico` | Paciente **inteiro** no médico da **primeira consulta**; sem consulta, no primeiro profissional que o atendeu. Mais de um no mesmo dia → o de maior valor naquele dia | "Quem trouxe o paciente?" — a visão para marketing |
| `separado` | Cada item no profissional do **próprio item**. Item sem profissional vai para o médico da consulta da mesma venda → senão quem atendeu na venda → senão a última consulta → senão a próxima → senão `(sem médico)`. Consulta sem profissional fica em `(sem médico)` — consulta não é pedida por outro médico | "Quanto cada médico produz, e quanto o paciente gasta com ele?" |

- **Consulta** é o item cuja descrição começa com `CONSULTA` e que tem profissional. Primeira, última e próxima consulta olham só consultas: o laboratório pago junto com a consulta e um ultrassom foi pedido na consulta, não pelo ultrassonografista
- **Com período**, primeira e última consulta olham o histórico do paciente até o `to`, inclusive antes do `from` — a consulta de junho continua dona do exame de julho. Nada depois do `to` entra, então um período fechado não muda quando chegam vendas novas
- **Só a atribuição muda**: receita, custo, margens, pacientes e visitas do resumo são os mesmos nas três visões; só `doctor_count` muda. A soma das receitas das linhas sempre bate com a receita total
- **Em `maior_gasto` e `primeiro_medico`** cada paciente aparece em uma linha só, e a cesta inteira dele (exames, produtos, itens de outros médicos) vai junto, com o custo. Por isso a receita do médico **não é a produção dele**
- **Em `separado`** o paciente conta em **cada médico que o atendeu**: a soma de `patient_count` das linhas passa do resumo — nunca some pacientes entre linhas. O LTV da linha é "quanto o paciente gasta com esse médico", não o valor total do paciente. Também não é o `get_cost_analysis(view=by_provider)`: os itens sem profissional (laboratório, imagem) entram no médico que os pediu
- **Profissional que não faz consulta** (ultrassonografista, fisioterapia, grupos como "Exames Cardiologicos") pode ficar com muitos pacientes em `maior_gasto`, porque faz o item caro. Em `primeiro_medico` só fica com quem nunca passou por consulta; em `separado`, só com a própria produção. Se o gestor estranhar um desses no topo, mostre a mesma pergunta em `primeiro_medico`
- **`(sem médico)`**: em `maior_gasto` e `primeiro_medico`, pacientes sem **nenhum** item com profissional — tipicamente quem só fez exame de laboratório. Em `separado`, também os itens sem profissional de pacientes que nunca passaram por consulta. Não é erro de cadastro por si só; se for grande, vale investigar (consulta sem profissional na importação da agenda, por exemplo)
- **Comparar visões** custa uma chamada por visão — a tool não tem cache. Faça em sequência, com o mesmo escopo
- `attribution` com `group_by=cliente` volta erro: cliente não é atribuído a médico

### Por médico × especialidade (`group_by=medico_especialidade`)

Responde "o médico X ganha mais como geriatra ou como clínico geral?". Mesmas colunas do modo médico, mais `especialidade`.

- A especialidade vem do **item de consulta**: descrição começando com `CONSULTA`, sem o prefixo e sem a modalidade no fim — "CONSULTA GERIATRIA PRESENCIAL" → `Geriatria`
- Dentro do médico a que foi atribuído, o paciente vai para **uma especialidade só**: a das consultas em que mais gastou com esse médico. Um médico que atende duas especialidades vira duas linhas, sem contar paciente duas vezes dentro dele — a soma das linhas bate com o modo médico na mesma visão
- Aceita `attribution` como o modo médico. Em `separado` a especialidade é decidida por paciente × médico: o mesmo paciente pode ser Geriatria num médico e Cardiologia em outro
- Paciente sem consulta com o médico no recorte herda a especialidade do médico quando ele só atende uma; com mais de uma, cai em **`(sem especialidade)`** — comum quando o paciente só fez exame ou procedimento com ele, ou a consulta ficou fora do período
- **Profissional que não faz consulta** (ultrassonografista, ecocardiograma, fisioterapia, grupos como "Exames Cardiologicos") leva como especialidade o **grupo do procedimento** em que mais fatura — o `Grupo` do Feegow: "Ultrassonografia Geral", "Fisioterapia", "Exames Cardiologicos". Fonte de venda sem grupo deixa esses profissionais em `(sem especialidade)`
- **`(sem médico)`** é sempre uma linha só, com `(sem especialidade)`: sem médico, dividir por especialidade só espalharia poucos pacientes em muitas linhas
- O resumo é o mesmo do modo médico

### Por cliente (`group_by=cliente`)

Responde "quem são meus melhores clientes?" e "quem parou de comprar?".

- **Identidade**: CPF quando existe; senão o nome (sem diferenciar maiúsculas); senão a referência do cliente. Mesmo cliente digitado com nomes diferentes e sem CPF vira dois clientes — LTV subestimado e recompra subnotificada
- Clínica também pode pedir `group_by=cliente` (ranking de pacientes). Ali a identidade é CPF/nome, não prontuário: as contagens podem diferir das do modo médico

## Métricas

**Por médico** (e por médico × especialidade) — linhas:

| Campo           | Significa |
|-----------------|-----------|
| `patient_count` | Pacientes distintos atribuídos ao médico (em `separado`, os que ele atendeu — a soma das linhas passa do resumo) |
| `visit_count`   | Vendas distintas desses pacientes |
| `frequency`     | Visitas por paciente (`visit_count ÷ patient_count`) |
| `avg_ticket`    | Receita por visita |
| `revenue`       | Receita total dos pacientes do médico |
| `ltv`           | Receita por paciente (`revenue ÷ patient_count` = ticket × frequência) |
| `last_visit`    | Última venda com item do próprio médico (`YYYY-MM-DD` — apresente como DD/MM/YYYY) |

Mais os campos de margem (§ Margem) e, por paciente e por visita: `gross_margin_per_patient`, `contribution_margin_per_patient`, `gross_margin_per_visit`, `contribution_margin_per_visit`.

Resumo: `total_revenue`, `patient_count` (distintos no conjunto inteiro), `visit_count`, `doctor_count`, `avg_ltv`, `avg_ticket`, os totais de margem, `premises_pct`, `avg_contribution_margin_per_patient` e `avg_contribution_margin_per_visit`.

**Por cliente** — linhas:

| Campo           | Significa |
|-----------------|-----------|
| `orders`        | Pedidos (vendas) distintos do cliente |
| `avg_ticket`    | Receita por pedido |
| `revenue` / `ltv` | Receita total do cliente — aqui os dois são o mesmo número |
| `last_purchase` | Data da última compra (`YYYY-MM-DD` — apresente como DD/MM/YYYY) |

Mais os campos de margem e, por pedido: `gross_margin_per_order`, `contribution_margin_per_order`. Em cliente a `contribution_margin` da linha já é "quanto esse cliente deixou".

Resumo: `total_revenue`, `customer_count`, `order_count`, `repeat_customers` (clientes com 2+ pedidos), `repeat_rate` (% recorrentes), `avg_ltv`, `avg_ticket`, `avg_orders`, os totais de margem, `premises_pct`, `avg_contribution_margin_per_customer` e `avg_contribution_margin_per_order`.

O **resumo é sempre do conjunto inteiro**: `search`, `min_orders`, `min_patients` e `active_since` filtram só as linhas (e o `total` de linhas), nunca o resumo.

## Margem

Campos em toda linha (e somados no resumo):

| Campo | Significa |
|-------|-----------|
| `total_cost` | Custo de todos os itens da linha |
| `gross_margin` / `gross_margin_pct` | Receita − custo; % sobre a receita |
| `premises` | Receita × `premises_pct` (premissas ativas do LTV — veja abaixo) |
| `contribution_margin` / `contribution_margin_pct` | Margem bruta − premissas; % sobre a receita |
| `revenue_without_cost` / `revenue_without_cost_pct` | Receita de itens sem custo cadastrado (custo R$ 0,00) — quanto da margem está inflada |

- **É o mesmo custo da `get_cost_analysis` na base venda**: mesma vigência (data da venda), histórico do executante antes do padrão, fator de unidade e quantidade, e pagamento dividido em duas formas contado como um procedimento só. Com `from`/`to`, a soma de `revenue` e de `gross_margin` das linhas bate com `get_cost_analysis(view=summary, basis=venda)` do mesmo período. **Não bate com a base produção**, que é o padrão da `get_cost_analysis` em clínica — ao conferir, passe `basis=venda`
- **Nunca calcule a margem do médico à mão** (LTV × `margin_pct` de `get_cost_analysis(view=by_provider)`). O `by_provider` mede só a produção do próprio médico; o paciente que ele traz compra também laboratório, imagem e exames de outros executantes, com margens muito diferentes. A margem certa já vem na linha
- **Para decidir onde investir, compare `contribution_margin_per_patient`** (ganho por paciente trazido) e `contribution_margin_per_visit` (ganho por atendimento), não o LTV: médico de LTV alto pode trazer paciente de cesta com margem baixa
- **As premissas são do LTV**: o gestor edita na tela do LTV (barra de premissas → Editar premissas). Enquanto não salvar as próprias, valem as da Análise de Custos; o botão "Copiar da Análise de Custos" traz as de lá de novo. `summary.premises_pct` diz o total usado — não assuma que é o mesmo da `get_cost_analysis`
- `premises_pct` em 0% → margem de contribuição igual à bruta; avise que o LTV está sem premissas e que elas se configuram na tela do LTV (ou copiando da Análise de Custos)

## Escopo: lifetime vs período

- **Sem `from`/`to`** → histórico completo (`scope: lifetime`). É o LTV de verdade e o padrão
- **Com `from` e `to`** (os dois, `YYYY-MM-DD`) → `scope: periodo`, filtrado pela data da venda. O número passa a ser **receita (e margem) por paciente no período**, não lifetime

Recorte de um mês quase sempre dá `frequency` perto de 1,0 e LTV ≈ ticket — pouco informativo. Para tendência, compare janelas longas e iguais (ano contra ano, 12 meses contra os 12 anteriores), nunca um mês contra o lifetime. Sempre diga ao gestor qual escopo o número mede.

## Filtros de linha

- `search`: nome do médico, da especialidade (em `medico_especialidade`) ou do cliente, sem diferenciar maiúsculas
- `min_patients` (só médico e médico × especialidade): esconde linhas com menos pacientes — base pequena distorce a margem por paciente. Use 10 como ponto de partida
- `active_since` (só médico e médico × especialidade, `YYYY-MM-DD`): só linhas com `last_visit` a partir da data — tira quem saiu da clínica. A importação pode estar atrasada: tome como "hoje" a maior `last_visit` que a tool devolver e recue daí (90 dias é um bom padrão)
- `min_orders` (só cliente): mínimo de pedidos — 2 = só recorrentes

Filtro da dimensão errada volta erro (ex.: `min_patients` em cliente).

## Receitas por pergunta

| Pergunta do gestor | Chamada |
|--------------------|---------|
| Onde investir em marketing? Qual médico traz o paciente que mais deixa margem? | `get_ltv_analysis(group_by=medico, attribution=primeiro_medico, sort_by=contribution_margin_per_patient, min_patients=10, active_since=...)` |
| Qual médico dá mais ganho por atendimento? | idem com `sort_by=contribution_margin_per_visit` |
| Quanto cada médico produz, contando os exames que pede? | `get_ltv_analysis(group_by=medico, attribution=separado, sort_by=revenue)` |
| Por que um grupo de exames ou um ultrassonografista aparece com tantos pacientes? | a mesma chamada em `maior_gasto` e depois em `primeiro_medico`, e compare `patient_count` |
| O médico X ganha mais em qual especialidade? | `get_ltv_analysis(group_by=medico_especialidade, search="nome")` |
| Qual especialidade dá mais margem por paciente? | `get_ltv_analysis(group_by=medico_especialidade, sort_by=contribution_margin_per_patient, min_patients=10)` e agrupe as linhas por `especialidade` somando receita, margem e pacientes |
| Quais médicos trazem os pacientes mais valiosos (em receita)? | `get_ltv_analysis(group_by=medico, attribution=primeiro_medico)` — já vem por `ltv` decrescente |
| Qual médico mais fideliza? | `get_ltv_analysis(group_by=medico, attribution=separado, sort_by=frequency)` — retornos ao próprio médico |
| Qual médico atende mais pacientes? | `get_ltv_analysis(group_by=medico, attribution=separado, sort_by=patient_count)` |
| Quem são meus melhores clientes? | `get_ltv_analysis(group_by=cliente, sort_by=contribution_margin, limit=20)` — por margem; por receita, o padrão (`ltv`) |
| Quanto da base volta a comprar? | `summary.repeat_rate` e `summary.avg_orders` do modo cliente |
| Clientes recorrentes que pararam de comprar | `get_ltv_analysis(group_by=cliente, min_orders=2, sort_by=last_purchase, direction=asc)` |
| Como está o médico X / o cliente Y? | `search="nome"` na dimensão certa |
| O LTV deste ano está melhor que o do ano passado? | duas chamadas com `from`/`to` de cada ano, uma depois da outra |

`sort_by` aceita só os campos da dimensão:

- médico e médico × especialidade: `ltv`, `revenue`, `patient_count`, `visit_count`, `frequency`, `avg_ticket`, `contribution_margin_per_patient`, `contribution_margin_per_visit`, `gross_margin_per_patient`, `contribution_margin_pct`
- cliente: `ltv`, `revenue`, `orders`, `avg_ticket`, `last_purchase`, `contribution_margin`, `contribution_margin_pct`, `contribution_margin_per_order`

Paginação por `limit` (padrão 20, máx. 200) e `offset`, com `has_more`.

## Leitura crítica

- **Amostra pequena engana.** LTV ou margem por paciente alta com 3 pacientes é ruído. Ao ranquear, mostre `patient_count` / `orders` ao lado e destaque só quem tem base razoável (`min_patients=10`)
- **`revenue_without_cost_pct` alto** → margem inflada por item sem custo cadastrado. Acima de ~10% na linha, avise antes de recomendar o médico e aponte `get_cost_analysis(view=missing_costs)` / `/pontoalto:costs` para cadastrar o custo. Uma médica com papanicolau e DIU sem custo pode ter um terço da receita "de graça"
- **Médico que saiu** → `last_visit` antiga. Não recomende investimento em quem não atende mais: use `active_since` ou confira a data
- **Frequência 1,0 em massa** → suspeita de prontuário (`customer_reference`) faltando na importação, não de falta de retorno
- **`(sem médico)` grande** → muitos pacientes só de exame/produto, ou `provider_name` não vindo na importação
- **Soma de pacientes em `separado`** → passa do resumo por construção (o paciente conta em cada médico). Nunca some `patient_count` das linhas nem divida a receita total por essa soma
- **"Médico" que não faz consulta no topo em `maior_gasto`** → ele fica com o paciente por fazer o item caro. Antes de recomendar, veja a mesma pergunta em `primeiro_medico`
- **`(sem especialidade)` grande** em um médico → ele atende várias especialidades e muitos pacientes só fizeram exame ou procedimento com ele no recorte. Em profissional que não faz consulta, é fonte de venda sem o grupo do procedimento
- **Cliente em duplicidade** (mesmo nome com grafias diferentes, sem CPF) → recompra subnotificada; aponte ao gestor em vez de somar por conta própria
- **Nunca compare escopos diferentes** (lifetime com período, ou períodos de tamanhos diferentes)
- **Não misture com DRE**: receita do LTV vem das vendas, a do DRE vem dos lançamentos por competência. Divergem legitimamente

## Tool pesada e sem cache

`get_ltv_analysis` varre **todos os itens de venda do histórico** e precifica cada um a cada chamada, e passa pelo semáforo de tools pesadas (máx. 2 simultâneas no servidor). Diferente de `get_cost_analysis`, **não há cache entre chamadas**: cada página, ordenação ou filtro refaz a varredura.

- Chame em sequência, nunca em paralelo com outra tool pesada
- Prefira **uma chamada com `limit` maior** (50-100) e responda várias perguntas com ela — reordenar e filtrar por base mínima ou por `last_visit` dá para fazer sobre as linhas recebidas quando `has_more=false`
- Em `Servidor ocupado, tente novamente em alguns segundos`: aguarde 10 s e repita a mesma chamada uma vez

## Dados pessoais (LGPD)

No modo cliente a resposta traz o **nome completo** do cliente (CPF não sai). Apresente ao gestor, mas não copie nomes para sugestões, regras ou textos que vão sair do PontoAlto (WhatsApp, e-mail) sem o gestor pedir.

## Escrita

Nenhuma. LTV é só leitura — não existe action de LTV e nada desta análise vira sugestão na inbox. Se a leitura revelar problema de dado (prontuário faltando, fornecedor sem nome, item sem custo), o conserto é na importação, nos cadastros ou em `set_item_cost` pela skill `cost-analysis`, não aqui.

## Regras de ouro

- Informe sempre `group_by`, `attribution` (por médico) e escopo (`lifetime` ou o período) junto do número
- Para marketing, `primeiro_medico`; para produção, `separado` (a visão que a tela abre); `maior_gasto` é o padrão da tool
- Para decidir investimento, compare margem de contribuição por paciente, não LTV — e nunca LTV × margem do `by_provider`
- Mostre a base (`patient_count` / `orders`) e `revenue_without_cost_pct` ao lado de qualquer margem
- Não recomende médico sem conferir `last_visit`
- Uma chamada bem dimensionada vale mais que várias pequenas: a tool não tem cache
- Priorize por impacto: o cliente de R$ 40.000,00 importa mais que dez de R$ 400,00
