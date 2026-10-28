---
name: fiscal-classification
description: "Notas fiscais do DRE Contábil no PontoAlto: classificar itens de NFS-e/NF-e emitidas e recebidas (analyze_fiscal_classification + classify_fiscal_items com regra) e vincular nota recebida ao pagamento do extrato (list_fiscal_documents + link_fiscal_payment), sempre via sugestões na inbox. Inclui o que o sistema já faz sozinho (regras e conciliação automática com certeza) e o cuidado com dados de paciente."
version: 0.1.0
---

# Notas Fiscais (DRE Contábil)

## O que já acontece sozinho — não refaça

As notas chegam do Ambiente Nacional (NFS-e), da SEFAZ (NF-e recebidas) ou por upload de XML. Na importação:

- **Regras de classificação** (CNPJ do emitente, código de serviço, NCM, CFOP) levam cada item à categoria do DRE Contábil
- **Conciliação nota × pagamento**: nota recebida com **um único** lançamento de saída (banco ou caixa) no mesmo valor, perto da emissão, **confirmado pelo fornecedor** — o PIX imprime o começo do CNPJ do recebedor ("Pagamento Pix 29.482.4") ou o lançamento já está no fornecedor da nota — é vinculada automaticamente. A categoria passa de um lado para o outro (lançamento categorizado → itens herdam; nota classificada → lançamento recebe) e o lançamento ganha o fornecedor
- Pagamento só **provável** (valor e data, sem confirmação) já vira sugestão `link_fiscal_payment` na inbox, feita pelo próprio sistema

O seu trabalho é o que sobra: itens sem regra e pagamentos que o sistema não conseguiu decidir.

## Classificar itens pendentes

```
analyze_fiscal_classification(date_from?, date_to?, role?)
```

Devolve os itens pendentes agrupados por papel (`emitente` = receita, `destinatario` = custo/despesa), código de serviço ou NCM e contraparte, do maior valor para o menor. Cada grupo traz `document_ids`, `item_ids`, total e `suggested_categories`:

| `source`                | Pista                                                        | Força |
|-------------------------|--------------------------------------------------------------|-------|
| `contraparte`           | Itens já classificados do mesmo CNPJ                          | Alta  |
| `pagamentos_fornecedor` | Categoria dos lançamentos do extrato do mesmo fornecedor      | Alta  |
| `servico`               | Itens já classificados do mesmo código de serviço/NCM         | Média |

Fluxo:

1. `list_categories` → mapear nome → `category_id`. Nota **recebida** só aceita categoria de despesa; **emitida**, de receita
2. `list_suggestions(status=pending)` → não repetir sugestão de nota já na inbox (uma pendente por nota e ação)
3. Para cada grupo, decidir a categoria:
   - `suggested_categories` com `contraparte` ou `pagamentos_fornecedor` e várias ocorrências → confidence 90-95
   - Só `servico` → 70-85, conforme o código seja específico (04.03.02 exame de imagem) ou genérico
   - Sem pista → perguntar ao gestor (`AskUserQuestion`) ou deixar pendente; não chutar
   - Remessa, retorno, imobilizado, devolução já tratada → `exclude: true` (sai do DRE)
4. `create_suggestion` em **modo batch**. Grupo recorrente (mesma contraparte e serviço): **uma** sugestão, na nota de maior valor, com `create_rule: true` — ao aprovar, a regra classifica as demais notas do grupo e as próximas. Caso pontual (sem regra): uma sugestão por nota:

```json
{
  "type": "classify_fiscal",
  "suggestable_type": "fiscal_document",
  "suggestable_id": 812,
  "action": "classify_fiscal_items",
  "action_params": {"fiscal_document_id": 812, "category_id": 31, "create_rule": true},
  "confidence": 92,
  "reasoning": "Mesmo CNPJ já classificado 6x em Honorários Médicos; serviço 04.01.01."
}
```

- `suggestable_id` **precisa** ser igual a `action_params.fiscal_document_id` — o servidor recusa a combinação trocada
- `create_rule: true` cria a regra do emitente + serviço (em nota emitida para pessoa física, a regra vale para o serviço de todos os tomadores pessoa física) e reaplica nas pendentes
- `item_ids` só quando a nota mistura naturezas (NF-e com material + serviço); sem ele, todos os itens da nota
- `presumption_group` só em nota emitida de Lucro Presumido com equiparação hospitalar (`servicos` para consulta, `hospitalar` para exame/procedimento)
- `bulk_create_suggestions` **não** serve aqui — é só para lançamentos do extrato

## Vincular nota ao pagamento

```
list_fiscal_documents(role=destinatario, payment=sem_pagamento, date_from?, date_to?)
```

Cada nota traz até 5 `payment_candidates`, com `confirmed_by`:

| `confirmed_by` | Significa                                         | Ação |
|----------------|---------------------------------------------------|------|
| `cnpj_extrato` | O PIX traz o CNPJ do emitente                       | Propor (confidence 90+) |
| `fornecedor`   | O lançamento está no fornecedor da nota             | Propor (confidence 90+) |
| `null`         | Só valor e data                                     | Propor se for o **único** candidato (65-75); com vários, listar para o gestor escolher |

O sistema só deixa para você casos em que ele não teve certeza — por isso candidato confirmado aqui geralmente significa **mais de um** confirmado (pagamentos mensais do mesmo valor ao mesmo fornecedor). Escolha o mais próximo da emissão e diga isso no `reasoning`.

```json
{
  "type": "link_fiscal_payment",
  "suggestable_type": "fiscal_document",
  "suggestable_id": 812,
  "action": "link_fiscal_payment",
  "action_params": {"fiscal_document_id": 812, "transaction_id": 45120},
  "confidence": 80,
  "reasoning": "Dois PIX de R$ 1.285,74 ao mesmo CNPJ; este é de 20/02, 3 dias após a emissão."
}
```

Sem candidato: a nota pode ter sido paga no cartão de crédito (Google, Facebook), em boleto com valor diferente, agrupada com outras ou ainda não paga — reporte ao gestor, não force vínculo.

## Dados de paciente (LGPD)

Nota emitida para pessoa física (paciente, cliente final) aparece só como **"Pessoa física"**: o MCP não devolve nome, CPF nem a descrição livre do serviço. Classifique pelo **código de serviço** (`04.01.01` consulta, `04.02` exames, `04.03` procedimentos) e nunca peça esses dados ao gestor para repassar.

## Depois de aprovar

- Aprovação é **manual**, na tela da inbox (tipos "Classificar Nota" e "Nota × Pagamento") — o plugin cria, não aprova
- O resultado aparece no DRE Contábil: `get_reports(report=dre_contabil, from, to)`. Item pendente não entra no DRE — receita de nota emitida sem classificação fica de fora até o gestor aprovar
- Desfazer na inbox devolve itens e lançamento ao estado anterior; com regra criada, a regra é desativada

## Regras de Ouro

- Toda escrita é sugestão: `classify_fiscal_items` e `link_fiscal_payment`, via `create_suggestion` (batch)
- Não refazer o que o sistema já resolveu: item com `source=regra|manual|pagamento` não precisa de sugestão
- Uma sugestão por nota e ação; checar `list_suggestions(status=pending)` antes
- Priorizar por valor (`total_value` / `gross_amount`)
- Categoria de despesa para nota recebida, de receita para emitida — o servidor recusa o contrário
- Sem pista de categoria, perguntar; sem candidato único, listar — nunca chutar
