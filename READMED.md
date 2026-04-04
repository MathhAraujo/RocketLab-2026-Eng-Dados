# Visagio Rocket Lab 2026 | Engenharia de Dados

## Pipeline ETL com Arquitetura Medallion no Databricks

**Autor:** Matheus Henrique de Melo Araujo  
**Atividade:** Estruturação de dados de e-commerce utilizando a arquitetura Medallion (Bronze, Silver, Gold) no Databricks Data Lakehouse.

## Sobre o Projeto

Este projeto implementa um pipeline ETL completo para um e-commerce, utilizando o dataset público da [Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) como fonte de dados. O pipeline foi construído seguindo a Arquitetura Medallion no Databricks, organizando os dados em três camadas progressivas de qualidade:

**Bronze:** Ingestão dos dados brutos (CSVs + API do Banco Central).  
**Silver:** Limpeza, padronização, deduplicação e transformações de negócio.  
**Gold:** Data Marts analíticos com KPIs prontos para consumo.

O projeto também inclui a orquestração automatizada dos notebooks via Databricks Workflows, com execução diária agendada.

## Arquitetura

```
┌─────────────────────────────────────────────────────────────────────┐
│                        FONTES DE DADOS                              │
│   CSVs (Kaggle - Olist)              API (Banco Central - PTAX)     │
└──────────────┬──────────────────────────────────┬───────────────────┘
               │                                  │
               ▼                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  CAMADA BRONZE                                                      │
│  Dados brutos + timestamp de ingestão                               │
│  9 tabelas CSV + 1 tabela de cotação do dólar (API)                 │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  CAMADA SILVER                                                      │
│  Limpeza, deduplicação, tradução e padronização                     │
│  Dimensões: consumidores, produtos, vendedores, cotação             │
│  Fatos: pedidos, itens, pagamentos, avaliações, pedido_total        │
│  Otimização: OPTIMIZE + ZORDER nas tabelas fato                     │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  CAMADA GOLD                                                        │
│  Data Marts analíticos com KPIs de negócio                          │
│  Projeto 1: Visão Comercial (receitas, rankings de produtos)        │
│  Projeto 2: Satisfação de Clientes (avaliações, rankings)           │
└─────────────────────────────────────────────────────────────────────┘
```

## Estrutura do Repositório

```
RocketLab-2026-Eng-Dados/
├── Notebooks/
│   ├── 01_Bronze.ipynb
│   ├── 02_Silver.ipynb
│   └── 03_Gold.ipynb
├── Job/
│   └── job.yaml
├── Prints/
│   └── execucao_job.png
└── README.md
```

## Camada Bronze

O notebook `01_Bronze` é responsável pela ingestão dos dados brutos:

1. Criação do database `bronze` no Data Lakehouse.
2. Ingestão automatizada dos 9 CSVs do dataset Olist via loop dinâmico, que gera os nomes das tabelas automaticamente a partir dos nomes dos arquivos, eliminando mapeamentos manuais e facilitando a manutenção.
3. Adição da coluna `timestamp_ingestion` em todas as tabelas, registrando o momento exato da ingestão.
4. Ingestão via API da cotação do dólar (PTAX) do Banco Central, com datas parametrizadas via `dbutils.widgets` para flexibilidade de execução.

### Tabelas Criadas

| Tabela | Origem |
|---|---|
| `bronze.tb_customers` | olist_customers_dataset.csv |
| `bronze.tb_geolocalizacao` | olist_geolocation_dataset.csv |
| `bronze.tb_order_items` | olist_order_items_dataset.csv |
| `bronze.tb_order_payments` | olist_order_payments_dataset.csv |
| `bronze.tb_order_reviews` | olist_order_reviews_dataset.csv |
| `bronze.tb_orders` | olist_orders_dataset.csv |
| `bronze.tb_products` | olist_products_dataset.csv |
| `bronze.tb_sellers` | olist_sellers_dataset.csv |
| `bronze.tb_product_category_name_translation` | product_category_name_translation.csv |
| `bronze.tb_cotacao_dolar` | API Banco Central (PTAX) |

## Camada Silver

O notebook `02_Silver` aplica transformações rigorosas de qualidade de dados:

**Renomeação de colunas** para português, utilizando dicionários de mapeamento para facilitar a manutenção e leitura do código.

**Deduplicação Sênior** com Window Functions particionadas por ID e ordenadas por `timestamp_ingestion` descendente, aplicada nas tabelas `dim_consumidores`, `dim_produtos` e `dim_vendedores`.

**Tradução de valores categóricos** (status de pedido e tipos de pagamento) de inglês para português, com tratamento de valores inesperados via `.otherwise()`.

**Conversão explícita de tipagem** em todas as tabelas, garantindo tipos corretos independentemente do `inferSchema`, seguindo uma abordagem de programação defensiva.

**Tolerância a falhas** com `try_to_timestamp` e `try_cast` para evitar quebras por dados sujos.

**Tratamento de nulos** nas avaliações, preenchendo títulos e comentários vazios com valores padrão.

**Padronização textual** com `upper()` aplicado defensivamente em cidades e estados.

**Colunas derivadas** na tabela de pedidos: tempo de entrega real, estimado, diferença e indicador de entrega no prazo.

**Calendário contínuo de cotação do dólar**, preenchendo fins de semana e feriados com a cotação da sexta-feira anterior via `last(ignorenulls=True)`.

**Tabela consolidada `fat_pedido_total`** com JOIN de pedidos, pagamentos e cotação, incluindo valores em BRL e USD com arredondamento de 2 casas decimais.

**Otimização física** com `OPTIMIZE` + `ZORDER` nas tabelas fato, indexando por `id_pedido` e `data_pedido`.

### Tabelas Criadas

| Tabela | Tipo | Origem |
|---|---|---|
| `silver.dim_consumidores` | Dimensão | tb_customers |
| `silver.fat_pedidos` | Fato | tb_orders |
| `silver.fat_itens_pedidos` | Fato | tb_order_items |
| `silver.fat_pagamentos_pedidos` | Fato | tb_order_payments |
| `silver.fat_avaliacoes_pedidos` | Fato | tb_order_reviews |
| `silver.dim_produtos` | Dimensão | tb_products |
| `silver.dim_vendedores` | Dimensão | tb_sellers |
| `silver.dim_categoria_produtos_traducao` | Dimensão | tb_product_category_name_translation |
| `silver.dim_cotacao_dolar` | Dimensão | tb_cotacao_dolar |
| `silver.fat_pedido_total` | Fato | JOIN de múltiplas tabelas |

## Camada Gold

O notebook `03_Gold` constrói os Data Marts analíticos para consumo do negócio.

### Projeto 1: Visão Comercial e Volume de Produtos

**Tabela `gold.fat_vendas_comercial`:** Agregação por ano, mês e categoria de produto, com as métricas `total_pedidos`, `qtd_itens_vendidos`, `receita_total_brl`, `receita_total_usd` e `ticket_medio_brl`.

**Rankings Comerciais:** Top 5 Produtos Mais Vendidos e Top 5 Produtos Menos Vendidos, exibidos via `display()`.

### Projeto 2: Satisfação de Clientes e Qualidade de Parceiros

**Tabela `gold.fat_avaliacoes_clientes`:** Agregação por categoria do produto, vendedor e estado, com as métricas `total_avaliacoes`, `avaliacao_media`, `total_avaliacoes_positivas`, `total_avaliacoes_negativas` e `percentual_satisfacao`.

**Rankings de Qualidade** com ordenação composta (nota + volume como desempate): produto mais e menos bem avaliado, vendedor mais e menos bem avaliado.

## Orquestração com Databricks Workflows

O pipeline é orquestrado via Databricks Workflows com 3 tasks sequenciais (Bronze, Silver, Gold), agendamento diário às 13:00 e tolerância a mudanças de schema via `.option("overwriteSchema", "true")`.

A configuração do Job está exportada em `Job/job.yaml` e o print da execução bem-sucedida está disponível em `Prints/`.

## Como Executar

### Pré-requisitos

Acesso a um workspace Databricks (Community Edition ou superior) e o dataset Olist baixado do [Kaggle](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce).

### Passo a Passo

1. **Upload dos dados:** No Databricks, crie um Volume em `Catalog > workspace > default` chamado `olist_files` e faça upload dos 9 CSVs.
2. **Importar notebooks:** Em `Workspace > Import`, importe os 3 arquivos `.ipynb` da pasta `Notebooks/`.
3. **Executar na ordem:** `01_Bronze` (ingestão), `02_Silver` (transformações), `03_Gold` (Data Marts).
4. **Configurar Workflow (opcional):** Importe o `job.yaml` em `Workflows` para automatizar a execução diária.

### Parâmetros do Notebook Bronze

| Parâmetro | Valor Padrão | Formato |
|---|---|---|
| `start_date` | 01-01-2016 | MM-DD-AAAA |
| `end_date` | 12-31-2018 | MM-DD-AAAA |

## Decisões Técnicas e Destaques

### Manutenibilidade
Utilização de dicionários de mapeamento para renomeação de colunas, centralizando as transformações e facilitando alterações futuras. Loop dinâmico na ingestão Bronze que gera nomes de tabelas automaticamente a partir dos arquivos, eliminando repetição de código.

### Programação Defensiva
Conversão explícita de tipagem mesmo quando o `inferSchema` acerta, protegendo contra mudanças nos dados de origem. Uso de `upper()` preventivo em cidades e estados, `try_cast` e `try_to_timestamp` para tolerância a dados sujos, e `.otherwise("desconhecido")` nas traduções de status para tratar valores inesperados.

### Observações sobre o Dataset
As colunas `customer_name`, `seller_name` e `product_name` constam no enunciado da atividade mas não existem no dataset original do Olist. Essa divergência foi documentada nos comentários do código. O mapeamento duplicado de `review_comment_title` e `review_comment_message` para o mesmo nome no enunciado foi tratado mantendo as colunas separadas como `titulo_avaliacao_comentario` e `mensagem_avaliacao_comentario`.

## Tecnologias Utilizadas

| Tecnologia | Uso |
|---|---|
| Databricks | Plataforma de Data Lakehouse |
| Apache Spark (PySpark) | Processamento distribuído de dados |
| Delta Lake | Formato de armazenamento otimizado |
| SQL | Criação de databases e otimização |
| Python | Lógica de transformações e ingestão de API |
| API Banco Central (PTAX) | Cotação do dólar |

## Dataset

**Brazilian E-Commerce Public Dataset by Olist**  
Disponível em: https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce

O dataset contém informações de aproximadamente 100 mil pedidos realizados entre 2016 e 2018 em marketplaces brasileiros, incluindo dados de clientes, pedidos, pagamentos, avaliações, produtos e vendedores.

---

*Desenvolvido como parte do Rocket Lab Visagio 2026.1.*