# TicketSage RAG Pipeline

TicketSage is an AI powered support intelligence platform built on top of Harborlight Software's helpdesk data. It combines a data engineering pipeline with a retrieval augmented generation layer so that support questions can be answered either through structured analytics or through grounded retrieval of similar past tickets and their resolutions.

## What this project does

Harborlight Software runs a SaaS helpdesk that generates a steady stream of support tickets across multiple channels, products, and regions. TicketSage ingests this raw ticket data, cleans and models it into an analytics ready warehouse, and layers a hybrid retrieval system on top so that support staff and analysts can ask natural language questions and get grounded answers, whether the question needs a number pulled from the database or a suggested resolution drawn from similar past tickets.

## Architecture

The pipeline is organized into four stages.

**Ingestion**
Raw ticket exports land in an S3 bucket as the raw zone. No transformation happens here, only capture with a load timestamp.

**Transformation**
Databricks runs the bronze to silver to gold pipeline. Silver cleans and standardizes the data: type casting, deduplication, and null profiling. Gold organizes the cleaned data into a star schema plus a dedicated text table for the unstructured fields.

**Storage and retrieval**
The gold tables are loaded into a Postgres instance running on an Oracle Cloud VM. The pgvector extension is enabled on this same instance, so structured tables and vector embeddings live side by side. A hybrid retrieval layer decides, per question, whether to query the structured tables directly or run a vector similarity search over embedded ticket text.

**Generation**
A locally hosted Qwen 7B instruct model handles two jobs: writing SQL against the gold tables when a question is structured, and drafting a grounded answer from retrieved ticket and resolution pairs when a question is more open ended.

## Data model

The gold layer is organized as a star schema.

- `fact_ticket`: ticket_id, customer_id, created_at, resolution_time_hours, reopened, csat_score, customer_sentiment, has_attachment, and foreign keys into the dimension tables below.
- `dim_customer`: customer_id, customer_segment.
- `dim_context`: channel, product_area, issue_type, priority, sla_plan, platform, region.
- `ticket_text`: ticket_id, initial_message, agent_first_reply, resolution_summary.

A ticket only qualifies as a trusted resolution, and only trusted resolutions are embedded for retrieval, when all of the following hold: status is resolved, reopened is false, and resolution_summary is not null. This keeps the retrieval index free of resolutions that did not actually hold up.

## Retrieval design

Two retrieval paths sit behind the same interface.

**Text to SQL path**
The question is embedded and matched against a small semantic layer describing the gold tables, their columns, and a handful of example question to SQL pairs. The retrieved context grounds the SQL that Qwen generates, which is then executed against the gold tables in Postgres.

**Ticket similarity path**
The question, or an incoming ticket's initial message, is embedded and matched against the trusted resolution embeddings. The closest past tickets and their resolutions are returned as grounding context, and Qwen drafts an answer from them.

A lightweight router decides which path a given question should take, based on whether it looks like an analytics question or a support question.

## Tech stack

- AWS S3 for raw storage
- Databricks with Spark SQL for the ETL pipeline
- Postgres with pgvector, hosted on an Oracle Cloud VM, for both structured tables and embeddings
- A local sentence embedding model for generating vectors
- Qwen 7B instruct, served locally through Ollama or vLLM, for SQL generation and answer synthesis
- FastAPI for the orchestration layer
- Streamlit for the chat interface

## Project structure

```
ticketsage/
├── ingestion/          # scripts for landing raw data in S3
├── etl/                 # Databricks notebooks and SQL for bronze, silver, gold
├── db/                  # schema definitions, pgvector setup, load scripts
├── embeddings/           # embedding generation for the trusted resolution set
├── retrieval/            # hybrid retrieval logic, schema and example indexing
├── llm/                   # Qwen serving config and prompt templates
├── api/                    # FastAPI orchestration layer
├── app/                     # Streamlit chat frontend
├── eval/                     # evaluation question set and scoring
└── README.md
```

## Setup

1. Provision an S3 bucket for raw ticket data and a Databricks workspace with access to it.
2. Provision an Oracle Cloud VM, install Postgres, and enable the pgvector extension.
3. Run the bronze, silver, gold Databricks jobs to populate the gold tables, then load them into Postgres.
4. Run the embedding job to populate the trusted resolution and semantic layer vector tables.
5. Pull and serve the Qwen 7B instruct model locally through Ollama or vLLM.
6. Start the FastAPI orchestration service, then start the Streamlit app.

Detailed configuration for each step lives in the corresponding subdirectory.

## Evaluation

A held out set of representative questions, covering both the text to SQL path and the ticket similarity path, is used to check retrieval and answer quality. Results and known limitations are tracked in the eval directory as the project evolves.

## Roadmap

- Orchestrate the pipeline end to end with Airflow or dbt
- Add monitoring for embedding freshness and SLA metrics
- Expand the semantic layer with more example question to SQL pairs
- Add a feedback loop so flagged answers improve future retrieval
