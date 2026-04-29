# ArxiVist AI

AI-powered RAG (Retrieval-Augmented Generation) platform for processing academic papers from arXiv and providing intelligent Q&A over structured knowledge. Built on Azure Databricks with a medallion architecture.

## Architecture Overview


┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Bronze     │───>│   Silver     │───>│    Gold      │───>│ Vector Search│
│  Raw Ingest  │    │  Cleansing   │    │  Embeddings  │    │    Index     │
│  (AutoLoader)│    │  & Chunking  │    │ (ai_embed)   │    │ (Delta Sync) │
└──────────────┘    └──────────────┘    └──────────────┘    └──────┬───────┘
                                                                   │
                                                            ┌──────▼───────┐
                                          User Question ───>│  RAG Chain   │
                                                            │  (Retrieval  │
                                                            │  Augmentation│
                                          Answer <──────────│  Generation) │
                                                            └──────┬───────┘
                                                                   │
                                                            ┌──────▼───────┐
                                                            │ Model Serving│
                                                            │ (LLM Endpoint│
                                                            └──────────────┘


## Stage 1: Medallion Pipeline (Data Engineering)

Transform raw documents into structured knowledge ready for search.

### Bronze Layer — Raw Ingestion

* **Action**: Load raw data (PDF documentation, JSON reviews) into cloud storage
* **Tool**: AutoLoader (`cloudFiles`) for automatic incremental file ingestion into Delta
* **Result**: Table with `path` (file path) and `content` (binary data or raw text) columns

### Silver Layer — Cleansing & Chunking

* **Action**: Clean text from artifacts and split long texts into chunks of 500–1000 tokens
* **Important**: Attach metadata to each chunk (file name, category) so the model has context
* **Result**: Delta table where one row = one meaningful text chunk

### Gold Layer — Embeddings

* **Action**: Convert each text chunk into a vector (list of numbers)
* **Tool**: Foundation Model APIs via `ai_embedding_vector()` SQL function (e.g., `databricks-bge-large-en`)
* **Result**: Final table with `text`, `metadata`, and `vector` columns. Change Data Feed enabled

## Stage 2: Vector Search

Make the system capable of instantly finding relevant vectors.

* **Vector Search Endpoint**: Create via Compute → Vector Search (acts as the search server)
* **Delta Sync Index**: Create an index pointing to the Gold table. Databricks auto-syncs — new rows in Gold are automatically vectorized and indexed

## Stage 3: Model Serving

Deploy the LLM that generates answers.

* Navigate to **Serving** and deploy a foundation model (e.g., Llama 3, DBRX)
* This provides an API endpoint that the RAG chain calls for generation

## Stage 4: RAG Orchestration

Python notebook that ties everything into a single chain:

1. **Retrieval**: Convert user question to vector → query Vector Search Index → get top-3 similar chunks
2. **Augmentation**: Insert retrieved chunks into a prompt template:
   > "You are a technical assistant. Using this data: {context from Gold table}, answer the question: {user question}."
3. **Generation**: Send the augmented prompt to the Serving Endpoint → return the final answer

## Infrastructure

* **Cloud**: Azure
* **Compute**: Databricks (serverless)
* **Storage**: ADLS Gen2 (`arxivist-ai` container on `sadlsdev`)
* **Governance**: Unity Catalog
* **Catalog**: `arxivist_ai_dev`
* **Storage Credential**: `dls_dev`

### Storage Layout


abfss://arxivist-ai@sadlsdev.dfs.core.windows.net/
├── raw/
├── bronze/
├── silver/
└── gold/


### Schemas

| Schema                  | Layer  | Description                                 |
|-------------------------|--------|---------------------------------------------|
| `arxivist_ai_dev.raw`   | Raw    | Landed files as-is from external sources    |
| `arxivist_ai_dev.bronze`| Bronze | AutoLoader output — raw content in Delta    |
| `arxivist_ai_dev.silver`| Silver | Cleaned, chunked text with metadata         |
| `arxivist_ai_dev.gold`  | Gold   | Embeddings with Change Data Feed enabled    |

## Setup

The initial infrastructure is provisioned via the `catalog-setup` notebook:

1. Creates an external location covering the ADLS Gen2 container
2. Creates the `arxivist_ai_dev` catalog with managed storage
3. Creates schemas (`raw`, `bronze`, `silver`, `gold`) with dedicated managed locations
4. Disables predictive optimization for cost control

### Prerequisites

* Databricks workspace with Unity Catalog enabled
* Azure ADLS Gen2 storage account (`sadlsdev`) with `arxivist-ai` container
* Storage credential `dls_dev` configured with appropriate access
* `CREATE CATALOG` and `CREATE EXTERNAL LOCATION` privileges
* Access to Foundation Model APIs for embeddings and serving

### Quick Start

1. Run the `catalog-setup` notebook to provision the catalog and schemas
2. Ingest documents into the Bronze layer via AutoLoader
3. Process and chunk text in the Silver layer
4. Generate embeddings in the Gold layer
5. Create a Vector Search endpoint and Delta Sync Index
6. Deploy an LLM via Model Serving
7. Run the RAG orchestration notebook to query the system