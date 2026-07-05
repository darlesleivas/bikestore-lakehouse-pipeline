# 🚲 Bikestore Lakehouse Pipeline

Pipeline de dados end-to-end utilizando arquitetura **Medallion (Bronze → Silver → Gold)** no **Databricks**, com armazenamento em **Azure Data Lake Storage Gen2 (ADLS)** e governança via **Unity Catalog**.

O projeto simula um cenário real de engenharia de dados para uma loja de bicicletas (Bikestore), processando dados de vendas, produtos, clientes, pedidos e estoque.

---

## 🏗️ Arquitetura

![Job Orchestration](docs/job_graph.png)

O pipeline segue o padrão de arquitetura em camadas (Medallion Architecture):

**Bronze** (dados brutos) → **Silver** (dados tratados) → **Gold** (dados agregados)

### 🥉 Camada Bronze
- Ingestão de dados brutos em formato **Delta Lake**
- Criação de tabelas temporárias (temp views) para cada entidade: `brands`, `categories`, `customers`, `order_items`, `orders`, `products`, `staffs`, `stocks`, `stores`
- Sem transformação de dados — fidelidade total à fonte original

### 🥈 Camada Silver
- Limpeza e padronização dos dados (remoção de nulos, tratamento de tipos)
- Joins entre entidades relacionadas (ex: produtos + categorias + marcas + estoque)
- Cálculos financeiros com precisão decimal exata (`DECIMAL(10,2)`), evitando erros de arredondamento em ponto flutuante
- Persistência como tabelas Delta físicas, registradas no Unity Catalog

### 🥇 Camada Gold
- Agregações orientadas a negócio, prontas para consumo analítico
- Exemplos: total de vendas por estado/data, pedidos pendentes por cliente
- Tabelas otimizadas para consulta direta via SQL ou dashboards

---

## ⚙️ Orquestração (Jobs & Workflows)

O pipeline é orquestrado via **Databricks Jobs**, com dependências explícitas entre tasks (`depends on`), garantindo que cada camada só seja processada após a validação da camada anterior:

### 🥉 Camada Bronze
- Ingestão de dados brutos em formato **Delta Lake**
- Criação de tabelas temporárias (temp views) para cada entidade: `brands`, `categories`, `customers`, `order_items`, `orders`, `products`, `staffs`, `stocks`, `stores`
- Sem transformação de dados — fidelidade total à fonte original

### 🥈 Camada Silver
- Limpeza e padronização dos dados (remoção de nulos, tratamento de tipos)
- Joins entre entidades relacionadas (ex: produtos + categorias + marcas + estoque)
- Cálculos financeiros com precisão decimal exata (`DECIMAL(10,2)`), evitando erros de arredondamento em ponto flutuante
- Persistência como tabelas Delta físicas, registradas no Unity Catalog

### 🥇 Camada Gold
- Agregações orientadas a negócio, prontas para consumo analítico
- Exemplos: total de vendas por estado/data, pedidos pendentes por cliente
- Tabelas otimizadas para consulta direta via SQL ou dashboards

---

## ⚙️ Orquestração (Jobs & Workflows)

O pipeline é orquestrado via **Databricks Jobs**, com dependências explícitas entre tasks (`depends on`), garantindo que cada camada só seja processada após a validação da camada anterior:

[Bronze Tables] --> [validate_bronze] --> [Silver Tables] --> [validate_silver] --> [Gold Tables]

- Execução **Serverless**
- Validações intermediárias (`validate_bronze`, `validate_silver`) garantem qualidade antes de avançar de camada
- Job monitorável via interface de **Runs**, com histórico de execução, duração e status por task

---

## 🛠️ Tecnologias utilizadas

| Categoria | Tecnologia |
|---|---|
| Processamento | Apache Spark (PySpark, Spark SQL) |
| Plataforma | Databricks (Serverless Compute) |
| Armazenamento | Azure Data Lake Storage Gen2 (ADLS) |
| Formato de dados | Delta Lake |
| Governança | Unity Catalog |
| Orquestração | Databricks Jobs & Workflows |
| Linguagens | Python, SQL |

---

## 📁 Estrutura do projeto

- `Projeto_Bikes/`
  - `1.bronze/`
  - `2.silver/`
  - `3.gold/`
- `docs/`
- `README.md`

---

## 📊 Principais desafios técnicos resolvidos

## 🏗️ Principais decisões técnicas e desafios de arquitetura

- **Provisionamento de infraestrutura no Azure**: criação da Storage Account (ADLS Gen2) com hierarchical namespace habilitado, estruturação de containers e diretórios seguindo a arquitetura Medallion (bronze/silver/gold), e definição de convenções de path para governança de dados em escala.

- **Autenticação e integração Databricks ↔ Azure**: configuração de acesso via Service Principal (App Registration no Azure AD), com client id, secret e tenant endpoint gerenciados via OAuth2 (`ClientCredsTokenProvider`), evitando exposição de credenciais em texto plano no código através de Databricks Secret Scopes.

- **Governança de dados com Unity Catalog**: registro de tabelas físicas (`EXTERNAL TABLE`) apontando para o Delta Lake no ADLS, permitindo que o catálogo funcione como camada de metadados centralizada, independente de onde o dado fisicamente reside.

- **Modelagem orientada a Delta Lake**: uso de tabelas temporárias (`createOrReplaceTempView`) como camada de abstração entre a leitura bruta em Delta e as transformações SQL/PySpark, permitindo reprocessamento idempotente via `mode('overwrite')` sem duplicidade de dados.

- **Pipeline data-driven via dicionários e loops**: em vez de repetir lógica de leitura tabela a tabela, o pipeline itera sobre um mapeamento (`{view_name: path}`), tornando a inclusão de novas entidades uma alteração de configuração, não de código.

- **Orquestração com dependências explícitas e gates de qualidade**: uso de Databricks Jobs com tasks de validação (`validate_bronze`, `validate_silver`) atuando como checkpoints entre camadas, garantindo que a camada seguinte só processe dados já validados — um padrão equivalente a circuit breakers em pipelines de dados.

---

## 🚀 Como executar

1. Configure as credenciais de acesso à sua Storage Account no Databricks (`spark.conf.set`)
2. Ajuste as variáveis de path (`bronze_path`, `silver_path`, `gold_path`) para o seu ambiente
3. Execute os notebooks na ordem: Bronze → Silver → Gold
4. (Opcional) Configure um Databricks Job para orquestração automática

---

## 📊 Resultados

Os dados finais da camada Gold estão disponíveis em [`data/gold/`](data/gold/), incluindo:

- [`gold_sales_ny.csv`](data/gold/gold_sales_ny.csv) — total de vendas agregado por data de envio, filtrado para o estado de NY
- [`gold_orders_pending.csv`](data/gold/gold_orders_pending.csv) — pedidos pendentes por cliente

Esses arquivos representam o resultado final do pipeline, prontos para consumo em ferramentas de BI ou análise exploratória.

---

## 👤 Autor

**Darles Leivas**  
Projeto desenvolvido como parte de estudos em Engenharia de Dados com Databricks, Spark e Azure.
