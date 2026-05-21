# NorthFirst Knowledgebase Architecture Diagrams

These diagrams summarize the current repository architecture and the memo `add_items_to_layer` ingestion flow. They intentionally include current robustness mechanisms and planned AI Guardrails as an extension point.

## 1. System Block Diagram

```mermaid
flowchart TB
    User["USER / CLIENT APPLICATIONS"] --> Gateway["API GATEWAY / LOAD BALANCER"]

    subgraph Platform["EXTERNAL PLATFORM SERVICES"]
        direction LR
        Blob["Blob Storage<br/>Source Documents"]
        AzureDI["Azure Document Intelligence<br/>PDF to Markdown"]
        AzureAI["Azure / OpenAI<br/>LLM + Embeddings"]
        TracingExt["LangSmith / Langfuse<br/>Tracing"]
    end

    subgraph System["NORTHFIRST KNOWLEDGEBASE ARCHITECTURE"]
        direction TB

        subgraph Serving["SERVING LAYER"]
            direction LR
            API["Knowledgebase API<br/>Layers | Processes | Search | Admin/Delete"]
            Status["Status and Result APIs<br/>process status | parent dependencies"]
        end

        subgraph Core["CORE APPLICATION SERVICES"]
            direction LR
            Orchestrator["Processing Orchestrator<br/>background jobs + restart recovery"]
            LayerConfig["Layer Configuration<br/>extensible layer to store mappings"]
            FileManager["File Manager<br/>download, stage, archive"]
            Cleanup["Lifecycle Management<br/>delete by parent_id + reset"]
        end

        subgraph Guardrails["AI GUARDRAILS - PLANNED EXTENSION POINT"]
            direction LR
            InputGuard["Input Guardrails<br/>file, metadata, prompt checks"]
            RetrievalGuard["Retrieval Guardrails<br/>grounding and source quality"]
            OutputGuard["Output Guardrails<br/>citation support, confidence, policy safety"]
            AuditGuard["Audit and Review<br/>traceable decisions + human review hooks"]
        end

        subgraph Ingestor["INGESTOR AND KNOWLEDGE PIPELINE"]
            direction LR
            Intake["Document Intake<br/>PDF parsing, markdown cleanup, hierarchy"]
            Indexing["Chunking and Indexing<br/>org-scoped knowledge collections"]
            Crosswalk["Crosswalk Creation<br/>memo to ecfr/manual/policies"]
            Analysis["Document Analysis<br/>scope, summaries, field extraction"]
        end

        subgraph AIData["AI DATA AND RETRIEVAL LAYER"]
            direction LR
            SearchEngine["Hybrid Search Engine<br/>dense + sparse + metadata filtering"]
            Qdrant["Qdrant Vector DB<br/>ecfr | manuals | memo | policies"]
            Reranker["Reranker Service<br/>cross-encoder relevance ranking"]
        end

        subgraph Datastores["DATABASES, CACHE, AND OUTPUTS"]
            direction LR
            PostgresKB["Postgres Knowledgebase<br/>layers, processes, files, linkages, verdicts"]
            LLMCache["LLM Cache<br/>repeatable and cost-aware LLM calls"]
            Outputs["Document Store<br/>uploaded files + combined JSON outputs"]
        end

        subgraph Ops["OPERATIONS AND EXTENSIBILITY"]
            direction LR
            Persistence["Persistent Volumes<br/>Postgres, Qdrant, downloads, models"]
            Observability["Observability<br/>structured logs + tracing"]
            Extensibility["Extension Points<br/>new layers, stores, pairings, guardrails"]
        end
    end

    Gateway --> API
    API --> Orchestrator
    Orchestrator --> Guardrails
    Orchestrator --> Ingestor
    Ingestor --> AIData
    AIData --> Datastores
    Datastores --> Ops

    Platform -. "document, AI, and tracing integrations" .-> System
```

## 2. Memo Upload and Crosswalk Creation Flow

```mermaid
flowchart TD
    Start["Memo uploaded to Knowledgebase"]
    Validate["Validate request<br/>organization, target layer, processing options"]
    Register["Register processing job<br/>persist job metadata and file references"]
    Stage["Stage source document<br/>download and store the uploaded memo"]
    Accepted["Return tracking ID<br/>client can monitor progress"]

    subgraph Background["BACKGROUND KNOWLEDGE PROCESSING"]
        direction TB
        Prepare["Prepare processing context<br/>organization, layer, and document scope"]
        Extract["Extract document structure<br/>convert PDF, identify sections, summarize content"]
        Index["Index memo knowledge<br/>chunk content and store searchable representations"]
        Crosswalk["Create crosswalks<br/>compare memo sections against reference knowledge layers"]
        ValidateAI["Validate AI-supported matches<br/>grounding, relevance, confidence, and audit trail"]
        Assemble["Assemble dependency map<br/>memo sections mapped to regulations, manuals, and policies"]
    end

    Store["Store final outputs<br/>search index, job status, and crosswalk results"]
    Complete["Mark job complete"]
    Retrieve["Client retrieves status and crosswalk results"]

    Resilience["Resilience controls<br/>restart unfinished jobs, continue independent comparisons, capture failures"]
    Guardrails["AI Guardrails - planned<br/>input checks, retrieval grounding, output validation, human review hooks"]
    Extensible["Extensible design<br/>new document layers, new reference sources, new comparison strategies"]

    Start --> Validate --> Register --> Stage --> Accepted
    Stage --> Prepare --> Extract --> Index --> Crosswalk --> ValidateAI --> Assemble --> Store --> Complete --> Retrieve

    Guardrails -.-> Validate
    Guardrails -.-> Crosswalk
    Guardrails -.-> ValidateAI
    Resilience -.-> Background
    Extensible -.-> Prepare
    Extensible -.-> Crosswalk
```
