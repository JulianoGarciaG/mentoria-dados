# mentoria-dados

Este repositório reúne os estudos práticos de uma mentoria de Engenharia de Dados. O projeto principal, `data-product-crypto`, é a construção de um produto de dados para o mercado de criptomoedas, usando **Databricks + Delta Lake** para ingestão e processamento e **Power BI** para consumo analítico.

A ideia aqui não é só "fazer o pipeline rodar", mas praticar o que um time de dados faz de verdade em produção: organizar os dados em camadas, pensar em nomenclatura e governança, garantir que reprocessar algo não gere duplicidade, e documentar o que foi feito — em vez de um monte de notebook solto sem contexto.

## Sumário

- [Contexto](#contexto)
- [Arquitetura](#arquitetura)
- [Roadmap de entregas](#roadmap-de-entregas-incrementais)
- [A lib zordon (governança de nomes)](#a-lib-zordon-governança-de-nomes)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Stack](#stack-tecnológica)
- [Status atual](#status-atual)
- [O que este projeto demonstra](#o-que-este-projeto-demonstra)

## Contexto

O projeto segue um escopo formal de entregas (documento incluído em [`Escopo de Entregas - Mentoria de Dados.pdf`](./data-product-crypto/Escopo%20de%20Entregas%20-%20Mentoria%20de%20Dados.pdf)), dividido em **três fases incrementais**. Cada fase só começa quando a anterior está validada de ponta a ponta — uma forma de reduzir risco que também é comum em projetos de dados reais fora do contexto de mentoria.

## Arquitetura

O pipeline segue a **Arquitetura Medallion** em cima do Delta Lake:

| Camada | Papel | Regras |
|---|---|---|
| **Bronze** | Ingestão crua, *append-only*, direto das APIs das exchanges | Timestamp de ingestão obrigatório, sem transformação |
| **Silver** | Dados limpos e conformados | Deduplicação, tipagem estrita, timezone padronizado em UTC |
| **Gold** | Tabelas de negócio, prontas para consumo | Métricas agregadas (preço, retorno, volatilidade, correlação, sentimento) |

O Power BI consome só a camada Gold. Toda a lógica analítica pesada — agregações, joins temporais, cálculos estatísticos — fica no Databricks, e o BI funciona como camada fina de visualização.

## Roadmap de entregas incrementais

| Entrega | Foco | Escopo técnico principal |
|---|---|---|
| **1 — MVP de Preço** | Validar a arquitetura ponta a ponta | Ingestão OHLCV de uma exchange, camadas Bronze/Silver/Gold, dashboard de série temporal |
| **2 — Comparativo de Mercado** | Análises entre múltiplos ativos | Múltiplas moedas, retorno normalizado (base 100), volatilidade, correlação, dominância do BTC |
| **3 — Sinais Externos e Sentimento** | Enriquecer a análise com dados comportamentais | Ingestão do Crypto Fear & Greed Index, alinhamento temporal preço x sentimento, dashboard executivo |

Hoje o código reflete o começo da Entrega 1: a ingestão Bronze de candles da Binance e da Poloniex já está implementada.

## A lib `zordon` (governança de nomes)

`zordon` é uma biblioteca interna já existente no ambiente da mentoria, usada aqui para padronizar como os notebooks leem e gravam tabelas no Unity Catalog. Não foi criada por mim — é uma ferramenta que faz parte do projeto e vale explicar o que ela resolve, já que aparece em todos os notebooks de ingestão.

O problema que ela ataca: quando várias pessoas escrevem notebooks em paralelo, é fácil cada um nomear catálogo/schema/tabela de um jeito diferente. A `zordon` tira essa decisão da mão de quem escreve o notebook — os nomes seguem sempre o padrão:

```
uc_{region}_{country}_{environment}.{layer}_{domain}_{subdomain}.{table}
```

Na prática, o uso é assim:

```python
import zordon

proj = zordon.Project(spark, country="br", region="sa", environment="dev")
bronze_binance = proj.client(layer="bronze", domain="binance", subdomain="ohlcv")

bronze_binance.upsert_table(
    df_novo,
    "klines",
    merge_keys=["symbol", "interval", "open_time"],
    partition_cols=["symbol"],
)
```

Alguns pontos que ela resolve por baixo dos panos:

- **Valida os nomes antes de criar qualquer coisa**, checando formato, palavras reservadas do Spark/SQL e regras da organização (nada de `_old`, `_v2`, nome de ambiente dentro do nome da tabela etc.).
- **Cada camada Medallion tem seu próprio vocabulário permitido** — Bronze é organizado por fonte (exchange), Silver por contexto (entidade já conformada), Gold por produto de negócio.
- **`upsert_table` faz o upsert de forma idempotente**: na primeira chamada cria a tabela, nas chamadas seguintes faz um `MERGE` do Delta Lake usando as chaves informadas, então rodar o mesmo notebook de novo não duplica dado.
- **`Project` evita repetir os parâmetros de catálogo** em notebooks que leem/escrevem em vários schemas ao mesmo tempo.

## Estrutura do repositório

```
mentoria-dados/
└── data-product-crypto/
    ├── Escopo de Entregas - Mentoria de Dados.pdf   # documento de escopo do projeto
    ├── src_brz_binance_candles.ipynb                # ingestão Bronze — Binance (OHLCV)
    ├── src_brz_poloniex_candles.ipynb               # ingestão Bronze — Poloniex (OHLCV)
    └── src/
        └── zordon/                                  # lib usada para governança de nomes (ver seção acima)
```

## Stack tecnológica

- **Databricks** (Workflows, Spark, Unity Catalog) como motor de processamento
- **Delta Lake** como formato de armazenamento transacional
- **Python** nos notebooks de ingestão
- **APIs REST públicas** (Binance, Poloniex, Alternative.me) como fontes de dados
- **Power BI** como camada de consumo analítico

Nos notebooks de ingestão já dá pra ver algumas práticas que valem destaque: tratamento de rate limit com backoff exponencial, carga incremental (busca só o que é novo desde o último dado já salvo) e upsert idempotente via `MERGE` do Delta Lake.

## Status atual

Projeto em andamento. A camada Bronze da Entrega 1 (ingestão de candles OHLCV) está pronta; as camadas Silver/Gold, os cálculos analíticos da Entrega 2 e a integração com o índice de sentimento da Entrega 3 ainda estão no roadmap descrito no documento de escopo.

## O que este projeto demonstra

- Organização de dados em camadas (Medallion) com regras claras por camada
- Uso consciente de uma ferramenta de governança de nomenclatura, entendendo o problema que ela resolve
- Integração com APIs externas com tratamento de erro e rate limiting
- Cargas incrementais e idempotentes usando Delta Lake (`MERGE`/upsert)
- Organização do trabalho em entregas incrementais com critérios de aceite formais
- Preocupação com documentação e rastreabilidade, não só com fazer o pipeline funcionar
