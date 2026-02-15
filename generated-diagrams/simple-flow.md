# ProofPack Simple Flow Diagram

```mermaid
flowchart LR
    A[👤 Operator PWA] -->|Upload| B[🚪 API Gateway]
    B --> C[⚡ Lambda Ingress]
    C --> D[📦 S3 Raw Media]
    C --> E[(💾 DynamoDB)]
    
    D --> F[⚡ Orchestrator]
    F --> G[🔄 Step Functions]
    
    G --> H[🎤 Transcribe Service]
    G --> I[📄 Textract Service]
    G --> J[🖼️ Image Processor]
    
    H --> E
    I --> E
    J --> E
    
    E --> K[⚖️ Rule Engine]
    L[📋 S3 Rules] --> K
    K --> E
    
    K --> M[🤖 Bedrock AI]
    M --> E
    
    E --> N[📑 ProofPack Generator]
    N --> O[📦 S3 ProofPacks]
    
    O --> P[🔐 OTP Service]
    P --> Q[(📊 Audit Log)]
    
    R[📤 Submit Adapter] --> S[🏛️ Gov Portal]
    R --> Q
    
    style A fill:#e1f5ff
    style B fill:#fff4e6
    style C fill:#ffe6e6
    style D fill:#e8f5e9
    style E fill:#f3e5f5
    style H fill:#e3f2fd
    style I fill:#e3f2fd
    style M fill:#e3f2fd
    style S fill:#fce4ec
```

## Simplified 3-Stage View

```mermaid
graph TD
    subgraph Stage1["📥 CAPTURE"]
        A1[Operator PWA]
        A2[Voice + Photos]
        A3[HTML5 Geolocation]
    end
    
    subgraph Stage2["🔍 PROCESS"]
        B1[Amazon Transcribe<br/>Hindi/English]
        B2[Amazon Textract<br/>Printed English]
        B3[Rule Engine<br/>JSON Rules]
        B4[Amazon Bedrock<br/>Explanations]
    end
    
    subgraph Stage3["✅ VERIFY & SUBMIT"]
        C1[ProofPack PDF+JSON]
        C2[OTP Attestation]
        C3[Government Portal]
    end
    
    Stage1 --> Stage2
    Stage2 --> Stage3
    
    style Stage1 fill:#e3f2fd
    style Stage2 fill:#fff3e0
    style Stage3 fill:#e8f5e9
```

## Data Flow with Emojis

```mermaid
sequenceDiagram
    participant 👤 Operator
    participant 📱 PWA
    participant ☁️ AWS
    participant 🤖 AI Services
    participant 🏛️ Gov Portal
    
    👤 Operator->>📱 PWA: Record voice + Capture photos
    📱 PWA->>☁️ AWS: Upload evidence
    ☁️ AWS->>🤖 AI Services: Process (Transcribe + Textract)
    🤖 AI Services->>☁️ AWS: Return extracted data
    ☁️ AWS->>☁️ AWS: Evaluate rules
    ☁️ AWS->>🤖 AI Services: Generate explanation (Bedrock)
    🤖 AI Services->>☁️ AWS: Return explanation
    ☁️ AWS->>📱 PWA: Show results
    👤 Operator->>📱 PWA: Review & Attest (OTP)
    📱 PWA->>☁️ AWS: Generate ProofPack
    ☁️ AWS->>🏛️ Gov Portal: Submit
    🏛️ Gov Portal->>📱 PWA: Confirmation
```

## Component Icons Legend

- 👤 = User/Operator
- 📱 = Mobile PWA
- 🚪 = API Gateway
- ⚡ = Lambda Functions
- 📦 = S3 Storage
- 💾 = DynamoDB
- 🔄 = Step Functions
- 🎤 = Transcribe (Voice)
- 📄 = Textract (OCR)
- 🖼️ = Image Processing
- ⚖️ = Rule Engine
- 🤖 = AI/Bedrock
- 📑 = ProofPack Generator
- 🔐 = OTP/Security
- 📊 = Audit Logs
- 🏛️ = Government Portal
- ☁️ = AWS Cloud

## View Instructions

These diagrams render automatically on:
- ✅ GitHub (just view the file)
- ✅ VS Code (with Mermaid extension)
- ✅ GitLab
- ✅ Notion
- ✅ https://mermaid.live/ (paste the code)
