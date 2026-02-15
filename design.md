## High-Level Architecture

```
Operator PWA → API Gateway → Lambda (Ingress) → S3 (Raw Media)
                                ↓
                    Lambda (Orchestrator) triggers:
                                ↓
        ┌───────────────────────┼───────────────────────┐
        ↓                       ↓                       ↓
  Amazon Transcribe      Amazon Textract        Image Preprocessor
  (Hindi/English ASR)    (Printed English)      (Boundary detection)
        ↓                       ↓                       ↓
        └───────────────────────┼───────────────────────┘
                                ↓
                        DynamoDB (Metadata)
                                ↓
                        Rule Engine Lambda
                        (Per-state JSON rules)
                                ↓
                        Amazon Bedrock
                        (Explanation only)
                                ↓
                        ProofPack Generator
                                ↓
                        S3 (ProofPack PDF+JSON)
                                ↓
                    OTP Attestation Service
                                ↓
                    Submission Adapter Lambda
                                ↓
                    Government Portal APIs
                                ↓
                    DynamoDB (Audit Log)
```

Component purposes:

- Operator PWA: Offline-first web app for scheme selection, voice recording, photo capture with HTML5 geolocation, and operator review/correction UI.
- API Gateway: REST API ingress with JWT authentication and request validation.
- Lambda Ingress: Accepts multipart uploads, generates ProofPack ID, stores raw media in S3, triggers orchestrator.
- S3 Raw Media: Encrypted storage for voice files and photos with 90-day retention policy.
- Lambda Orchestrator: Coordinates processing pipeline via S3 event triggers and Step Functions.
- Amazon Transcribe: Converts Hindi/English voice to text with word-level timestamps and confidence scores.
- Amazon Textract: Extracts printed English fields from documents (Aadhaar numbers, Job Card fields). Not used for Hindi handwriting.
- Image Preprocessor Lambda: Detects document boundaries, applies perspective correction, flags blurry images.
- DynamoDB Metadata: Stores ProofPack status, extracted fields, transcript, and processing state.
- Rule Engine Lambda: Loads per-state JSON rule packs, evaluates deterministic eligibility rules, outputs decision with violations.
- Amazon Bedrock: Generates human-readable explanations and remediation templates using Claude 3 Haiku. Does not make eligibility decisions.
- ProofPack Generator Lambda: Renders PDF with cover page and evidence pages, generates companion JSON, stores in S3.
- OTP Attestation Service: Generates and validates 6-digit OTP for operator attestation.
- Submission Adapter Lambda: Transforms ProofPack JSON into scheme-specific payloads for government APIs.
- DynamoDB Audit Log: Immutable log of all processing steps, operator corrections, and attestations.

## Detailed Processing Pipeline

1. Operator submits case via PWA: POST /ingest with voice file, photos, scheme_id, state_code, operator_id, and device geolocation.

2. Lambda Ingress validates request, generates ProofPack ID (format: PP-{STATE}-{SCHEME}-{YYYYMMDD}-{RAND}), uploads media to S3 bucket with prefix s3://proofpack-raw/{proofpack_id}/, writes initial record to DynamoDB with status "ingested", returns ProofPack ID to PWA.

3. S3 PutObject event triggers Lambda Orchestrator, which starts Step Functions workflow with ProofPack ID.

4. Step Functions invokes processing Lambdas in parallel:
   - Transcribe Lambda submits voice file to Amazon Transcribe with language code "hi-IN" or "en-IN", polls for completion, stores transcript JSON in DynamoDB.
   - Textract Lambda submits each photo to Amazon Textract AnalyzeDocument API with FORMS and TABLES features, extracts key-value pairs (e.g., "Job Card Number: 1234567890"), stores in DynamoDB.
   - Image Preprocessor Lambda runs OpenCV boundary detection, calculates blur score, stores quality metrics in DynamoDB.

5. Step Functions waits for all parallel tasks to complete, then invokes Rule Engine Lambda.

6. Rule Engine Lambda loads rule pack from S3 (s3://proofpack-rules/{state_code}/{scheme_id}.json), evaluates rules against extracted fields and transcript, writes eligibility decision and violations to DynamoDB, updates status to "evaluated".

7. If eligibility is "ineligible" or "incomplete", Step Functions invokes Bedrock Lambda to generate explanation and remediation guidance, stores in DynamoDB.

8. Operator reviews extracted data in PWA via GET /proofpack/{id}. If corrections needed, operator submits PATCH /proofpack/{id}/corrections with field updates. Lambda writes corrections to DynamoDB audit log and updates extracted fields.

9. Operator finalizes case in PWA, which calls POST /attest/{id}/request. OTP Service generates 6-digit code, sends via SMS (integration TBD), returns session token.

10. Operator enters OTP in PWA, which calls POST /attest/{id}/verify with OTP and session token. OTP Service validates, Lambda writes attestation record to DynamoDB with operator_id and timestamp, updates status to "attested".

11. Step Functions invokes ProofPack Generator Lambda, which renders PDF using template engine, generates JSON, uploads both to S3 (s3://proofpack-final/{proofpack_id}/), updates status to "ready".

12. Operator downloads ProofPack via GET /proofpack/{id}/download or submits directly via POST /submit/{id}. Submission Adapter Lambda transforms JSON and calls government portal API, stores submission receipt in DynamoDB, updates status to "submitted".

## Rule Engine & Per-State Rule Packs

Rule packs are JSON files stored in S3 with schema:

```json
{
  "scheme_id": "string",
  "state_code": "string",
  "version": "string",
  "rules": [
    {
      "rule_id": "string",
      "description": "string",
      "condition": "JSONLogic expression",
      "severity": "error | warning"
    }
  ],
  "required_documents": ["string"],
  "required_fields": ["string"]
}
```

### Example: MGNREGA Wage Grievance (Maharashtra)

```json
{
  "scheme_id": "MGNREGA_WAGE_GRIEVANCE",
  "state_code": "MH",
  "version": "1.0.0",
  "rules": [
    {
      "rule_id": "MH_MGNREGA_001",
      "description": "Job Card number must be present",
      "condition": {"!!": [{"var": "job_card_number"}]},
      "severity": "error"
    },
    {
      "rule_id": "MH_MGNREGA_002",
      "description": "Wage payment delay must exceed 15 days",
      "condition": {">": [{"var": "days_since_work"}, 15]},
      "severity": "error",
      "note": "TODO: Verify payment timeline from MGNREGA Act Section 3"
    },
    {
      "rule_id": "MH_MGNREGA_003",
      "description": "Bank account number must be 10-18 digits",
      "condition": {"and": [
        {">=": [{"length": {"var": "bank_account"}}, 10]},
        {"<=": [{"length": {"var": "bank_account"}}, 18]}
      ]},
      "severity": "error"
    }
  ],
  "required_documents": [
    "job_card_photo",
    "bank_passbook_photo"
  ],
  "required_fields": [
    "job_card_number",
    "claimant_name",
    "bank_account",
    "days_since_work"
  ]
}
```

TODO: Verify official required documents from https://nrega.nic.in/Circular_Archive/archive/

### Example: Widow Pension (Uttar Pradesh)

```json
{
  "scheme_id": "WIDOW_PENSION",
  "state_code": "UP",
  "version": "1.0.0",
  "rules": [
    {
      "rule_id": "UP_WIDOW_001",
      "description": "Applicant age must be 18-60 years",
      "condition": {"and": [
        {">=": [{"var": "applicant_age"}, 18]},
        {"<=": [{"var": "applicant_age"}, 60]}
      ]},
      "severity": "error",
      "note": "TODO: Verify age range from official UP widow pension guidelines"
    },
    {
      "rule_id": "UP_WIDOW_002",
      "description": "Annual household income must be below poverty line",
      "condition": {"<": [{"var": "annual_income"}, 46080]},
      "severity": "error",
      "note": "TODO: Verify income threshold from official UP poverty line definition"
    }
  ],
  "required_documents": [
    "spouse_death_certificate",
    "age_proof",
    "income_certificate"
  ],
  "required_fields": [
    "applicant_name",
    "applicant_age",
    "spouse_name",
    "spouse_death_date",
    "annual_income"
  ]
}
```

TODO: Verify official required documents from https://sspy-up.gov.in/

## ProofPack Data Model

Full JSON schema:

```json
{
  "proofpack_id": "string (PP-{STATE}-{SCHEME}-{YYYYMMDD}-{RAND})",
  "scheme_id": "string",
  "state_code": "string (ISO 3166-2:IN)",
  "created_at": "ISO 8601 timestamp",
  "updated_at": "ISO 8601 timestamp",
  "status": "ingested | processing | evaluated | attested | ready | submitted",
  "claimant": {
    "name": "string",
    "aadhaar_last4": "string (4 digits)",
    "mobile": "string (E.164 format)",
    "age": "integer (optional)"
  },
  "operator": {
    "operator_id": "string",
    "csc_location": "string",
    "device_geolocation": {
      "latitude": "number",
      "longitude": "number",
      "accuracy_meters": "number"
    }
  },
  "evidence": [
    {
      "evidence_id": "string",
      "type": "photo | voice",
      "filename": "string",
      "s3_uri": "string",
      "captured_at": "ISO 8601 timestamp",
      "geolocation": {
        "latitude": "number",
        "longitude": "number",
        "accuracy_meters": "number"
      },
      "quality_metrics": {
        "blur_score": "number (0-1)",
        "boundary_detected": "boolean"
      }
    }
  ],
  "transcript": {
    "text": "string",
    "language": "hi-IN | en-IN",
    "confidence": "number (0-1)",
    "words": [
      {
        "word": "string",
        "start_time": "number (seconds)",
        "end_time": "number (seconds)",
        "confidence": "number (0-1)"
      }
    ]
  },
  "extracted_fields": {
    "job_card_number": "string",
    "bank_account": "string",
    "aadhaar_number": "string (encrypted)",
    "custom_fields": {}
  },
  "eligibility": {
    "decision": "eligible | ineligible | incomplete",
    "confidence": "number (0-1)",
    "rule_violations": [
      {
        "rule_id": "string",
        "description": "string",
        "severity": "error | warning"
      }
    ],
    "explanation": "string (LLM-generated)",
    "remediation": "string (LLM-generated)"
  },
  "missing_fields": ["string"],
  "submission_json": {
    "scheme_specific_payload": {}
  },
  "attestations": [
    {
      "attestation_id": "string",
      "operator_id": "string",
      "attested_at": "ISO 8601 timestamp",
      "otp_verified": "boolean",
      "signature": "string (HMAC)"
    }
  ],
  "audit_log": [
    {
      "timestamp": "ISO 8601 timestamp",
      "actor": "system | operator_id",
      "action": "string",
      "details": {}
    }
  ]
}
```

## APIs & Endpoints

### POST /ingest

Accepts multipart form data with voice file, photos, and metadata.

Request:
```
POST /ingest
Content-Type: multipart/form-data
Authorization: Bearer {jwt_token}

scheme_id=MGNREGA_WAGE_GRIEVANCE
state_code=MH
operator_id=CSC-MH-1234
device_lat=19.0760
device_lon=72.8777
device_accuracy=12
voice_file=@narrative.webm
photo_1=@job_card.jpg
photo_2=@passbook.jpg
```

Response:
```json
{
  "proofpack_id": "PP-MH-MGNREGA-20260215-A7B3",
  "status": "ingested",
  "created_at": "2026-02-15T10:23:45Z"
}
```

### GET /proofpack/{id}

Returns ProofPack metadata and extracted fields for operator review.

Request:
```
GET /proofpack/PP-MH-MGNREGA-20260215-A7B3
Authorization: Bearer {jwt_token}
```

Response:
```json
{
  "proofpack_id": "PP-MH-MGNREGA-20260215-A7B3",
  "status": "evaluated",
  "claimant": {
    "name": "Ramesh Kumar",
    "aadhaar_last4": "8234"
  },
  "extracted_fields": {
    "job_card_number": "1234567890",
    "bank_account": "12345678901234"
  },
  "transcript": {
    "text": "मेरा नाम रमेश कुमार है...",
    "confidence": 0.87
  },
  "eligibility": {
    "decision": "eligible",
    "explanation": "All required documents present and eligibility criteria met."
  }
}
```

### POST /attest/{id}

Requests OTP for attestation.

Request:
```
POST /proofpack/PP-MH-MGNREGA-20260215-A7B3/attest/request
Authorization: Bearer {jwt_token}
```

Response:
```json
{
  "session_token": "sess_abc123",
  "expires_at": "2026-02-15T10:33:45Z"
}
```

### POST /attest/{id}/verify

Verifies OTP and records attestation.

Request:
```json
POST /proofpack/PP-MH-MGNREGA-20260215-A7B3/attest/verify
Authorization: Bearer {jwt_token}

{
  "session_token": "sess_abc123",
  "otp": "123456"
}
```

Response:
```json
{
  "attested": true,
  "attested_at": "2026-02-15T10:25:30Z"
}
```

### POST /submit/{id}

Submits ProofPack to government portal.

Request:
```
POST /proofpack/PP-MH-MGNREGA-20260215-A7B3/submit
Authorization: Bearer {jwt_token}
```

Response:
```json
{
  "submission_id": "GOV-MH-2026-98765",
  "submitted_at": "2026-02-15T10:30:00Z",
  "receipt_url": "https://portal.gov.in/receipt/98765"
}
```

## UI/UX: Operator PWA Flow

1. Scheme Selection Screen: Operator selects scheme (MGNREGA Grievance / Widow Pension) and state from dropdown. Taps "Start New Case".

2. Voice Recording Screen: Large red record button. Operator taps to start recording beneficiary narrative. Timer displays elapsed time. Tap again to stop. Playback controls to review. "Re-record" or "Continue" buttons.

3. Photo Capture Screen: Camera viewfinder with document frame overlay. Operator captures Job Card, passbook, Aadhaar, etc. HTML5 geolocation captured automatically with each photo. Thumbnail gallery shows captured photos. "Add Photo" or "Continue" buttons.

4. Review & Correct Screen: Displays extracted fields in editable form (Job Card Number, Bank Account, Name, etc.). Displays transcript with editable text area. Operator corrects OCR/ASR errors. "Save Corrections" button.

5. Eligibility Summary Screen: Shows eligibility decision (Eligible / Ineligible / Incomplete) with explanation. If incomplete, lists missing documents/fields with remediation guidance. "Request OTP" button.

6. Attestation Screen: Displays OTP input field. Operator enters 6-digit code received via SMS. "Verify OTP" button. On success, shows "Attested" checkmark.

7. ProofPack Ready Screen: Shows ProofPack ID and QR code. "Download PDF", "Download JSON", "Submit to Portal" buttons. Submission status indicator.

PWA emphasizes operator-first capture with HTML5 geolocation embedded in photo metadata. Offline queue stores cases locally and syncs when connectivity returns.

## Security & Privacy Design

- Encryption at Rest: All S3 buckets use SSE-KMS with customer-managed keys. DynamoDB tables encrypted with AWS-managed keys.
- Encryption in Transit: All API endpoints require HTTPS. JWT tokens signed with RS256.
- OTP Attestation: 6-digit OTP valid for 10 minutes. HMAC signature generated using operator_id, proofpack_id, timestamp, and secret key. Signature stored in attestation record for audit.
- Retention Policy: Raw media (voice, photos) auto-deleted after 90 days via S3 lifecycle policy. ProofPack JSON retained for 7 years. Audit logs immutable and retained indefinitely.
- PII Minimization: ProofPack PDF redacts full Aadhaar (shows last 4 digits only) and full bank account (shows last 4 digits only). Full values encrypted in JSON and accessible only to authorized government systems.
- RBAC: Operators can create and attest ProofPacks. Supervisors can view all ProofPacks for their CSC. Government users can view submitted ProofPacks only. IAM roles enforce least privilege.

## Deployment & Infrastructure

AWS Services:
- API Gateway: REST API with JWT authorizer, request validation, and throttling (1000 req/sec burst).
- Lambda: Node.js 20.x runtime. Ingress (512 MB), Orchestrator (256 MB), Transcribe (256 MB), Textract (512 MB), Rule Engine (512 MB), Bedrock (256 MB), ProofPack Generator (1024 MB), Submission Adapter (256 MB). All Lambdas in VPC with NAT Gateway for external API calls.
- S3: Three buckets (proofpack-raw, proofpack-rules, proofpack-final) with versioning and lifecycle policies. CloudFront distribution for ProofPack downloads.
- DynamoDB: Two tables (ProofPackMetadata with proofpack_id partition key, AuditLog with proofpack_id partition key and timestamp sort key). On-demand billing.
- Amazon Transcribe: Hindi and English language models. Automatic language detection disabled (operator selects language).
- Amazon Textract: AnalyzeDocument API with FORMS and TABLES features. Used only for printed English fields, not Hindi handwriting.
- Amazon Bedrock: Claude 3 Haiku model for explanation generation. Prompt template stored in S3. Does not make eligibility decisions.
- CloudWatch: Logs for all Lambdas with 30-day retention. Metrics for API latency, Lambda duration, Transcribe/Textract errors. Alarms for error rate >5%.
- Step Functions: Standard workflow for orchestration. Average execution time 60-90 seconds.

Ballpark Prototype Cost (100 cases/day for 30 days):
- Lambda: 3000 executions × 7 functions × $0.20/million = $0.04
- S3: 3000 cases × 5 MB × $0.023/GB = $0.35
- DynamoDB: 3000 writes × $1.25/million = $0.004
- Transcribe: 3000 × 2 min × $0.024/min = $144
- Textract: 3000 × 4 pages × $0.0015/page = $18
- Bedrock: 3000 × 500 tokens × $0.00025/1K tokens = $0.38
- API Gateway: 3000 × 10 requests × $3.50/million = $0.11
- Total: ~$163/month for 3000 cases (100/day)

## Demo Plan: 2-Minute Showstopper

Timing:
- 0:00-0:20 (20s): Introduce ProofPack and problem statement. Show operator PWA on mobile device.
- 0:20-0:50 (30s): Scenario 1 (Eligible): Operator selects MGNREGA Grievance, records 10-second Hindi voice clip ("मेरा नाम रमेश है, मुझे 30 दिन से वेतन नहीं मिला"), captures Job Card photo and passbook photo. PWA shows "Processing..." spinner.
- 0:50-1:10 (20s): Show extracted fields on Review screen (Job Card Number: 1234567890, Bank Account: 12345678901234, Name: Ramesh Kumar). Show transcript. Operator taps "Request OTP", enters code, taps "Verify".
- 1:10-1:30 (20s): Show Eligibility Summary: "Eligible - All criteria met." Show ProofPack PDF with cover page, Job Card photo, transcript excerpt, and QR code. Operator taps "Submit to Portal". Show success message.
- 1:30-1:50 (20s): Scenario 2 (Incomplete): Operator selects Widow Pension, records voice, captures only age proof photo (missing death certificate). Show Eligibility Summary: "Incomplete - Missing spouse death certificate." Show remediation guidance: "Please provide death certificate issued by municipal authority."
- 1:50-2:00 (10s): Show architecture diagram and AWS services. Emphasize operator-first PWA, HTML5 geolocation, deterministic rules, and LLM explanations only.

Sample Inputs:
- Voice Transcript 1 (Hindi, Eligible): "मेरा नाम रमेश कुमार है। मैं महाराष्ट्र के ठाणे जिले से हूं। मुझे MGNREGA के तहत 15 दिन काम किया लेकिन 30 दिन से वेतन नहीं मिला। मेरा जॉब कार्ड नंबर 1234567890 है।"
- Voice Transcript 2 (Hindi, Incomplete): "मेरा नाम सीता देवी है। मेरे पति की मृत्यु 2 साल पहले हुई थी। मुझे विधवा पेंशन चाहिए।"
- Image 1: Job Card front page with printed Job Card Number, Name, and Photo.
- Image 2: Bank passbook page with printed Account Number and IFSC Code.
- Image 3: Aadhaar card with printed 12-digit number and Name.
- Image 4: Age proof (ration card) with printed Date of Birth.

Judges will see:
- Live PWA demo on mobile device with real camera and geolocation capture.
- Real-time processing with AWS service calls (Transcribe, Textract, Bedrock).
- Generated ProofPack PDF with cover page, evidence pages, and QR code.
- Eligibility decision with human-readable explanation for both scenarios.
- Architecture diagram showing operator-first design and deterministic rule engine.

## Observability & Metrics

CloudWatch Logs:
- API Gateway access logs (request ID, latency, status code).
- Lambda execution logs (proofpack_id, processing step, duration, errors). No PII in logs.
- Transcribe/Textract API call logs (job ID, status, confidence scores).
- Rule Engine decision logs (proofpack_id, rule violations, eligibility decision).

CloudWatch Dashboards:
- ProofPack Pipeline: Ingestion rate, processing duration (p50, p95, p99), success rate, error rate by step.
- Extraction Accuracy: Textract confidence distribution, Transcribe confidence distribution, manual correction rate.
- Eligibility Decisions: Eligible/Ineligible/Incomplete breakdown by scheme and state. Top rule violations.
- Operator Activity: Cases per operator, average time per case, attestation rate.

Rejection Reasons Analytics:
- DynamoDB query on eligibility.decision="ineligible" or "incomplete".
- Aggregate by eligibility.rule_violations.rule_id to identify top rejection reasons.
- Generate weekly report for scheme administrators with remediation recommendations.

Alarms:
- Error rate >5% for any Lambda function.
- API Gateway 5xx rate >1%.
- Transcribe/Textract API error rate >2%.
- ProofPack processing duration >120 seconds (p95).

## Limitations & Future Work

Limitations:
- WhatsApp EXIF Caveat: WhatsApp compresses images and strips EXIF metadata including geolocation unless files are uploaded as documents. Operators must be trained to use "Upload as Document" feature. WhatsApp is a degraded fallback channel.
- Textract Hindi Handwriting: Amazon Textract cannot OCR Hindi handwriting. Hindi vernacular inputs (handwritten forms, signatures) are handled via operator manual entry after reviewing photos.
- Manual Correction Requirement: OCR and ASR are not 100% accurate. Operators must review and correct extracted fields before attestation. System does not auto-submit without operator review.
- MVP Scheme Coverage: MVP covers only MGNREGA wage grievance and Widow Pension. Additional schemes require new rule packs and field mappings.
- Government API Integration: Submission adapter is a stub in MVP. Production requires integration with state-specific government portal APIs (authentication, payload format, error handling).

Future Work:
- Expand to 10+ schemes (Ration Card, PM-KISAN, Old Age Pension, Disability Pension, etc.) with per-state rule packs.
- Add regional language support (Tamil, Telugu, Bengali, Marathi) for voice narratives and UI.
- Integrate government portal APIs for automated submission and status tracking.
- Add beneficiary-facing mobile app for self-service case tracking and document upload.
- Implement advanced image quality checks (glare detection, shadow removal, super-resolution for low-quality captures).
- Add fraud detection rules (duplicate Aadhaar, suspicious geolocation patterns, voice deepfake detection).
- Build analytics dashboard for scheme administrators with case volume trends, rejection reasons, and operator performance metrics.
- Explore on-device ASR and OCR for offline processing in zero-connectivity areas.

## Appendix

### Sample Rule Pack JSON (MGNREGA - Karnataka)

```json
{
  "scheme_id": "MGNREGA_WAGE_GRIEVANCE",
  "state_code": "KA",
  "version": "1.0.0",
  "rules": [
    {
      "rule_id": "KA_MGNREGA_001",
      "description": "Job Card must be registered in Karnataka",
      "condition": {"==": [{"substr": [{"var": "job_card_number"}, 0, 2]}, "KA"]},
      "severity": "error"
    },
    {
      "rule_id": "KA_MGNREGA_002",
      "description": "Minimum 7 days of work required for wage grievance",
      "condition": {">=": [{"var": "days_worked"}, 7]},
      "severity": "error"
    },
    {
      "rule_id": "KA_MGNREGA_003",
      "description": "Payment delay must exceed 15 days from work completion",
      "condition": {">": [{"var": "days_since_work"}, 15]},
      "severity": "error",
      "note": "TODO: Verify payment timeline from MGNREGA Act Section 3"
    }
  ],
  "required_documents": [
    "job_card_photo",
    "muster_roll_photo",
    "bank_passbook_photo"
  ],
  "required_fields": [
    "job_card_number",
    "claimant_name",
    "bank_account",
    "ifsc_code",
    "days_worked",
    "days_since_work",
    "work_description"
  ]
}
```

TODO: Verify official required documents from https://nrega.karnataka.gov.in/

### Sample ProofPack PDF Cover Layout

```
┌─────────────────────────────────────────────┐
│  ProofPack                                  │
│  Evidence Verification Report               │
├─────────────────────────────────────────────┤
│  ProofPack ID: PP-MH-MGNREGA-20260215-A7B3  │
│  Generated: 15 Feb 2026, 10:25 IST          │
├─────────────────────────────────────────────┤
│  Scheme: MGNREGA Wage Grievance             │
│  State: Maharashtra (MH)                    │
├─────────────────────────────────────────────┤
│  Claimant Details:                          │
│    Name: Ramesh Kumar                       │
│    Aadhaar: XXXX-XXXX-8234                  │
│    Mobile: +91-98765-43210                  │
├─────────────────────────────────────────────┤
│  Operator Details:                          │
│    Operator ID: CSC-MH-1234                 │
│    CSC Location: Thane, Maharashtra         │
│    Capture Location: 19.0760°N, 72.8777°E  │
│    Capture Date: 15 Feb 2026, 10:20 IST    │
├─────────────────────────────────────────────┤
│  Eligibility Decision: ELIGIBLE             │
│  Confidence: 95%                            │
│                                             │
│  Explanation:                               │
│  All required documents are present and     │
│  meet eligibility criteria. Job Card        │
│  number verified. Payment delay exceeds     │
│  15-day threshold. Bank account validated.  │
├─────────────────────────────────────────────┤
│  Attestation:                               │
│    Attested by: CSC-MH-1234                 │
│    Attested at: 15 Feb 2026, 10:25 IST     │
│    OTP Verified: Yes                        │
├─────────────────────────────────────────────┤
│  Evidence Summary:                          │
│    - Voice narrative (2:15 duration)        │
│    - Job Card photo (captured 10:20 IST)   │
│    - Bank passbook photo (captured 10:21)  │
├─────────────────────────────────────────────┤
│  [QR Code]                                  │
│  Scan to view full JSON metadata            │
└─────────────────────────────────────────────┘
```
