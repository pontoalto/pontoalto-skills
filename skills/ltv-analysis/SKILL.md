---
name: ltv-analysis
description: "LTV (lifetime value) no PontoAlto: get_ltv_analysis por médico (clínica) ou por cliente (negócio de produto), atribuição do paciente ao médico principal, identidade do cliente, lifetime vs período, recompra e clientes recorrentes que pararam de comprar. Só leitura."
version: 0.1.0
---

# LTV — Lifetime Value

## O que o LTV mede — e o que ele não mede

O LTV **não sai do extrato bancário**. Ele lê as **vendas importadas** (`SaleItem` dos itens não cancelados) e soma o `total_paid` de cada paciente ou cliente ao longo do tempo. Três consequências:

- **LTV é receita, não lucro.** Não há custo no cálculo. Um médico com LTV alto pode ter margem baixa — para margem, use a skill `cost-analysis`. Não subtraia custo do LTV à mão: as duas análises atribuem receita de jeitos diferentes (veja abaixo) e os números não se encaixam
- **Não depende de categorização nem de conciliação** — só das importações de vendas. Mês sem venda importada é mês sem LTV
- **O histórico começa na primeira importação.** "Lifetime" é desde que o tenant começou a importar vendas no PontoAlto: paciente antigo aparece como novo e o LTV dele fica subestimado. Se o gestor estranhar números baixos, confira com ele desde quando as vendas estão no sistema

## As duas dimensões

`get_ltv_analysis` agrega por **médico** ou por **cliente**. O padrão acompanha a tela: `medico` para clínica, `cliente` para negócio de produto. A resposta devolve o `group_by` usado — sempre informe ao gestor.

### Por médico (`group_by=medico`)

Responde "qual médico traz pacientes que voltam e gastam mais ao longo da vida?".

- **Atribuição por paciente**: cada paciente vai **inteiro** para o **médico com quem mais gastou** (empate → ordem alfabética). Toda a receita dele — inclusive exames, produtos e itens de outros médicos — entra na linha desse médico principal. Cada paciente aparece em uma linha só, e a soma das receitas por médico bate com a receita total
- Por isso a receita de um médico no LTV **não é a produção dele**. O faturamento por executante está em `get_cost_analysis(view=by_provider)`; não compare os dois números
- **`(sem médico)`** agrupa pacientes sem **nenhum** item com médico — tipicamente quem só fez exame de laboratório. Não é erro de cadastro por si só; se for grande, vale investigar
- **Identidade do paciente**: `customer_reference` (prontuário). Venda sem prontuário conta como **paciente novo a cada venda** — a frequência dele é sempre 1,0 e o LTV vira o ticket. Se um médico tem muitos pacientes e `frequency` colada em 1,0, desconfie de prontuário faltando na importação antes de concluir que ele não fideliza

### Por cliente (`group_by=cliente`)

Responde "quem são meus melhores clientes?" e "quem parou de comprar?".

- **Identidade**: CPF quando existe; senão o nome (sem diferenciar maiúsculas); senão a referência do cliente. Mesmo cliente digitado com nomes diferentes e sem CPF vira dois clientes — LTV subestimado e recompra subnotificada
- Clínica também pode pedir `group_by=cliente` (ranking de pacientes). Ali a identidade é CPF/nome, não prontuário: as contagens podem diferir das do modo médico

## Métricas

**Por médico** — linhas:

| Campo           | Significa |
|-----------------|-----------|
| `patient_count` | Pacientes distintos atribuídos ao médico |
| `visit_count`   | Vendas distintas desses pacientes |
| `frequency`     | Visitas por paciente (`visit_count ÷ patient_count`) |
| `avg_ticket`    | Receita por visita |
| `revenue`       | Receita total dos pacientes do médico |
| `ltv`           | Receita por paciente (`revenue ÷ patient_count` = ticket × frequência) |

Resumo: `total_revenue`, `patient_count` (distintos no conjunto inteiro), `visit_count`, `doctor_count`, `avg_ltv`, `avg_ticket`.

**Por cliente** — linhas:

| Campo           | Significa |
|-----------------|-----------|
| `orders`        | Pedidos (vendas) distintos do cliente |
| `avg_ticket`    | Receita por pedido |
| `revenue` / `ltv` | Receita total do cliente — aqui os dois são o mesmo número |
| `last_purchase` | Data da última compra (`YYYY-MM-DD` — apresente como DD/MM/YYYY) |

Resumo: `total_revenue`, `customer_count`, `order_count`, `repeat_customers` (clientes com 2+ pedidos), `repeat_rate` (% recorrentes), `avg_ltv`, `avg_ticket`, `avg_orders`.

O **resumo é sempre do conjunto inteiro**: `search` e `min_orders` filtram só as linhas (e o `total` de linhas), nunca o resumo.

## Escopo: lifetime vs período

- **Sem `from`/`to`** → histórico completo (`scope: lifetime`). É o LTV de verdade e o padrão
- **Com `from` e `to`** (os dois, `YYYY-MM-DD`) → `scope: periodo`, filtrado pela data da venda. O número passa a ser **receita por paciente no período**, não lifetime

Recorte de um mês quase sempre dá `frequency` perto de 1,0 e LTV ≈ ticket — pouco informativo. Para tendência, compare janelas longas e iguais (ano contra ano, 12 meses contra os 12 anteriores), nunca um mês contra o lifetime. Sempre diga ao gestor qual escopo o número mede.

## Receitas por pergunta

| Pergunta do gestor | Chamada |
|--------------------|---------|
| Quais médicos trazem os pacientes mais valiosos? | `get_ltv_analysis(group_by=medico)` — já vem por `ltv` decrescente |
| Qual médico mais fideliza? | `get_ltv_analysis(group_by=medico, sort_by=frequency)` |
| Qual médico atende mais pacientes? | `get_ltv_analysis(group_by=medico, sort_by=patient_count)` |
| Quem são meus melhores clientes? | `get_ltv_analysis(group_by=cliente, limit=20)` |
| Quanto da base volta a comprar? | `summary.repeat_rate` e `summary.avg_orders` do modo cliente |
| Clientes recorrentes que pararam de comprar | `get_ltv_analysis(group_by=cliente, min_orders=2, sort_by=last_purchase, direction=asc)` |
| Como está o médico X / o cliente Y? | `search="nome"` na dimensão certa |
| O LTV deste ano está melhor que o do ano passado? | duas chamadas com `from`/`to` de cada ano, uma depois da outra |

`sort_by` aceita só os campos da dimensão: `ltv`, `revenue`, `patient_count`, `visit_count`, `frequency`, `avg_ticket` em médico; `ltv`, `revenue`, `orders`, `avg_ticket`, `last_purchase` em cliente. `min_orders` só existe em cliente. Paginação por `limit` (padrão 20, máx. 200) e `offset`, com `has_more`.

## Leitura crítica

- **Amostra pequena engana.** LTV alto com 3 pacientes é ruído. Ao ranquear, mostre `patient_count` / `orders` ao lado do LTV e destaque só quem tem base razoável (ordem de grandeza: 10+ pacientes)
- **Frequência 1,0 em massa** → suspeita de prontuário (`customer_reference`) faltando na importação, não de falta de retorno
- **`(sem médico)` grande** → muitos pacientes só de exame/produto, ou `provider_name` não vindo na importação
- **Cliente em duplicidade** (mesmo nome com grafias diferentes, sem CPF) → recompra subnotificada; aponte ao gestor em vez de somar por conta própria
- **Nunca compare escopos diferentes** (lifetime com período, ou períodos de tamanhos diferentes)
- **Não misture com DRE**: receita do LTV vem das vendas, a do DRE vem dos lançamentos por competência. Divergem legitimamente

## Tool pesada e sem cache

`get_ltv_analysis` varre **todos os itens de venda do histórico** a cada chamada e passa pelo semáforo de tools pesadas (máx. 2 simultâneas no servidor). Diferente de `get_cost_analysis`, **não há cache entre chamadas**: cada página, ordenação ou filtro refaz a varredura.

- Chame em sequência, nunca em paralelo com outra tool pesada
- Prefira **uma chamada com `limit` maior** (50-100) e responda várias perguntas com ela, em vez de paginar ou reordenar várias vezes
- Em `Servidor ocupado, tente novamente em alguns segundos`: aguarde 10 s e repita a mesma chamada uma vez

## Dados pessoais (LGPD)

No modo cliente a resposta traz o **nome completo** do cliente (CPF não sai). Apresente ao gestor, mas não copie nomes para sugestões, regras ou textos que vão sair do PontoAlto (WhatsApp, e-mail) sem o gestor pedir.

## Escrita

Nenhuma. LTV é só leitura — não existe action de LTV e nada desta análise vira sugestão na inbox. Se a leitura revelar problema de dado (prontuário faltando, fornecedor sem nome), o conserto é na importação ou nos cadastros, não aqui.

## Regras de ouro

- Informe sempre `group_by` e escopo (`lifetime` ou o período) junto do número
- LTV é receita, não margem — margem é `cost-analysis`
- Mostre a base (`patient_count` / `orders`) ao lado de qualquer LTV
- Uma chamada bem dimensionada vale mais que várias pequenas: a tool não tem cache
- Priorize por receita: o cliente de R$ 40.000,00 importa mais que dez de R$ 400,00
