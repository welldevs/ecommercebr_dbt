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

## Camada de staging

Um model por fonte (`stg_olist__<entidade>`), sem joins nem agregações — só renomeação de colunas e cast de tipos. Cada model tem sua própria `schema.yml` com description, grão declarado e teste de chave primária (simples ou composta, via `dbt_utils.unique_combination_of_columns` quando aplicável):

| Model | Grão |
| --- | --- |
| `stg_olist__orders` | `order_id` |
| `stg_olist__customers` | `customer_id` |
| `stg_olist__sellers` | `seller_id` |
| `stg_olist__products` | `product_id` |
| `stg_olist__order_items` | `order_id` + `order_item_id` |
| `stg_olist__order_payments` | `order_id` + `payment_sequential` |
| `stg_olist__order_reviews` | `review_id` + `order_id` (o `review_id` sozinho não é único no dado bruto) |
| `stg_olist__geolocation` | sem chave primária natural — várias coordenadas por prefixo de CEP |

Valide tudo com:

```bash
dbt build -s staging
```

## Limitações conhecidas

- **`dbt-labs/codegen` não é compatível com o padrão de sources deste projeto.** As macros `generate_source` e `generate_base_model` fazem introspecção de colunas via `information_schema`/`DESCRIBE` assumindo tabelas já carregadas num schema físico. Como as sources aqui usam `external_location` + `read_csv` (sem carga física), ambas falham no DuckDB. Alternativa usada: `DESCRIBE SELECT * FROM read_csv(...)` via `dbt show --inline`, manual.
- **Views sobre `read_csv` com caminho relativo só funcionam dentro do contexto do dbt.** Abrir o arquivo `.duckdb` direto em outra ferramenta (ex.: DBeaver) e consultar um model de staging pode falhar, porque o caminho relativo do CSV é resolvido a partir do diretório de execução do dbt (`external_root`), não do cliente SQL. Models pensados para consumo externo devem ser materializados como `table`.

## Status atual

- **Fase 0 (fundação):** concluída — perfis, schemas por ambiente, sources e pacotes configurados.
- **Fase 1 (staging):** em andamento — DATA-01 e DATA-02 concluídos (8 models de staging com testes). Faltam DATA-03 (relatório de qualidade) e DATA-04 (tempo simulado / `data_corte`).
