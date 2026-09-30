---
description: "LTV (lifetime value) no PontoAlto: valor de cada paciente/cliente ao longo do tempo, por médico (clínica) ou por cliente (produto), com recompra e clientes recorrentes que pararam de comprar. Só consulta."
argument-hint: "[--local] [medico|cliente] [YYYY|YYYY-MM]"
---

# PontoAlto — LTV

Atalho para a análise de LTV. Use quando o gestor quer saber quais médicos trazem os pacientes mais valiosos, quem são os melhores clientes, quanto da base volta a comprar ou quais clientes recorrentes sumiram. Apenas consulta — não cria sugestões.

Responda em português. Use a skill `ltv-analysis` para a leitura detalhada e `financial-domain` para o contexto de domínio.

## MCP Server

Argumento recebido: `$ARGUMENTS`

- Sem `--local`: tools com prefixo `mcp__claude_ai_Ponto_Alto__` (produção)
- Com `--local`: tools com prefixo `mcp__pontoalto-local__` (desenvolvimento)

**Dimensão**: `medico` ou `cliente` nos argumentos força o `group_by`. Sem isso, deixe a tool escolher (médico em clínica, cliente em negócio de produto).

**Escopo**: sem período nos argumentos → histórico completo (lifetime, não passar `from`/`to`). `YYYY` → o ano inteiro (`YYYY-01-01` a `YYYY-12-31`). `YYYY-MM` → o mês, avisando que em um mês a frequência fica perto de 1 e o LTV se aproxima do ticket.

## Inicialização

1. `list_tenants` — escolher via `AskUserQuestion` se 2+, automático se 1
2. Confirmar conexão: nome do tenant + organização
3. Ir direto ao diagnóstico

## Diagnóstico

Uma única chamada, já dimensionada para responder as perguntas seguintes sem refazer a varredura:

```
get_ltv_analysis(tenant_id, [group_by], [from, to], limit=50)
```

A tool é pesada e **não tem cache**: cada chamada varre todo o histórico de vendas. Não chame em paralelo com outra tool pesada e não pagine à toa. Se vier `Servidor ocupado, tente novamente em alguns segundos`, aguarde 10 s e repita a mesma chamada uma vez.

Apresentar o resumo e as 10 primeiras linhas, e em seguida as observações da skill `ltv-analysis` § Leitura crítica que se aplicarem (amostra pequena, `frequency` colada em 1,0, `(sem médico)` grande).

## Escolha do subfluxo

Apresentar via `AskUserQuestion`, conforme o `group_by` devolvido.

**Por médico:**

1. **Quem fideliza (Recomendado)** — reordenar por `frequency` as linhas já recebidas (se `has_more=false`; senão chamar com `sort_by=frequency`), só entre médicos com base razoável de pacientes
2. **Tendência** — este ano contra o ano anterior, duas chamadas com `from`/`to`, em sequência
3. **Um médico específico** — `search="nome"`

**Por cliente:**

1. **Reativação (Recomendado)** — `get_ltv_analysis(group_by=cliente, min_orders=2, sort_by=last_purchase, direction=asc)`: clientes recorrentes que estão há mais tempo sem comprar
2. **Melhores clientes** — o top por LTV com pedidos, ticket e última compra
3. **Um cliente específico** — `search="nome"`

Reordenar, recortar o top ou filtrar por base mínima pode ser feito sobre as linhas que já vieram — só chame a tool de novo quando precisar de outro filtro no servidor (`min_orders`, `search`, período) ou de linhas além das recebidas.

## Formato de Saída

```
💎 LTV — [por Médico | por Cliente]
Tenant: [nome] · Escopo: [histórico completo | DD/MM/YYYY a DD/MM/YYYY]

Receita        R$ ...
Pacientes      ...          (clientes: ... · recorrentes: ... (..%))
Visitas        ...          (pedidos: ...)
LTV médio      R$ ...
Ticket médio   R$ ...
```

Depois, tabela com o top, sempre com a base ao lado do LTV:

- Por médico: Médico · Pacientes · Visitas/pac. · Ticket médio · Receita · LTV
- Por cliente: Cliente · Pedidos · Ticket médio · LTV · Última compra (DD/MM/YYYY)

Valores em R$ formato brasileiro.

## Regras de Ouro

- **Só consulta** — nada deste command vira sugestão na inbox
- Informar sempre dimensão e escopo junto do número; nunca comparar lifetime com período
- LTV é receita, não margem — para margem, `/pontoalto:costs`
- Mostrar `patient_count` / `orders` ao lado do LTV; amostra pequena não entra em ranking sem aviso
- Nomes de clientes são dado pessoal: mostrar ao gestor, não copiar para fora do PontoAlto sem ele pedir
