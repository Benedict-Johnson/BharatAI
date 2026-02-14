# Design Document: MSME Payment Enforcement System

## Overview

The MSME Payment Enforcement System is a serverless, AI-powered platform built on AWS that automates payment enforcement under the MSMED Act. The system uses a multi-agent architecture orchestrated through AWS Lambda, with Claude 3.5 Sonnet (via Amazon Bedrock) providing intelligent document processing, communication generation, and legal drafting capabilities.

The architecture follows a serverless, event-driven design pattern where each component operates as an independent Lambda function, communicating through Amazon EventBridge and storing state in Amazon Aurora. This ensures scalability, cost-efficiency, and resilience while maintaining compliance with Indian data privacy standards.

### Key Design Principles

1. **Serverless-First**: All compute operations run on AWS Lambda for automatic scaling and cost optimization
2. **Event-Driven**: Components communicate asynchronously through EventBridge for loose coupling
3. **AI-Augmented**: Claude 3.5 Sonnet handles natural language understanding, generation, and legal drafting
4. **Security-First**: End-to-end encryption, secure Udyam/Aadhaar validation, and compliance with Indian data privacy laws
5. **Vernacular-Native**: Multi-language support (Tamil/Hindi/English) at every layer
6. **Offline-Resilient**: Mobile app with local caching and sync capabilities

## Architecture

### High-Level Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        Mobile[Mobile App - React Native]
        Voice[Voice Interface]
    end
    
    subgraph "API Gateway Layer"
        APIGW[Amazon API Gateway]
        Auth[Amazon Cognito]
    end
    
    subgraph "Orchestration Layer"
        EventBridge[Amazon EventBridge]
    end
    
    subgraph "Compute Layer - Lambda Functions"
        InvoiceProc[Invoice Processor]
        PaymentMon[Payment Monitor]
        CommAgent[Communication Agent]
        LegalGen[Legal Document Generator]
        RiskAnalyzer[Risk Analyzer]
        SamaInteg[Samadhaan Integrator]
    end
    
    subgraph "AI Layer"
        Bedrock[Amazon Bedrock - Claude 3.5]
        Textract[Amazon Textract]
        Polly[Amazon Polly]
        Transcribe[Amazon Transcribe]
    end
    
    subgraph "Data Layer"
        Aurora[(Amazon Aurora PostgreSQL)]
        S3[Amazon S3 - Document Storage]
    end
    
    subgraph "External Integrations"
        WhatsApp[WhatsApp Business API]
        Email[Amazon SES]
        Samadhaan[MSME Samadhaan Portal]
        Udyam[Udyam Portal API]
    end
    
    Mobile --> APIGW
    Voice --> APIGW
    APIGW --> Auth
    APIGW --> InvoiceProc
    APIGW --> PaymentMon
    APIGW --> RiskAnalyzer
    
    InvoiceProc --> Textract
    InvoiceProc --> Bedrock
    InvoiceProc --> S3
    InvoiceProc --> Aurora
    InvoiceProc --> EventBridge
    
    EventBridge --> PaymentMon
    EventBridge --> CommAgent
    EventBridge --> LegalGen
    
    PaymentMon --> Aurora
    PaymentMon --> EventBridge
    
    CommAgent --> Bedrock
    CommAgent --> WhatsApp
    CommAgent --> Email
    CommAgent --> Aurora
    
    LegalGen --> Bedrock
    LegalGen --> S3
    LegalGen --> Aurora
    
    RiskAnalyzer --> Aurora
    RiskAnalyzer --> Bedrock
    
    SamaInteg --> Udyam
    SamaInteg --> Samadhaan
    SamaInteg --> Aurora
    
    Voice --> Transcribe
    Voice --> Polly
    Voice --> Bedrock
```


### Component Interaction Flow

1. **Invoice Ingestion Flow**: Mobile App → API Gateway → Invoice Processor Lambda → Textract → Bedrock (validation) → S3 (storage) → Aurora (metadata) → EventBridge (invoice.created event)
2. **Payment Monitoring Flow**: EventBridge (scheduled) → Payment Monitor Lambda → Aurora (query overdue invoices) → EventBridge (reminder.due or notice.due events)
3. **Communication Flow**: EventBridge (reminder.due) → Communication Agent Lambda → Bedrock (generate message) → WhatsApp/SES → Aurora (log delivery)
4. **Legal Escalation Flow**: EventBridge (notice.due) → Legal Document Generator Lambda → Bedrock (draft notice) → S3 (store PDF) → Aurora (update status) → API Gateway (notify retailer)
5. **Samadhaan Filing Flow**: Mobile App → API Gateway → Samadhaan Integrator Lambda → Udyam API (validate) → Bedrock (format submission) → Samadhaan Portal API → Aurora (store confirmation)

## Components and Interfaces

### 1. Invoice Processor Lambda

**Responsibility**: Processes uploaded invoices, extracts mandatory fields, and stores invoice records.

**Inputs**:
- Invoice image/PDF from S3 (triggered by S3 event)
- Retailer ID and metadata from API Gateway

**Processing**:
1. Call Amazon Textract to extract text and form data
2. Use Claude 3.5 via Bedrock to identify and validate mandatory fields:
   - Buyer GSTIN (format: 2 digits + 10 alphanumeric + 1 letter + 1 digit + 1 letter + 1 alphanumeric)
   - Udyam Number (format: UDYAM-XX-00-0000000)
   - Invoice Date (various formats, normalized to ISO 8601)
   - Principal Amount (numeric with currency symbol)
3. Calculate confidence scores for each extracted field
4. Store original document in S3 with encryption
5. Store invoice metadata in Aurora
6. Publish "invoice.created" event to EventBridge

**Outputs**:
- Invoice record in Aurora with extracted fields
- Document reference in S3
- EventBridge event for downstream processing

**Error Handling**:
- If confidence < 80% for any field, flag for manual review
- If Textract fails, retry up to 3 times with exponential backoff
- If document is corrupted, return error to user with localized message

**Interface**:
```typescript
interface InvoiceProcessorInput {
  retailerId: string;
  documentS3Key: string;
  documentType: 'image' | 'pdf';
  preferredLanguage: 'ta' | 'hi' | 'en';
}

interface InvoiceProcessorOutput {
  invoiceId: string;
  extractedFields: {
    buyerGSTIN: string;
    udyamNumber: string;
    invoiceDate: string; // ISO 8601
    principalAmount: number;
    confidence: {
      buyerGSTIN: number;
      udyamNumber: number;
      invoiceDate: number;
      principalAmount: number;
    };
  };
  requiresManualReview: boolean;
  reviewFields: string[];
}
```

### 2. Payment Monitor Lambda

**Responsibility**: Continuously monitors payment deadlines and triggers enforcement actions.

**Trigger**: EventBridge scheduled rule (runs every 6 hours)

**Processing**:
1. Query Aurora for all active (unpaid) invoices
2. For each invoice, calculate days since invoice date
3. Determine appropriate action based on days elapsed:
   - Day 30: Publish "reminder.first" event
   - Day 40: Publish "reminder.second" event
   - Day 44: Publish "reminder.final" event
   - Day 46: Publish "notice.generate" event
4. Calculate compound interest penalty for overdue invoices
5. Update invoice status in Aurora

**Outputs**:
- EventBridge events triggering reminders or legal notices
- Updated invoice records with calculated penalties

**Interface**:
```typescript
interface PaymentMonitorEvent {
  eventType: 'reminder.first' | 'reminder.second' | 'reminder.final' | 'notice.generate';
  invoiceId: string;
  retailerId: string;
  buyerGSTIN: string;
  invoiceDate: string;
  principalAmount: number;
  daysElapsed: number;
  compoundInterestPenalty?: number; // Only for overdue invoices
  preferredLanguage: 'ta' | 'hi' | 'en';
}
```

### 3. Communication Agent Lambda

**Responsibility**: Generates and sends diplomatic reminders to buyers via WhatsApp and Email.

**Trigger**: EventBridge events (reminder.first, reminder.second, reminder.final)

**Processing**:
1. Retrieve invoice details from Aurora
2. Use Claude 3.5 via Bedrock to generate contextual, diplomatic message:
   - Prompt includes: reminder stage, invoice details, cultural context, language preference
   - Tone: professional, respectful, escalating urgency
3. Translate message to target language if needed
4. Send via WhatsApp Business API (primary channel)
5. Send via Amazon SES (backup channel)
6. Log delivery status in Aurora
7. Implement retry logic with exponential backoff (max 3 attempts)

**Outputs**:
- Sent messages via WhatsApp and Email
- Delivery logs in Aurora

**Interface**:
```typescript
interface CommunicationAgentInput {
  reminderType: 'first' | 'second' | 'final';
  invoiceId: string;
  buyerContact: {
    phone: string; // For WhatsApp
    email: string;
  };
  invoiceDetails: {
    invoiceNumber: string;
    invoiceDate: string;
    principalAmount: number;
    dueDate: string;
  };
  language: 'ta' | 'hi' | 'en';
}

interface BedrockPromptTemplate {
  systemPrompt: string;
  userPrompt: string;
  temperature: number;
  maxTokens: number;
}
```


### 4. Legal Document Generator Lambda

**Responsibility**: Drafts formal Section 15 notices under the MSMED Act using AI-powered legal language generation.

**Trigger**: EventBridge event (notice.generate)

**Processing**:
1. Retrieve invoice and retailer details from Aurora
2. Calculate final compound interest penalty (3x bank rate, compounded monthly)
3. Use Claude 3.5 via Bedrock to draft Section 15 notice:
   - Prompt includes: legal template, invoice details, penalty calculation, MSMED Act references
   - Ensure formal legal tone and compliance with statutory requirements
4. Generate notice in retailer's preferred language
5. Convert to PDF format
6. Store PDF in S3 with encryption
7. Update invoice status to "legal_notice_generated"
8. Send notification to retailer for approval

**Outputs**:
- PDF document stored in S3
- Invoice status updated in Aurora
- Notification to retailer via API Gateway WebSocket

**Interface**:
```typescript
interface LegalDocumentGeneratorInput {
  invoiceId: string;
  retailerDetails: {
    name: string;
    udyamNumber: string;
    address: string;
    contact: string;
  };
  buyerDetails: {
    name: string;
    gstin: string;
    address: string;
    contact: string;
  };
  invoiceDetails: {
    invoiceNumber: string;
    invoiceDate: string;
    principalAmount: number;
    dueDate: string;
    overdueBy: number; // days
  };
  penaltyCalculation: {
    bankRate: number;
    penaltyRate: number; // 3x bank rate
    compoundInterest: number;
    totalAmountDue: number;
  };
  language: 'ta' | 'hi' | 'en';
}

interface Section15Notice {
  noticeId: string;
  generatedDate: string;
  pdfS3Key: string;
  content: {
    header: string;
    body: string;
    footer: string;
    legalReferences: string[];
  };
}
```

### 5. Risk Analyzer Lambda

**Responsibility**: Calculates buyer reliability scores based on historical payment patterns.

**Trigger**: API Gateway request (on-demand when retailer searches for buyer)

**Processing**:
1. Query Aurora for all historical invoices associated with buyer GSTIN
2. Calculate metrics:
   - Average payment delay (days)
   - On-time payment percentage
   - Number of disputes filed
   - Total transaction volume
3. Use Claude 3.5 via Bedrock to generate risk assessment:
   - Input: aggregated anonymized payment patterns
   - Output: reliability score (0-100) and risk category
4. Cache result in Aurora for 24 hours

**Outputs**:
- Buyer reliability score and risk assessment
- Cached result in Aurora

**Interface**:
```typescript
interface RiskAnalyzerInput {
  buyerGSTIN: string;
}

interface RiskAnalyzerOutput {
  buyerGSTIN: string;
  reliabilityScore: number; // 0-100
  riskCategory: 'low' | 'medium' | 'high';
  metrics: {
    averagePaymentDelay: number; // days
    onTimePaymentPercentage: number;
    disputeCount: number;
    totalTransactionCount: number;
    dataConfidence: 'high' | 'medium' | 'low'; // Based on sample size
  };
  recommendation: string; // Generated by Claude
  lastUpdated: string; // ISO 8601
}
```

### 6. Samadhaan Integrator Lambda

**Responsibility**: Validates Udyam registration and submits pre-filled dispute forms to MSME Samadhaan portal.

**Trigger**: API Gateway request (when retailer initiates dispute filing)

**Processing**:
1. Validate Udyam Number via Udyam Portal API
2. Retrieve invoice, Section 15 notice, and all communication logs from Aurora/S3
3. Use Claude 3.5 via Bedrock to format dispute submission:
   - Map invoice data to Samadhaan form fields
   - Generate case summary and description
4. Attach supporting documents (invoice PDF, Section 15 notice)
5. Submit to MSME Samadhaan portal via API
6. Store submission confirmation and reference number in Aurora

**Outputs**:
- Validated Udyam registration
- Submitted dispute with reference number
- Updated invoice status in Aurora

**Interface**:
```typescript
interface SamaadhaanIntegratorInput {
  invoiceId: string;
  retailerUdyamNumber: string;
}

interface UdyamValidationResponse {
  valid: boolean;
  enterpriseDetails?: {
    name: string;
    type: 'micro' | 'small' | 'medium';
    registrationDate: string;
  };
  error?: string;
}

interface SamaadhaanSubmission {
  referenceNumber: string;
  submissionDate: string;
  status: 'submitted' | 'pending_review' | 'accepted';
  portalUrl: string;
}
```

### 7. Voice Interface Handler Lambda

**Responsibility**: Processes voice commands and provides voice responses for low-literacy users.

**Trigger**: API Gateway request from mobile app

**Processing**:
1. Receive audio stream from mobile app
2. Use Amazon Transcribe to convert speech to text
3. Use Claude 3.5 via Bedrock to interpret intent and extract parameters
4. Route to appropriate Lambda function (Invoice Processor, Payment Monitor, Risk Analyzer)
5. Format response for voice output
6. Use Amazon Polly to convert text response to speech
7. Return audio stream to mobile app

**Outputs**:
- Transcribed text and interpreted intent
- Audio response stream

**Interface**:
```typescript
interface VoiceInterfaceInput {
  audioS3Key: string; // Audio uploaded to S3
  language: 'ta' | 'hi' | 'en';
  retailerId: string;
}

interface VoiceInterfaceOutput {
  transcription: string;
  intent: 'upload_invoice' | 'check_status' | 'search_buyer' | 'unknown';
  parameters: Record<string, any>;
  responseText: string;
  responseAudioS3Key: string;
}
```

## Data Models

### Aurora PostgreSQL Schema

```sql
-- Retailers table
CREATE TABLE retailers (
  retailer_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  udyam_number VARCHAR(50) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL,
  phone VARCHAR(15) NOT NULL,
  email VARCHAR(255),
  preferred_language VARCHAR(2) NOT NULL CHECK (preferred_language IN ('ta', 'hi', 'en')),
  address TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Buyers table
CREATE TABLE buyers (
  buyer_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  gstin VARCHAR(15) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL,
  phone VARCHAR(15),
  email VARCHAR(255),
  address TEXT,
  reliability_score INTEGER CHECK (reliability_score BETWEEN 0 AND 100),
  risk_category VARCHAR(10) CHECK (risk_category IN ('low', 'medium', 'high')),
  score_updated_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Invoices table
CREATE TABLE invoices (
  invoice_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  retailer_id UUID NOT NULL REFERENCES retailers(retailer_id),
  buyer_id UUID NOT NULL REFERENCES buyers(buyer_id),
  invoice_number VARCHAR(100),
  invoice_date DATE NOT NULL,
  principal_amount DECIMAL(15, 2) NOT NULL,
  due_date DATE NOT NULL, -- invoice_date + 45 days
  payment_date DATE,
  payment_reference VARCHAR(100),
  status VARCHAR(20) NOT NULL CHECK (status IN ('active', 'paid', 'overdue', 'legal_notice_sent', 'dispute_filed')),
  document_s3_key VARCHAR(500) NOT NULL,
  extraction_confidence JSONB, -- Stores confidence scores for each field
  requires_manual_review BOOLEAN DEFAULT FALSE,
  compound_interest_penalty DECIMAL(15, 2),
  total_amount_due DECIMAL(15, 2),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Communications table (audit trail)
CREATE TABLE communications (
  communication_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  invoice_id UUID NOT NULL REFERENCES invoices(invoice_id),
  communication_type VARCHAR(20) NOT NULL CHECK (communication_type IN ('reminder_first', 'reminder_second', 'reminder_final', 'legal_notice')),
  channel VARCHAR(10) NOT NULL CHECK (channel IN ('whatsapp', 'email')),
  recipient VARCHAR(255) NOT NULL,
  message_content TEXT NOT NULL,
  delivery_status VARCHAR(20) NOT NULL CHECK (delivery_status IN ('sent', 'delivered', 'failed', 'read')),
  sent_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  delivered_at TIMESTAMP,
  retry_count INTEGER DEFAULT 0
);

-- Legal notices table
CREATE TABLE legal_notices (
  notice_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  invoice_id UUID NOT NULL REFERENCES invoices(invoice_id),
  notice_type VARCHAR(20) DEFAULT 'section_15',
  generated_date DATE NOT NULL,
  approved_by_retailer BOOLEAN DEFAULT FALSE,
  approved_at TIMESTAMP,
  sent_at TIMESTAMP,
  pdf_s3_key VARCHAR(500) NOT NULL,
  language VARCHAR(2) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Samadhaan submissions table
CREATE TABLE samadhaan_submissions (
  submission_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  invoice_id UUID NOT NULL REFERENCES invoices(invoice_id),
  reference_number VARCHAR(100) UNIQUE NOT NULL,
  submission_date DATE NOT NULL,
  status VARCHAR(20) NOT NULL CHECK (status IN ('submitted', 'pending_review', 'accepted', 'hearing_scheduled', 'resolved')),
  portal_url VARCHAR(500),
  resolution_details TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Audit logs table
CREATE TABLE audit_logs (
  log_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  entity_type VARCHAR(50) NOT NULL, -- 'invoice', 'communication', 'notice', etc.
  entity_id UUID NOT NULL,
  action VARCHAR(100) NOT NULL,
  performed_by UUID, -- retailer_id or 'system'
  details JSONB,
  timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for performance
CREATE INDEX idx_invoices_retailer ON invoices(retailer_id);
CREATE INDEX idx_invoices_buyer ON invoices(buyer_id);
CREATE INDEX idx_invoices_status ON invoices(status);
CREATE INDEX idx_invoices_due_date ON invoices(due_date);
CREATE INDEX idx_communications_invoice ON communications(invoice_id);
CREATE INDEX idx_buyers_gstin ON buyers(gstin);
```


### S3 Document Storage Structure

```
msme-payment-enforcement/
├── invoices/
│   ├── {retailer_id}/
│   │   ├── {invoice_id}/
│   │   │   ├── original.{ext}  # Original uploaded document
│   │   │   └── pages/          # Individual pages if multi-page
│   │   │       ├── page_1.jpg
│   │   │       └── page_2.jpg
├── legal_notices/
│   ├── {invoice_id}/
│   │   └── section_15_notice_{timestamp}.pdf
└── voice_recordings/
    ├── {retailer_id}/
    │   ├── input_{timestamp}.mp3
    │   └── response_{timestamp}.mp3
```

All objects stored with:
- Server-side encryption (SSE-S3 or SSE-KMS)
- Lifecycle policy: Transition to Glacier after 1 year, retain for 7 years
- Versioning enabled for legal documents

## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Property Reflection

After analyzing all acceptance criteria, I identified several areas where properties can be consolidated:

1. **Reminder timing properties (4.1, 4.2, 4.3)** can be combined into a single comprehensive property about reminder scheduling
2. **Penalty calculation properties (6.1, 6.2, 6.3, 6.4, 6.5)** can be consolidated into properties about calculation correctness and real-time updates
3. **Language selection properties (4.4, 5.5, 9.4, 10.2, 10.3)** share the same underlying behavior and can be unified
4. **Audit logging properties (14.1, 14.2, 14.3, 14.4)** all test that events are logged and can be combined
5. **Offline sync properties (15.2, 15.4)** both test queuing and synchronization behavior

### Core Properties

**Property 1: File Format Validation**
*For any* uploaded document, the Invoice_Processor should accept files with JPEG, PNG, HEIC, or PDF extensions and reject all other formats with an appropriate error message.
**Validates: Requirements 1.1, 1.2**

**Property 2: File Size Validation**
*For any* uploaded PDF document, the Invoice_Processor should accept files up to 10MB and reject larger files with an appropriate error message.
**Validates: Requirements 1.2**

**Property 3: Processing Time Bound**
*For any* valid invoice document, the Invoice_Processor should complete extraction and storage within 30 seconds.
**Validates: Requirements 1.3**

**Property 4: Multi-Page Consolidation**
*For any* set of invoice page images uploaded for the same invoice, the Invoice_Processor should create exactly one invoice record with all pages referenced.
**Validates: Requirements 1.4**

**Property 5: Low Confidence Field Flagging**
*For any* extracted field with confidence score below 80%, the Invoice_Processor should flag that field for manual review and prompt the retailer to verify it.
**Validates: Requirements 2.5**

**Property 6: Extraction Completeness**
*For any* processed invoice, the Invoice_Processor should return all four mandatory fields (Buyer_GSTIN, Udyam_Number, Invoice_Date, Principal_Amount) in the response for retailer confirmation.
**Validates: Requirements 2.6**

**Property 7: Due Date Calculation**
*For any* invoice date, the Payment_Monitor should calculate the due date as exactly 45 days after the invoice date.
**Validates: Requirements 3.1**

**Property 8: Days Remaining Calculation**
*For any* invoice with a due date, the Payment_Monitor should correctly calculate the number of days remaining (or overdue) relative to the current date.
**Validates: Requirements 3.2, 3.5**

**Property 9: Payment Stops Monitoring**
*For any* invoice marked as paid, the Payment_Monitor should not generate any further reminder or notice events for that invoice.
**Validates: Requirements 3.3, 11.2**

**Property 10: Reminder Scheduling**
*For any* active (unpaid) invoice, the Communication_Agent should send reminders at exactly day 30, day 40, and day 44 after the invoice date, and the Legal_Document_Generator should generate a notice after day 45.
**Validates: Requirements 4.1, 4.2, 4.3, 5.1**

**Property 11: Language Consistency**
*For any* retailer with a preferred language setting, all generated communications, documents, UI elements, and voice responses should be in that language (Tamil, Hindi, or English).
**Validates: Requirements 4.4, 5.5, 9.4, 10.2, 10.3**

**Property 12: Required Message Fields**
*For any* generated reminder message, the message content should include the invoice number, principal amount, and payment due date.
**Validates: Requirements 4.6**

**Property 13: Communication Retry Logic**
*For any* failed communication attempt, the Communication_Agent should retry up to 3 times with exponential backoff before marking as permanently failed.
**Validates: Requirements 4.7**

**Property 14: Legal Notice Mandatory Elements**
*For any* generated Section 15 notice, the document should include all mandatory legal elements: retailer details, buyer details, invoice details, MSMED Act references, compound interest penalty calculation, and formal demand for payment.
**Validates: Requirements 5.2**

**Property 15: Notice Includes Penalty**
*For any* generated Section 15 notice, the document should include the calculated compound interest penalty amount.
**Validates: Requirements 5.4**

**Property 16: Notice Approval Required**
*For any* generated Section 15 notice, the notice should not be sent to the buyer until the retailer explicitly approves it.
**Validates: Requirements 5.6**

**Property 17: Approved Notice Delivery**
*For any* Section 15 notice approved by the retailer, the Communication_Agent should send it via registered email with delivery confirmation tracking.
**Validates: Requirements 5.7**

**Property 18: Compound Interest Calculation**
*For any* overdue invoice, the Payment_Monitor should calculate the compound interest penalty as: Principal_Amount × (1 + (3 × bank_rate / 12))^months - Principal_Amount, where months is the number of complete months overdue.
**Validates: Requirements 6.1, 6.2, 6.3**

**Property 19: Total Amount Calculation**
*For any* overdue invoice, the displayed total amount owed should equal the principal amount plus the compound interest penalty.
**Validates: Requirements 6.4**

**Property 20: Real-Time Penalty Updates**
*For any* overdue invoice, querying the penalty amount at different times should reflect the updated calculation based on the current number of overdue days.
**Validates: Requirements 6.5**

**Property 21: Udyam Validation**
*For any* dispute filing attempt, the Samadhaan_Integrator should validate the Udyam number with the government portal before proceeding with form pre-filling.
**Validates: Requirements 7.1**

**Property 22: Dispute Form Pre-filling**
*For any* validated Udyam number, the Samadhaan_Integrator should pre-fill the dispute form with all invoice details including Buyer_GSTIN, Invoice_Date, Principal_Amount, and Compound_Interest_Penalty.
**Validates: Requirements 7.2, 7.3**

**Property 23: Dispute Document Attachment**
*For any* pre-filled dispute form, the Samadhaan_Integrator should attach both the original invoice document and the Section 15 notice.
**Validates: Requirements 7.4**

**Property 24: Dispute Form Review Required**
*For any* pre-filled dispute form, the form should not be submitted to the MSME Samadhaan portal until the retailer reviews and approves it.
**Validates: Requirements 7.5**

**Property 25: Dispute Submission After Approval**
*For any* dispute form approved by the retailer, the Samadhaan_Integrator should submit it to the MSME Samadhaan portal and store the confirmation reference number.
**Validates: Requirements 7.6**

**Property 26: Buyer Score Retrieval**
*For any* valid Buyer_GSTIN, the Risk_Analyzer should return a Buyer_Reliability_Score when queried.
**Validates: Requirements 8.1**

**Property 27: Score Metric Sensitivity**
*For any* buyer, if the average payment delay increases, or the on-time payment percentage decreases, or the dispute count increases, the Buyer_Reliability_Score should decrease (or stay the same, but not increase).
**Validates: Requirements 8.3, 8.4, 8.5**

**Property 28: Score Range Validation**
*For any* calculated Buyer_Reliability_Score, the score should be between 0 and 100 inclusive.
**Validates: Requirements 8.6**

**Property 29: Risk Category Assignment**
*For any* Buyer_Reliability_Score, the Risk_Analyzer should assign exactly one risk category from: Low Risk, Medium Risk, or High Risk.
**Validates: Requirements 8.7**

**Property 30: Voice Language Support**
*For any* voice command in Tamil, Hindi, or English, the Voice_Interface should successfully transcribe and process the command.
**Validates: Requirements 9.1**

**Property 31: Voice Transcription Performance**
*For any* voice command, the Voice_Interface should complete transcription within 3 seconds.
**Validates: Requirements 9.2**

**Property 32: Voice Intent Recognition**
*For any* recognized voice command, the Voice_Interface should identify the correct intent (upload_invoice, check_status, search_buyer) and execute the corresponding action.
**Validates: Requirements 9.3**

**Property 33: Voice Command Coverage**
*For any* voice command requesting invoice upload, status check, or buyer search, the Voice_Interface should recognize and execute the command.
**Validates: Requirements 9.6**

**Property 34: UI Language Update**
*For any* language preference change, all UI elements should immediately update to display in the newly selected language.
**Validates: Requirements 10.4**

**Property 35: Currency Formatting**
*For any* displayed currency amount, the system should format it according to Indian Rupee conventions (₹ symbol, comma separators for thousands/lakhs).
**Validates: Requirements 10.5**

**Property 36: Payment Marking Prompt**
*For any* invoice being marked as paid, the system should prompt for both payment date and payment reference number before completing the operation.
**Validates: Requirements 11.1**

**Property 37: Payment Duration Calculation**
*For any* invoice marked as paid, the system should calculate the actual payment duration as the number of days between invoice date and payment date.
**Validates: Requirements 11.3**

**Property 38: Payment History Update**
*For any* invoice marked as paid, the buyer's payment history should be updated to include this payment record for future reliability score calculations.
**Validates: Requirements 11.4**

**Property 39: Penalty Payment Recording**
*For any* invoice marked as paid after the 45-day deadline, the system should record whether the compound interest penalty was paid along with the principal amount.
**Validates: Requirements 11.5**

**Property 40: Retailer Notification on Reminder Sent**
*For any* reminder sent to a buyer, the system should send a notification to the retailer confirming the reminder was sent.
**Validates: Requirements 12.1**

**Property 41: Notice Generation Notification**
*For any* automatically generated Section 15 notice, the system should notify the retailer and request approval before sending.
**Validates: Requirements 12.2**

**Property 42: Deadline Approaching Alert**
*For any* invoice with a due date exactly 3 days in the future, the system should send an alert to the retailer.
**Validates: Requirements 12.3**

**Property 43: Overdue Alert**
*For any* invoice that becomes overdue (current date > due date), the system should send an alert to the retailer.
**Validates: Requirements 12.4**

**Property 44: Daily Summary Notifications**
*For any* retailer with notifications enabled, the system should send a daily summary of all active invoices and their current status.
**Validates: Requirements 12.5**

**Property 45: Buyer Response Notification**
*For any* response received from a buyer to a reminder or notice, the system should immediately notify the retailer.
**Validates: Requirements 12.6**

**Property 46: Sensitive Data Encryption**
*For any* invoice stored in the database, sensitive fields (Principal_Amount, Buyer_GSTIN) should be encrypted at rest.
**Validates: Requirements 13.1**

**Property 47: Anonymized Risk Analysis**
*For any* Buyer_Reliability_Score calculation, the Risk_Analyzer should use only anonymized payment pattern data without retailer-identifying information.
**Validates: Requirements 13.2**

**Property 48: Authentication Required**
*For any* system access attempt, the system should require successful authentication via mobile OTP or biometric verification before granting access.
**Validates: Requirements 13.3**

**Property 49: Document Encryption**
*For any* invoice document stored in S3, the document should be encrypted at rest using AES-256 or equivalent encryption.
**Validates: Requirements 13.4**

**Property 50: TLS Transport Security**
*For any* external API communication (WhatsApp, Email, Samadhaan, Udyam), the connection should use TLS 1.3 or higher.
**Validates: Requirements 13.5**

**Property 51: Data Anonymization on Deletion**
*For any* retailer data deletion request, the system should anonymize all retailer-identifying information while preserving aggregated payment statistics for buyer reliability scoring.
**Validates: Requirements 13.7**

**Property 52: Communication Audit Logging**
*For any* sent reminder or notice, the system should log the timestamp, recipient, delivery status, and message content in the audit trail.
**Validates: Requirements 14.1**

**Property 53: Notice Timestamping and Signing**
*For any* generated Section 15 notice, the system should store a timestamped copy with a digital signature for legal validity.
**Validates: Requirements 14.2**

**Property 54: Action Audit Logging**
*For any* action taken by a retailer (invoice upload, payment marking, dispute filing), the system should log the action with timestamp and user identification.
**Validates: Requirements 14.3**

**Property 55: Dispute Submission Recording**
*For any* dispute filed with MSME Samadhaan, the system should store the submission confirmation and reference number in the audit trail.
**Validates: Requirements 14.4**

**Property 56: Audit Trail Generation**
*For any* invoice, the system should be able to generate a chronological report of all activities (uploads, reminders, notices, payments, disputes) associated with that invoice.
**Validates: Requirements 14.5**

**Property 57: Offline Image Capture**
*For any* invoice image capture attempt when internet connectivity is unavailable, the system should store the image locally and allow the capture to complete.
**Validates: Requirements 15.1**

**Property 58: Automatic Sync on Reconnection**
*For any* pending offline operations (image uploads, status updates), the system should automatically synchronize them when internet connectivity is restored.
**Validates: Requirements 15.2, 15.4**

**Property 59: Offline Data Access**
*For any* previously synced invoice, the system should display cached invoice information when operating offline.
**Validates: Requirements 15.3**

