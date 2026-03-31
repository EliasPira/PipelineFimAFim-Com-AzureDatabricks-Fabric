# 🚀 Pipeline Fim a Fim com Azure Databricks, ADLS, Fabric e Power BI

![Status](https://img.shields.io/badge/Status-Ativo-brightgreen)
![Azure](https://img.shields.io/badge/Azure-Databricks-blue)
![Fabric](https://img.shields.io/badge/Microsoft-Fabric-purple)
![Python](https://img.shields.io/badge/Python-3.10+-yellow)
![DeltaLake](https://img.shields.io/badge/Delta-Lake-blue)
![License](https://img.shields.io/badge/License-MIT-green)

Este repositório contém um pipeline completo de engenharia de dados utilizando:

- **Azure SQL Database (AdventureWorks)**
- **Azure Data Lake Storage (ADLS)**
- **Azure Databricks (Unity Catalog + Delta Lake)**
- **Fabric Data Factory (orquestração)**
- **Fabric Lakehouse (consumo)**
- **Power BI no Fabric (visualização)**

O objetivo é demonstrar um fluxo **fim a fim**, desde a ingestão dos dados transacionais até a disponibilização da camada Gold para consumo analítico.

---

# 🏗️ Arquitetura da Solução

![Arquitetura](.Imagens/Arquitetura.png)

A solução é composta pelos seguintes blocos:

### 🔹 1. Origem de Dados
- **Azure SQL Database**
  - Servidor: `sqlsrvengdados`
  - Banco: `AdventureWorks`
  - Tabelas transacionais utilizadas como fonte

### 🔹 2. Infraestrutura no Azure
- Resource Group dedicado
- Azure Data Lake Storage Gen2 com containers:
  - **bronze** (dados brutos)
  - **silver** (dados limpos)
  - **gold** (dados analíticos)
- Azure Databricks Workspace
- Access Connector for Databricks
- Unity Catalog + Metastore
- Storage Credentials + External Locations
- Fabric Workspace

### 🔹 3. Processamento no Azure Databricks
- Notebook **Bronze to Silver**
- Notebook **Silver to Gold**
- Escrita em Delta Lake

### 🔹 4. Orquestração no Fabric Data Factory
- Ingestão SQL → ADLS (Bronze)
- Execução dos notebooks no Databricks
- Publicação da camada Gold

### 🔹 5. Disponibilização no Fabric Lakehouse
- Shortcuts apontando para o ADLS (Gold)
- Consumo direto via Power BI (DirectLake)

---

# 🖼️ Diagrama Visual da Arquitetura

```text
                         ┌──────────────────────────────┐
                         │      Azure SQL Database       │
                         │   AdventureWorks (Fonte)      │
                         └───────────────┬──────────────┘
                                         │
                                         ▼
                         ┌──────────────────────────────┐
                         │     Fabric Data Factory       │
                         │  (Ingestão: Copy Activity)    │
                         └───────────────┬──────────────┘
                                         │
                                         ▼
                         ┌──────────────────────────────┐
                         │             ADLS              │
                         │     Container: BRONZE         │
                         └───────────────┬──────────────┘
                                         │
                                         ▼
                         ┌──────────────────────────────┐
                         │       Azure Databricks        │
                         │  Notebook: Bronze → Silver    │
                         └───────────────┬──────────────┘
                                         │
                                         ▼
                         ┌──────────────────────────────┐
                         │             ADLS              │
                         │     Container: SILVER         │
                         └───────────────┬──────────────┘
                                         │
                                         ▼
                         ┌──────────────────────────────┐
                         │       Azure Databricks        │
                         │   Notebook: Silver → Gold     │
                         │  (External Location + MI)     │
                         └───────────────┬──────────────┘
                                         │
                                         ▼
                         ┌──────────────────────────────┐
                         │             ADLS              │
                         │      Container: GOLD          │
                         └───────────────┬──────────────┘
                                         │
                                         ▼
                         ┌──────────────────────────────┐
                         │      Fabric Lakehouse         │
                         │   Shortcuts → GOLD (ADLS)     │
                         └───────────────┬──────────────┘
                                         │
                                         ▼
                         ┌──────────────────────────────┐
                         │      Power BI (Fabric)        │
                         │     DirectLake / Semantic     │
                         └──────────────────────────────┘
```

---

# 📂 Estrutura do Repositório

```
PipelineFimAFim-Com-AzureDatabricks-Fabric/
│
├── Databricks/
│   ├── Bronze to Silver.ipynb
│   ├── Silver to Gold.ipynb
│   └── Connect Databricks to ADLS.ipynb
│
├── Fabric/
│   ├── pipeline.json
│   ├── lakehouse_shortcuts.md
│   └── triggers.md
│
├── Infrastructure/
│   ├── adls.json
│   ├── databricks_workspace.json
│   ├── access_connector.json
│   ├── storage_credentials.sql
│   └── external_locations.sql
│
├── .gitignore
├── .gitattributes
└── README.md
```

---

# 🔗 Conexão Azure ↔ Azure Databricks

Neste projeto foram demonstradas **duas abordagens complementares**:

---

## ✔ Abordagem 1 — OAuth com Client Secret (uso pedagógico)

Utilizada para leitura e transformação nas camadas **Bronze** e **Silver**.

```python
scope="secretscope"
key="engdados-secret"
storage_account="stoengdados"

application_id = dbutils.secrets.get(scope, "application-id")
directory_id = dbutils.secrets.get(scope, "tenant-id")
service_credential = dbutils.secrets.get(scope=scope, key=key)

spark.conf.set(f"fs.azure.account.auth.type.{storage_account}.dfs.core.windows.net", "OAuth")
spark.conf.set(f"fs.azure.account.oauth.provider.type.{storage_account}.dfs.core.windows.net", "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider")
spark.conf.set(f"fs.azure.account.oauth2.client.id.{storage_account}.dfs.core.windows.net", application_id)
spark.conf.set(f"fs.azure.account.oauth2.client.secret.{storage_account}.dfs.core.windows.net", service_credential)
spark.conf.set(f"fs.azure.account.oauth2.client.endpoint.{storage_account}.dfs.core.windows.net", f"https://login.microsoftonline.com/{directory_id}/oauth2/token")
```

---

## ✔ Abordagem 2 — Access Connector + Storage Credential + External Location (moderna)

Utilizada para escrita da camada **GOLD**.

Elimina a necessidade de:

- Client Secret  
- SAS Token  
- Shared Key  

Exemplo:

```sql
CREATE EXTERNAL LOCATION gold_loc
URL 'abfss://gold@<storage>.dfs.core.windows.net/'
WITH (STORAGE_CREDENTIAL sc_stoengdados);
```

---

# 🧹 Processamento: Bronze → Silver

Notebook: **Bronze to Silver**

- Leitura dos dados brutos
- Padronização
- Conversão de tipos
- Escrita em Delta Lake (Silver)

---

# 🔧 Processamento: Silver → Gold

Notebook: **Silver to Gold**

- Criação de dimensões e fatos
- Aplicação de regras de negócio
- Escrita em Delta Lake (Gold)

---

# 🔄 Orquestração no Fabric Data Factory

O pipeline executa:

1. Ingestão SQL → ADLS (Bronze)  
2. Execução dos notebooks no Databricks  
3. Publicação da camada Gold via Shortcuts  

---

# 🏁 Disponibilização da Camada Gold no Fabric

- Shortcuts no Lakehouse apontando para o ADLS  
- Consumo direto via Power BI (DirectLake)  
- Sem cópia de dados  

---

# 📊 Consumo no Power BI (Fabric)

A camada Gold fica disponível para:

- Relatórios Power BI  
- Dashboards  
- Dataflows  
- SQL Endpoint  
- Notebooks Python  

---

# 🤝 Contribuições

Contribuições são bem-vindas.  
Sinta-se à vontade para abrir issues ou enviar pull requests.

---

# 📄 Licença

Este projeto é distribuído sob a licença MIT.
```

---
