# Plataforma de Ingestao Automatizada — Hub-Spoke Multicloud

> Baseado na arquitetura de referencia Google Cloud:
> [Build a multicloud open data lakehouse](https://cloud.google.com/architecture/agentic-ai-build-multicloud-open-data-lakehouse)

## Visao Geral

Plataforma de ingestao multicloud (Google Cloud + AWS), onde:

- **Google Cloud** = hub central de governanca, analytics e serving
- **AWS** = spoke de dados tratados por Databricks ou Glue, armazenados em S3 como Apache Iceberg
- **Federacao** = Google Cloud Lakehouse + BigLake conectam ao Databricks Unity Catalog via REST catalog federation
- **Formato aberto** = Apache Iceberg como formato universal, garantindo interoperabilidade entre engines

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          GOOGLE CLOUD (Hub Central)                          │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │              Google Cloud Lakehouse (Governanca Unificada)              │  │
│  │   Catalog Federation · IAM · Audit Trail · Schema Resolution          │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌──────────────┐  ┌─────────────────────┐  ┌────────────────────────────┐  │
│  │  BigQuery    │  │ Managed Spark       │  │  Dataplex                  │  │
│  │  (Serving +  │  │ (Lightning Engine)  │  │  (Quality + Profiling +   │  │
│  │   Analytics) │  │  Cross-cloud joins  │  │   Lineage + Discovery)    │  │
│  └──────────────┘  └─────────────────────┘  └────────────────────────────┘  │
│         │                    │                          │                     │
├─────────┼────────────────────┼──────────────────────────┼─────────────────────┤
│         │           Cross-Cloud Interconnect            │                     │
│         │          (ou VPN / Cloud NAT)                  │                     │
└─────────┼────────────────────┼──────────────────────────┼─────────────────────┘
          │                    │                          │
┌─────────┼────────────────────┼──────────────────────────┼─────────────────────┐
│         ▼                    ▼                          ▼                     │
│  ┌──────────────┐  ┌─────────────────────┐  ┌────────────────────────────┐  │
│  │  S3 Buckets  │  │ Databricks /        │  │  Unity Catalog            │  │
│  │  (Iceberg)   │  │ AWS Glue            │  │  (Metadata + Schema)      │  │
│  └──────────────┘  └─────────────────────┘  └────────────────────────────┘  │
│                                                                              │
│                            AWS (Spoke de Dados)                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Lado AWS — Dados em S3 Iceberg

### 1.1 Pipelines de Escrita (AWS)

Dados sao ingeridos e tratados na AWS por dois engines:

| Engine | Uso | Output |
|--------|-----|--------|
| **Databricks** | ETL/ELT complexo, ML pipelines, streaming | Tabelas Iceberg no S3 |
| **AWS Glue** | ETL serverless, crawlers, jobs PySpark | Tabelas Iceberg no S3 |

Ambos escrevem em formato **Apache Iceberg** no S3, registrados no **Databricks Unity Catalog**.

### 1.2 Formato Iceberg no S3

```
s3://lakehouse-spoke-aws/
├── vendas/
│   ├── metadata/
│   │   ├── v1.metadata.json
│   │   ├── v2.metadata.json
│   │   └── snap-*.avro
│   └── data/
│       ├── part-00000.parquet
│       └── part-00001.parquet
├── clientes/
│   ├── metadata/
│   └── data/
└── produtos/
    ├── metadata/
    └── data/
```

**Vantagens Iceberg:**
- ACID transactions
- Schema evolution sem reescrita
- Time travel / rollback
- Partition evolution
- Engine-agnostico (Spark, Trino, Flink, BigQuery, Databricks, Glue)

### 1.3 Unity Catalog como Metastore

Unity Catalog gerencia:
- Schema das tabelas Iceberg
- Permissoes de acesso (ACLs)
- Linhagem intra-AWS
- Endpoints REST catalog para federacao externa

```
Databricks/Glue ──write──> S3 (Iceberg) ──register──> Unity Catalog
                                                            │
                                                    REST Catalog API
                                                            │
                                                            ▼
                                              Google Cloud Lakehouse
                                                (schema resolution)
```

---

## 2. Lado Google Cloud — Hub Central

### 2.1 Google Cloud Lakehouse + BigLake

Componente central que federea metadados cross-cloud:

```
┌─────────────────────────────────────────────────────┐
│            Google Cloud Lakehouse                    │
│                                                     │
│  1. Conecta ao Unity Catalog via REST federation    │
│  2. Resolve schemas das tabelas Iceberg no S3       │
│  3. Credenciais AWS via Secret Manager              │
│  4. BigLake cria tabelas externas apontando p/ S3   │
│  5. IAM controla acesso unificado                   │
└─────────────────────────────────────────────────────┘
```

**Fluxo de federacao:**

```
BigQuery Query
    │
    ▼
BigLake (tabela externa)
    │
    ▼
Google Cloud Lakehouse
    │
    ├──> Secret Manager (credenciais AWS)
    ├──> Unity Catalog (schema resolution via REST)
    │
    ▼
S3 Bucket (leitura direta dos Parquet/Iceberg)
```

### 2.2 Managed Spark com Lightning Engine

Para joins complexos cross-cloud:

```python
# Spark job no Managed Service for Apache Spark
# Join entre dados GCS (Google) e S3 (AWS) via Iceberg

from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("cross-cloud-join") \
    .config("spark.sql.catalog.aws_catalog", "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.aws_catalog.type", "rest") \
    .config("spark.sql.catalog.aws_catalog.uri", "https://unity-catalog.databricks.com/api/2.1/unity-catalog/iceberg") \
    .config("spark.sql.catalog.aws_catalog.credential", "secret:aws-unity-token") \
    .config("spark.sql.catalog.gcp_catalog", "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.gcp_catalog.type", "rest") \
    .config("spark.sql.catalog.gcp_catalog.uri", "https://biglake.googleapis.com/iceberg") \
    .getOrCreate()

# Dados AWS (Iceberg no S3)
vendas_aws = spark.read.table("aws_catalog.vendas.orders")

# Dados GCP (Iceberg no GCS ou BigQuery)
clientes_gcp = spark.read.table("gcp_catalog.crm.customers")

# Cross-cloud join
perfil_unificado = vendas_aws.join(
    clientes_gcp,
    vendas_aws.customer_id == clientes_gcp.id,
    "inner"
)

# Resultado no BigQuery
perfil_unificado.write \
    .format("bigquery") \
    .option("table", "projeto.analytics.perfil_unificado") \
    .mode("overwrite") \
    .save()
```

### 2.3 BigQuery como Serving Layer

BigQuery serve como camada final de analytics:

| Modo de Acesso | Quando Usar |
|---------------|-------------|
| **Federated Query (BigLake)** | Consultas ad-hoc direto no S3, sem copia |
| **Materialized (Spark → BQ)** | Dados frequentes, joins pre-computados |
| **BigQuery BI Engine** | Dashboards real-time sobre dados materializados |

---

## 3. Governanca Unificada

### 3.1 Camada Dupla de Governanca

```
┌─────────────────────────────────────────────────────────┐
│                  GOVERNANCA UNIFICADA                     │
│                                                          │
│  ┌─────────────────────┐   ┌──────────────────────────┐ │
│  │  Dataplex (GCP)     │   │  Unity Catalog (AWS)     │ │
│  │                     │   │                          │ │
│  │  - Data Quality     │   │  - Schema management     │ │
│  │  - Data Profiling   │   │  - ACLs Databricks       │ │
│  │  - Auto Discovery   │   │  - Lineage intra-AWS     │ │
│  │  - Lineage GCP-side │   │  - REST catalog API      │ │
│  │  - Security Policies│   │                          │ │
│  │  - Classification   │   │                          │ │
│  └─────────┬───────────┘   └────────────┬─────────────┘ │
│            │                            │                │
│            └──────────┬─────────────────┘                │
│                       │                                  │
│            ┌──────────▼──────────┐                       │
│            │  Google Cloud       │                       │
│            │  Lakehouse          │                       │
│            │  (Federacao)        │                       │
│            └─────────────────────┘                       │
└─────────────────────────────────────────────────────────┘
```

### 3.2 Dataplex — Lakes e Zones Multicloud

```
Dataplex
│
├── Lake: lake-corporativo
│   │
│   ├── Zone: raw-gcp (RAW_ZONE)
│   │   ├── Asset: gcs://bucket-raw/jdbc/
│   │   ├── Asset: gcs://bucket-raw/cdc/
│   │   └── Asset: gcs://bucket-raw/api/
│   │
│   ├── Zone: raw-aws (RAW_ZONE) — via BigLake external tables
│   │   ├── Asset: bigquery://projeto.aws_vendas (→ s3://lakehouse/vendas/)
│   │   ├── Asset: bigquery://projeto.aws_clientes (→ s3://lakehouse/clientes/)
│   │   └── Asset: bigquery://projeto.aws_produtos (→ s3://lakehouse/produtos/)
│   │
│   ├── Zone: trusted (CURATED_ZONE)
│   │   └── Asset: bigquery://projeto.trusted_*
│   │
│   └── Zone: refined (CURATED_ZONE)
│       └── Asset: bigquery://projeto.refined_*
```

### 3.3 Regras de Qualidade Cross-Cloud

```yaml
# dataplex_dq_multicloud.yaml

# Regras para dados vindos da AWS (via BigLake)
rules:
  - rule_type: NON_NULL_EXPECTATION
    column: order_id
    dimension: COMPLETENESS
    description: "order_id nunca nulo — origem AWS Databricks"

  - rule_type: SQL_ASSERTION
    dimension: CONSISTENCY
    description: "Todas orders AWS devem ter customer valido no GCP"
    sql_expression: >
      SELECT COUNT(*) = 0
      FROM `projeto.aws_vendas.orders` aws
      LEFT JOIN `projeto.crm.customers` gcp ON aws.customer_id = gcp.id
      WHERE gcp.id IS NULL

  - rule_type: SQL_ASSERTION
    dimension: FRESHNESS
    description: "Dados Iceberg AWS atualizados nas ultimas 24h"
    sql_expression: >
      SELECT COUNT(*) > 0
      FROM `projeto.aws_vendas.orders`
      WHERE DATE(_commit_timestamp) >= CURRENT_DATE() - 1
```

---

## 4. Conectividade Cross-Cloud

### 4.1 Rede

| Componente | Funcao |
|-----------|--------|
| **Cross-Cloud Interconnect** | Link dedicado AWS ↔ GCP (baixa latencia, sem egress público) |
| **Cloud NAT + Cloud Router** | Alternativa via internet para ambientes menores |
| **VPC Peering** | Rede privada dentro do GCP |
| **AWS PrivateLink** | Acesso privado ao S3 e Unity Catalog |

### 4.2 Autenticacao Cross-Cloud

```
┌────────────────────┐         ┌────────────────────┐
│  GCP               │         │  AWS               │
│                    │         │                    │
│  Secret Manager    │────────>│  IAM Role          │
│  (AWS credentials) │         │  (cross-account)   │
│                    │         │                    │
│  Service Account   │         │  S3 Bucket Policy  │
│  (least privilege) │         │  (allow GCP SA)    │
└────────────────────┘         └────────────────────┘
```

**Configuracao:**

```bash
# 1. Criar secret com credenciais AWS no GCP
gcloud secrets create aws-s3-credentials \
    --data-file=aws-credentials.json

# 2. Criar BigLake connection
bq mk --connection \
    --connection_type=AWS \
    --properties='{"crossCloudConfig":{"accessRole":{"iamRoleId":"arn:aws:iam::123456789:role/biglake-reader"}}}' \
    --location=us \
    aws-lakehouse-conn

# 3. Criar tabela externa Iceberg via BigLake
bq mk --table \
    --external_table_definition='{"sourceFormat":"ICEBERG","sourceUris":["s3://lakehouse-spoke-aws/vendas/metadata/v2.metadata.json"],"connectionId":"aws-lakehouse-conn"}' \
    projeto:aws_vendas.orders
```

---

## 5. Fluxo End-to-End Multicloud

```
                    ┌──────────────────────────────────────┐
                    │           AWS (Spoke)                  │
                    │                                        │
  Origens AWS       │  ┌────────────┐   ┌───────────────┐  │
  (RDS, DynamoDB,   │  │ Databricks │   │   AWS Glue    │  │
   APIs, Kinesis)───┼─>│ (ETL/ML)   │   │ (ETL serverl.)│  │
                    │  └─────┬──────┘   └──────┬────────┘  │
                    │        │                 │            │
                    │        └────────┬────────┘            │
                    │                 ▼                     │
                    │     ┌──────────────────────┐         │
                    │     │  S3 (Apache Iceberg)  │         │
                    │     └──────────┬───────────┘         │
                    │                │                      │
                    │     ┌──────────▼───────────┐         │
                    │     │  Unity Catalog        │         │
                    │     │  (REST API)           │         │
                    │     └──────────┬───────────┘         │
                    └────────────────┼──────────────────────┘
                                     │
                        Cross-Cloud Interconnect
                                     │
                    ┌────────────────┼──────────────────────┐
                    │                ▼                      │
                    │  ┌──────────────────────────┐        │
                    │  │  Google Cloud Lakehouse   │        │
                    │  │  + BigLake + Secret Mgr   │        │
                    │  │  (REST Catalog Federation)│        │
                    │  └─────────────┬────────────┘        │
                    │                │                      │
                    │       ┌────────┴─────────┐           │
                    │       ▼                  ▼           │
                    │  ┌──────────┐    ┌─────────────────┐ │
                    │  │ BigQuery │    │ Managed Spark   │ │
                    │  │ (Serving)│    │ Lightning Engine│ │
                    │  └────┬─────┘    │ (Cross-cloud    │ │
                    │       │          │  joins)         │ │
                    │       │          └────────┬────────┘ │
                    │       │                   │          │
  Origens GCP      │       │                   │          │
  (JDBC, CDC,  ────┼───────┼───────────────────┘          │
   MongoDB, API,   │       │                              │
   FTP)            │       ▼                              │
                    │  ┌──────────────────────────────┐    │
                    │  │  Dataplex (Governanca)        │    │
                    │  │  Quality · Lineage · Catalog  │    │
                    │  │  Profiling · Security         │    │
                    │  └──────────────────────────────┘    │
                    │                                      │
                    │        GOOGLE CLOUD (Hub)             │
                    └──────────────────────────────────────┘
```

---

## 6. Comparativo: V1 (Single Cloud) vs V2 (Multicloud)

| Aspecto | V1 — GCP Only | V2 — Multicloud |
|---------|--------------|-----------------|
| **Processamento** | Dataproc Serverless | Managed Spark (Lightning Engine) + Databricks/Glue na AWS |
| **Formato** | Parquet/Avro no GCS | Apache Iceberg no S3 + GCS |
| **Metastore** | Dataplex auto-discovery | Unity Catalog (AWS) + Lakehouse Federation (GCP) |
| **Cross-cloud** | N/A | BigLake external tables + REST catalog federation |
| **CDC** | Datastream → GCS | Datastream (GCP) + DMS/Glue (AWS) → S3 Iceberg |
| **Governanca** | Dataplex only | Dataplex (GCP) + Unity Catalog (AWS), federados |
| **Rede** | VPC interna | Cross-Cloud Interconnect |
| **Custo egress** | Zero | Minimizado via Interconnect; federated queries evitam copia |
| **Serving** | BigQuery | BigQuery (federated + materialized) |

---

## 7. Stack Completa Multicloud

### Google Cloud (Hub)

| Camada | Servico | Funcao |
|--------|---------|--------|
| **Federacao** | Google Cloud Lakehouse + BigLake | REST catalog federation com Unity Catalog |
| **Compute** | Managed Spark (Lightning Engine) | Joins cross-cloud, transformacoes pesadas |
| **Serving** | BigQuery | Analytics, dashboards, federated queries |
| **Governanca** | Dataplex | Quality, profiling, lineage, catalog, security |
| **Ingestao GCP** | Datastream, Cloud Functions, Storage Transfer | CDC, APIs, FTP |
| **Armazenamento** | GCS (Iceberg) | Landing zone dados GCP-nativos |
| **Seguranca** | Secret Manager + IAM | Credenciais AWS, controle acesso unificado |
| **Rede** | Cross-Cloud Interconnect | Link privado AWS ↔ GCP |
| **Orquestracao** | Cloud Composer 2 | Scheduling pipelines cross-cloud |
| **Monitoramento** | Cloud Monitoring + Logging | Observabilidade unificada |

### AWS (Spoke)

| Camada | Servico | Funcao |
|--------|---------|--------|
| **ETL** | Databricks | Pipelines complexos, ML, streaming |
| **ETL Serverless** | AWS Glue | Jobs PySpark serverless, crawlers |
| **Armazenamento** | S3 (Apache Iceberg) | Data lakehouse open format |
| **Metastore** | Databricks Unity Catalog | Schema, ACLs, REST API para federacao |
| **CDC** | AWS DMS | Change Data Capture de RDS/Aurora |
| **Rede** | PrivateLink + Direct Connect | Conectividade privada |

---

## 8. Proximos Passos

1. **Provisionar Cross-Cloud Interconnect** — Link AWS ↔ GCP (ou VPN como MVP)
2. **Configurar Unity Catalog** — Registrar tabelas Iceberg existentes no S3
3. **Criar BigLake connections** — External tables apontando para S3 via REST federation
4. **Validar federated queries** — BigQuery lendo Iceberg do S3 sem copia
5. **Implementar Managed Spark jobs** — Cross-cloud joins com Lightning Engine
6. **Configurar Dataplex** — Quality rules cross-cloud, lineage unificada
7. **POC** — Escolher 1 dominio (ex: vendas) e validar fluxo end-to-end
