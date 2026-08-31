# BRD Agent — Databricks RAG-Based Business Requirements Document Generator

An agentic RAG application built entirely within the **Databricks ecosystem** that automatically generates a Business Requirements Document (BRD) from an existing Databricks notebook.

The system combines **Databricks Vector Search, Delta tables, embeddings, retrieval-augmented generation (RAG), Databricks-hosted LLMs, and automated DOCX generation** to transform technical notebook logic into a structured business-facing document.

---

## 🚀 Overview

Data and analytics projects often contain the actual business logic inside Databricks notebooks, while the corresponding business requirements documentation is created separately and manually.

This project automates that documentation workflow.

Given a Databricks notebook path, the BRD Agent:

1. Reads the current notebook source.
2. Retrieves relevant notebook logic from a Vector Search index.
3. Retrieves examples and structural guidance from a separate "gold BRD" knowledge base.
4. Combines the retrieved context with structured system instructions.
5. Sends the context to a Databricks-hosted LLM.
6. Generates a structured BRD.
7. Exports the result as HTML and Microsoft Word (`.docx`).

The architecture is designed around **grounded generation**, so the generated BRD is based on the notebook's actual implementation and retrieved reference material rather than relying solely on the LLM's general knowledge.

---

## 🏗️ Architecture

```text
                    ┌─────────────────────────┐
                    │  Databricks Notebook    │
                    │  containing ETL /       │
                    │  transformation logic   │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │ Notebook Chunking /     │
                    │ Delta Table             │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │ Databricks Vector       │
                    │ Search                  │
                    │ Notebook Index          │
                    └────────────┬────────────┘
                                 │
                                 │
        ┌────────────────────────┘
        │
        │                    ┌─────────────────────────┐
        │                    │ Gold BRD Documents     │
        │                    │ / BRD Structure        │
        │                    └────────────┬────────────┘
        │                                 │
        │                                 ▼
        │                    ┌─────────────────────────┐
        │                    │ Databricks Vector       │
        │                    │ Search                  │
        │                    │ Gold BRD Index          │
        │                    └────────────┬────────────┘
        │                                 │
        ▼                                 ▼
┌─────────────────────────────────────────────────────┐
│                 RAG Context                         │
│                                                     │
│  Notebook Source + Notebook Retrieval + Gold BRD   │
│  Retrieval + System Instructions                    │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │ Databricks LLM       │
              │ databricks-gpt-5-2   │
              └──────────┬───────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │ Generated BRD        │
              │ HTML / DOCX          │
              └──────────────────────┘
```

---

## 🧠 RAG Design

The project uses **two separate retrieval sources**.

### 1. Notebook Logic Knowledge Base

The notebook source is represented as chunks and stored in a Delta table:

```text
main.doc_test.notebook_chunks
```

The Vector Search index is:

```text
main.doc_test.brd_notebook_index
```

The index uses `chunk_id` as its primary key and generates embeddings from the `content` column using:

```text
databricks-bge-large-en
```

Additional metadata such as notebook name, schema, domain, chunk type, and cell index is synchronized with the index.

This allows the agent to retrieve relevant technical logic from the notebook.

---

### 2. Gold BRD Knowledge Base

A separate Delta table stores BRD reference chunks:

```text
main.doc_test.brd_chunks
```

The corresponding Vector Search index is:

```text
main.doc_test.brd_gold_brd_index
```

This knowledge base provides examples and structural guidance for how BRDs should be organized and mapped.

Keeping the notebook knowledge and BRD knowledge in separate indexes allows the system to distinguish between:

* **What the notebook actually does**
* **How that information should be documented**

---

## 🔎 Retrieval Pipeline

At runtime, the agent performs two similarity searches.

### Notebook retrieval

The agent queries the notebook Vector Search index and retrieves the top:

```text
25 chunks
```

The retrieved content is combined into the notebook logic context.

### BRD retrieval

The agent separately queries the gold BRD index and retrieves the top:

```text
10 chunks
```

These chunks provide BRD structure and mapping guidance.

---

## 🤖 LLM Generation

The generated context is passed to a Databricks-hosted LLM:

```text
databricks-gpt-5-2
```

with:

```text
temperature = 0.3
```

The final prompt contains four major components:

```text
System Instructions
        +
Notebook Content
        +
Retrieved Notebook Context
        +
Retrieved Gold BRD Context
        ↓
     LLM Generation
        ↓
      BRD Output
```

The prompt explicitly separates these contexts and instructs the model to generate the BRD according to the system instructions.

---

## 🛡️ Context Engineering & Grounding

Rather than relying on a single large prompt, the project uses structured context engineering.

The system prompt is split into four independently maintained files:

```text
01_identity.txt
02_parsing_rules.txt
03_brd_structure.txt
04_docx_formatting.txt
```

These are loaded from a Databricks Volume and combined at runtime.

The system also performs integrity checks to ensure critical instructions such as anti-hallucination rules and final projection/mapping rules are present before generation.

This provides a stronger grounding mechanism than relying purely on prompt wording.

---

## 📄 BRD Generation

The agent accepts two runtime parameters through Databricks widgets:

| Parameter       | Description                                 |
| --------------- | ------------------------------------------- |
| `notebook_path` | Path of the Databricks notebook to document |
| `brd_file_name` | Name of the generated BRD                   |

The notebook source is retrieved directly through the Databricks Workspace API using `WorkspaceClient` and exported as source code.

The generated response is then converted into document output.

### Output formats

The agent generates:

* HTML
* Microsoft Word (`.docx`)

The DOCX generation uses `python-docx` and parses the generated HTML using BeautifulSoup.

Generated documents are written to a Databricks Volume.

---

## 🗄️ Databricks Data Layer

The project keeps the RAG data layer inside Unity Catalog / Delta.

### Notebook chunks

```text
main.doc_test.notebook_chunks
```

Contains fields including:

```text
chunk_id
notebook_name
schema
domain
chunk_type
cell_index
content
```

### BRD chunks

```text
main.doc_test.brd_chunks
```

Contains:

```text
chunk_id
brd_name
section
domain
content
```

Both tables use Delta format.

---

## 🔄 Vector Search Synchronization

The Vector Search indexes are configured as **Delta Sync indexes** with a triggered pipeline.

```text
Delta Table
    ↓
Delta Sync
    ↓
Vector Search Index
```

Both indexes use `chunk_id` as their primary key and generate embeddings from the `content` column.

The implementation also enables **Delta Change Data Feed** on the source tables, allowing changes to the underlying Delta data to be propagated through the synchronization workflow.

Index synchronization can be explicitly triggered using:

```python
notebook_index.sync()
brd_index.sync()
```

and the resulting indexes can be validated through the Vector Search client.

---

## 🔧 Technology Stack

| Component           | Technology                         |
| ------------------- | ---------------------------------- |
| Cloud / Platform    | Databricks                         |
| Data Storage        | Delta Lake                         |
| Governance          | Unity Catalog                      |
| Vector Database     | Databricks Vector Search           |
| Embeddings          | `databricks-bge-large-en`          |
| LLM                 | `databricks-gpt-5-2`               |
| LLM Framework       | LangChain / `databricks_langchain` |
| Notebook Access     | Databricks Workspace SDK           |
| Document Generation | Python-docx                        |
| HTML Parsing        | BeautifulSoup                      |
| Language            | Python                             |
| Runtime             | Databricks Notebook                |

---

## 📁 Project Structure

```text
BRD-Agent/
│
├── 01_create_vector_database.ipynb
│
├── 02_create_vector_search_indexes.ipynb
│
├── 03_runtime_brd_agent.ipynb
│
├── system_prompt/
│   ├── 01_identity.txt
│   ├── 02_parsing_rules.txt
│   ├── 03_brd_structure.txt
│   └── 04_docx_formatting.txt
│
└── README.md
```

> Rename the notebook filenames above to match the actual filenames in your repository.

---

## 🚀 Setup

### 1. Databricks Environment

The project requires access to a Databricks workspace with:

* Unity Catalog
* Delta Lake
* Databricks Vector Search
* A Databricks-hosted embedding endpoint
* A Databricks-hosted LLM endpoint
* Databricks Volumes

---

### 2. Install Vector Search SDK

Inside the Databricks notebook:

```python
%pip install databricks-vectorsearch
```

Restart the Python environment after installation.

---

### 3. Configure Vector Search

The project uses:

```text
Endpoint:
brd-agent-vs-endpoint

Embedding model:
databricks-bge-large-en

Notebook index:
main.doc_test.brd_notebook_index

BRD index:
main.doc_test.brd_gold_brd_index
```

These values can be changed in the configuration section of the notebooks.

---

## ▶️ Running the Project

Run the notebooks in the following order.

### Step 1 — Prepare the Vector Database

Run the notebook responsible for preparing the Delta-backed chunk tables.

This stage:

1. Creates the Unity Catalog schema.
2. Creates the notebook and BRD chunk tables.
3. Loads chunk data into Delta.
4. Removes duplicate notebook chunks.
5. Enables Delta Change Data Feed.

---

### Step 2 — Create Vector Search Indexes

Create the Databricks Vector Search endpoint and indexes.

The project creates two indexes:

```text
brd_notebook_index
brd_gold_brd_index
```

Both are Delta Sync indexes using the configured embedding model.

Trigger synchronization after loading the source data.

---

### Step 3 — Run the BRD Agent

Provide:

```text
notebook_path
brd_file_name
```

The agent then:

```text
Read Notebook
      ↓
Retrieve Notebook Context
      ↓
Retrieve Gold BRD Context
      ↓
Build Grounded Prompt
      ↓
Call Databricks LLM
      ↓
Generate BRD
      ↓
Save HTML
      ↓
Generate DOCX
```

The runtime notebook implements this flow directly.

---

## 🔍 Example

Input:

```text
Databricks Notebook:
<path to notebook>
```

Output:

```text
BRD_<project_name>.html
BRD_<project_name>.docx
```

The generated document contains structured sections such as:

* Document Control
* Overview and Scope
* Functional Requirements
* Business Context
* Data / Transformation Logic
* Source Dependencies
* Target Mapping
* Other sections defined by the BRD system instructions

The exact BRD structure is controlled by the system prompt and retrieved gold-BRD context.

---

## ✅ Validation

The implementation includes validation at multiple layers.

### Vector database validation

The Vector Search client is used to verify that the expected indexes exist:

```python
vsc.list_indexes(VS_ENDPOINT_NAME)
```

The resulting indexes are validated as `DELTA_SYNC` indexes with `chunk_id` as the primary key.

### Data validation

The notebook checks for duplicate chunk IDs before writing the deduplicated dataset back to Delta.

### Prompt validation

Critical system instructions are checked before the LLM is called to prevent silent prompt-loading failures.

### Output validation

The generated HTML and DOCX files are written to the configured Databricks Volume and the generation path is reported after successful creation.

---

## 🎯 Key Design Decisions

### Separate retrieval indexes

Notebook logic and BRD examples are stored separately rather than combining them into one knowledge base.

This allows the agent to retrieve:

```text
Technical truth
      +
Documentation structure
```

independently.

### Retrieval before generation

The LLM receives retrieved evidence before generating the BRD, reducing reliance on its pretrained knowledge.

### Structured system instructions

The system prompt is modularized into multiple files rather than maintaining one large hard-coded prompt.

### Delta-backed Vector Search

Using Delta tables as the source of the Vector Search indexes keeps the RAG knowledge base within the Databricks data platform.

### Triggered synchronization

Vector indexes can be synchronized after changes to the source Delta tables rather than rebuilding the complete vector database manually.

---

## ⚠️ Current Limitations

The current implementation is a working prototype and has several areas that can be improved.

* Retrieval currently uses fixed `TOP_K` values.
* Notebook retrieval is based on a query derived from the notebook content.
* There is currently no explicit retrieval evaluation framework such as Recall@K or Precision@K.
* The DOCX conversion focuses primarily on structured text and may require additional handling for complex HTML elements such as advanced tables or formatting.
* Authentication currently uses Databricks notebook authentication during development.
* The current workflow requires the Databricks environment and configured endpoints.
* The system prompt and Volume paths are environment-specific and should be parameterized for production deployment.

---

## 🔮 Future Improvements

Potential extensions include:

* Hybrid keyword + vector retrieval
* Retrieval reranking
* Better semantic chunking
* Retrieval evaluation with a golden question/answer dataset
* Automated BRD quality scoring
* Citation / source lineage inside generated BRDs
* Improved DOCX table and formatting support
* Model evaluation and comparison
* MLflow tracing and evaluation
* Production authentication using service principals
* Databricks Model Serving deployment
* Automated index refresh pipelines
* Multi-notebook project-level BRD generation
* Human-in-the-loop BRD review and approval

---

## 💡 What This Project Demonstrates

This project demonstrates practical implementation of:

* Retrieval-Augmented Generation (RAG)
* Vector databases
* Semantic search
* Embedding-based retrieval
* Context engineering
* Prompt engineering
* Agentic document generation
* Databricks Vector Search
* Delta Lake
* Unity Catalog
* LLM integration
* Automated document generation
* Grounded AI workflows

More importantly, it demonstrates how an LLM can be integrated with an enterprise data platform to transform **existing technical implementation into structured business documentation** while grounding generation in project-specific context.

---

## 👤 Author

**Shivam Kumar**

Built as a practical exploration of **RAG, context engineering, Databricks Vector Search, and agentic LLM workflows** for automating technical-to-business documentation.

