# Plataforma de Ingestao Automatizada — Hub-Spoke

## Visao Geral

Plataforma de ingestao de dados 100% Google Cloud, agnóstica quanto a origens, com governanca centralizada via Dataplex e processamento distribuído via Spark no Dataproc Serverless.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DATAPLEX (Governanca)                        │
│  Catalogacao · Qualidade · Linhagem · Politicas · Classificacao     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────┐    ┌──────────────────┐    ┌──────────────────────┐  │
│  │  ORIGENS  │───>│  ORQUESTRACAO    │───>│  DESTINO (BigQuery / │  │
│  │           │    │  Cloud Composer  │    │  GCS / Bigtable)     │  │
│  └───────────┘    └──────────────────┘    └──────────────────────┘  │
│       │                    │                                        │
│       │           ┌───────────────┐                                 │
│       └──────────>│ DATAPROC      │                                 │
│                   │ SERVERLESS    │                                 │
│                   │ (Spark Jobs)  │                                 │
│                   └───────────────┘                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. Conectores de Origem

### 1.1 Bases Transacionais (RDBMS)

| Metodo | Tecnologia | Uso |
|--------|-----------|-----|
| **JDBC** | Spark JDBC Reader | Leitura batch/incremental de PostgreSQL, MySQL, SQL Server, Oracle |
| **ODBC** | Spark ODBC via Simba/driver nativo | Fontes legadas que expõem apenas ODBC |
| **CDC** | Datastream (GCP nativo) | Captura de mudancas em tempo real de MySQL, PostgreSQL, Oracle, SQL Server |

**CDC com Datastream:**
- Captura eventos INSERT/UPDATE/DELETE em near real-time
- Entrega em GCS (Avro/JSON) ou diretamente BigQuery
- Spark job consome arquivos CDC do GCS para transformacoes complexas

```
[RDBMS] ──CDC──> [Datastream] ──> [GCS /raw/cdc/] ──> [Spark/Dataproc] ──> [BigQuery]
[RDBMS] ──JDBC─> [Spark/Dataproc Serverless] ──────────────────────────> [BigQuery]
```

### 1.2 Bases NoSQL — MongoDB

| Cenario | Conector | Detalhes |
|---------|----------|---------|
| **MongoDB Community (self-hosted)** | MongoDB Spark Connector (`mongo-spark-connector`) | Conecta via connection string padrao. Spark le colecoes como DataFrames |
| **MongoDB Atlas** | MongoDB Spark Connector + Atlas Data Federation | Mesma lib, connection string `mongodb+srv://`. Suporta leitura de clusters M0 (free) ate dedicados |

**Configuracao Spark:**
```python
spark.read.format("mongodb") \
    .option("connection.uri", "mongodb+srv://user:pass@cluster.mongodb.net") \
    .option("database", "meu_db") \
    .option("collection", "minha_colecao") \
    .load()
```

**Change Streams (CDC para MongoDB):**
- MongoDB 3.6+ suporta Change Streams nativamente
- Spark Structured Streaming consome change streams para ingestao incremental
- Atlas: requer replica set (incluido em todos os tiers, inclusive free M0)

### 1.3 APIs REST/GraphQL

| Componente | Funcao |
|-----------|--------|
| **Cloud Functions (2nd gen)** | Extrator leve — chama API, grava resposta raw em GCS |
| **Cloud Run** | Extrator pesado — APIs com paginacao complexa, autenticacao OAuth, rate limiting |
| **Spark (Dataproc Serverless)** | Le arquivos JSON/Parquet do GCS, aplica schema, transforma e carrega |

```
[API externa] ──> [Cloud Functions/Run] ──> [GCS /raw/api/] ──> [Spark] ──> [BigQuery]
```

**Padroes suportados:**
- REST com paginacao (offset, cursor, next_token)
- GraphQL com queries parametrizadas
- Webhooks (Cloud Run como receptor, grava em Pub/Sub → GCS)
- APIs autenticadas (OAuth2, API Key, JWT) — secrets no Secret Manager

### 1.4 Servidores de Transferencia

| Protocolo | Tecnologia GCP | Detalhes |
|-----------|---------------|---------|
| **FTP/SFTP** | Storage Transfer Service | Transferencia agendada de servidor FTP/SFTP para GCS |
| **AWS S3 (Direct Connect equiv.)** | Storage Transfer Service | Transferencia cross-cloud S3 → GCS |
| **Azure Blob** | Storage Transfer Service | Transferencia cross-cloud Azure → GCS |
| **On-prem (Direct Connect)** | Transfer Appliance + Interconnect | Transferencia via link dedicado para GCS |

```
[FTP/SFTP] ──> [Storage Transfer Service] ──> [GCS /raw/ftp/] ──> [Spark] ──> [BigQuery]
[S3/Azure] ──> [Storage Transfer Service] ──> [GCS /raw/cloud/] ──> [Spark] ──> [BigQuery]
```

---

## 2. Camada de Processamento — Dataproc Serverless

### Arquitetura dos Jobs Spark

```
GCS (raw/)
  │
  ├── batch_ingestion_job.py      → Leitura JDBC/ODBC, full/incremental load
  ├── cdc_processing_job.py       → Processa arquivos CDC do Datastream
  ├── nosql_ingestion_job.py      → Leitura MongoDB via Spark Connector
  ├── api_transform_job.py        → Schema enforcement + transform de dados de API
  └── file_processing_job.py      → Processa arquivos de FTP/transfer
        │
        ▼
  GCS (trusted/) ──> BigQuery (curated)
```

### Configuracao Dataproc Serverless

```bash
gcloud dataproc batches submit pyspark \
    gs://bucket/jobs/batch_ingestion_job.py \
    --region=us-central1 \
    --subnet=default \
    --jars=gs://bucket/jars/mongo-spark-connector.jar,gs://bucket/jars/jdbc-drivers.jar \
    --properties="spark.dynamicAllocation.enabled=true" \
    --labels="domain=vendas,layer=raw" \
    -- --source=postgresql --table=orders --mode=incremental
```

**Vantagens Dataproc Serverless:**
- Zero gerenciamento de cluster
- Autoscaling automatico
- Paga por uso (por DCU-hora)
- Suporta jars customizados (JDBC drivers, Mongo connector)

---

## 3. Orquestracao — Cloud Composer (Airflow)

### Estrutura de DAGs

```
dags/
├── ingestao/
│   ├── dag_jdbc_batch.py          # Ingestao batch via JDBC
│   ├── dag_cdc_processing.py      # Processamento CDC (triggered por GCS event)
│   ├── dag_mongodb_sync.py        # Sync MongoDB → BigQuery
│   ├── dag_api_extraction.py      # Extracao de APIs + transform
│   └── dag_ftp_transfer.py        # Transfer FTP → GCS → BigQuery
├── qualidade/
│   ├── dag_dataplex_dq.py         # Executa checks de qualidade Dataplex
│   └── dag_data_profiling.py      # Profiling automatico
└── governanca/
    └── dag_catalog_update.py      # Atualiza catalogo Dataplex
```

### Exemplo DAG — Ingestao JDBC

```python
from airflow.decorators import dag, task
from airflow.providers.google.cloud.operators.dataproc import DataprocSubmitBatchOperator
from airflow.providers.google.cloud.operators.dataplex import DataplexCreateTaskOperator

@dag(schedule="0 6 * * *", catchup=False, tags=["ingestao", "jdbc"])
def ingestao_jdbc_vendas():

    ingest = DataprocSubmitBatchOperator(
        task_id="spark_jdbc_ingest",
        batch={
            "pyspark_batch": {
                "main_python_file_uri": "gs://bucket/jobs/batch_ingestion_job.py",
                "args": ["--source=postgresql", "--table=orders", "--mode=incremental"],
                "jar_file_uris": ["gs://bucket/jars/postgresql-42.7.jar"],
            },
            "runtime_config": {"properties": {"spark.dynamicAllocation.enabled": "true"}},
            "labels": {"domain": "vendas", "layer": "raw"},
        },
        region="us-central1",
    )

    quality_check = DataplexCreateTaskOperator(
        task_id="dataplex_quality",
        project_id="meu-projeto",
        region="us-central1",
        lake_id="lake-vendas",
        dataplex_task_id="dq-orders",
        body={
            "spark": {"python_script_file": "gs://bucket/dq/check_orders.py"},
            "trigger_spec": {"type_": "ON_DEMAND"},
        },
    )

    ingest >> quality_check
```

---

## 4. Governanca — Dataplex

### Estrutura de Lakes e Zones

```
Dataplex Organization
│
├── Lake: lake-corporativo
│   ├── Zone: raw (RAW_ZONE)
│   │   ├── Asset: gcs://bucket-raw/jdbc/
│   │   ├── Asset: gcs://bucket-raw/cdc/
│   │   ├── Asset: gcs://bucket-raw/nosql/
│   │   ├── Asset: gcs://bucket-raw/api/
│   │   └── Asset: gcs://bucket-raw/ftp/
│   │
│   ├── Zone: trusted (CURATED_ZONE)
│   │   ├── Asset: gcs://bucket-trusted/
│   │   └── Asset: bigquery://projeto.dataset_trusted
│   │
│   └── Zone: refined (CURATED_ZONE)
│       └── Asset: bigquery://projeto.dataset_refined
```

### Funcionalidades Dataplex Utilizadas

| Recurso | Aplicacao |
|---------|-----------|
| **Data Catalog** | Catalogacao automatica de todos os assets (GCS, BigQuery, MongoDB metadata) |
| **Auto Data Discovery** | Detecta schema, tipo de dado, classificacao automatica |
| **Data Quality** | Regras de qualidade executadas pos-ingestao (nulls, ranges, unicidade, freshness) |
| **Data Lineage** | Linhagem automatica Dataproc → BigQuery. Rastreia origem ate destino |
| **Data Profiling** | Profiling automatico de novas tabelas/particoes |
| **Security Policies** | Controle de acesso por lake/zone/asset via IAM |
| **Custom Metadata** | Tags de dominio, SLA, owner, classificacao (PII, financeiro) |

### Regras de Qualidade (Exemplo)

```yaml
# dataplex_dq_rules.yaml
rules:
  - rule_type: NON_NULL_EXPECTATION
    column: customer_id
    dimension: COMPLETENESS

  - rule_type: RANGE_EXPECTATION
    column: order_total
    dimension: VALIDITY
    min_value: "0"

  - rule_type: UNIQUENESS_EXPECTATION
    column: order_id
    dimension: UNIQUENESS

  - rule_type: SQL_ASSERTION
    dimension: FRESHNESS
    sql_expression: >
      SELECT COUNT(*) = 0
      FROM dataset.orders
      WHERE DATE(ingestion_timestamp) < CURRENT_DATE()
```

---

## 5. Fluxo End-to-End

```
                         ┌─────────────────────────┐
                         │    Cloud Composer        │
                         │    (Orquestracao)         │
                         └────────┬────────────────┘
                                  │ trigger
        ┌─────────────────────────┼─────────────────────────┐
        │                         │                         │
        ▼                         ▼                         ▼
┌──────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│ Datastream   │  │ Storage Transfer Svc │  │ Cloud Functions/Run  │
│ (CDC)        │  │ (FTP/S3/Azure)       │  │ (APIs)               │
└──────┬───────┘  └──────────┬───────────┘  └──────────┬───────────┘
       │                     │                         │
       └─────────┬───────────┴─────────────────────────┘
                 │
                 ▼
        ┌────────────────┐
        │   GCS (raw/)   │
        └───────┬────────┘
                │
                ▼
     ┌─────────────────────┐
     │  Dataproc Serverless │
     │  (Spark Jobs)        │
     │  + MongoDB Connector │
     │  + JDBC Drivers      │
     └─────────┬───────────┘
               │
         ┌─────┴──────┐
         ▼            ▼
   ┌──────────┐ ┌──────────┐
   │ GCS      │ │ BigQuery │
   │ trusted/ │ │ curated  │
   └──────────┘ └──────────┘
         │            │
         └─────┬──────┘
               ▼
     ┌──────────────────┐
     │    DATAPLEX       │
     │ Catalog+Quality   │
     │ Lineage+Security  │
     └──────────────────┘
```

---

## 6. Stack Completa

| Camada | Servico GCP | Funcao |
|--------|------------|--------|
| **Ingestao CDC** | Datastream | Change Data Capture de RDBMS |
| **Ingestao Batch** | Dataproc Serverless (Spark) | JDBC, ODBC, MongoDB, arquivos |
| **Ingestao API** | Cloud Functions / Cloud Run | Extracao de APIs REST/GraphQL |
| **Transferencia** | Storage Transfer Service | FTP, SFTP, S3, Azure Blob |
| **Processamento** | Dataproc Serverless (Spark) | Transformacao, schema enforcement |
| **Armazenamento Raw** | Cloud Storage (GCS) | Landing zone em Parquet/Avro/JSON |
| **Armazenamento Curated** | BigQuery | Data Warehouse analitico |
| **Orquestracao** | Cloud Composer 2 (Airflow) | Scheduling, dependencias, retry |
| **Governanca** | Dataplex | Catalog, qualidade, linhagem, seguranca |
| **Seguranca** | Secret Manager + IAM | Credenciais e controle de acesso |
| **Monitoramento** | Cloud Monitoring + Logging | Alertas, dashboards, logs centralizados |
| **Rede** | VPC + Private Service Connect | Conectividade segura com fontes on-prem |

---

## 7. Estimativa de Custos (Referencia)

| Componente | Modelo de Cobranca | Referencia |
|-----------|-------------------|-----------|
| Dataproc Serverless | DCU-hora (~$0.069/DCU-hr) | Paga só enquanto job roda |
| Datastream | GB processado (~$0.10/GB) | Apenas dados CDC capturados |
| Cloud Composer 2 | Ambiente + workers (~$350-500/mes small) | Ambiente sempre ativo |
| BigQuery | Armazenamento ($0.02/GB/mes) + Queries ($6.25/TB) | Slot pricing alternativo |
| GCS | $0.020/GB/mes (Standard) | Storage intermediario |
| Storage Transfer Service | Gratuito (custo apenas do GCS destino) | Transferencias agendadas |

---

## 8. Proximos Passos

1. **Definir domínios e fontes** — Mapear todas as origens por spoke
2. **Provisionar infraestrutura base** — Terraform/Pulumi para GCS buckets, Dataplex lakes, VPC
3. **Implementar conectores** — Começar por JDBC batch (menor risco), depois CDC, MongoDB, APIs
4. **Configurar Dataplex** — Lakes, zones, regras de qualidade, politicas de acesso
5. **Criar DAGs de orquestracao** — Cloud Composer com templates reutilizaveis por tipo de fonte
6. **Validar com dados reais** — POC com 2-3 fontes representativas
