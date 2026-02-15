# Requirements Document

## Project Overview

ProofPack is a government scheme eligibility verification platform designed for rural India. The system enables CSC (Common Service Centre) operators, VLEs (Village Level Entrepreneurs), and NGO field workers to capture beneficiary evidence through a Progressive Web App (PWA), process it through automated pipelines, and generate standardized eligibility verification packages. The primary intake mechanism is the operator-first ProofPack PWA which captures uncompressed photos, HTML5 geolocation data, and voice attestations. WhatsApp serves as a degraded fallback channel with explicit limitations on EXIF metadata and geolocation reliability unless documents are uploaded as files rather than compressed images.

The MVP demonstrates end-to-end verification for two representative schemes: (1) MGNREGA wage/payment grievance resolution (focused on Job Card verification) and (2) Widow Pension as an archetype for social security schemes. The system architecture is designed for national extensibility through modular per-state JSON rule packs, though the MVP does not claim nationwide coverage.

Eligibility decisions are made deterministically by per-state JSON rule packs that encode official scheme requirements. Large Language Models are used exclusively for generating human-readable explanations, remediation guidance, and summaries—never for final eligibility determinations.

## Goals

- Enable CSC operators to capture complete beneficiary evidence packages in under 10 minutes per case using the PWA interface
- Achieve 85%+ automated extraction accuracy for printed English fields (Aadhaar numbers, bank account numbers) using Amazon Textract
- Process Hindi vernacular voice inputs through Amazon Transcribe ASR with operator review and correction workflow
- Generate standardized ProofPack PDFs with embedded audit trails within 2 minutes of operator finalization
- Support deterministic eligibility verification for MGNREGA 