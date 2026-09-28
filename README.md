# Olist Analytics

Projeto dbt construído em cima do [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce): dados reais e anonimizados de ~100 mil pedidos de um marketplace brasileiro, entre 2016 e 2018.

Stack: **dbt Core 1.12.x** + **dbt-duckdb** (tudo roda localmente, sem custo e sem conta em nuvem).

## Pré-requisitos

- Python 3.11+
- Git
- Os 9 arquivos CSV do dataset da Olist (baixados à parte, não vão para o repositório)

## Setup do zero

### 1. Clonar o repositório e criar o ambiente virtual

```bash
git clone git@github.com:welldevs/ecommercebr_dbt.git
cd ecommercebr_dbt/olist_analytics
python -m venv venv
source venv/bin/activate        # Linux/Mac
# ou, no Windows:
# .\venv\Scripts\Activate.ps1

pip install "dbt-core>=1.12,<1.13" dbt-duckdb
```

### 2. Baixar os dados

Baixe o dataset do Kaggle e extraia os 9 CSVs para `data/raw/` (pasta ignorada pelo git — tratada como somente leitura):

```
data/raw/
├── olist_customers_dataset.csv
├── olist_geolocation_dataset.csv
├── olist_order_items_dataset.csv
├── olist_order_payments_dataset.csv
├── olist_order_reviews_dataset.csv
├── olist_orders_dataset.csv
├── olist_products_dataset.csv
├── olist_sellers_dataset.csv
└── product_category_name_translation.csv
```

> `product_category_name_translation.csv` também precisa de uma cópia em `seeds/` — ela é carregada como *seed* do dbt, não como source (decisão de design documentada no PR correspondente).

### 3. Configurar variáveis de ambiente

Crie um arquivo `.env` na raiz do projeto (`olist_analytics/.env`, ignorado pelo git) com:

```
DBT_DATABASE_PATH=olist_analytics.duckdb
```

Esse valor é o nome do arquivo DuckDB único, compartilhado entre os targets `dev` e `prod` (o que muda entre eles é o schema, não o arquivo).

### 4. Configurar o `profiles.yml`

O `profiles.yml` não é versionado (é específico de cada máquina/usuário). Crie `~/.dbt/profiles.yml` com:

```yaml
olist_analytics:
  outputs:
    dev:
      type: duckdb
      path: "{{ env_var('DBT_DATABASE_PATH') }}"
      schema: "<seu_prefixo_pessoal>"   # ex.: dbt_seunome
      threads: 1

    prod:
      type: duckdb
      path: "{{ env_var('DBT_DATABASE_PATH') }}"
      schema: "prod"
      threads: 4

  target: dev
```

### 5. Instalar as dependências do dbt e construir o projeto

Rode todos os comandos a partir de dentro de `olist_analytics/` (é onde está o `dbt_project.yml`):

```bash
dbt deps          # instala os pacotes declarados em packages.yml
dbt debug         # confirma a conexão no target dev
dbt debug --target prod   # confirma a conexão no target prod
dbt seed          # carrega os seeds (ex.: tradução de categorias)
dbt build         # roda seeds, models e testes no target dev
```

## Arquitetura

Três camadas — staging, intermediate e marts — descritas em detalhe no README do trilhamento do projeto. Sources declaradas com `external_location` + `read_csv`, sem carregar os dados brutos no banco; seeds para dados de referência pequenos e estáticos.

## Status atual

- **Fase 0 (fundação):** concluída — perfis, schemas por ambiente, sources e pacotes configurados.
- **Fase 1 (staging) em diante:** ainda não iniciada.
