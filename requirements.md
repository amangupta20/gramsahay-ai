# Requirements Document: GramSahay AI Clinical Scribe

## Introduction

GramSahay AI is a B2G voice agent system designed to transform voice notes from ASHA (Accredited Social Health Activist) workers into structured medical records for rural healthcare in India. The system employs a two-stage AI processing pipeline on Amazon Bedrock: (1) speech-to-text transcription using OpenAI Whisper or Amazon Transcribe, followed by (2) structured data extraction using Google Gemma 3 (27B) Instruct. A novel pseudonymization middleware operates on the transcribed text to ensure patient privacy while maintaining comprehensive audit trails. The system supports Hinglish (Hindi + English mix) and operates in an offline-first mobile environment.

## Glossary

- **ASHA_Worker**: Accredited Social Health Activist, a community health worker in rural India who records patient encounters
- **Doctor**: Medical professional who reviews structured patient data and has access to audit trails
- **System**: The GramSahay AI Clinical Scribe application
- **Gemma_3**: Google Gemma 3 (27B) Instruct model running on Amazon Bedrock
- **Pseudonymization_Middleware**: Privacy layer that replaces PII with reversible pseudonyms
- **Presidio**: Microsoft's PII detection library used as the primary obfuscation layer
- **Bedrock_Guardrails**: Amazon Bedrock's built-in PII detection used as a safety net that blocks ANY detected PII
- **Transcription_Service**: Speech-to-text service (OpenAI Whisper or Amazon Transcribe) running on Amazon Bedrock
- **PII_Vault**: Secure database table storing pseudonym-to-real-identity mappings
- **Audit_Trail**: Three-layer verification system (raw audio, transcript, structured data)
- **FHIR**: Fast Healthcare Interoperability Resources, a standard for healthcare data exchange
- **Hinglish**: Mixed Hindi and English language commonly spoken in India
- **Amazon_Bedrock**: AWS managed service providing secure, scalable access to foundation models

## Requirements

### Requirement 1: Voice Note Capture and Recording

**User Story:** As an ASHA worker, I want to record patient encounters with a single button press, so that I can capture clinical information quickly without interrupting patient care.

#### Acceptance Criteria

1. WHEN an ASHA worker presses the record button, THE System SHALL begin capturing audio in .wav PCM format
2. WHEN audio recording is active, THE System SHALL provide visual feedback indicating recording status
3. WHEN an ASHA worker stops recording, THE System SHALL save the audio file locally on the device
4. WHEN the device is offline, THE System SHALL queue the audio file for later processing
5. THE System SHALL support audio recordings up to 5 minutes in duration
6. WHEN recording fails, THE System SHALL display an error message and allow retry

### Requirement 2: Speech-to-Text Transcription via Amazon Bedrock

**User Story:** As a system architect, I want audio transcribed using OpenAI Whisper or Amazon Transcribe running on Amazon Bedrock, so that we maintain security and get accurate Hinglish transcription.

#### Acceptance Criteria

1. WHEN audio is uploaded, THE System SHALL send it to the Transcription_Service (Whisper or Amazon Transcribe) on Amazon_Bedrock
2. THE Transcription_Service SHALL return a verbatim transcript in text format
3. THE Transcription_Service SHALL support Hinglish (Hindi + English code-mixing)
4. THE System SHALL complete transcription within 10 seconds for 5-minute audio files
5. THE Transcription_Service SHALL handle audio quality variations including background noise and regional accents
6. WHEN transcription fails, THE System SHALL log the error and retry up to 3 times
7. THE System SHALL validate that the transcript is not empty before proceeding to PII detection

### Requirement 3: Dual-Layer Pseudonymization with Presidio and Bedrock Guardrails

**User Story:** As a compliance officer, I want patient names and locations replaced with pseudonyms using a dual-layer approach (Presidio primary, Bedrock Guardrails safety net), so that we maintain defense-in-depth privacy protection with HIPAA-equivalent standards.

#### Acceptance Criteria

1. WHEN a transcript is received from the Transcription_Service, THE System SHALL send it to the Pseudonymization_Middleware
2. THE Pseudonymization_Middleware SHALL use Presidio as the primary PII detection engine on the text transcript
3. WHEN PII is detected by Presidio, THE Pseudonymization_Middleware SHALL generate a unique UUID-based pseudonym for each entity
4. THE Pseudonymization_Middleware SHALL store the mapping in the PII_Vault table with timestamp and user context
5. WHEN the same entity appears in future transcripts, THE System SHALL use the same pseudonym for consistency
6. THE System SHALL replace all detected PII with pseudonyms before sending data to Gemma_3
7. WHEN pseudonymized text is sent to Amazon_Bedrock, THE System SHALL enable Bedrock_Guardrails PII detection as a safety net
8. WHEN Bedrock_Guardrails detects PII that Presidio missed, THE System SHALL block the request and log the detection
9. IF pseudonymization fails at either layer, THEN THE System SHALL reject the request and log the error

### Requirement 4: AI Structured Data Extraction via Amazon Bedrock with FHIR R4 Output

**User Story:** As an ASHA worker, I want pseudonymized transcripts automatically converted to FHIR R4-compliant structured medical records securely and quickly, so that I don't have to manually type patient information.

#### Acceptance Criteria

1. WHEN pseudonymized transcript text is ready, THE System SHALL send it to Gemma_3 via Amazon_Bedrock API
2. THE System SHALL use Amazon_Bedrock for its enterprise-grade security features including encryption at rest and in transit
3. THE System SHALL use Amazon_Bedrock for its auto-scaling capabilities to maintain fast response times during heavy usage sessions
4. THE System SHALL include the medical scribe system prompt configured for text input with each Bedrock API request
5. WHEN Gemma_3 processes the transcript, THE System SHALL receive FHIR R4-compliant JSON output
6. THE System SHALL validate that the JSON output conforms to FHIR R4 specification
7. THE System SHALL map vitals to FHIR Observation resources, symptoms to FHIR Condition resources, medications to FHIR MedicationStatement resources, and referrals to FHIR ServiceRequest resources
8. WHEN JSON validation fails, THE System SHALL log the error and retry up to 3 times
9. THE System SHALL support Hinglish language input (mixed Hindi and English)
10. WHEN Gemma_3 returns null for missing values, THE System SHALL preserve these nulls without hallucination
11. THE System SHALL monitor Amazon_Bedrock API quotas and implement rate limiting to prevent service disruption

### Requirement 5: PII Re-hydration for Authorized Users

**User Story:** As a doctor, I want to see real patient names in the medical records, so that I can provide personalized care while knowing the system protects privacy during processing.

#### Acceptance Criteria

1. WHEN a doctor with valid JWT token requests patient data, THE System SHALL verify the user role is "doctor"
2. WHEN authorization is confirmed, THE System SHALL query the PII_Vault for pseudonym mappings
3. THE System SHALL replace all pseudonyms in the structured data with original identifiers
4. WHEN an ASHA worker requests data, THE System SHALL return re-hydrated data only for patients assigned to them as primary ASHA worker
5. THE System SHALL verify patient ownership via the primary_asha_worker_id field before granting access
6. WHEN re-hydration fails for a pseudonym, THE System SHALL display the pseudonym with a warning indicator
7. THE System SHALL log all re-hydration requests with user ID and timestamp

### Requirement 6: Glass Box Audit Trail Storage with Compression

**User Story:** As a medical auditor, I want access to original audio, transcript, and structured data for each encounter, so that I can verify AI accuracy and investigate discrepancies.

#### Acceptance Criteria

1. WHEN audio is recorded, THE System SHALL compress the audio using FLAC or Opus codec before storage
2. THE System SHALL store the compressed audio file in AWS S3 with AES-256 encryption
3. THE System SHALL implement S3 lifecycle policies: Standard storage (0-90 days), Glacier (90 days-1 year), Deep Archive (1-7 years)
4. WHEN the Transcription_Service returns results, THE System SHALL store the verbatim transcript in the database
5. WHEN Gemma_3 returns results, THE System SHALL store the FHIR R4-compliant JSON in the database linked to the same encounter
6. THE System SHALL create a unique encounter ID linking all three data layers (audio, transcript, FHIR JSON)
7. WHEN a doctor accesses the audit trail, THE System SHALL display all three layers side-by-side
8. THE System SHALL retain audit trail data for a minimum of 7 years
9. WHEN audit data is accessed, THE System SHALL log the access with user ID and timestamp
10. THE System SHALL provide audio playback controls within the audit trail interface
11. THE System SHALL display the verbatim transcript alongside the audio player for simultaneous review
12. THE System SHALL track and log compression ratios for performance monitoring

### Requirement 7: Human-in-the-Loop Review Interface

**User Story:** As a doctor reviewing patient records, I want to access the original audio and transcript when I notice unusual data, so that I can verify AI accuracy and catch potential hallucinations or errors.

#### Acceptance Criteria

1. WHEN a doctor views a structured medical record, THE System SHALL display a "Review Audit Trail" button
2. WHEN the review button is clicked, THE System SHALL open an interface showing audio player, transcript, and structured data side-by-side
3. THE System SHALL provide audio playback controls (play, pause, seek, speed adjustment)
4. THE System SHALL highlight the current audio position in the transcript during playback
5. WHEN a doctor identifies an error, THE System SHALL provide a "Flag for Review" button
6. WHEN data is flagged, THE System SHALL record the flag with doctor ID, timestamp, and optional notes
7. THE System SHALL display a visual indicator on medical cards that have been flagged for review
8. WHEN a doctor corrects structured data, THE System SHALL preserve the original AI output in the audit trail
9. THE System SHALL allow doctors to add correction notes that appear alongside the structured data

### Requirement 8: Structured Medical Record Display

**User Story:** As a doctor, I want to view patient encounters as structured medical cards, so that I can quickly review vitals, symptoms, and medications without reading lengthy notes.

#### Acceptance Criteria

1. WHEN a doctor selects a patient encounter, THE System SHALL display a medical card with structured fields
2. THE System SHALL display vitals (blood pressure, temperature, pulse) in a dedicated section
3. THE System SHALL display symptoms as a bulleted list
4. THE System SHALL display medications given as a bulleted list
5. THE System SHALL display referral status with a clear visual indicator
6. WHEN structured data contains null values, THE System SHALL display "Not recorded" instead of empty fields
7. THE System SHALL provide a link to view the full audit trail from the medical card

### Requirement 9: Authentication and Role-Based Access Control

**User Story:** As a system administrator, I want role-based access control using JWT tokens, so that ASHA workers and doctors have appropriate access levels.

#### Acceptance Criteria

1. WHEN a user logs in, THE System SHALL issue a JWT token with role claim (asha_worker or doctor)
2. THE System SHALL validate JWT tokens on every API request
3. WHEN a token is expired, THE System SHALL reject the request and return 401 Unauthorized
4. WHEN an ASHA worker attempts to access audit trails, THE System SHALL reject the request with 403 Forbidden
5. WHEN a doctor attempts to access another doctor's patients without permission, THE System SHALL reject the request
6. THE System SHALL support token refresh without requiring re-login
7. THE System SHALL log all authentication failures with IP address and timestamp

### Requirement 10: Offline-First Mobile Experience

**User Story:** As an ASHA worker in a rural area, I want to record and view patient data without internet connectivity, so that I can work effectively in areas with poor network coverage.

#### Acceptance Criteria

1. WHEN the device is offline, THE System SHALL allow audio recording and local storage
2. THE System SHALL queue all pending uploads for automatic sync when connectivity returns
3. WHEN connectivity is restored, THE System SHALL automatically upload queued recordings in chronological order
4. THE System SHALL cache previously viewed patient records for offline access
5. WHEN offline, THE System SHALL display a clear indicator showing sync status
6. THE System SHALL allow viewing cached patient data without network connectivity
7. WHEN offline, THE System SHALL allow audio recording and viewing cached records only
8. WHEN online, THE System SHALL require connectivity for processing (transcription, PII detection, AI extraction) and syncing new data
9. THE System SHALL process the sync queue immediately when connectivity is restored
10. WHEN sync fails after 3 attempts, THE System SHALL notify the user and allow manual retry

### Requirement 11: Data Validation and Error Handling

**User Story:** As a system architect, I want comprehensive validation and error handling, so that the system gracefully handles edge cases and prevents data corruption.

#### Acceptance Criteria

1. WHEN audio file size exceeds 50MB, THE System SHALL reject the upload and notify the user
2. WHEN JSON parsing fails, THE System SHALL log the raw response and return a user-friendly error
3. WHEN database connection fails, THE System SHALL retry with exponential backoff up to 5 times
4. WHEN Bedrock API returns an error, THE System SHALL log the error details and display a generic message to the user
5. THE System SHALL validate all user inputs against expected formats before processing
6. WHEN validation fails, THE System SHALL return specific error messages indicating which fields are invalid
7. IF critical errors occur, THEN THE System SHALL preserve local data and prevent data loss

### Requirement 12: Performance and Scalability

**User Story:** As a product manager, I want the system to process voice notes efficiently with clear performance metrics, so that ASHA workers can document encounters without delays.

#### Acceptance Criteria

1. WHEN audio is submitted, THE System SHALL complete server-side processing (transcription + PII detection + extraction + storage) within 20 seconds for 95% of requests
2. THE System SHALL complete end-to-end processing (including upload on 3G networks) within 60 seconds for 95% of requests
3. THE System SHALL separate network latency from processing latency in performance metrics
4. THE System SHALL support concurrent processing of at least 100 audio files
5. WHEN system load exceeds capacity, THE System SHALL queue requests and provide estimated wait time
6. THE System SHALL use connection pooling for database connections
7. THE System SHALL implement caching for frequently accessed patient records
8. WHEN response time exceeds the performance threshold, THE System SHALL log performance metrics for analysis

### Requirement 13: FHIR Compliance

**User Story:** As a healthcare interoperability specialist, I want structured data to follow FHIR standards, so that the system can integrate with other healthcare systems.

#### Acceptance Criteria

1. THE System SHALL structure clinical data according to FHIR R4 specification
2. THE System SHALL use FHIR Observation resource for vitals
3. THE System SHALL use FHIR Condition resource for symptoms
4. THE System SHALL use FHIR MedicationStatement resource for medications given
5. THE System SHALL use FHIR ServiceRequest resource for referrals
6. WHEN exporting data, THE System SHALL generate valid FHIR JSON bundles
7. THE System SHALL validate FHIR resources against official schemas before storage

### Requirement 14: Enterprise Security and Infinite Scalability via Amazon Bedrock

**User Story:** As a system architect, I want to leverage Amazon Bedrock's managed infrastructure, so that the system can scale from hundreds to millions of users without infrastructure changes while maintaining state-of-the-art security.

#### Acceptance Criteria

1. THE System SHALL use Amazon_Bedrock as the exclusive AI inference platform for production workloads
2. THE System SHALL benefit from Amazon_Bedrock's automatic scaling without manual capacity planning
3. THE System SHALL leverage Amazon_Bedrock's built-in encryption for data at rest and in transit
4. THE System SHALL use Amazon_Bedrock's VPC endpoints to keep AI traffic within AWS private network
5. THE System SHALL implement Amazon_Bedrock's CloudWatch integration for monitoring and alerting
6. THE System SHALL use Amazon_Bedrock's IAM integration for fine-grained access control
7. WHEN system load increases, THE Amazon_Bedrock SHALL automatically provision additional capacity
8. THE System SHALL maintain sub-20-second server-side processing times even under peak load through Bedrock's auto-scaling
9. THE System SHALL use Amazon_Bedrock's multi-region availability for disaster recovery
10. THE System SHALL comply with AWS's HIPAA-eligible services requirements through Bedrock's compliance certifications
