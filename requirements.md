## Project Overview

ProofPack is an operator-first evidence capture and eligibility verification system for Indian social welfare schemes. The system enables CSC operators, VLEs, and NGO field workers to collect voice narratives, supporting documents, and geolocation data via a Progressive Web App, then automatically processes this evidence through AWS services to generate structured ProofPacks for scheme administrators.

Primary intake is the ProofPack PWA, which captures uncompressed photos with HTML5 geolocation and voice attestations. WhatsApp is a degraded fallback channel only and is unreliable for EXIF metadata and geolocation unless files are uploaded as documents rather than compressed images.

The MVP demonstrates end-to-end processing for two schemes: MGNREGA wage/payment grievances (Job Card focus) and Widow Pension applications. The architecture supports national extensibility through modular per-state rule packs.

## Goals

- Reduce evidence collection time from 45+ minutes to under 10 minutes per case
- Achieve 85%+ automated field extraction accuracy for printed English documents
- Generate submission-ready ProofPacks with zero manual data entry for 70%+ of cases
- Support Hindi voice narratives with 80%+ transcription accuracy via Amazon Transcribe
- Enable offline-first PWA capture with automatic sync when connectivity returns
- Provide deterministic eligibility decisions via per-state JSON rule packs
- Deliver human-readable explanations and remediation guidance for all rejection cases
- Maintain end-to-end audit trail with cryptographic attestation for all evidence

## Target Personas

Primary: CSC operators, VLEs, and NGO field workers who assist beneficiaries with scheme applications and grievances. These operators have basic smartphone literacy, intermittent connectivity, and handle 5-15 cases per day across multiple schemes.

Secondary: Scheme beneficiaries (rural residents, widows, MGNREGA workers) who provide voice narratives in Hindi or regional languages and present physical documents for capture.

## MVP Scope

The MVP covers two schemes:

1. MGNREGA wage/payment grievance processing (Job Card verification focus)
2. Widow Pension application archetype (age verification, spouse death certificate, income proof)

The system architecture uses modular per-state JSON rule packs to enable national extensibility. The MVP does not claim nationwide coverage but demonstrates the technical pattern for scaling to additional schemes and states.

## Inputs & Outputs

Inputs:
- Voice narrative (Hindi/English, 1-3 minutes, WebM/Opus or M4A format)
- Document photos (JPEG/PNG, uncompressed, captured via PWA with HTML5 geolocation)
- Operator metadata (CSC ID, timestamp, device geolocation)
- Scheme selection and state code
- OTP for attestation (6-digit numeric)

Outputs:
- ProofPack PDF (cover page + evidence pages + eligibility summary)
- ProofPack JSON (structured metadata, transcripts, extracted fields, eligibility decision, audit log)
- Submission payload (scheme-specific JSON for government portal APIs)
- Remediation guidance (human-readable explanation of missing/invalid evidence)

## Functional Requirements

1. Ingest & Storage: Accept multipart uploads via POST /ingest with voice file, images, and metadata. Store raw media in S3 with SSE-KMS encryption. Generate unique ProofPack ID and return to operator.

2. ASR Processing: Invoke Amazon Transcribe for Hindi and English voice narratives. Store transcript with word-level timestamps and confidence scores. Flag low-confidence segments for operator review.

3. PWA Capture: Progressive Web App captures uncompressed photos using device camera API. Capture HTML5 geolocation (latitude, longitude, accuracy) at time of photo capture. Support offline queue with background sync.

4. OCR Processing: Use Amazon Textract for printed English fields only (Aadhaar numbers, account numbers, printed names, dates). Do not use Textract for Hindi handwriting. Extract key-value pairs and tables from Job Cards, pension forms, and identity documents.

5. Image Preprocessing: Detect document boundaries, apply perspective correction, enhance contrast for low-light captures. Flag blurry or unreadable images for operator re-capture.

6. Rule Engine: Load per-state JSON rule packs for selected scheme. Evaluate deterministic eligibility rules against extracted fields and transcript. Rules check document presence, field completeness, age thresholds, income limits, and cross-field consistency. Output eligibility decision (eligible/ineligible/incomplete) with specific rule violations.

7. LLM Explanation: Use Amazon Bedrock (Claude 3 Haiku) to generate human-readable explanations of eligibility decisions and remediation templates. LLM does not make eligibility decisions, only explains rule engine output.

8. OTP Attestation: Generate 6-digit OTP sent to operator mobile. Operator enters OTP to cryptographically attest that evidence was collected in their presence. Store attestation timestamp and operator ID in audit log.

9. ProofPack Generator: Render PDF with cover page (claimant details, scheme, eligibility summary), evidence pages (photos with captions, transcript excerpts), and footer with QR code linking to JSON. Generate companion JSON with full structured data.

10. Submission Adapter: Transform ProofPack JSON into scheme-specific submission payloads for government portal APIs. Support manual download of PDF+JSON for offline submission.

11. Manual Correction UI: Operator reviews extracted fields, transcript, and eligibility decision. Operator can correct OCR errors, edit transcript, and override missing field flags. All corrections logged in audit trail.

## Non-Functional Requirements

- Latency: End-to-end processing (upload to ProofPack generation) completes in under 90 seconds for typical case (2-minute voice, 4 photos)
- Security: All media encrypted at rest with SSE-KMS. API endpoints require JWT authentication. OTP attestation required before ProofPack finalization. No PII in CloudWatch logs.
- Retention: Raw media retained for 90 days, then auto-deleted. ProofPack JSON retained for 7 years per government audit requirements.
- Scalability: System handles 1000 concurrent operators during peak hours. S3 and Lambda auto-scale. DynamoDB provisioned for 500 writes/sec.
- Accessibility: PWA meets WCAG 2.1 AA for operator UI. Voice prompts and large touch targets for low-literacy operators.

## Acceptance Tests

1. Given a Job Card photo with printed Aadhaar number, When Textract processes the image, Then the 12-digit Aadhaar is extracted with 95%+ confidence.

2. Given a case missing spouse death certificate, When rule engine evaluates Widow Pension rules, Then eligibility is "incomplete" and missing_fields includes "spouse_death_certificate".

3. Given an operator enters valid OTP, When POST /attest/{id} is called, Then attestation timestamp is recorded and ProofPack status changes to "attested".

4. Given PWA captures photo with HTML5 geolocation, When photo is uploaded, Then ProofPack JSON includes latitude, longitude, and accuracy in evidence metadata.

5. Given a Job Card image with printed English fields, When Textract extracts key-value pairs, Then job_card_number and registration_date are populated in ProofPack JSON.

6. Given operator corrects misrecognized transcript word, When correction is saved, Then audit_log includes operator_id, timestamp, field_name, old_value, and new_value.

## Constraints & External Dependencies

AWS Services:
- API Gateway (REST API ingress)
- Lambda (processing pipeline, rule engine, ProofPack generator)
- S3 (raw media storage, ProofPack storage)
- DynamoDB (ProofPack metadata, audit logs)
- Amazon Transcribe (Hindi/English ASR)
- Amazon Textract (printed English OCR only, not for Hindi handwriting)
- Amazon Bedrock (Claude 3 Haiku for explanations only, not eligibility decisions)
- CloudWatch (logs, metrics, dashboards)
- KMS (encryption keys)

WhatsApp Caveat: WhatsApp compresses images and strips EXIF metadata including geolocation unless files are sent as documents. WhatsApp is a degraded fallback channel and operators must be trained to upload as documents, not images.

External: Government scheme portal APIs for submission (integration TBD per state).

## Success Metrics

- ProofPack generation success rate: 90%+ of ingested cases produce valid ProofPacks
- Operator time per case: Median under 10 minutes from scheme selection to ProofPack download
- Automated extraction accuracy: 85%+ of printed English fields extracted without operator correction
- Transcription accuracy: 80%+ word-level accuracy for Hindi voice narratives
- Attestation compliance: 95%+ of ProofPacks attested with OTP before submission
- System uptime: 99.5%+ availability during business hours (9 AM - 6 PM IST)

## Ethical & Legal Considerations

- Informed Consent: Operators must obtain verbal consent from beneficiaries before recording voice narratives. Consent statement included in PWA workflow.
- LLM Limitations: LLMs generate explanations only and do not make eligibility decisions. All decisions are deterministic rule-based. LLM outputs are reviewed by operators before delivery to beneficiaries.
- PII Minimization: ProofPacks include only minimal claimant identifiers (name, Aadhaar last 4 digits, mobile). Full Aadhaar and bank account numbers are redacted in PDF, stored only in encrypted JSON.
- Data Retention: Raw media auto-deleted after 90 days. Beneficiaries can request deletion of ProofPack data via operator portal.

## Appendix

### Sample ProofPack Cover Fields

- ProofPack ID
- Scheme Name and State Code
- Claimant Name and Aadhaar (last 4 digits)
- Operator ID and CSC Location
- Capture Date and Geolocation
- Eligibility Decision (Eligible / Ineligible / Incomplete)
- Missing Fields (if incomplete)
- Attestation Timestamp and OTP Verification Status
- QR Code (links to JSON)

### Sample ProofPack Metadata JSON

```json
{
  "proofpack_id": "PP-MH-MGNREGA-20260215-A7B3",
  "scheme_id": "MGNREGA_WAGE_GRIEVANCE",
  "state_code": "MH",
  "created_at": "2026-02-15T10:23:45Z",
  "claimant": {
    "name": "Ramesh Kumar",
    "aadhaar_last4": "8234",
    "mobile": "+919876543210"
  },
  "evidence": [
    {
      "type": "photo",
      "filename": "job_card_front.jpg",
      "captured_at": "2026-02-15T10:20:12Z",
      "geolocation": {
        "latitude": 19.0760,
        "longitude": 72.8777,
        "accuracy_meters": 12
      }
    }
  ],
  "eligibility": {
    "decision": "eligible",
    "confidence": 0.95
  },
  "attestation": {
    "operator_id": "CSC-MH-1234",
    "attested_at": "2026-02-15T10:25:30Z",
    "otp_verified": true
  }
}
```
