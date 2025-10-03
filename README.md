# Assistly Architectures

##  1. High-Level System Architecture

```mermaid
  graph TB
      subgraph "Client Layer"
          User[👤 User Browser]
          UI[Streamlit UI<br/>main.py]
      end

      subgraph "Application Core - RAGPipeline"
          Pipeline[RAGPipeline<br/>Orchestrator]
          RAG[AtlanRAG<br/>Query Processing]
          Classifier[TicketClassifier<br/>Ticket Classification]
          Memory[ConversationMemoryManager<br/>LangChain InMemory]
      end

      subgraph "Production Features Layer"
          Validator[Input Validator<br/>Security & Sanitization]
          RateLimit[Rate Limiter<br/>Token Bucket Algorithm]
          Cache[Cache Manager<br/>3-Tier TTL Cache]
          Retry[Retry Logic<br/>Exponential Backoff]
          Metrics[Metrics Collector<br/>Query & Connection Tracking]
          Health[Connection Health<br/>Service Monitoring]
      end

      subgraph "External AI Services"
          OpenAI[OpenAI GPT-4o<br/>Classification & Generation<br/>httpx Connection Pool]
          Embed[FastEmbed<br/>BAAI/bge-small-en-v1.5<br/>Local Embedding]
      end

      subgraph "Vector & Storage Services"
          Qdrant[Qdrant Cloud<br/>Vector Database<br/>gRPC Connection Pool]
          MongoDB[MongoDB Atlas<br/>Document Store<br/>Scraped Content]
      end

      subgraph "Data Pipeline Layer"
          Scraper[scrape.py<br/>Firecrawl API<br/>Web Scraping]
          Ingestion[qdrant_ingestion.py<br/>Text Splitting & Vectorization]
          Utils[utils.py<br/>MongoDB Helper]
      end

      subgraph "Singleton Managers"
          GM[get_memory_manager]
          GC[get_cache]
          GR[get_rate_limiter]
          GMC[get_metrics_collector]
      end

      %% User Flow
      User --> UI
      UI --> Validator
      Validator --> Pipeline

      %% Pipeline Orchestration
      Pipeline --> RAG
      Pipeline --> Classifier
      Pipeline --> Memory

      %% RAG Component Flow
      RAG --> RateLimit
      RAG --> Cache
      RAG --> Retry
      RAG --> Metrics

      %% Classifier Flow
      Classifier --> RateLimit
      Classifier --> Cache

      %% External Service Connections
      RateLimit --> OpenAI
      RAG --> Embed
      Retry --> Qdrant
      Health --> Qdrant
      Health --> OpenAI

      %% Data Pipeline Flow
      Scraper --> Utils
      Utils --> MongoDB
      MongoDB --> Ingestion
      Ingestion --> Embed
      Embed --> Qdrant

      %% Singleton Access
      Pipeline --> GM
      Pipeline --> GC
      RAG --> GR
      RAG --> GMC

      %% Styling
      style User fill:#42a5f5,stroke:#0d47a1,stroke-width:3px,color:#fff
      style UI fill:#1e88e5,stroke:#0d47a1,stroke-width:2px,color:#fff

      style Pipeline fill:#7b1fa2,stroke:#4a148c,stroke-width:3px,color:#fff
      style RAG fill:#8e24aa,stroke:#4a148c,stroke-width:2px,color:#fff
      style Classifier fill:#8e24aa,stroke:#4a148c,stroke-width:2px,color:#fff
      style Memory fill:#8e24aa,stroke:#4a148c,stroke-width:2px,color:#fff

      style Validator fill:#ef6c00,stroke:#bf360c,stroke-width:2px,color:#fff
      style RateLimit fill:#f57c00,stroke:#bf360c,stroke-width:2px,color:#fff
      style Cache fill:#f57c00,stroke:#bf360c,stroke-width:2px,color:#fff
      style Retry fill:#f57c00,stroke:#bf360c,stroke-width:2px,color:#fff
      style Metrics fill:#f57c00,stroke:#bf360c,stroke-width:2px,color:#fff
      style Health fill:#f57c00,stroke:#bf360c,stroke-width:2px,color:#fff

      style OpenAI fill:#ff6f00,stroke:#e65100,stroke-width:3px,color:#fff
      style Embed fill:#ffa726,stroke:#e65100,stroke-width:2px,color:#fff

      style Qdrant fill:#ab47bc,stroke:#6a1b9a,stroke-width:3px,color:#fff
      style MongoDB fill:#66bb6a,stroke:#2e7d32,stroke-width:3px,color:#fff

      style Scraper fill:#26a69a,stroke:#00695c,stroke-width:2px,color:#fff
      style Ingestion fill:#26a69a,stroke:#00695c,stroke-width:2px,color:#fff
      style Utils fill:#4db6ac,stroke:#00695c,stroke-width:2px,color:#fff

      style GM fill:#78909c,stroke:#455a64,stroke-width:2px,color:#fff
      style GC fill:#78909c,stroke:#455a64,stroke-width:2px,color:#fff
      style GR fill:#78909c,stroke:#455a64,stroke-width:2px,color:#fff
      style GMC fill:#78909c,stroke:#455a64,stroke-width:2px,color:#fff
```

## 2. RAG Pipeline Flow - Complete Request Lifecycle

```mermaid
  flowchart TB
      %% Data Ingestion Layer
      subgraph DataIngestion["📥 DATA INGESTION PIPELINE"]
          WS[("🌐 Web Sources<br/>docs.atlan.com<br/>developer.atlan.com")]
          FC["🔥 Firecrawl API<br/>scrape.py"]
          MDB[("🗄️ MongoDB Atlas<br/>Raw Documents<br/>Metadata Storage")]

          WS -->|HTTP Request| FC
          FC -->|Store Documents| MDB
      end

      %% Vector Processing Layer
      subgraph VectorProcessing["🧬 VECTOR PROCESSING PIPELINE"]
          direction TB
          QI["⚙️ qdrant_ingestion.py"]

          subgraph Chunking["📄 Enhanced Chunking"]
              CP["Code Preservation"]
              RCS["RecursiveCharacterTextSplitter<br/>15+ separators"]
              QM["Quality Metrics<br/>has_code, has_headers"]
          end

          FE["🔢 FastEmbed<br/>BAAI/bge-small-en-v1.5<br/>384-dim vectors"]
          QC[("🎯 Qdrant Cloud<br/>Vector Collections<br/>atlan_docs_enhanced")]

          QI --> Chunking
          Chunking --> FE
          FE -->|Batch Upload| QC
      end

      %% RAG Query Pipeline
      subgraph RAGPipeline["🤖 RAG QUERY PIPELINE"]
          direction TB

          subgraph Input["📝 Input Processing"]
              UQ["User Query"]
              IV["Input Validation<br/>validators.py"]
              RL["Rate Limiter<br/>rate_limiter.py"]
          end

          subgraph Enhancement["🔍 Query Enhancement (Optional)"]
              QE["GPT-4o<br/>Term Expansion"]
              CC["Cache Check<br/>cache_manager.py"]
          end

          subgraph Search["🔎 Vector Search"]
              QG["Query Embedding<br/>FastEmbed"]
              VS["Qdrant Vector Search<br/>Cosine Similarity"]
              RC["Retry Logic<br/>retry_logic.py"]
          end

          subgraph Response["💬 Response Generation"]
              CTX["Context Assembly<br/>Top-K chunks"]
              MEM["Conversation Memory<br/>memory_manager.py"]
              GPT["GPT-4o Response<br/>+Citations"]
          end

          UQ --> IV
          IV --> RL
          RL --> QE
          QE --> CC
          CC --> QG
          QG --> VS
          VS --> RC
          RC --> CTX
          CTX --> MEM
          MEM --> GPT
      end

      %% Classification Pipeline
      subgraph Classification["🏷️ TICKET CLASSIFICATION"]
          direction TB
          TC["Ticket Content"]
          CL["GPT-4o Classifier<br/>rag_pipeline.py"]

          subgraph ClassOutput["Classification Output"]
              TT["Topic Tags<br/>How-to, API, SSO, etc."]
              ST["Sentiment<br/>Frustrated, Curious"]
              PR["Priority<br/>P0, P1, P2"]
          end

          TC --> CL
          CL --> ClassOutput
      end

      %% Streamlit Application Layer
      subgraph StreamlitApp["🖥️ STREAMLIT APPLICATION"]
          direction TB

          subgraph UI["User Interface"]
              DB["📊 Dashboard<br/>Bulk Classification"]
              CA["💬 Chat Agent<br/>Interactive Q&A"]
              SET["⚙️ Settings<br/>Dynamic Config"]
              AN["📈 Analytics<br/>Metrics & Health"]
          end

          subgraph Backend["Backend Services"]
              RAG["AtlanRAG Class<br/>rag_pipeline.py"]
              CACHE["Multi-Level Cache<br/>Embeddings, Search,<br/>Classification, Response"]     
              METRICS["Performance Metrics<br/>metrics.py"]
              HEALTH["Connection Health<br/>connection_health.py"]
          end

          UI --> Backend
      end

      %% External Services
      subgraph ExternalServices["☁️ EXTERNAL SERVICES"]
          OAI["🤖 OpenAI GPT-4o<br/>Classification<br/>Query Enhancement<br/>Response
  Generation"]
          QDB[("🎯 Qdrant Cloud<br/>Vector Database<br/>gRPC Connection")]
          MONGO[("🗄️ MongoDB Atlas<br/>Document Storage<br/>Connection Pool")]
      end

      %% Data Flow Connections
      MDB -.->|Read Documents| QI
      QC -.->|Vector Search| VS

      RAG <-->|API Calls| OAI
      RAG <-->|Vector Search| QDB

      Backend -->|Caching| CACHE
      Backend -->|Tracking| METRICS
      Backend -->|Monitoring| HEALTH

      RAGPipeline -->|Response| StreamlitApp
      Classification -->|Results| StreamlitApp

      %% Production Infrastructure
      subgraph ProdInfra["🛡️ PRODUCTION INFRASTRUCTURE"]
          direction LR
          CM["Cache Manager<br/>1h-30min TTL"]
          RLM["Rate Limiter<br/>Token Bucket"]
          RET["Retry Logic<br/>Exp. Backoff"]
          VAL["Validators<br/>Security Checks"]
          MET["Metrics Collector<br/>Real-time Stats"]
      end

      %% Enhanced Styling with Better Contrast
      classDef dataStore fill:#0d47a1,stroke:#fff,stroke-width:3px,color:#fff
      classDef process fill:#1565c0,stroke:#fff,stroke-width:3px,color:#fff
      classDef ai fill:#f57c00,stroke:#fff,stroke-width:3px,color:#fff
      classDef ui fill:#2e7d32,stroke:#fff,stroke-width:3px,color:#fff
      classDef infra fill:#c62828,stroke:#fff,stroke-width:3px,color:#fff
      classDef chunking fill:#4a148c,stroke:#fff,stroke-width:3px,color:#fff
      classDef input fill:#006064,stroke:#fff,stroke-width:3px,color:#fff
      classDef enhancement fill:#bf360c,stroke:#fff,stroke-width:3px,color:#fff
      classDef search fill:#004d40,stroke:#fff,stroke-width:3px,color:#fff
      classDef response fill:#1b5e20,stroke:#fff,stroke-width:3px,color:#fff
      classDef classOutput fill:#4a148c,stroke:#fff,stroke-width:3px,color:#fff
      classDef uiComponents fill:#1a237e,stroke:#fff,stroke-width:3px,color:#fff
      classDef backend fill:#311b92,stroke:#fff,stroke-width:3px,color:#fff

      class MDB,QC,QDB,MONGO dataStore
      class FC,QI,FE process
      class OAI,QE,GPT,CL ai
      class WS ui
      class CM,RLM,RET,VAL,MET infra
      class CP,RCS,QM chunking
      class UQ,IV,RL input
      class CC enhancement
      class QG,VS,RC search
      class CTX,MEM response
      class TT,ST,PR classOutput
      class DB,CA,SET,AN uiComponents
      class RAG,CACHE,METRICS,HEALTH backend
```

##  3. Security & Input Validation Architecture

```mermaid
 graph TB
      subgraph "Input Processing"
          Input[User Input]
          Sanitize[Text Sanitization]
          Validate[Validation Rules]
      end

      subgraph "Security Checks"
          Length[Length Validation<br/>Min: 3, Max: 5000]
          Injection[Prompt Injection Detection<br/>15+ Patterns]
          Special[Special Character Check<br/>Max 30% ratio]
          Session[Session ID Validation<br/>UUID Format]
      end

      subgraph "Sanitization"
          Null[Remove Null Bytes]
          Whitespace[Normalize Whitespace]
          Control[Strip Control Chars]
          Error[Error Sanitization<br/>Redact API Keys]
      end

      subgraph "Actions"
          Block[Block Request]
          Allow[Allow Request]
          Log[Security Logging]
      end

      Input --> Sanitize
      Sanitize --> Null
      Null --> Whitespace
      Whitespace --> Control
      Control --> Validate

      Validate --> Length
      Validate --> Injection
      Validate --> Special
      Validate --> Session

      Length --> |Invalid| Block
      Injection --> |Detected| Block
      Special --> |High Ratio| Log
      Session --> |Invalid| Block

      Length --> |Valid| Allow
      Injection --> |Clean| Allow
      Session --> |Valid| Allow

      Block --> Log

      %% High-contrast colors
      style Block fill:#e53935,color:#fff,stroke:#b71c1c,stroke-width:2px
      style Allow fill:#43a047,color:#fff,stroke:#1b5e20,stroke-width:2px
      style Log fill:#fbc02d,color:#000,stroke:#f57f17,stroke-width:2px
```

##  4. Caching Strategy - Multi-Level Architecture

```mermaid
graph TB
      subgraph "Entry Point"
          User[User Query/Request]
      end

      subgraph "RAG Pipeline Operations"
          Classify[Classify Ticket<br/>OpenAI API]
          Search[Search Qdrant<br/>Vector Search]
          Generate[Generate Response<br/>OpenAI API]
      end

      subgraph "Cache Layer - RAGCache Singleton"
          CC[Classification Cache<br/>TTLCache<br/>500 items, 3600s TTL]
          SC[Search Cache<br/>TTLCache<br/>500 items, 1800s TTL]
          RC[Response Cache<br/>TTLCache<br/>500 items, 3600s TTL]
      end

      subgraph "Cache Key Generation"
          SHA1[SHA-256 Hash<br/>ticket text]
          SHA2[SHA-256 Hash<br/>query + top_k + score + collection]
          SHA3[SHA-256 Hash<br/>query + context_hash]
      end

      subgraph "Thread Safety"
          Lock1[Classification Lock]
          Lock2[Search Lock]
          Lock3[Response Lock]
      end

      subgraph "Cache Statistics"
          Stats[Hit/Miss Tracking<br/>Size Monitoring<br/>Hit Rate Calculation]
      end

      subgraph "Benefits"
          Cost[💰 Cost Reduction<br/>Reduce OpenAI API Calls]
          Speed[⚡ Performance<br/>Instant Cache Response]
          Load[📉 Load Reduction<br/>Lower Qdrant Traffic]
      end

      User --> Classify
      User --> Search
      User --> Generate

      Classify --> SHA1
      Search --> SHA2
      Generate --> SHA3

      SHA1 --> CC
      SHA2 --> SC
      SHA3 --> RC

      CC --> Lock1
      SC --> Lock2
      RC --> Lock3

      Lock1 --> Stats
      Lock2 --> Stats
      Lock3 --> Stats

      CC -->|Cache Hit| Speed
      SC -->|Cache Hit| Speed
      RC -->|Cache Hit| Speed

      CC -->|Reduce Calls| Cost
      RC -->|Reduce Calls| Cost
      SC -->|Reduce Queries| Load

      %% Styling
      style CC fill:#66bb6a,color:#fff,stroke:#388e3c,stroke-width:3px
      style SC fill:#ab47bc,color:#fff,stroke:#8e24aa,stroke-width:3px
      style RC fill:#42a5f5,color:#fff,stroke:#1e88e5,stroke-width:3px

      style Classify fill:#ffa726,color:#fff,stroke:#f57c00,stroke-width:2px
      style Search fill:#ffa726,color:#fff,stroke:#f57c00,stroke-width:2px
      style Generate fill:#ffa726,color:#fff,stroke:#f57c00,stroke-width:2px

      style SHA1 fill:#78909c,color:#fff,stroke:#546e7a,stroke-width:2px
      style SHA2 fill:#78909c,color:#fff,stroke:#546e7a,stroke-width:2px
      style SHA3 fill:#78909c,color:#fff,stroke:#546e7a,stroke-width:2px

      style Lock1 fill:#607d8b,color:#fff,stroke:#455a64,stroke-width:2px
      style Lock2 fill:#607d8b,color:#fff,stroke:#455a64,stroke-width:2px
      style Lock3 fill:#607d8b,color:#fff,stroke:#455a64,stroke-width:2px

      style Cost fill:#ef5350,color:#fff,stroke:#c62828,stroke-width:2px
      style Speed fill:#66bb6a,color:#fff,stroke:#2e7d32,stroke-width:2px
      style Load fill:#29b6f6,color:#fff,stroke:#0277bd,stroke-width:2px

      style User fill:#fff176,color:#000,stroke:#f9a825,stroke-width:3px
      style Stats fill:#90a4ae,color:#fff,stroke:#546e7a,stroke-width:2px
```

##  5. Rate Limiting System

```mermaid
graph TB
      subgraph "Rate Limiters"
          OL[OpenAI Limiter<br/>50 calls/min<br/>Token Bucket]
          QL[Qdrant Limiter<br/>100 calls/min<br/>Token Bucket]
          UL[Query Limiter<br/>30 calls/min<br/>Token Bucket]
          CL[Classification Limiter<br/>40 calls/min<br/>Token Bucket]
      end

      subgraph "Request Flow"
          R[Incoming Request]
          Check{Check Limit}
          Window[Time Window<br/>60 seconds]
          Count[Count Calls]
          Wait[Calculate Wait Time]
      end

      subgraph "Actions"
          Allow[Allow Request<br/>Add to Bucket]
          Deny[Deny Request<br/>Return Wait Time]
          Log[Log Rate Limit Event]
      end

      R --> Check
      Check --> |OpenAI Call| OL
      Check --> |Qdrant Call| QL
      Check --> |User Query| UL
      Check --> |Classification| CL

      OL --> Window
      QL --> Window
      UL --> Window
      CL --> Window

      Window --> Count
      Count --> |Under Limit| Allow
      Count --> |Over Limit| Wait
      Wait --> Deny

      Deny --> Log
      Allow --> Log

      %% Improved contrast colors
      style Allow fill:#388E3C,fontColor:#ffffff,stroke:#1B5E20
      style Deny fill:#D32F2F,fontColor:#ffffff,stroke:#B71C1C
      style Log fill:#FBC02D,fontColor:#000000,stroke:#F57F17

      style OL fill:#90CAF9,stroke:#1565C0
      style QL fill:#CE93D8,stroke:#6A1B9A
      style UL fill:#A5D6A7,stroke:#2E7D32
      style CL fill:#FFAB91,stroke:#D84315
```

## 6. Retry Logic & Error Handling

```mermaid
graph TB
      subgraph "Operation Types"
          O1[OpenAI Call<br/>3 retries<br/>1s base delay]
          Q1[Qdrant Call<br/>3 retries<br/>0.5s base delay]
          E1[Embedding Gen<br/>2 retries<br/>1s base delay]
      end

      subgraph "Retry Strategy"
          Attempt[Execute Operation]
          Fail{Failed?}
          Type{Error Type}
          Backoff[Exponential Backoff<br/>delay = base × 2^attempt]
          MaxRetry{Max Retries?}
      end

      subgraph "Error Handling"
          Transient[Transient Error<br/>Connection/Timeout]
          Permanent[Permanent Error<br/>Validation/Auth]
          Retry[Retry with Delay]
          Abort[Abort & Return Error]
          Log[Error Logging]
      end

      O1 --> Attempt
      Q1 --> Attempt
      E1 --> Attempt

      Attempt --> Fail
      Fail --> |No| Success[Return Result]
      Fail --> |Yes| Type

      Type --> Transient
      Type --> Permanent

      Transient --> MaxRetry
      Permanent --> Abort

      MaxRetry --> |No| Backoff
      MaxRetry --> |Yes| Abort

      Backoff --> Retry
      Retry --> Attempt

      Abort --> Log
      Success --> Log

      %% High contrast styles
      style Success fill:#388E3C,fontColor:#ffffff,stroke:#1B5E20
      style Abort fill:#D32F2F,fontColor:#ffffff,stroke:#B71C1C
      style Retry fill:#FBC02D,fontColor:#000000,stroke:#F57F17
      style Log fill:#1976D2,fontColor:#ffffff,stroke:#0D47A1

      style Transient fill:#81D4FA,stroke:#0277BD
      style Permanent fill:#FFAB91,stroke:#D84315
      style Backoff fill:#CE93D8,stroke:#6A1B9A
```

##   7. Memory Management & Conversation Flow

```mermaid
graph TB
    subgraph SessionMgmt["Session Management"]
        Create["Create Session\nUUID Generation"]
        Store["Session Storage\nIn-Memory Dict"]
        Expire["Auto Cleanup\n60 min timeout"]
    end

    subgraph MessageMgmt["Message Management"]
        User[User Message]
        AI[AI Response]
        History["Chat History\nLangChain InMemory"]
        Trim["Auto Trim\nMax 20 messages"]
        Remove[Remove Oldest Pairs]
    end

    subgraph ContextRet["Context Retrieval"]
        Get[Get Context]
        Last5["Last 5 Exchanges\n10 messages"]
        Format["Format for Prompt\nUser: ... Assistant: ..."]
    end

    subgraph Cleanup["Cleanup"]
        Auto["Auto Cleanup\nEvery 100 ops"]
        Manual["Manual Cleanup\nForce Clear"]
        Stats["Cleanup Stats\nTracking"]
    end

    Create --> Store
    Store --> Expire

    User --> History
    AI --> History
    History --> Trim
    Trim --> Remove

    Get --> Last5
    Last5 --> Format
    Format --> RAG[RAG Pipeline]

    Expire --> Auto
    Expire --> Manual
    Auto --> Stats
    Manual --> Stats

    %% Styles for better contrast
    style Create fill:#42A5F5,color:#ffffff,stroke:#1565C0
    style Store fill:#90CAF9,stroke:#1E88E5
    style Expire fill:#64B5F6,stroke:#1976D2

    style History fill:#BA68C8,color:#ffffff,stroke:#6A1B9A
    style Trim fill:#CE93D8,stroke:#8E24AA
    style Remove fill:#AB47BC,color:#ffffff,stroke:#4A148C

    style Format fill:#81C784,color:#ffffff,stroke:#388E3C
    style Last5 fill:#2E7D32,color:#ffffff,stroke:#1B5E20
    style Get fill:#66BB6A,color:#ffffff,stroke:#1B5E20
    style RAG fill:#43A047,color:#ffffff,stroke:#1B5E20

    style Auto fill:#FFB74D,stroke:#EF6C00
    style Manual fill:#FFA726,stroke:#E65100
    style Stats fill:#FFD54F,stroke:#F57F17
```

## 8. Metrics Collection & Monitoring

```mermaid
  graph TB
      subgraph "Metrics Types"
          QM[Query Metrics<br/>Response Time<br/>Tokens Used<br/>Search Method]
          CM[Connection Pool<br/>Active Connections<br/>Wait Times<br/>Errors]
          PM[Performance<br/>P50/P95/P99<br/>Percentiles]
          Cost[Cost Estimation<br/>Token Usage<br/>API Calls]
      end

      subgraph "Collection"
          Event[Operation Event]
          Timer[Performance Timer<br/>Context Manager]
          Record[Record Metrics<br/>Thread-Safe]
          Aggregate[Aggregate Stats<br/>Real-time]
      end

      subgraph "Analysis"
          HitRate[Cache Hit Rate]
          ErrorRate[Error Rate %]
          AvgTime[Avg Response Time]
          TokenCost[Cost per Query]
      end

      subgraph "Reporting"
          JSON[JSON Export]
          Logs[Structured Logs]
          Dashboard[Metrics Dashboard]
      end

      Event --> Timer
      Timer --> QM
      Timer --> CM
      Timer --> PM
      Timer --> Cost

      QM --> Record
      CM --> Record
      PM --> Record
      Cost --> Record

      Record --> Aggregate

      Aggregate --> HitRate
      Aggregate --> ErrorRate
      Aggregate --> AvgTime
      Aggregate --> TokenCost

      HitRate --> JSON
      ErrorRate --> JSON
      AvgTime --> JSON
      TokenCost --> JSON

      JSON --> Logs
      JSON --> Dashboard

      %% Improved color contrast
      style QM fill:#42A5F5,fontColor:#ffffff,stroke:#1565C0
      style CM fill:#64B5F6,fontColor:#000000,stroke:#1E88E5
      style PM fill:#1E88E5,fontColor:#ffffff,stroke:#0D47A1
      style Cost fill:#FFB74D,fontColor:#000000,stroke:#EF6C00
      style Record fill:#CE93D8,fontColor:#000000,stroke:#8E24AA
      style Aggregate fill:#9C27B0,fontColor:#ffffff,stroke:#6A1B9A
      style HitRate fill:#81C784,fontColor:#ffffff,stroke:#388E3C
      style ErrorRate fill:#E57373,fontColor:#ffffff,stroke:#D32F2F
      style AvgTime fill:#64B5F6,fontColor:#000000,stroke:#1976D2
      style TokenCost fill:#FFD54F,fontColor:#000000,stroke:#F57F17
      style JSON fill:#FFF59D,fontColor:#000000,stroke:#FBC02D
      style Logs fill:#90A4AE,fontColor:#ffffff,stroke:#607D8B
      style Dashboard fill:#81C784,fontColor:#ffffff,stroke:#388E3C
```

##   9. Data Pipeline - Scraping to Vector Storage

```mermaid
graph LR
    subgraph DataSources["Data Sources"]
        D1["docs.atlan.com\n~1078 pages"]
        D2["developer.atlan.com\n~611 pages"]
    end
    subgraph ScrapingLayer["Scraping Layer"]
        F["Firecrawl API\nRate Limited"]
        Extract["Content Extraction\nMarkdown + Metadata"]
        M["MongoDB Storage\nRaw Documents"]
    end
    subgraph Processing["Processing"]
        Read[Read from MongoDB]
        Mode{Ingestion Mode}
        Inc["Incremental\nTimestamp-based\n~0.1s"]
        Full["Full Reindex\nScroll-based\nWeekly"]
    end
    subgraph Chunking["Chunking"]
        Split["Enhanced Splitting\n1200 tokens\n200 overlap"]
        Code["Code Block\nPreservation"]
        Quality["Quality Metrics\nCode/Headers/Words"]
    end
    subgraph Vectorization["Vectorization"]
        Embed["FastEmbed BGE\n384 dimensions"]
        Batch["Batch Processing\n50 chunks/batch"]
        Q["Qdrant Upload\nVector + Payload"]
    end
    D1 --> F
    D2 --> F
    F --> Extract
    Extract --> M
    M --> Read
    Read --> Mode
    Mode --> Inc
    Mode --> Full
    Inc --> Split
    Full --> Split
    Split --> Code
    Code --> Quality
    Quality --> Embed
    Embed --> Batch
    Batch --> Q
    
    %% Improved colors for maximum contrast and visibility
    style D1 fill:#1976D2,color:#ffffff,stroke:#0D47A1,stroke-width:3px
    style D2 fill:#1976D2,color:#ffffff,stroke:#0D47A1,stroke-width:3px
    
    style F fill:#F57C00,color:#ffffff,stroke:#E65100,stroke-width:3px
    style Extract fill:#FB8C00,color:#ffffff,stroke:#E65100,stroke-width:3px
    style M fill:#388E3C,color:#ffffff,stroke:#1B5E20,stroke-width:3px
    
    style Read fill:#43A047,color:#ffffff,stroke:#2E7D32,stroke-width:3px
    style Mode fill:#F9A825,color:#000000,stroke:#F57F17,stroke-width:3px
    style Inc fill:#FBC02D,color:#000000,stroke:#F57F17,stroke-width:3px
    style Full fill:#FDD835,color:#000000,stroke:#F57F17,stroke-width:3px
    
    style Split fill:#0288D1,color:#ffffff,stroke:#01579B,stroke-width:3px
    style Code fill:#0277BD,color:#ffffff,stroke:#01579B,stroke-width:3px
    style Quality fill:#01579B,color:#ffffff,stroke:#004D40,stroke-width:3px
    
    style Embed fill:#7B1FA2,color:#ffffff,stroke:#4A148C,stroke-width:3px
    style Batch fill:#6A1B9A,color:#ffffff,stroke:#4A148C,stroke-width:3px
    style Q fill:#8E24AA,color:#ffffff,stroke:#4A148C,stroke-width:3px
```

##  10. Classification & Routing Logic

```mermaid
graph TB
    subgraph Input["Input"]
        Ticket["Support Ticket\nSubject + Body"]
    end

    subgraph Classification["Classification"]
        GPT["GPT-4o Classifier\nTemp: 0.1"]
        Topics["Topic Tags\n9 Categories"]
        Sentiment["Sentiment\n4 Types"]
        Priority["Priority\nP0/P1/P2"]
    end

    subgraph TopicCategories["Topic Categories"]
        T1[How-to]
        T2[Product]
        T3[Connector]
        T4[Lineage]
        T5[API/SDK]
        T6[SSO]
        T7[Glossary]
        T8[Best Practices]
        T9[Sensitive Data]
    end

    subgraph RoutingDecision["Routing Decision"]
        Check{"Topic in\nRAG List?"}
        RAG["RAG Pipeline\nAI Response"]
        Route["Team Routing\nReference ID"]
    end

    subgraph RAGTopicsDefault["RAG Topics Default"]
        R1[How-to]
        R2[Product]
        R3[Best Practices]
        R4[API/SDK]
        R5[SSO]
    end

    Ticket --> GPT
    GPT --> Topics
    GPT --> Sentiment
    GPT --> Priority

    Topics --> T1 & T2 & T3 & T4 & T5 & T6 & T7 & T8 & T9

    T1 --> Check
    T2 --> Check
    T3 --> Check
    T4 --> Check
    T5 --> Check
    T6 --> Check
    T7 --> Check
    T8 --> Check
    T9 --> Check

    Check --> |Match| RAG
    Check --> |No Match| Route

    R1 & R2 & R3 & R4 & R5 -.->|RAG Topics| Check

    style Ticket fill:#1976D2,color:#ffffff,stroke:#0D47A1,stroke-width:3px
    style GPT fill:#1E88E5,color:#ffffff,stroke:#0D47A1,stroke-width:3px
    style Topics fill:#42A5F5,color:#ffffff,stroke:#1565C0,stroke-width:3px
    style Sentiment fill:#42A5F5,color:#ffffff,stroke:#1565C0,stroke-width:3px
    style Priority fill:#42A5F5,color:#ffffff,stroke:#1565C0,stroke-width:3px
    
    style T1 fill:#7B1FA2,color:#ffffff,stroke:#4A148C,stroke-width:2px
    style T2 fill:#7B1FA2,color:#ffffff,stroke:#4A148C,stroke-width:2px
    style T3 fill:#7B1FA2,color:#ffffff,stroke:#4A148C,stroke-width:2px
    style T4 fill:#7B1FA2,color:#ffffff,stroke:#4A148C,stroke-width:2px
    style T5 fill:#7B1FA2,color:#ffffff,stroke:#4A148C,stroke-width:2px
    style T6 fill:#7B1FA2,color:#ffffff,stroke:#4A148C,stroke-width:2px
    style T7 fill:#7B1FA2,color:#ffffff,stroke:#4A148C,stroke-width:2px
    style T8 fill:#7B1FA2,color:#ffffff,stroke:#4A148C,stroke-width:2px
    style T9 fill:#7B1FA2,color:#ffffff,stroke:#4A148C,stroke-width:2px
    
    style Check fill:#F9A825,color:#000000,stroke:#F57F17,stroke-width:3px
    style RAG fill:#2E7D32,color:#ffffff,stroke:#1B5E20,stroke-width:3px
    style Route fill:#FBC02D,color:#000000,stroke:#F57F17,stroke-width:3px
    
    style R1 fill:#43A047,color:#ffffff,stroke:#2E7D32,stroke-width:2px
    style R2 fill:#43A047,color:#ffffff,stroke:#2E7D32,stroke-width:2px
    style R3 fill:#43A047,color:#ffffff,stroke:#2E7D32,stroke-width:2px
    style R4 fill:#43A047,color:#ffffff,stroke:#2E7D32,stroke-width:2px
    style R5 fill:#43A047,color:#ffffff,stroke:#2E7D32,stroke-width:2px
```

##  11. Settings & Configuration Management

```mermaid
graph TB
    subgraph SettingsUI["Settings UI"]
        UI["Settings Page\nStreamlit Tabs"]
        Search["Search Settings\nCollection/Top-K/Threshold"]
        Model["Model Settings\nLLM/Temp/Tokens"]
        Features["Feature Toggles\nQuery Enhancement"]
        Routing["RAG Topics Config\nMulti-select"]
        UISet["UI Preferences\nShow Analysis"]
    end

    subgraph Validation["Validation"]
        Pydantic["Pydantic Validation\nType Safety"]
        Ranges["Range Checks\nMin/Max Values"]
        Warnings["Warning System\nConfig Issues"]
    end

    subgraph Application["Application"]
        Session["Session State\nStreamlit"]
        Pipeline["RAG Pipeline\nSettings Update"]
        Apply["Dynamic Apply\nNo Restart"]
    end

    subgraph Persistence["Persistence"]
        Export["Export JSON\nBackup Config"]
        Import["Import JSON\nRestore Config"]
        Defaults[Reset to Defaults]
    end

    UI --> Search
    UI --> Model
    UI --> Features
    UI --> Routing
    UI --> UISet

    Search --> Pydantic
    Model --> Pydantic
    Features --> Pydantic

    Pydantic --> Ranges
    Ranges --> Warnings

    Warnings --> Session
    Session --> Pipeline
    Pipeline --> Apply

    Session --> Export
    Import --> Session
    Defaults --> Session

    style UI fill:#1976D2,color:#ffffff,stroke:#0D47A1,stroke-width:3px
    style Search fill:#42A5F5,color:#ffffff,stroke:#1565C0,stroke-width:2px
    style Model fill:#42A5F5,color:#ffffff,stroke:#1565C0,stroke-width:2px
    style Features fill:#42A5F5,color:#ffffff,stroke:#1565C0,stroke-width:2px
    style Routing fill:#42A5F5,color:#ffffff,stroke:#1565C0,stroke-width:2px
    style UISet fill:#42A5F5,color:#ffffff,stroke:#1565C0,stroke-width:2px
    
    style Pydantic fill:#1E88E5,color:#ffffff,stroke:#0D47A1,stroke-width:3px
    style Ranges fill:#42A5F5,color:#ffffff,stroke:#1565C0,stroke-width:2px
    style Warnings fill:#F9A825,color:#000000,stroke:#F57F17,stroke-width:3px
    
    style Session fill:#7B1FA2,color:#ffffff,stroke:#4A148C,stroke-width:2px
    style Pipeline fill:#7B1FA2,color:#ffffff,stroke:#4A148C,stroke-width:2px
    style Apply fill:#2E7D32,color:#ffffff,stroke:#1B5E20,stroke-width:3px
    
    style Export fill:#00796B,color:#ffffff,stroke:#004D40,stroke-width:2px
    style Import fill:#00796B,color:#ffffff,stroke:#004D40,stroke-width:2px
    style Defaults fill:#00796B,color:#ffffff,stroke:#004D40,stroke-width:2px
```

##   12. Connection Pool & Resource Management

```mermaid
graph TB
    subgraph HTTPClient["HTTP Client - OpenAI"]
        HTTP["HTTPX Client\nConnection Pooling"]
        MaxConn[Max Connections: 20]
        KeepAlive[Keep-Alive: 5]
        Timeout[Timeout: 30s]
    end

    subgraph QdrantClient["Qdrant Client"]
        GRPC[gRPC Options]
        MaxMsg[Max Message: 100MB]
        KeepAliveQ[Keep-Alive: 30s]
        TimeoutQ[Timeout: 10s]
    end

    subgraph MongoDBClient["MongoDB Client"]
        Mongo[PyMongo MongoClient]
        AutoPool["Auto Connection Pool\n✅ Default: 100 connections"]
        ThreadSafe["Thread-Safe\n✅ Automatic"]
        Reuse["Connection Reuse\n✅ Built-in"]
    end

    subgraph Monitoring["Monitoring"]
        Track[Connection Tracking]
        Slow["Slow Operation Detection\n>5s threshold"]
        Errors["Connection Errors\nLogging"]
        Metrics["Pool Metrics\nUtilization/Wait Time"]
    end

    HTTP --> MaxConn
    MaxConn --> KeepAlive
    KeepAlive --> Timeout

    GRPC --> MaxMsg
    MaxMsg --> KeepAliveQ
    KeepAliveQ --> TimeoutQ

    Mongo --> AutoPool
    AutoPool --> ThreadSafe
    ThreadSafe --> Reuse

    HTTP --> Track
    GRPC --> Track
    Mongo --> Track

    Track --> Slow
    Track --> Errors
    Track --> Metrics

    style HTTP fill:#1E88E5,color:#ffffff,stroke:#0D47A1,stroke-width:3px
    style MaxConn fill:#42A5F5,color:#ffffff,stroke:#1565C0,stroke-width:2px
    style KeepAlive fill:#42A5F5,color:#ffffff,stroke:#1565C0,stroke-width:2px
    style Timeout fill:#42A5F5,color:#ffffff,stroke:#1565C0,stroke-width:2px
    
    style GRPC fill:#7B1FA2,color:#ffffff,stroke:#4A148C,stroke-width:3px
    style MaxMsg fill:#9C27B0,color:#ffffff,stroke:#6A1B9A,stroke-width:2px
    style KeepAliveQ fill:#9C27B0,color:#ffffff,stroke:#6A1B9A,stroke-width:2px
    style TimeoutQ fill:#9C27B0,color:#ffffff,stroke:#6A1B9A,stroke-width:2px
    
    style Mongo fill:#2E7D32,color:#ffffff,stroke:#1B5E20,stroke-width:3px
    style AutoPool fill:#388E3C,color:#ffffff,stroke:#2E7D32,stroke-width:2px
    style ThreadSafe fill:#388E3C,color:#ffffff,stroke:#2E7D32,stroke-width:2px
    style Reuse fill:#388E3C,color:#ffffff,stroke:#2E7D32,stroke-width:2px
    
    style Track fill:#D32F2F,color:#ffffff,stroke:#B71C1C,stroke-width:3px
    style Slow fill:#F44336,color:#ffffff,stroke:#C62828,stroke-width:2px
    style Errors fill:#F44336,color:#ffffff,stroke:#C62828,stroke-width:2px
    style Metrics fill:#388E3C,color:#ffffff,stroke:#2E7D32,stroke-width:2px
```
