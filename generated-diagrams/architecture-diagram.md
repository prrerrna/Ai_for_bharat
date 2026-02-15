# ProofPack AWS Serverless Architecture Diagram

## Visual Architecture Flow

```
┌─────────────────┐
│  Operator PWA   │
│ (Offline-first) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  API Gateway    │
│  (JWT Auth)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Lambda Ingress  │
│ (Upload Handler)│
└────┬───────┬────┘
     │       │
     │       └──────────────────┐
     ▼                          ▼
┌─────────────────┐    ┌─────────────────┐
│  S3 Raw Media   │    │    DynamoDB     │
│ (90-day retain) │    │    Metadata     │
└────────┬────────┘    └────────┬────────┘
         │                      │
         ▼                      │
┌─────────────────┐             │
│    Lambda       │             │
│  Orchestrator   │             │
└────────┬────────┘             │
         │                      │
         ▼                      │
┌─────────────────┐             │
│ Step Functions  │             │
│ (Orchestration) │             │
└────┬───┬───┬────┘             │
     │   │   │                  │
     │   │   └──────────────────┼──────────────┐
     │   │                      │              │
     │   └──────────────────────┼──────┐       │
     │                          │      │       │
     ▼                          │      ▼       ▼
┌─────────────────┐             │  ┌──────┐ ┌──────┐
│    Lambda       │             │  │Lambda│ │Lambda│
│  Transcribe     │             │  │Text- │ │Image │
│   Processor     │             │  │tract │ │Prep  │
└────────┬────────┘             │  └──┬───┘ └──┬───┘
         │                      │     │        │
         ▼                      │     ▼        │
┌─────────────────┐             │  ┌──────┐   │
│    Amazon       │             │  │Amazon│   │
│   Transcribe    │             │  │Text- │   │
│ (Hindi/English) │             │  │tract │   │
└────────┬────────┘             │  └──┬───┘   │
         │                      │     │        │
         └──────────────────────┼─────┴────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │    DynamoDB     │
                       │    Metadata     │
                       └────────┬────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │     Lambda      │
                       │  Rule Engine    │
                       │ (JSON Rules)    │
                       └────┬───────┬────┘
                            │       │
                            │       └────────────┐
                            ▼                    │
                       ┌─────────────────┐       │
                       │  S3 Rule Packs  │       │
                       │ (Per-state JSON)│       │
                       └─────────────────┘       │
                                                 │
                                                 ▼
                                        ┌─────────────────┐
                                        │     Lambda      │
                                        │  Bedrock Proc   │
                                        └────────┬────────┘
                                                 │
                                                 ▼
                                        ┌─────────────────┐
                                        │  Amazon Bedrock │
                                        │ (Claude 3 Haiku)│
                                        │ (Explain only)  │
                                        └────────┬────────┘
                                                 │
                                                 ▼
                                        ┌─────────────────┐
                                        │    DynamoDB     │
                                        │    Metadata     │
                                        └────────┬────────┘
                                                 │
                                                 ▼
                                        ┌─────────────────┐
                                        │     Lambda      │
                                        │  ProofPack Gen  │
                                        └────────┬────────┘
                                                 │
                                                 ▼
                                        ┌─────────────────┐
                                        │ S3 ProofPacks   │
                                        │  (PDF + JSON)   │
                                        └────────┬────────┘
                                                 │
                                                 ▼
                                        ┌─────────────────┐
                                        │     Lambda      │
                                        │ OTP Attestation │
                                        └────────┬────────┘
                                                 │
                                                 ▼
                                        ┌─────────────────┐
                                        │    DynamoDB     │
                                        │   Audit Log     │
                                        └────────┬────────┘
                                                 │
                                                 ▼
                                        ┌─────────────────┐
                                        │     Lambda      │
                                        │ Submission      │
                                        │   Adapter       │
                                        └────────┬────────┘
                                                 │
                                                 ▼
                                        ┌─────────────────┐
                                        │  Government     │
                                        │  Portal APIs    │
                                        └─────────────────┘
```

## Component Legend

### Compute Layer
- **Operator PWA**: Progressive Web App for offline-first evidence capture
- **API Gateway**: REST API with JWT authentication and request validation
- **Lambda Functions**:
  - Ingress: Handles multipart uploads and ProofPack ID generation
  - Orchestrator: Coordinates processing pipeline via Step Functions
  - Transcribe Processor: Submits voice files to Amazon Transcribe
  - Textract Processor: Submits photos to Amazon Textract
  - Image Preprocessor: Document boundary detection and quality checks
  - Rule Engine: Evaluates per-state JSON rule packs for eligibility
  - Bedrock Processor: Generates human-readable explanations
  - ProofPack Generator: Renders PDF and JSON outputs
  - OTP Attestation: Validates operator attestation
  - Submission Adapter: Transforms data for government portal APIs

### Storage Layer
- **S3 Buckets**:
  - Raw Media: Encrypted storage with 90-day retention
  - Rule Packs: Per-state JSON eligibility rules
  - ProofPacks: Final PDF and JSON outputs
- **DynamoDB Tables**:
  - Metadata: ProofPack status and extracted fields
  - Audit Log: Immutable processing and attestation log

### AI/ML Services
- **Amazon Transcribe**: Hindi/English voice-to-text conversion
- **Amazon Textract**: Printed English field extraction (not for Hindi handwriting)
- **Amazon Bedrock**: Claude 3 Haiku for explanations only (not eligibility decisions)

### Orchestration
- **Step Functions**: Coordinates parallel processing and sequential workflow

### External Integration
- **Government Portal APIs**: Scheme-specific submission endpoints

## Data Flow Summary

1. Operator captures evidence via PWA (voice + photos with HTML5 geolocation)
2. API Gateway authenticates and routes to Ingress Lambda
3. Ingress stores raw media in S3 and creates DynamoDB record
4. S3 event triggers Orchestrator Lambda → Step Functions workflow
5. Step Functions invokes parallel processing:
   - Transcribe Lambda → Amazon Transcribe → DynamoDB
   - Textract Lambda → Amazon Textract → DynamoDB
   - Image Preprocessor → DynamoDB
6. Rule Engine loads per-state rules from S3 and evaluates eligibility
7. Bedrock Lambda generates human-readable explanation (if needed)
8. ProofPack Generator creates PDF + JSON and stores in S3
9. OTP Attestation validates operator and logs to DynamoDB
10. Submission Adapter transforms data and calls government portal API

## Key Design Principles

- **Operator-first**: PWA captures uncompressed photos with HTML5 geolocation
- **Deterministic decisions**: JSON rule packs make eligibility decisions, not LLMs
- **LLM for explanation only**: Bedrock generates human-readable text, not decisions
- **Textract limitation**: Used only for printed English fields, not Hindi handwriting
- **Audit trail**: All processing steps and operator corrections logged immutably
- **Security**: SSE-KMS encryption, JWT auth, OTP attestation, PII minimization
