# ProofPack: AI-Powered Evidence Verification for Indian Social Welfare Schemes

> Empowering CSC operators to capture, verify, and submit welfare scheme applications with AI-driven automation

---

## 🎯 Problem Statement

Millions of rural Indians struggle to access social welfare schemes due to:
- Complex documentation requirements across multiple schemes and states
- Language barriers (Hindi/regional languages vs. English forms)
- Manual verification delays taking weeks or months
- High rejection rates due to incomplete or incorrect documentation

CSC operators and NGO field workers spend 45+ minutes per case manually filling forms, often leading to errors and rejections.

---

## 💡 Our Solution

ProofPack is an **operator-first Progressive Web App** that uses AWS AI services to:

1. **Capture evidence** via voice narratives (Hindi/English) and photos with HTML5 geolocation
2. **Extract data** using Amazon Transcribe (ASR) and Amazon Textract (OCR for printed English)
3. **Verify eligibility** through deterministic per-state JSON rule packs
4. **Generate ProofPacks** - structured PDF + JSON evidence bundles with audit trails
5. **Submit directly** to government portals with OTP attestation

### Key Features

- PWA captures uncompressed photos with HTML5 geolocation metadata
- Offline-first architecture - works in zero-connectivity areas, syncs later
- Voice-first intake - beneficiaries narrate their case in Hindi/regional languages
- Deterministic rule engine - JSON rule packs make eligibility decisions, NOT LLMs
- LLMs for explanation only - Amazon Bedrock generates human-readable guidance

---

## 🏗️ Architecture

### High-Level Flow

```
Operator PWA → API Gateway → Lambda → S3 → AI Processing (Transcribe/Textract) 
→ Rule Engine → Bedrock (Explanations) → ProofPack Generator → OTP Attestation 
→ Government Portal Submission
```

### Architecture Diagrams

📊 **[View Full Architecture Diagram](./generated-diagrams/architecture-mermaid.md)** - Complete AWS serverless architecture

📊 **[View Simplified Flow Diagrams](./generated-diagrams/simple-flow.md)** - 3-stage flow and sequence diagrams

📊 **[View Text-Based Diagram](./generated-diagrams/architecture-diagram.md)** - ASCII diagram with component details

**Note on Diagrams**: We used Mermaid diagrams instead of PNG because the AWS diagram generation tool has a Windows compatibility issue (Unix-specific signal handling). Mermaid diagrams render automatically on GitHub and can be exported to PNG via https://mermaid.live/ if needed.

---

## 🎯 MVP Scope

### Two Schemes Demonstrated

1. **MGNREGA Wage/Payment Grievance** (Job Card focus)
2. **Widow Pension Application** (Age verification, spouse death certificate, income proof)

### National Extensibility

The architecture uses **modular per-state JSON rule packs** to enable easy addition of new schemes and state-specific eligibility rules.

---

## 🛠️ Technology Stack

### AWS Services

| Service | Purpose |
|---------|---------|
| **API Gateway** | REST API with JWT authentication |
| **Lambda** | Serverless compute (Node.js 20.x) |
| **S3** | Raw media, rule packs, ProofPacks storage |
| **DynamoDB** | Metadata and audit logs |
| **Step Functions** | Workflow orchestration |
| **Amazon Transcribe** | Hindi/English voice-to-text |
| **Amazon Textract** | Printed English field extraction |
| **Amazon Bedrock** | Claude 3 Haiku for explanations |
| **CloudWatch** | Logging, metrics, alarms |

### Key Design Decisions

✅ **Textract Limitation**: Used ONLY for printed English fields. Hindi handwriting handled via operator review.

✅ **LLM Guardrails**: Bedrock generates explanations ONLY. Eligibility decisions made by deterministic JSON rules.

✅ **WhatsApp Caveat**: WhatsApp compresses images and strips EXIF/geolocation. PWA is primary channel.

✅ **Security**: SSE-KMS encryption, JWT auth, OTP attestation, PII minimization.

---

## � Cost Estimate

**Prototype (100 cases/day for 30 days = 3000 cases): ~$163/month**

Main cost driver: Amazon Transcribe at $0.024/minute

---

## 📁 Project Structure

```
Ai_for_bharat/
├── README.md                          # This file
├── requirements.md                    # Detailed requirements document
├── design.md                          # Technical design document
└── generated-diagrams/                # Architecture diagrams
    ├── architecture-mermaid.md
    ├── simple-flow.md
    └── architecture-diagram.md
```

---

## 📖 Documentation

1. **[Requirements Document](./requirements.md)** - Project overview, goals, functional requirements, acceptance tests
2. **[Design Document](./design.md)** - Architecture, APIs, data models, rule engine, security design
3. **[Architecture Diagrams](./generated-diagrams/)** - Visual representations of the system

---

## ⚠️ Known Limitations

1. **Textract Hindi Limitation**: Cannot OCR Hindi handwriting (operator manual entry required)
2. **Manual Review Required**: OCR/ASR not 100% accurate, operators must review before attestation
3. **MVP Scheme Coverage**: Only MGNREGA and Widow Pension in MVP
4. **Government API Integration**: Production requires state-specific portal integrations

---

Built for AI for Bharat Hackathon
