# ProofPack AWS Architecture Diagram (Mermaid)

This diagram will render automatically on GitHub, VS Code with Mermaid extension, and many other markdown viewers.

```mermaid
graph TB
    subgraph "Client Layer"
        PWA[Operator PWA<br/>Offline-first capture]
    end
    
    subgraph "API Layer"
        APIGW[API Gateway<br/>JWT Auth]
    end
    
    subgraph "Ingestion Layer"
        LambdaIngress[Lambda: Ingress<br/>Upload Handler]
        S3Raw[S3: Raw Media<br/>90-day retention]
        DDBMeta1[DynamoDB: Metadata<br/>Initial record]
    end
    
    subgraph "Orchestration Layer"
        LambdaOrch[Lambda: Orchestrator<br/>Workflow coordinator]
        StepFunc[Step Functions<br/>Parallel processing]
    end
    
    subgraph "AI Processing Layer"
        LambdaTrans[Lambda: Transcribe<br/>Voice processor]
        Transcribe[Amazon Transcribe<br/>Hindi/English ASR]
        
        LambdaText[Lambda: Textract<br/>Document processor]
        Textract[Amazon Textract<br/>Printed English only]
        
        LambdaImg[Lambda: Image<br/>Preprocessor]
    end
    
    subgraph "Storage Layer"
        DDBMeta2[DynamoDB: Metadata<br/>Extracted data]
        S3Rules[S3: Rule Packs<br/>Per-state JSON]
    end
    
    subgraph "Decision Layer"
        LambdaRules[Lambda: Rule Engine<br/>Deterministic rules]
        LambdaBedrock[Lambda: Bedrock<br/>Explanation generator]
        Bedrock[Amazon Bedrock<br/>Claude 3 Haiku]
    end
    
    subgraph "Output Layer"
        LambdaProof[Lambda: ProofPack<br/>PDF + JSON generator]
        S3Final[S3: ProofPacks<br/>Final outputs]
    end
    
    subgraph "Attestation Layer"
        LambdaOTP[Lambda: OTP<br/>Attestation service]
        DDBaudit[DynamoDB: Audit Log<br/>Immutable trail]
    end
    
    subgraph "Submission Layer"
        LambdaSubmit[Lambda: Submission<br/>Adapter]
        GovPortal[Government Portal<br/>APIs]
    end
    
    %% Flow connections
    PWA -->|POST /ingest| APIGW
    APIGW --> LambdaIngress
    LambdaIngress --> S3Raw
    LambdaIngress --> DDBMeta1
    
    S3Raw -->|S3 event| LambdaOrch
    LambdaOrch --> StepFunc
    
    StepFunc -->|Parallel| LambdaTrans
    StepFunc -->|Parallel| LambdaText
    StepFunc -->|Parallel| LambdaImg
    
    LambdaTrans --> Transcribe
    Transcribe --> DDBMeta2
    
    LambdaText --> Textract
    Textract --> DDBMeta2
    
    LambdaImg --> DDBMeta2
    
    DDBMeta2 --> LambdaRules
    S3Rules --> LambdaRules
    LambdaRules --> DDBMeta2
    
    LambdaRules -->|If incomplete| LambdaBedrock
    LambdaBedrock --> Bedrock
    Bedrock --> DDBMeta2
    
    DDBMeta2 --> LambdaProof
    LambdaProof --> S3Final
    
    S3Final --> LambdaOTP
    LambdaOTP --> DDBaudit
    
    LambdaSubmit --> GovPortal
    LambdaSubmit --> DDBaudit
    
    %% Styling
    classDef awsCompute fill:#FF9900,stroke:#232F3E,stroke-width:2px,color:#fff
    classDef awsStorage fill:#3F8624,stroke:#232F3E,stroke-width:2px,color:#fff
    classDef awsDatabase fill:#3B48CC,stroke:#232F3E,stroke-width:2px,color:#fff
    classDef awsAI fill:#01A88D,stroke:#232F3E,stroke-width:2px,color:#fff
    classDef awsIntegration fill:#E7157B,stroke:#232F3E,stroke-width:2px,color:#fff
    classDef client fill:#879196,stroke:#232F3E,stroke-width:2px,color:#fff
    classDef external fill:#232F3E,stroke:#FF9900,stroke-width:2px,color:#fff
    
    class PWA client
    class APIGW awsIntegration
    class LambdaIngress,LambdaOrch,LambdaTrans,LambdaText,LambdaImg,LambdaRules,LambdaBedrock,LambdaProof,LambdaOTP,LambdaSubmit awsCompute
    class S3Raw,S3Rules,S3Final awsStorage
    class DDBMeta1,DDBMeta2,DDBaudit awsDatabase
    class Transcribe,Textract,Bedrock awsAI
    class StepFunc awsIntegration
    class GovPortal external
```

## How to View This Diagram

1. **On GitHub**: Push this file to your repo and view it - GitHub renders Mermaid automatically
2. **In VS Code**: Install the "Markdown Preview Mermaid Support" extension
3. **Online**: Copy the mermaid code block to https://mermaid.live/

## Architecture Flow Summary

1. **Operator PWA** → API Gateway → Lambda Ingress → S3 Raw Media + DynamoDB
2. **S3 Event** → Lambda Orchestrator → Step Functions (parallel processing)
3. **Parallel AI Processing**:
   - Transcribe Lambda → Amazon Transcribe → DynamoDB
   - Textract Lambda → Amazon Textract → DynamoDB
   - Image Preprocessor → DynamoDB
4. **Rule Engine** loads per-state rules from S3 → evaluates → DynamoDB
5. **Bedrock Lambda** (if needed) → Amazon Bedrock → explanation → DynamoDB
6. **ProofPack Generator** → creates PDF + JSON → S3 Final
7. **OTP Attestation** → validates operator → DynamoDB Audit Log
8. **Submission Adapter** → Government Portal APIs + Audit Log

## Key Components

- **Orange**: Lambda functions (compute)
- **Green**: S3 buckets (storage)
- **Blue**: DynamoDB tables (database)
- **Teal**: AI/ML services (Transcribe, Textract, Bedrock)
- **Pink**: Integration services (API Gateway, Step Functions)
- **Gray**: Client (Operator PWA)
- **Dark**: External systems (Government Portal)
