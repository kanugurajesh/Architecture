# Assistly

##  1. High-Level System Architecture

```mermaid
graph TB
      subgraph "Client Layer"
          User[👤 User Browser]
          UI[Streamlit UI]
      end

      subgraph "Application Layer"
          Main[main.py<br/>Streamlit App]
          Pipeline[rag_pipeline.py<br/>RAG Engine]
          Classifier[TicketClassifier]
          Memory[ConversationMemoryManager]
      end

      subgraph "Production Features"
          RateLimit[Rate Limiter<br/>Multi-tier]
          Cache[Cache Manager<br/>TTL-based]
          Retry[Retry Logic<br/>Exponential Backoff]
          Metrics[Metrics Collector<br/>Performance Tracking]
          Validator[Input Validator<br/>Security]
      end

      subgraph "External Services"
          OpenAI[OpenAI GPT-4o<br/>Classification & RAG]
          Qdrant[Qdrant Cloud<br/>Vector DB]
          MongoDB[MongoDB Atlas<br/>Document Store]
      end

      subgraph "Data Pipeline"
          Scraper[scrape.py<br/>Firecrawl API]
          Ingestion[qdrant_ingestion.py<br/>Vector Processing]
      end

      User --> UI
      UI --> Main
      Main --> Validator
      Main --> Pipeline
      Main --> Classifier
      Main --> Memory

      Pipeline --> RateLimit
      Pipeline --> Cache
      Pipeline --> Retry
      Pipeline --> Metrics

      RateLimit --> OpenAI
      Pipeline --> Qdrant
      Classifier --> OpenAI

      Scraper --> MongoDB
      MongoDB --> Ingestion
      Ingestion --> Qdrant

      style User fill:#42a5f5,stroke:#0d47a1,stroke-width:2px
      style OpenAI fill:#ffb74d,stroke:#e65100,stroke-width:2px
      style Qdrant fill:#ce93d8,stroke:#6a1b9a,stroke-width:2px
      style MongoDB fill:#81c784,stroke:#2e7d32,stroke-width:2px
```

## 2. RAG Pipeline Flow - Complete Request Lifecycle

```mermaid
sequenceDiagram
      participant U as User
      participant UI as Streamlit UI
      participant V as Validator
      participant RL as Rate Limiter
      participant C as Cache
      participant P as RAG Pipeline
      participant CL as Classifier
      participant M as Memory Manager
      participant Q as Qdrant
      participant O as OpenAI
      participant Met as Metrics

      U->>UI: Submit Query
      UI->>V: Validate Input
      V->>V: Check Length/Security
      V->>V: Sanitize Text
      V-->>UI: Validation Result

      UI->>RL: Check Rate Limit
      RL-->>UI: Allow/Deny

      UI->>CL: Classify Ticket
      CL->>C: Check Cache
      alt Cache Hit
          C-->>CL: Cached Classification
      else Cache Miss
          CL->>O: GPT-4o Classification
          O-->>CL: Topic/Sentiment/Priority
          CL->>C: Store in Cache
      end
      CL-->>UI: Classification

      UI->>UI: Determine Response Type<br/>(RAG vs Routing)

      alt RAG Response
          UI->>P: Generate RAG Response
          P->>M: Get Conversation History
          M-->>P: Context (last 5 exchanges)

          P->>P: Enhance Query (optional)
          P->>C: Check Search Cache

          alt Search Cache Hit
              C-->>P: Cached Results
          else Search Cache Miss
              P->>Q: Vector Search
              Q-->>P: Top-K Documents
              P->>C: Cache Results
          end

          P->>O: Generate Response with Context
          O-->>P: AI Response
          P->>M: Store Conversation Turn
          P-->>UI: Response + Sources
      else Routing Response
          UI->>UI: Generate Routing Message
          UI-->>U: Team Routing Info
      end

      UI->>Met: Record Metrics
      Met->>Met: Update Stats
      UI-->>U: Display Response
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

      style Block fill:#ffcdd2
      style Allow fill:#c8e6c9
      style Log fill:#fff9c4
```

##  4. Caching Strategy - Multi-Level Architecture

```mermaid
graph LR
      subgraph "Cache Layers"
          EC[Embedding Cache<br/>1000 items<br/>TTL: 1 hour]
          SC[Search Cache<br/>500 items<br/>TTL: 30 min]
          CC[Classification Cache<br/>500 items<br/>TTL: 1 hour]
      end

      subgraph "Operations"
          Q1[Query Request]
          E1[Embedding Generation]
          S1[Search Request]
          C1[Classification Request]
      end

      subgraph "Storage"
          SHA[SHA-256 Hash Keys]
          TTL[TTL Expiration]
          Lock[Thread-Safe Locks]
      end

      subgraph "Benefits"
          Cost[Cost Reduction<br/>Save API Calls]
          Speed[Performance<br/>Sub-ms Response]
          Load[Load Reduction<br/>Decrease Traffic]
      end

      Q1 --> E1
      E1 --> EC
      EC --> |Hit| Speed
      EC --> |Miss| E1

      Q1 --> S1
      S1 --> SC
      SC --> |Hit| Speed
      SC --> |Miss| S1

      Q1 --> C1
      C1 --> CC
      CC --> |Hit| Speed
      CC --> |Miss| C1

      EC --> SHA
      SC --> SHA
      CC --> SHA

      SHA --> TTL
      TTL --> Lock

      EC --> Cost
      SC --> Cost
      CC --> Cost

      style EC fill:#e3f2fd
      style SC fill:#f3e5f5
      style CC fill:#e8f5e9
      style Cost fill:#fff3e0
      style Speed fill:#c8e6c9
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

      style Allow fill:#c8e6c9
      style Deny fill:#ffcdd2
      style Log fill:#fff9c4
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

      style Success fill:#c8e6c9
      style Abort fill:#ffcdd2
      style Retry fill:#fff9c4
```

##   7. Memory Management & Conversation Flow

```mermaid
graph TB
    subgraph "Session Management"
        Create[Create Session - UUID Generation]
        Store[Session Storage - In-Memory Dict]
        Expire[Auto Cleanup - 60 min timeout]
    end

    subgraph "Message Management"
        User[User Message]
        AI[AI Response]
        History[Chat History - LangChain InMemory]
        Trim[Auto Trim - Max 20 messages]
        Remove[Remove Oldest Pairs]
    end

    subgraph "Context Retrieval"
        Get[Get Context]
        Last5[Last 5 Exchanges - 10 messages]
        Format[Format for Prompt - User: ... Assistant: ...]
    end

    subgraph "Cleanup"
        Auto[Auto Cleanup - Every 100 ops]
        Manual[Manual Cleanup - Force Clear]
        Stats[Cleanup Stats - Tracking]
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

    style Create fill:#e3f2fd
    style History fill:#f3e5f5
    style Format fill:#e8f5e9
    style Stats fill:#fff9c4
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

      style QM fill:#e3f2fd
      style Cost fill:#fff3e0
      style Dashboard fill:#c8e6c9
```

##   9. Data Pipeline - Scraping to Vector Storage

```mermaid
graph LR
      subgraph "Data Sources"
          D1[docs.atlan.com<br/>~1078 pages]
          D2[developer.atlan.com<br/>~611 pages]
      end

      subgraph "Scraping Layer"
          F[Firecrawl API<br/>Rate Limited]
          Extract[Content Extraction<br/>Markdown + Metadata]
          M[MongoDB Storage<br/>Raw Documents]
      end

      subgraph "Processing"
          Read[Read from MongoDB]
          Mode{Ingestion Mode}
          Inc[Incremental<br/>Timestamp-based<br/>~0.1s]
          Full[Full Reindex<br/>Scroll-based<br/>Weekly]
      end

      subgraph "Chunking"
          Split[Enhanced Splitting<br/>1200 tokens<br/>200 overlap]
          Code[Code Block<br/>Preservation]
          Quality[Quality Metrics<br/>Code/Headers/Words]
      end

      subgraph "Vectorization"
          Embed[FastEmbed BGE<br/>384 dimensions]
          Batch[Batch Processing<br/>50 chunks/batch]
          Q[Qdrant Upload<br/>Vector + Payload]
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

      style F fill:#fff3e0
      style M fill:#e8f5e9
      style Embed fill:#e3f2fd
      style Q fill:#f3e5f5
```

##  10. Classification & Routing Logic

```mermaid
graph TB
      subgraph "Input"
          Ticket[Support Ticket<br/>Subject + Body]
      end

      subgraph "Classification"
          GPT[GPT-4o Classifier<br/>Temp: 0.1]
          Topics[Topic Tags<br/>9 Categories]
          Sentiment[Sentiment<br/>4 Types]
          Priority[Priority<br/>P0/P1/P2]
      end

      subgraph "Topic Categories"
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

      subgraph "Routing Decision"
          Check{Topic in<br/>RAG List?}
          RAG[RAG Pipeline<br/>AI Response]
          Route[Team Routing<br/>Reference ID]
      end

      subgraph "RAG Topics Default"
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

      style RAG fill:#c8e6c9
      style Route fill:#fff9c4
      style GPT fill:#e3f2fd
```

##  11. Settings & Configuration Management

```mermaid
graph TB
      subgraph "Settings UI"
          UI[Settings Page<br/>Streamlit Tabs]
          Search[Search Settings<br/>Collection/Top-K/Threshold]
          Model[Model Settings<br/>LLM/Temp/Tokens]
          Features[Feature Toggles<br/>Query Enhancement]
          Routing[RAG Topics Config<br/>Multi-select]
          UISet[UI Preferences<br/>Show Analysis]
      end

      subgraph "Validation"
          Pydantic[Pydantic Validation<br/>Type Safety]
          Ranges[Range Checks<br/>Min/Max Values]
          Warnings[Warning System<br/>Config Issues]
      end

      subgraph "Application"
          Session[Session State<br/>Streamlit]
          Pipeline[RAG Pipeline<br/>Settings Update]
          Apply[Dynamic Apply<br/>No Restart]
      end

      subgraph "Persistence"
          Export[Export JSON<br/>Backup Config]
          Import[Import JSON<br/>Restore Config]
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

      style Pydantic fill:#e3f2fd
      style Apply fill:#c8e6c9
      style Warnings fill:#fff9c4
```

##   12. Connection Pool & Resource Management

```mermaid
graph TB
      subgraph "HTTP Client - OpenAI"
          HTTP[HTTPX Client<br/>Connection Pooling]
          MaxConn[Max Connections: 20]
          KeepAlive[Keep-Alive: 5]
          Timeout[Timeout: 30s]
      end

      subgraph "Qdrant Client"
          GRPC[gRPC Options]
          MaxMsg[Max Message: 100MB]
          KeepAliveQ[Keep-Alive: 30s]
          TimeoutQ[Timeout: 10s]
      end

      subgraph "MongoDB Client"
          Mongo[PyMongo MongoClient]
          AutoPool[Auto Connection Pool<br/>✅ Default: 100 connections]
          ThreadSafe[Thread-Safe<br/>✅ Automatic]
          Reuse[Connection Reuse<br/>✅ Built-in]
      end

      subgraph "Monitoring"
          Track[Connection Tracking]
          Slow[Slow Operation Detection<br/>>5s threshold]
          Errors[Connection Errors<br/>Logging]
          Metrics[Pool Metrics<br/>Utilization/Wait Time]
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

      style HTTP fill:#e3f2fd
      style GRPC fill:#f3e5f5
      style AutoPool fill:#c8e6c9
      style Metrics fill:#c8e6c9
```
