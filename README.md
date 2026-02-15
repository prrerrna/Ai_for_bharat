# ProofPack: AI-Powered Evidence Verification for Indian Social Welfare Schemes

> Empowering CSC operators to capture, verify, and submit welfare scheme applications with AI-driven automation

[![AWS](https://img.shields.io/badge/AWS-Serverless-orange)](https://aws.amazon.com/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/Status-MVP%20Design-green)](.)

---

## 🎯 Problem Statement

Millions of rural Indians struggle to access social welfare schemes due to:
- **Complex documentation requirements** across multiple schemes and states
- **Language barriers** (Hindi/regional languages vs. English forms)
- **Manual verification delays** taking weeks or months
- **High rejection rates** due to incomplete or incorrect documentation
- **Limited digital literacy** among beneficiaries

CSC operators and NGO field workers spend 45+ minutes per case manually filling forms, often leading to errors and rejections.

---

## 💡 Our Solution: ProofPack

ProofPack is an **operator-first Progressive Web App** that uses AWS AI services to:

1. **Capture evidence** via voice narratives (Hindi/English) and photos with HTML5 geolocation
2. **Extract data** using Amazon Transcribe (ASR) and Amazon Textract (OCR for printed English)
3. **Verify eligibility** through deterministic per-state JSON rule packs
4. **Generate ProofPacks** - structured PDF + JSON evidence bundles with audit trails
5. **Submit directly** to government portals with OTP attestation

### Key Innovation: Operator-First Design

- **PWA captures uncompressed photos** with HTML5 geolocation metadata
- **Offline-first architecture** - works in zero-connectivity areas, syncs later
- **Voice-first intake** - beneficiaries narrate their case in Hindi/regional languages
- **Deterministic rule engine** - JSON rule packs make eligibility decisions, NOT LLMs
- **LLMs for explanation only** - Amazon Bedrock generates human-readable guidance

---

## 🏗️ Architecture

### High-Level Flow

```
Operator PWA → API Gateway → Lambda → S3 → AI Processing (Transcribe/Textract) 
→ Rule Engine → Bedrock (Explanations) → ProofPack Generator → OTP Attestation 
→ Government Portal Submission
```

### Architecture Diagrams

We've created **Mermaid diagrams** that render automatically on GitHub:

📊 **[View Full Architecture Diagram](./generated-diagrams/architecture-mermaid.md)** - Complete AWS serverless architecture with color-coded components

📊 **[View Simplified Flow Diagrams](./generated-diagrams/simple-flow.md)** - Multiple views including 3-stage flow and sequence diagrams

📊 **[View Text-Based Diagram](./generated-diagrams/architecture-diagram.md)** - ASCII art diagram with detailed component descriptions

#### Why Mermaid Diagrams?

We initially attempted to generate PNG diagrams using the AWS Diagram MCP Server tool, but encountered a **Windows compatibility issue**:

- The Python `diagrams` library uses Unix-specific signal handling (`SIGALRM`)
- This signal is not available on Windows systems
- Error: `AttributeError: module 'signal' has no attribute 'SIGALRM'`

**Solution**: We created Mermaid diagrams instead, which:
- ✅ Render automatically on GitHub, GitLab, VS Code
- ✅ Are version-controllable (text-based)
- ✅ Can be edited easily without specialized tools
- ✅ Export to PNG/SVG via https://mermaid.live/
- ✅ Work cross-platform (Windows, Mac, Linux)

---

## 🎯 MVP Scope

### Two Schemes Demonstrated

1. **MGNREGA Wage/Payment Grievance** (Job Card focus)
   - Wage payment delays
   - Job Card verification
   - Bank account validation

2. **Widow Pension Application** (Archetype)
   - Age verification (18-60 years)
   - Spouse death certificate
   - Income proof below poverty line

### National Extensibility

The architecture uses **modular per-state JSON rule packs** to enable:
- Easy addition of new schemes (Ration Card, PM-KISAN, Old Age Pension, etc.)
- State-specific eligibility rules
- Regional language support (Tamil, Telugu, Bengali, Marathi)

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
| **KMS** | Encryption key management |

### Key Design Decisions

✅ **Textract Limitation Acknowledged**: Used ONLY for printed English fields (Aadhaar, account numbers). Hindi handwriting handled via operator review.

✅ **LLM Guardrails**: Bedrock generates explanations and remediation guidance ONLY. Eligibility decisions made by deterministic JSON rules.

✅ **WhatsApp Caveat**: WhatsApp compresses images and strips EXIF/geolocation unless uploaded as documents. PWA is primary channel.

✅ **Security-First**: SSE-KMS encryption, JWT auth, OTP attestation, PII minimization, 90-day raw media retention.

---

## 📊 Cost Estimate

**Prototype (100 cases/day for 30 days = 3000 cases)**

| Service | Cost |
|---------|------|
| Lambda | $0.04 |
| S3 | $0.35 |
| DynamoDB | $0.004 |
| Transcribe | $144 |
| Textract | $18 |
| Bedrock | $0.38 |
| API Gateway | $0.11 |
| **Total** | **~$163/month** |

*Transcribe is the primary cost driver at $0.024/minute*

---

## 📁 Project Structure

```
Ai_for_bharat/
├── README.md                          # This file
├── requirements.md                    # Detailed requirements document
├── design.md                          # Technical design document
├── generated-diagrams/
│   ├── architecture-mermaid.md       # Full Mermaid architecture
│   ├── simple-flow.md                # Simplified flow diagrams
│   └── architecture-diagram.md       # ASCII text diagram
├── .kiro/
│   ├── specs/proofpack-system/       # Spec files
│   └── settings/mcp.json             # MCP configuration
└── .gitignore
```

---

## 🚀 Getting Started

### Prerequisites

- AWS Account with appropriate permissions
- Node.js 20.x or later
- AWS CLI configured

### Documentation

1. **[Requirements Document](./requirements.md)** - Project overview, goals, functional requirements, acceptance tests
2. **[Design Document](./design.md)** - Architecture, APIs, data models, rule engine, security design
3. **[Architecture Diagrams](./generated-diagrams/)** - Visual representations of the system

### Key Features

- ✅ Operator-first PWA with offline support
- ✅ HTML5 geolocation capture for evidence provenance
- ✅ Hindi/English voice narrative processing
- ✅ Printed English OCR (Textract limitation acknowledged)
- ✅ Deterministic per-state rule packs
- ✅ LLM-generated explanations (not decisions)
- ✅ OTP attestation for operator accountability
- ✅ PDF + JSON ProofPack generation
- ✅ Government portal submission adapter
- ✅ Immutable audit trail

---

## 🎬 Demo Plan (2-Minute Showstopper)

**Scenario 1: Eligible MGNREGA Case** (1 minute)
- Operator records Hindi voice: "मेरा नाम रमेश है, मुझे 30 दिन से वेतन नहीं मिला"
- Captures Job Card and passbook photos
- System extracts fields, evaluates rules
- Shows "Eligible" with explanation
- Generates ProofPack PDF with QR code
- Submits to portal

**Scenario 2: Incomplete Widow Pension** (30 seconds)
- Missing spouse death certificate
- System shows "Incomplete" with remediation guidance
- Lists exact missing documents

**Architecture Overview** (30 seconds)
- Show AWS services diagram
- Emphasize operator-first PWA, deterministic rules, LLM explanations only

---

## 🔒 Security & Privacy

- **Encryption**: SSE-KMS at rest, HTTPS in transit
- **Authentication**: JWT tokens (RS256)
- **Attestation**: OTP-based operator verification
- **PII Minimization**: Aadhaar/bank account redacted in PDF (last 4 digits only)
- **Retention**: Raw media auto-deleted after 90 days
- **Audit Trail**: Immutable DynamoDB logs
- **RBAC**: Operators, supervisors, government users with least privilege

---

## 📈 Success Metrics

- **90%+** ProofPack generation success rate
- **<10 minutes** median operator time per case
- **85%+** automated field extraction accuracy
- **80%+** Hindi transcription accuracy
- **95%+** OTP attestation compliance
- **99.5%+** system uptime during business hours

---

## ⚠️ Known Limitations

1. **WhatsApp EXIF Caveat**: WhatsApp strips geolocation metadata unless files uploaded as documents
2. **Textract Hindi Limitation**: Cannot OCR Hindi handwriting (operator manual entry required)
3. **Manual Review Required**: OCR/ASR not 100% accurate, operators must review before attestation
4. **MVP Scheme Coverage**: Only MGNREGA and Widow Pension in MVP
5. **Government API Stubs**: Production requires state-specific portal integrations

---

## 🛣️ Roadmap

### Phase 1: MVP (Current)
- ✅ Two schemes (MGNREGA, Widow Pension)
- ✅ Hindi/English support
- ✅ Operator PWA with offline support
- ✅ AWS serverless architecture

### Phase 2: Expansion
- 🔲 10+ schemes (Ration Card, PM-KISAN, Old Age Pension, Disability Pension)
- 🔲 Regional languages (Tamil, Telugu, Bengali, Marathi)
- 🔲 Government portal API integrations
- 🔲 Beneficiary-facing mobile app

### Phase 3: Advanced Features
- 🔲 Advanced image quality checks (glare detection, super-resolution)
- 🔲 Fraud detection (duplicate Aadhaar, suspicious patterns)
- 🔲 Analytics dashboard for scheme administrators
- 🔲 On-device ASR/OCR for zero-connectivity areas

---

## 🤝 Contributing

This is a hackathon/competition project. Contributions, suggestions, and feedback are welcome!

---

## 📄 License

MIT License - See [LICENSE](LICENSE) file for details

---

## 👥 Team

Built with ❤️ for AI for Bharat Hackathon

---

## 📞 Contact

For questions or collaboration opportunities, please open an issue or reach out via GitHub.

---

## 🙏 Acknowledgments

- **AWS** for serverless infrastructure and AI services
- **AI for Bharat** for organizing the hackathon
- **CSC operators and NGO field workers** who inspired this solution
- **Rural beneficiaries** whose challenges we aim to solve

---

**Note**: This is a design document and MVP architecture. Implementation is in progress. All TODO items in requirements.md and design.md reference official government sources for verification.
