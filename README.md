# mentoria-dados

Repositório de estudos práticos de **Engenharia de Dados**, desenvolvido no contexto de uma mentoria técnica. O projeto principal, `data-product-crypto`, consiste na construção de um **produto de dados analítico para o mercado de criptomoedas**, usando **Databricks + Delta Lake** como motor de ingestão/processamento e **Power BI** como camada de consumo analítico.

O objetivo do repositório não é só "rodar um pipeline", mas exercitar de ponta a ponta as práticas que um time de dados usa em produção: arquitetura em camadas, governança de nomenclatura, idempotência, controle de qualidade e documentação — em vez de notebooks soltos e scripts descartáveis.

## Sumário

- [Contexto](#contexto)
- [Arquitetura](#arquitetura)
- [Roadmap de entregas](#roadmap-de-entregas-incrementais)
- [Destaque técnico: zordon](#destaque-técnico-zordon)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Stack](#stack-tecnológica)
- [Status atual](#status-atual)
- [O que este projeto demonstra](#o-que-este-projeto-demonstra)

## Contexto

O projeto simula a construção de um produto de dados real, do zero, sob um escopo formal de entregas (documento incluído em [`Escopo de Entregas - Mentoria de Dados.pdf`](./data-product-crypto/Escopo%20de%20Entregas%20-%20Mentoria%20de%20Dados.pdf)). A entrega é dividida em **três fases incrementais**, cada uma só avança quando a anterior está validada de ponta a ponta — uma disciplina de gerenciamento de risco comum em projetos de dados reais.

## Arquitetura

O pipeline segue a **Arquitetura Medallion** sobre Delta Lake:

| Camada | Papel | Regras |
|---|---|---|
| **Bronze** | Ingestão crua, *append-only*, direto das APIs das exchanges | Timestamp de ingestão obrigatório, sem transformação |
| **Silver** | Dados limpos e conformados | Deduplicação, tipagem estrita, timezone padronizado em UTC |
| **Gold** | Tabelas de negócio, prontas para consumo | Métricas agregadas (preço, retorno, volatilidade, correlação, sentimento) |

O Power BI consome **exclusivamente** a camada Gold — toda a lógica analítica (agregações, joins temporais, cálculos estatísticos) é resolvida no Databricks, mantendo o BI como camada fina de visualização.

## Roadmap de entregas incrementais

| Entrega | Foco | Escopo técnico principal |
|---|---|---|
| **1 — MVP de Preço** | Validar a arquitetura ponta a ponta | Ingestão OHLCV de uma exchange, camadas Bronze/Silver/Gold, dashboard de série temporal |
| **2 — Comparativo de Mercado** | Análises entre múltiplos ativos | Múltiplas moedas, retorno normalizado (base 100), volatilidade, correlação, dominância do BTC |
| **3 — Sinais Externos e Sentimento** | Enriquecer a análise com dados comportamentais | Ingestão do Crypto Fear & Greed Index, alinhamento temporal preço x sentimento, dashboard executivo |

O estado atual do código reflete o início da **Entrega 1**: ingestão Bronze de candles da Binance e da Poloniex já implementada e versionada.

## Destaque técnico: `zordon`

A parte mais representativa de senioridade técnica do repositório não é o pipeline em si, mas a biblioteca `zordon` (em [`src/zordon`](./data-product-crypto/src/zordon)), criada para resolver um problema comum em times de dados: **como garantir que todo mundo nomeie catálogos, schemas e tabelas do Unity Catalog da mesma forma**, mesmo trabalhando em paralelo.

Principais decisões de design:

- **Convenção de nomenclatura como código, não como wiki.** O padrão `uc_{region}_{country}_{environment}.{layer}_{domain}_{subdomain}.{table}` é validado programaticamente, não depende de disciplina manual.
- **Validação em camadas explícitas** (`naming.py`): restrições nativas do Unity Catalog → convenção de formato (snake_case) → palavras reservadas → regras de governança (proíbe sufixos como `_old`, `_v2`, termos de ambiente dentro do nome de tabela etc.), cada uma com mensagem de erro específica sobre *por que* o nome foi rejeitado.
- **Vocabulário controlado por camada** (`vocabularies.py`): Bronze é organizado por *fonte* (exchange), Silver por *contexto* (entidade conformada), Gold por *produto de negócio* — refletindo no código a própria filosofia da arquitetura Medallion.
- **API ergonômica para o consumidor** (`client.py`, `project.py`): `DataClient.upsert_table()` encapsula lógica de `MERGE` do Delta Lake com idempotência (cria a tabela no primeiro uso, faz upsert nas chamadas seguintes), e `Project` evita repetição dos parâmetros de catálogo ao longo de um notebook que lê/escreve em vários schemas.
- **Erros como classe própria** (`errors.py`), permitindo capturar falhas de governança de forma diferenciada de outras exceções.

Isso mostra a diferença entre "escrever um notebook que funciona" e **desenhar uma ferramenta interna reutilizável** que impõe padrões de qualidade a um time inteiro.

## Estrutura do repositório

```
mentoria-dados/
└── data-product-crypto/
    ├── Escopo de Entregas - Mentoria de Dados.pdf   # documento de escopo do projeto
    ├── src_brz_binance_candles.ipynb                # ingestão Bronze — Binance (OHLCV)
    ├── src_brz_poloniex_candles.ipynb               # ingestão Bronze — Poloniex (OHLCV)
    └── src/
        └── zordon/                                  # biblioteca interna de governança de dados
            ├── __init__.py
            ├── governance.py    # validação e construção de nomes (catálogo/schema/FQN)
            ├── naming.py        # regras de validação em camadas
            ├── vocabularies.py  # vocabulário controlado por camada Medallion
            ├── client.py        # leitura/escrita/upsert de tabelas Delta
            ├── project.py       # factory de clientes por schema
            └── errors.py        # exceções de governança
```

## Stack tecnológica

- **Databricks** (Workflows, Spark, Unity Catalog) como motor de processamento
- **Delta Lake** como formato de armazenamento transacional
- **Python** para a biblioteca de governança e os notebooks de ingestão
- **APIs REST públicas** (Binance, Poloniex, Alternative.me) como fontes de dados
- **Power BI** como camada de consumo analítico

Nos notebooks de ingestão já é possível ver práticas de engenharia relevantes: **tratamento de rate limit com backoff exponencial**, **carga incremental** (busca só o que é novo a partir do último dado já ingerido) e **upsert idempotente** via `MERGE` do Delta Lake.

## Status atual

Projeto em andamento. A camada Bronze da Entrega 1 (ingestão de candles OHLCV) está implementada; as camadas Silver/Gold, os cálculos analíticos da Entrega 2 e a integração do índice de sentimento da Entrega 3 fazem parte do roadmap descrito no documento de escopo.

## O que este projeto demonstra

- Modelagem de arquitetura de dados em camadas (Medallion) com regras de negócio claras por camada
- Design de uma biblioteca interna de governança (nomenclatura, validação, vocabulário controlado)
- Integração com APIs externas com tratamento de erros e rate limiting
- Cargas incrementais e idempotentes usando Delta Lake (`MERGE`, `upsert`)
- Organização de projeto orientada a entregas incrementais com critérios de aceite formais
- Preocupação com documentação técnica e rastreabilidade, não só com "fazer o pipeline rodar"
