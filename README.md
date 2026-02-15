# ProofPack: AI-Powered Evidence Verification for Indian Social Welfare Schemes

Operator-first Progressive Web App for CSC operators to capture, process, and verify welfare scheme applications using AWS AI services.

## Problem Statement

Rural beneficiaries face barriers accessing social welfare schemes due to complex documentation, language gaps, and manual verification delays. CSC operators spend 45+ minutes per case manually filling forms, leading to errors and rejections.

## Solution

ProofPack is an operator-first PWA that:

1. Captures voice narratives (Hindi/English) and photos with HTML5 geolocation
2. Extracts data using Amazon Transcribe and Amazon Textract (printed English only)
3. Evaluates eligibility via deterministic per-state JSON rule packs
4. Generates structured PDF + JSON ProofPacks with audit trails
5. Provides submission adapter for government portals (stub in MVP)

## Architecture

```
Operator PWA → API Gateway → Lambda → S3 → AI Processing (Transcribe/Textract) 
→ Rule Engine → Bedrock (Explanations) → ProofPack Generator → OTP Attestation 
→ Submission Adapter
```

Architecture diagrams available in [generated-diagrams/](./generated-diagrams/).

Note: Mermaid diagrams used due to Windows compatibility limitations with AWS diagram generation tool.

## MVP Scope

Two schemes demonstrated:
- MGNREGA Wage/Payment Grievance (Job Card verification)
- Widow Pension Application (age, death certificate, income proof)

Architecture supports extensibility via modular per-state JSON rule packs. MVP does not claim nationwide coverage.

## Technology Stack

| Service | Purpose |
|---------|---------|
| API Gateway | REST API with JWT authentication |
| Lambda | Serverless compute (Node.js 20.x) |
| S3 | Raw media, rule packs, ProofPacks storage |
| DynamoDB | Metadata and audit logs |
| Step Functions | Workflow orchestration |
| Amazon Transcribe | Hindi/English voice-to-text |
| Amazon Textract | Printed English field extraction |
| Amazon Bedrock | Claude 3 Haiku for explanations only |
| CloudWatch | Logging, metrics, alarms |

## Key Design Decisions

- Textract used only for printed English fields. Hindi handwriting requires operator review.
- LLM (Bedrock) generates explanations only. Eligibility decisions made by deterministic JSON rules.
- WhatsApp is degraded fallback (strips EXIF/geolocation). PWA is primary channel.
- Security: SSE-KMS encryption, JWT auth, OTP attestation, no PII in logs.

## Cost Estimate

Estimated $150-$200/month for 100 cases/day prototype workload. Main cost driver: Amazon Transcribe.

## Documentation

- [requirements.md](./requirements.md) - Detailed requirements and acceptance tests
- [design.md](./design.md) - Architecture, APIs, data models, security design
- [generated-diagrams/](./generated-diagrams/) - Visual architecture representations

## Known Limitations

- Textract cannot OCR Hindi handwriting (operator manual entry required)
- OCR/ASR accuracy requires operator review before attestation
- MVP covers only MGNREGA and Widow Pension
- Government API integration requires state-specific implementation
- Submission adapter is stub in MVP

Built for AI for Bharat Hackathon by AWS
