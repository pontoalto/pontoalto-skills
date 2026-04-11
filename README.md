# PontoAlto — Plugin Claude Code

Assistente financeiro para gestores do [PontoAlto](https://pontoalto.app) no terminal. Conecta o Claude Code aos MCP servers do PontoAlto e guia você no fluxo diário de fechamento de mês: importações, categorização, competência, fornecedores, conciliação de vendas, custos e relatórios.

## Para quem é

Gestores financeiros que operam o PontoAlto diariamente e querem:

- Fechar o mês sem precisar navegar entre dezenas de telas
- Categorizar dezenas/centenas de lançamentos em lote, com regras automáticas
- Perguntar dúvidas específicas ("quais transações sem fornecedor em março?") e receber resposta direta
- Gerar relatórios prontos (DRE, orçado vs realizado, margem por serviço) por comando

Você continua no controle: toda operação de escrita passa por sugestões na inbox, que você aprova ou rejeita.

## Commands

| Command                  | Quando usar                                                        |
|--------------------------|--------------------------------------------------------------------|
| `/pontoalto:manager`     | Fluxo completo do mês — quando estiver fechando ou não souber por onde começar. Mostra um checklist do status e executa a primeira pendência. |
| `/pontoalto:categorize`  | Só categorizar lançamentos pendentes (automático, consulta WhatsApp ou manual). |
| `/pontoalto:reconcile`   | Liquidar repasses de cartão/dinheiro e conciliar vendas com o extrato bancário. |
| `/pontoalto:suppliers`   | Vincular fornecedores a pagamentos e ajustar datas de competência para o DRE. |
| `/pontoalto:report`      | Gerar relatório mensal consolidado: DRE, orçado vs realizado e análise de custos. |
| `/pontoalto:sale-source` | Montar ou ajustar uma fonte de venda customizada (CSV de sistema ainda não integrado), com loop de preview iterativo. |

Todos os commands aceitam a flag `--local` para apontar para o MCP server de desenvolvimento local (útil para testes).

## Como funciona

1. Você invoca um command (ex: `/pontoalto:manager`)
2. O Claude conecta ao MCP server do PontoAlto e lista seus tenants
3. Você escolhe o tenant (se tiver mais de um)
4. O Claude diagnostica o que está pendente e pergunta o próximo passo
5. Ações de escrita criam **sugestões** na inbox — você revisa e aprova
6. O Claude reporta o resultado e pergunta se quer continuar

## Exemplos

**Fechar o mês do zero:**
```
/pontoalto:manager
```

**Só categorizar pendências:**
```
/pontoalto:categorize
```

**Relatório de um mês específico:**
```
/pontoalto:report 2026-03
```

**Testar em desenvolvimento local:**
```
/pontoalto:manager --local
```

**Montar uma fonte de venda nova a partir de um CSV:**
```
/pontoalto:sale-source
```
Funciona sem acesso ao código do sistema — o Claude guia você por um loop de preview iterativo até a spec bater com o seu CSV, e só salva após sua confirmação. Requer perfil `admin` no tenant.

## Requisitos

- [Claude Code](https://claude.com/claude-code) instalado
- Acesso ao MCP server do PontoAlto (produção via `claude.ai` ou local via `cdc-dashboard`)
- Ao menos um tenant ativo no PontoAlto

## Instalação

Claude Code plugin — consulte a [documentação oficial de plugins](https://code.claude.com/docs/en/plugins.md) para instalar.

## Modelo recomendado

O fluxo do PontoAlto é majoritariamente orquestração de tools MCP (listar, analisar, criar sugestões). Para isso, **Sonnet 4.6** é o melhor custo/benefício — bem mais rápido que Opus e com qualidade suficiente para categorização, conciliação e análise de DRE.

| Modelo                  | Quando usar |
|-------------------------|-------------|
| `claude-sonnet-4-6`     | **Padrão recomendado.** Rápido e preciso para o dia a dia (`/pontoalto:manager`, `/categorize`, `/reconcile`, `/suppliers`). |
| `claude-haiku-4-5`      | Máxima velocidade em tarefas diretas (ex: `/pontoalto:report`, categorizações simples via regras). Pode perder nuance em análises mais abertas. |
| `claude-opus-4-6`       | Reserve para raciocínio pesado: inconsistências complexas no fechamento, decisões de competência ambíguas ou análises de custo não triviais. |

Troque de modelo com `/model claude-sonnet-4-6`. Se quiser ainda mais velocidade sem mudar de modelo, `/fast` acelera o output.

## Regras de ouro do assistente

O plugin opera com estas garantias:

- **Você tem a última palavra** — toda operação de escrita vai para a inbox como sugestão. Exceções diretas (sem inbox): liquidações de repasses (`create_settlements`), recebíveis em dinheiro (`create_cash_settlements`) e gerenciamento de fontes de venda (`save_sale_source_definition`, `delete_sale_source_definition`) — nessas, o Claude sempre pede confirmação explícita antes de gravar.
- **Prioriza impacto monetário** — maiores valores primeiro em qualquer listagem.
- **Regras automáticas** — ao categorizar ou vincular fornecedor, cria regra (`contains`) para automatizar os próximos meses. Pattern ambíguo com histórico claro? Propõe atualizar ou desativar a regra antiga (via inbox). Pattern ambíguo sem padrão? Não cria regra e avisa.
- **Confidence transparente** — matches abaixo de 55% são ignorados; entre 55-69% são advisory; ≥ 70% viram sugestão.
- **Não bloqueia em perguntas** — se o gestor não responde um pedido de consulta, o workflow continua para a próxima etapa.

## Desenvolvimento

Para contribuir ou iterar neste plugin, veja o `CLAUDE.md` na raiz do projeto.
