---
marp: true
theme: gcloud
paginate: true
footer: 'Hub-Spoke Analítico · Google Cloud'
---

<!-- _class: title -->

# Hub-Spoke Analítico no Google Cloud

### Da engenharia centralizada ao domínio com governança

*Arquitetura de dados para escala com controle*

---

<!-- _class: section -->

# Situação

---

## Dados analíticos são infraestrutura crítica em todos os domínios de negócio

- Vendas, finanças, logística, marketing — todos dependem de dados confiáveis
- **Qualidade e acesso** determinam a velocidade das decisões
- Cada domínio tem contexto que a engenharia central não tem

---

<!-- _class: section -->

# Complicação

---

## Data Lake centralizou o gargalo; Data Mesh distribuiu o caos

*Três modelos, três problemas — nenhum resolveu autonomia e governança ao mesmo tempo*

- **Data Lake:** equipe central como único produtor — gargalo inevitável
- **Data Mesh:** silos, padrões divergentes, governança impossível
- Cada solução criou o próximo problema

---

<!-- _class: invert -->

## Dados inconsistentes entre domínios custam mais do que toda a engenharia que os produz

- Decisões baseadas em definições conflitantes de métricas
- **Retrabalho** de consolidação manual entre áreas
- Auditoria e compliance inviáveis sem linhagem centralizada

---

<!-- _class: lead -->

# Como ter autonomia de domínio sem sacrificar governança?

---

<!-- _class: section -->

# Resposta
## Hub-Spoke encerra o ciclo de trade-offs entre controle e velocidade analítica

---

## O Hub centraliza ingestão, governança e marketplace como infraestrutura compartilhada

*Dataflow · Pub/Sub · Composer · Dataplex · Analytics Hub (Sharing)*

- **Ingestão:** Dataflow e Dataproc Serverless com self-service por spoke
- **Governança:** Dataplex — catálogo, qualidade, linhagem, IAM centralizado
- **Publicação:** Analytics Hub como marketplace interno de produtos de dados

---

<!-- _class: invert -->

## A Equipe de Plataforma multiplica capacidade analítica sem crescer proporcionalmente

- A antiga engenharia central deixa de atender chamados e passa a **construir plataforma**
- Self-service de ingestão, onboarding e acesso — automação elimina trabalho manual
- Cada spoke onboardado aumenta a capacidade sem aumentar o time de plataforma

---

## Spokes transformam dados brutos em produtos certificados via Dataform e BigQuery

*Cada spoke é dono do conhecimento do seu domínio — não da infraestrutura*

- Recebem dados brutos padronizados do hub
- Aplicam regras de negócio com **Dataform** — SQL versionado com testes e linhagem
- Camada refined publicada somente após validação automática do Dataplex

---

<!-- _class: invert -->

## Domínios publicam produtos de dados sem depender de filas da engenharia central

- Ciclo completo dentro do domínio: transformar → certificar → publicar
- **Analytics Hub** entrega linked datasets — sem mover dados, sem duplicação
- Consumidores encontram tudo em um marketplace com catálogo único

---

## Agentes de IA geram transformações precisas alimentados pelos metadados do Dataplex

*O hub provê o contexto que torna os agentes confiáveis e auditáveis*

- Esquemas, linhagem e contratos de dados como contexto para os agentes
- **Dataplex** valida o código gerado pelo agente antes de publicar
- Ciclo: analista descreve → agente gera → Dataplex certifica

---

<!-- _class: invert -->

## Analistas de negócio produzem produtos de dados sem precisar dominar SQL ou Spark

- O conhecimento de domínio se torna a **única competência necessária** no spoke
- Agentes geram, testam e documentam as transformações
- O hub garante que a velocidade dos agentes não sacrifique qualidade nem governança

---

## Cada spoke adere ao hub incrementalmente — sem big-bang, sem paralisação

*Hub pronto primeiro — spokes migram por ondas, priorizados por impacto*

- Infraestrutura central operacional antes do primeiro spoke migrar
- **Legado convive** com o hub durante toda a transição
- Padrão se consolida a cada onda — risco decai, velocidade aumenta

---

<!-- _class: invert -->

## Cada domínio migrado gera valor imediato e reduz o custo e risco da próxima onda

- Primeiro spoke valida o padrão e documenta o processo de onboarding
- **Cada migração** é prova de conceito para a seguinte
- Sem esperar migração completa para gerar retorno — valor começa no primeiro spoke

---

<!-- _class: closing -->

# Hub-Spoke: governança que habilita, autonomia que escala

*Centralizar o que é plataforma. Distribuir o que é domínio.*
