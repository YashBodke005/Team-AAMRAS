# Design Document: MediConnect AI

## Overview

MediConnect AI is a voice-first AI healthcare platform connecting 1 million ASHA workers to 1.3 million doctors across India. The system addresses critical healthcare gaps in rural India through AI-powered diagnosis (87% accuracy), instant telemedicine connectivity, and automated clinical documentation.

### Design Goals

1. **Accessibility**: Voice-first interface supporting 10+ Indian languages with offline capability
2. **Clinical Accuracy**: 87% diagnosis accuracy through ensemble AI models, improving to 95% over 2 years
3. **Performance**: <3s diagnosis, <200ms API response, <2s voice transcription
4. **Scalability**: Support 10,000 concurrent users, 100,000 assessments/day
5. **Cost Efficiency**: Leverage free tiers (AWS, Groq, Supabase) to maintain free ASHA access
6. **Compliance**: DPDP Act 2023, ABDM standards, HIPAA-aligned practices

### Key Technical Decisions

- **AI Strategy**: Ensemble voting (BioBERT + PubMed-BERT + SetFit) for robustness over single model
- **Voice Processing**: Groq Whisper (14,400 free req/day) for cost efficiency
- **Clinical Documentation**: Groq Llama 3.1-8B (free tier) for SOAP notes generation
- **Offline Strategy**: Quantized TinyBERT (50MB) with IndexedDB caching
- **Database**: Supabase PostgreSQL (free 500MB) + Upstash Redis (10K commands/day)
- **Deployment**: AWS Lambda (serverless) + Vercel (frontend) for auto-scaling
- **Telemedicine**: Twilio Video API (45K free minutes) for video consultations

## Architecture

### System Architecture Overview

MediConnect AI follows a 5-layer architecture optimized for serverless deployment and free-tier resource utilization:

```
┌─────────────────────────────────────────────────────────────┐
│                    LAYER 1: PRESENTATION                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   WhatsApp   │  │  PWA (React) │  │    Doctor    │      │
│  │     Bot      │  │   + Vite +   │  │  Dashboard   │      │
│  │   (Twilio)   │  │   Tailwind   │  │   (React)    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                   LAYER 2: API GATEWAY                       │
│              AWS API Gateway / Kong Gateway                  │
│     Rate Limiting • JWT Auth • CORS • Logging               │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                 LAYER 3: APPLICATION LAYER                   │
│                      (FastAPI Services)                      │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐             │
│  │  Symptom   │ │    AI      │ │    Risk    │             │
│  │ Processing │ │ Prediction │ │Stratifier  │             │
│  └────────────┘ └────────────┘ └────────────┘             │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐             │
│  │Telemedicine│ │  Clinical  │ │ Knowledge  │             │
│  │   Bridge   │ │    Docs    │ │    Base    │             │
│  └────────────┘ └────────────┘ └────────────┘             │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                   LAYER 4: AI/ML LAYER                       │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐             │
│  │  BioBERT   │ │ PubMed-BERT│ │   SetFit   │             │
│  │  (85% acc) │ │  (78% acc) │ │  (72% acc) │             │
│  └────────────┘ └────────────┘ └────────────┘             │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐             │
│  │    Groq    │ │  Bedrock   │ │ TinyBERT   │             │
│  │   Whisper  │ │Nova Vision │ │  (Offline) │             │
│  └────────────┘ └────────────┘ └────────────┘             │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    LAYER 5: DATA LAYER                       │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐             │
│  │  Supabase  │ │   AWS S3   │ │  Upstash   │             │
│  │ PostgreSQL │ │  (5GB free)│ │   Redis    │             │
│  │ (500MB)    │ │            │ │ (10K/day)  │             │
│  └────────────┘ └────────────┘ └────────────┘             │
│  ┌────────────┐ ┌────────────┐                             │
│  │ IndexedDB  │ │  AWS SQS   │                             │
│  │ (Offline)  │ │  (Queue)   │                             │
│  └────────────┘ └────────────┘                             │
└─────────────────────────────────────────────────────────────┘
```

### Layer 1: Presentation Layer

**WhatsApp Bot (Primary ASHA Interface)**
- Platform: Twilio WhatsApp Business API
- Capabilities: Text messages, voice notes, images, interactive buttons
- Message Flow:
  1. ASHA sends voice note with symptoms
  2. Bot acknowledges receipt, shows "processing" status
  3. Bot returns diagnosis with risk level, treatment, nearest hospital
  4. Interactive buttons: "Connect to Doctor", "View Details", "New Assessment"

**Progressive Web App (PWA)**
- Framework: React 18 + Vite + Tailwind CSS
- Offline Support: Service Workers + IndexedDB
- Features:
  - Voice recording with waveform visualization
  - Image capture for anemia/jaundice detection
  - Assessment history with search/filter
  - Offline indicator with sync status
  - Multi-language UI (10+ languages)
- Deployment: Vercel (100GB bandwidth free)

**Doctor Dashboard**
- Framework: React 18 + Tailwind CSS
- Features:
  - Real-time consultation transcription
  - SOAP notes editor with AI suggestions
  - Voice-to-prescription interface
  - Telemedicine video interface
  - Patient history viewer
  - Analytics dashboard (consultations, time saved, accuracy)
- Deployment: Vercel

### Layer 2: API Gateway

**AWS API Gateway Configuration**
- Endpoints:
  - `/api/v1/assess` - Symptom assessment
  - `/api/v1/transcribe` - Voice transcription
  - `/api/v1/telemedicine` - Doctor connection
  - `/api/v1/clinical-docs` - SOAP notes generation
  - `/api/v1/surveillance` - Epidemic data
- Rate Limiting:
  - ASHA tier: 100 requests/minute
  - Doctor tier: 200 requests/minute
  - Health Officer tier: 50 requests/minute
- Authentication: JWT validation with RS256 signing
- CORS: Whitelist domains (*.mediconnect.ai, WhatsApp webhook)
- Logging: CloudWatch with request/response bodies (PII redacted)

### Layer 3: Application Layer (FastAPI Services)

**Service Architecture**

Each service is a separate FastAPI module deployed as AWS Lambda function:

**1. Symptom Processing Service**
```python
# Endpoint: POST /api/v1/assess
# Input: { "text": "fever and cough for 3 days", "language": "hi", "patient_id": "..." }
# Output: { "symptoms": [...], "processed_text": "...", "confidence": 0.92 }
```
- Responsibilities:
  - Text normalization and cleaning
  - Language detection (if not provided)
  - Medical entity extraction using spaCy medical NER
  - Symptom standardization (map regional terms to medical terms)
  - Duration and severity extraction
- Dependencies: spaCy, langdetect, custom medical dictionary

**2. AI Prediction Service**
```python
# Endpoint: POST /api/v1/predict
# Input: { "symptoms": [...], "patient_context": {...} }
# Output: { "predictions": [{"disease": "...", "confidence": 0.87}], "top_3": [...] }
```
- Responsibilities:
  - Load 3 models (BioBERT, PubMed-BERT, SetFit)
  - Parallel inference execution
  - Ensemble voting (weighted average: BioBERT 50%, PubMed-BERT 30%, SetFit 20%)
  - Confidence calibration
  - Top-3 disease ranking
- Model Loading: Lazy loading with caching (Lambda /tmp directory)
- Timeout: 10 seconds

**3. Risk Stratification Service**
```python
# Endpoint: POST /api/v1/stratify
# Input: { "symptoms": [...], "predictions": [...], "patient_age": 45 }
# Output: { "risk_level": "EMERGENCY", "reasoning": "...", "action": "..." }
```
- Responsibilities:
  - Emergency keyword detection (chest pain, difficulty breathing, severe bleeding, unconsciousness)
  - Context evaluation (age, existing conditions, symptom severity)
  - Risk level assignment (EMERGENCY/URGENT/ROUTINE)
  - Action recommendation generation
  - Hospital/ambulance contact retrieval
- Rules Engine:
  - EMERGENCY: Any life-threatening symptom OR high-risk combination
  - URGENT: Moderate symptoms requiring medical attention within 24hrs
  - ROUTINE: Mild symptoms manageable with home care
- Sensitivity Target: >99% for EMERGENCY detection (zero false negatives)

**4. Telemedicine Bridge Service**
```python
# Endpoint: POST /api/v1/telemedicine/connect
# Input: { "asha_id": "...", "patient_id": "...", "case_summary": {...} }
# Output: { "doctor_id": "...", "video_room_id": "...", "estimated_wait": 0 }
```
- Responsibilities:
  - Query available doctors (status=online, current_load<5)
  - Prioritize by: same district > same state > any available
  - Generate AI case summary (symptoms, diagnosis, risk, vitals)
  - Create Twilio Video room
  - Send notifications to doctor and ASHA
  - Track consultation start time
- Matching Algorithm:
  ```
  1. Filter doctors: online AND current_load < 5
  2. Score doctors: district_match(10) + state_match(5) + specialization_match(3)
  3. Sort by score DESC, then by current_load ASC
  4. Return top match
  5. If no match, queue request and return estimated_wait
  ```

**5. Clinical Documentation Service**
```python
# Endpoint: POST /api/v1/clinical-docs/generate
# Input: { "transcription": "...", "consultation_type": "telemedicine" }
# Output: { "soap_notes": {...}, "icd10_codes": [...], "prescription": {...} }
```
- Responsibilities:
  - Parse transcription into structured data
  - Generate SOAP notes using Groq Llama 3.1-8B
  - Extract diagnoses and map to ICD-10 codes
  - Extract prescriptions with drug names, dosages, frequency
  - Drug interaction checking
  - Format output for EHR export (HL7 FHIR)
- Groq API Configuration:
  - Model: llama-3.1-8b-instant
  - Temperature: 0.3 (deterministic for medical docs)
  - Max tokens: 2048
  - System prompt: "You are a medical documentation assistant..."

**6. Knowledge Base Service**
```python
# Endpoint: GET /api/v1/knowledge/treatment?disease=malaria
# Output: { "treatment": "...", "medications": [...], "precautions": [...] }
```
- Responsibilities:
  - Query ICMR/NRHM treatment protocols
  - Retrieve medication information
  - Provide home care instructions
  - Return contraindications and warnings
- Data Source: GraphRAG index of ICMR guidelines (300+ pages)
- Caching: Redis cache with 24-hour TTL

**7. Notification Service**
```python
# Endpoint: POST /api/v1/notify
# Input: { "recipient": "...", "type": "whatsapp", "message": "...", "template": "..." }
```
- Responsibilities:
  - Send WhatsApp messages via Twilio
  - Send SMS for critical alerts
  - Format messages based on templates
  - Handle delivery status callbacks
  - Retry failed deliveries (3 attempts with exponential backoff)

### Layer 4: AI/ML Layer

**ASHA Diagnosis Pipeline**

```
Input: "बुखार और खांसी 3 दिन से" (Hindi voice)
  ↓
[1] Voice Transcription (Groq Whisper Large-v3)
  → Output: "fever and cough for 3 days"
  → Latency: <2s
  ↓
[2] Symptom Extraction (spaCy Medical NER)
  → Entities: [fever, cough]
  → Duration: 3 days
  → Severity: not specified (assume moderate)
  ↓
[3] Parallel Model Inference
  ┌─────────────┬─────────────┬─────────────┐
  │  BioBERT    │ PubMed-BERT │   SetFit    │
  │  Input: symptoms + context              │
  │  Output: disease probabilities          │
  └─────────────┴─────────────┴─────────────┘
  → BioBERT: [URI: 0.82, Flu: 0.15, COVID: 0.03]
  → PubMed-BERT: [URI: 0.75, Flu: 0.20, Bronchitis: 0.05]
  → SetFit: [URI: 0.68, Flu: 0.25, COVID: 0.07]
  ↓
[4] Ensemble Voting (Weighted Average)
  → Weights: BioBERT(0.5), PubMed-BERT(0.3), SetFit(0.2)
  → URI: 0.82*0.5 + 0.75*0.3 + 0.68*0.2 = 0.771
  → Flu: 0.15*0.5 + 0.20*0.3 + 0.25*0.2 = 0.185
  → Final: URI (77.1%), Flu (18.5%)
  ↓
[5] Risk Stratification
  → Symptoms: fever, cough (no emergency keywords)
  → Duration: 3 days (not acute)
  → Risk: ROUTINE
  → Action: Home care, monitor for worsening
  ↓
[6] Treatment Retrieval (Groq API / Knowledge Base)
  → Query: "URI treatment ICMR protocol"
  → Response: Rest, fluids, paracetamol, monitor temperature
  ↓
[7] Hospital Finder (Google Places API)
  → Location: ASHA's current GPS
  → Radius: 5km
  → Results: [PHC Rampur (2.3km), District Hospital (4.8km)]
  ↓
Output (WhatsApp Message):
  "🏥 Assessment Complete
  
  Diagnosis: Upper Respiratory Infection (77% confidence)
  Risk Level: ROUTINE ✅
  
  Treatment:
  • Rest and drink plenty of fluids
  • Paracetamol 500mg every 6 hours for fever
  • Monitor temperature
  
  When to seek help:
  • If fever persists >5 days
  • Difficulty breathing
  • Chest pain
  
  Nearest PHC: Rampur PHC (2.3km)
  📞 Contact: 9876543210"
```

**Model Details**

**BioBERT (Primary Model - 85% accuracy)**
- Base: BERT-base (110M parameters)
- Pre-training: PubMed abstracts (4.5B words) + PMC full-text (13.5B words)
- Fine-tuning: MedQA dataset (12K medical Q&A)
- Input: [CLS] symptom_text [SEP] patient_context [SEP]
- Output: 30-class disease classification
- Inference: 150ms on CPU, 50ms on GPU
- Deployment: ONNX quantized model (INT8) for faster inference

**PubMed-BERT (Secondary Model - 78% accuracy)**
- Base: BERT-base (110M parameters)
- Pre-training: PubMed abstracts only (21B words)
- Specialization: Better for rare diseases and research-backed conditions
- Same architecture as BioBERT, different training corpus

**SetFit (Tertiary Model - 72% accuracy)**
- Base: Sentence-BERT (few-shot learning)
- Training: 8 examples per disease class (240 total examples)
- Advantage: Fast training, good for new diseases
- Inference: 80ms
- Use case: Backup model, handles edge cases

**Offline Model: TinyBERT (Quantized)**
- Base: TinyBERT-4L-312D (14.5M parameters)
- Quantization: INT8 post-training quantization
- Model size: 15MB (compressed)
- Disease coverage: Top 50 diseases (85% of rural cases)
- Accuracy: 78% (trade-off for size)
- Deployment: IndexedDB in browser, TensorFlow.js for inference
- Inference: 500ms on mobile CPU

**Voice Processing: Groq Whisper Large-v3**
- Model: OpenAI Whisper Large-v3 (1.5B parameters)
- Hosting: Groq API (free 14,400 requests/day)
- Languages: Hindi, English, Tamil, Telugu, Bengali, Marathi, Gujarati, Kannada, Malayalam, Punjabi
- Accuracy: >85% for medical terms
- Latency: <2s for 30-second audio
- Input format: MP3, WAV, OGG (max 25MB)
- Fallback: AWS Transcribe Medical (if Groq quota exceeded)

**Image Analysis: AWS Bedrock Nova Vision / ResNet**
- Primary: AWS Bedrock Nova Vision (if credits available)
  - Input: Patient photo (face, eyes, skin)
  - Output: Anemia probability, jaundice probability
  - Latency: <3s
- Fallback: ResNet-50 fine-tuned on medical images
  - Training data: 10K labeled images (anemia, jaundice, normal)
  - Accuracy: 82% for anemia, 88% for jaundice
  - Deployment: ONNX model on Lambda

**Clinical Documentation: Groq Llama 3.1-8B**
- Model: Meta Llama 3.1-8B-Instant
- Hosting: Groq API (free 14,400 requests/day)
- Use cases:
  - SOAP notes generation
  - Prescription formatting
  - Case summary generation
- Prompt Engineering:
  ```
  System: You are a medical documentation assistant. Generate structured SOAP notes from consultation transcripts.
  
  User: Transcription: "Patient complains of fever for 3 days, temperature 101F, prescribed paracetamol..."
  
  Assistant: 
  SUBJECTIVE: Patient reports fever for 3 days
  OBJECTIVE: Temperature 101°F
  ASSESSMENT: Likely viral fever
  PLAN: Paracetamol 500mg TID, rest, fluids
  ```
- Temperature: 0.3 (deterministic)
- Max tokens: 2048

### Layer 5: Data Layer

**Supabase PostgreSQL (Primary Database)**

Schema Design:

```sql
-- Users table (ASHA, Doctors, Health Officers, Admins)
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  phone VARCHAR(15) UNIQUE NOT NULL,
  role VARCHAR(20) NOT NULL CHECK (role IN ('asha', 'doctor', 'health_officer', 'admin')),
  name VARCHAR(100) NOT NULL,
  language VARCHAR(10) DEFAULT 'en',
  district VARCHAR(50),
  state VARCHAR(50),
  created_at TIMESTAMP DEFAULT NOW(),
  last_active TIMESTAMP,
  status VARCHAR(20) DEFAULT 'offline' CHECK (status IN ('online', 'offline', 'busy'))
);

-- Patients table (anonymized for privacy)
CREATE TABLE patients (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  age INT,
  gender VARCHAR(10),
  district VARCHAR(50),
  state VARCHAR(50),
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW()
);

-- Assessments table (symptom assessments by ASHA)
CREATE TABLE assessments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  patient_id UUID REFERENCES patients(id),
  asha_id UUID REFERENCES users(id),
  symptoms JSONB NOT NULL, -- [{name: "fever", duration: "3 days", severity: "moderate"}]
  voice_note_url VARCHAR(255), -- S3 URL
  image_url VARCHAR(255), -- S3 URL for anemia/jaundice photos
  transcription TEXT,
  language VARCHAR(10),
  created_at TIMESTAMP DEFAULT NOW(),
  is_offline BOOLEAN DEFAULT FALSE,
  synced_at TIMESTAMP
);

-- Predictions table (AI diagnosis results)
CREATE TABLE predictions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  assessment_id UUID REFERENCES assessments(id),
  disease VARCHAR(100) NOT NULL,
  confidence DECIMAL(5,4) NOT NULL, -- 0.8765
  model_version VARCHAR(20), -- "ensemble-v1.2"
  top_3_diseases JSONB, -- [{"disease": "URI", "confidence": 0.77}, ...]
  risk_level VARCHAR(20) CHECK (risk_level IN ('EMERGENCY', 'URGENT', 'ROUTINE')),
  treatment_plan TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Consultations table (telemedicine sessions)
CREATE TABLE consultations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  assessment_id UUID REFERENCES assessments(id),
  asha_id UUID REFERENCES users(id),
  doctor_id UUID REFERENCES users(id),
  patient_id UUID REFERENCES patients(id),
  video_room_id VARCHAR(100), -- Twilio room ID
  case_summary TEXT,
  started_at TIMESTAMP,
  ended_at TIMESTAMP,
  duration_seconds INT,
  video_recording_url VARCHAR(255), -- S3 URL
  status VARCHAR(20) CHECK (status IN ('pending', 'active', 'completed', 'cancelled'))
);

-- Clinical notes table (SOAP notes, prescriptions)
CREATE TABLE clinical_notes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  consultation_id UUID REFERENCES consultations(id),
  doctor_id UUID REFERENCES users(id),
  transcription TEXT,
  soap_notes JSONB, -- {subjective: "...", objective: "...", assessment: "...", plan: "..."}
  icd10_codes VARCHAR(50)[], -- ["J06.9", "R50.9"]
  prescription JSONB, -- [{drug: "Paracetamol", dosage: "500mg", frequency: "TID", duration: "5 days"}]
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP
);

-- Feedback table (physician validation for accuracy tracking)
CREATE TABLE feedback (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  prediction_id UUID REFERENCES predictions(id),
  doctor_id UUID REFERENCES users(id),
  correct_diagnosis VARCHAR(100),
  is_accurate BOOLEAN,
  comments TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Surveillance data table (anonymized for epidemic tracking)
CREATE TABLE surveillance_data (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  district VARCHAR(50) NOT NULL,
  state VARCHAR(50) NOT NULL,
  disease VARCHAR(100) NOT NULL,
  case_count INT DEFAULT 1,
  date DATE NOT NULL,
  week_number INT, -- ISO week number
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(district, disease, date)
);

-- Outbreak alerts table
CREATE TABLE outbreak_alerts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  district VARCHAR(50) NOT NULL,
  state VARCHAR(50) NOT NULL,
  disease VARCHAR(100) NOT NULL,
  case_count INT NOT NULL,
  threshold INT NOT NULL,
  confidence DECIMAL(5,4), -- 0.85
  status VARCHAR(20) CHECK (status IN ('active', 'resolved', 'false_alarm')),
  created_at TIMESTAMP DEFAULT NOW(),
  resolved_at TIMESTAMP
);
```

**Indexes for Performance:**
```sql
CREATE INDEX idx_assessments_asha ON assessments(asha_id, created_at DESC);
CREATE INDEX idx_assessments_patient ON assessments(patient_id, created_at DESC);
CREATE INDEX idx_predictions_disease ON predictions(disease, created_at DESC);
CREATE INDEX idx_consultations_doctor ON consultations(doctor_id, status, started_at DESC);
CREATE INDEX idx_surveillance_district_date ON surveillance_data(district, date DESC);
CREATE INDEX idx_surveillance_disease_date ON surveillance_data(disease, date DESC);
```

**AWS S3 (Object Storage)**

Bucket Structure:
```
mediconnect-ai-storage/
├── audio/
│   ├── {year}/{month}/{day}/{assessment_id}.mp3
├── images/
│   ├── {year}/{month}/{day}/{assessment_id}.jpg
├── videos/
│   ├── {year}/{month}/{day}/{consultation_id}.mp4
├── documents/
│   ├── prescriptions/{consultation_id}.pdf
│   ├── clinical-notes/{consultation_id}.pdf
└── knowledge-base/
    ├── icmr-guidelines.json
    ├── nrhm-protocols.json
    └── drug-database.json
```

Lifecycle Policies:
- Audio files: Delete after 90 days
- Images: Delete after 180 days
- Videos: Delete after 30 days (compliance requirement)
- Documents: Retain for 7 years (medical records law)
- Knowledge base: Never delete

**Upstash Redis (Caching)**

Cache Keys:
```
user:session:{user_id} → JWT session data (TTL: 24h)
disease:treatment:{disease_name} → Treatment protocols (TTL: 24h)
doctor:availability:{doctor_id} → Online status (TTL: 5min)
rate_limit:{user_id}:{endpoint} → Request count (TTL: 1min)
common_diseases → Top 50 diseases for offline mode (TTL: 7 days)
hospital:nearby:{lat}:{lng} → Hospital list (TTL: 1h)
```

**IndexedDB (Offline Storage)**

Stores:
```javascript
{
  diseases: [
    { id: 1, name: "URI", symptoms: [...], treatment: "...", model: <TinyBERT weights> }
  ],
  assessments_queue: [
    { id: "...", patient_id: "...", symptoms: [...], timestamp: "...", synced: false }
  ],
  user_preferences: {
    language: "hi",
    district: "Rampur",
    offline_mode_enabled: true
  }
}
```

**AWS SQS (Message Queue)**

Queues:
- `offline-sync-queue`: Assessments made offline, pending sync
- `notification-queue`: WhatsApp/SMS notifications to send
- `surveillance-aggregation-queue`: Symptom data for epidemic tracking
- `model-retraining-queue`: Feedback data for model improvement

## Components and Interfaces

### Voice Processing Component

**Interface:**
```python
class VoiceProcessor:
    def transcribe(self, audio_file: bytes, language: str) -> TranscriptionResult:
        """
        Transcribe audio to text using Groq Whisper or AWS Transcribe Medical
        
        Args:
            audio_file: Audio bytes (MP3, WAV, OGG)
            language: ISO language code (hi, en, ta, etc.)
        
        Returns:
            TranscriptionResult with text, confidence, detected_language
        """
        pass
    
    def detect_language(self, audio_file: bytes) -> str:
        """Auto-detect language from audio"""
        pass
```

**Implementation:**
```python
import httpx
from typing import Optional

class GroqVoiceProcessor(VoiceProcessor):
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.groq.com/openai/v1/audio/transcriptions"
        self.model = "whisper-large-v3"
    
    async def transcribe(self, audio_file: bytes, language: str) -> TranscriptionResult:
        async with httpx.AsyncClient() as client:
            response = await client.post(
                self.base_url,
                headers={"Authorization": f"Bearer {self.api_key}"},
                files={"file": audio_file},
                data={"model": self.model, "language": language}
            )
            result = response.json()
            return TranscriptionResult(
                text=result["text"],
                confidence=result.get("confidence", 0.9),
                detected_language=result.get("language", language)
            )
```

### AI Diagnosis Component

**Interface:**
```python
class DiagnosisEngine:
    def predict(self, symptoms: List[Symptom], patient_context: PatientContext) -> DiagnosisResult:
        """
        Generate disease predictions using ensemble of 3 models
        
        Args:
            symptoms: List of extracted symptoms with duration, severity
            patient_context: Age, gender, existing conditions, location
        
        Returns:
            DiagnosisResult with top_3 predictions, confidence scores, ensemble details
        """
        pass
    
    def stratify_risk(self, symptoms: List[Symptom], predictions: List[Prediction]) -> RiskLevel:
        """Classify case as EMERGENCY, URGENT, or ROUTINE"""
        pass
```

**Implementation:**
```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch
import numpy as np

class EnsembleDiagnosisEngine(DiagnosisEngine):
    def __init__(self):
        self.biobert = self._load_model("dmis-lab/biobert-v1.1")
        self.pubmedbert = self._load_model("microsoft/BiomedNLP-PubMedBERT-base-uncased-abstract")
        self.setfit = self._load_setfit_model("setfit-uri-model")
        self.weights = {"biobert": 0.5, "pubmedbert": 0.3, "setfit": 0.2}
        self.disease_classes = self._load_disease_classes()  # 30 diseases
    
    def predict(self, symptoms: List[Symptom], patient_context: PatientContext) -> DiagnosisResult:
        # Format input text
        input_text = self._format_input(symptoms, patient_context)
        
        # Parallel inference
        biobert_probs = self._infer_biobert(input_text)
        pubmedbert_probs = self._infer_pubmedbert(input_text)
        setfit_probs = self._infer_setfit(input_text)
        
        # Ensemble voting (weighted average)
        ensemble_probs = (
            biobert_probs * self.weights["biobert"] +
            pubmedbert_probs * self.weights["pubmedbert"] +
            setfit_probs * self.weights["setfit"]
        )
        
        # Get top 3 predictions
        top_3_indices = np.argsort(ensemble_probs)[-3:][::-1]
        top_3_predictions = [
            Prediction(
                disease=self.disease_classes[idx],
                confidence=ensemble_probs[idx],
                model_breakdown={
                    "biobert": biobert_probs[idx],
                    "pubmedbert": pubmedbert_probs[idx],
                    "setfit": setfit_probs[idx]
                }
            )
            for idx in top_3_indices
        ]
        
        return DiagnosisResult(
            top_prediction=top_3_predictions[0],
            top_3=top_3_predictions,
            ensemble_confidence=float(ensemble_probs[top_3_indices[0]])
        )
    
    def stratify_risk(self, symptoms: List[Symptom], predictions: List[Prediction]) -> RiskLevel:
        # Emergency keywords
        emergency_keywords = [
            "chest pain", "difficulty breathing", "severe bleeding",
            "unconscious", "seizure", "stroke symptoms", "severe headache"
        ]
        
        # Check for emergency keywords
        symptom_texts = [s.name.lower() for s in symptoms]
        if any(keyword in " ".join(symptom_texts) for keyword in emergency_keywords):
            return RiskLevel.EMERGENCY
        
        # Check prediction severity
        if predictions[0].disease in ["heart attack", "stroke", "sepsis", "meningitis"]:
            return RiskLevel.EMERGENCY
        
        # Check symptom severity and duration
        severe_symptoms = [s for s in symptoms if s.severity == "severe"]
        if severe_symptoms and len(severe_symptoms) >= 2:
            return RiskLevel.URGENT
        
        # Default to routine
        return RiskLevel.ROUTINE
```

### Telemedicine Component

**Interface:**
```python
class TelemedicineBridge:
    def find_available_doctor(self, district: str, state: str, specialization: Optional[str]) -> Optional[Doctor]:
        """Find available doctor matching criteria"""
        pass
    
    def create_video_room(self, consultation_id: str) -> VideoRoom:
        """Create Twilio video room for consultation"""
        pass
    
    def generate_case_summary(self, assessment: Assessment, prediction: DiagnosisResult) -> str:
        """Generate AI case summary for doctor"""
        pass
```

**Implementation:**
```python
from twilio.rest import Client
from twilio.jwt.access_token import AccessToken
from twilio.jwt.access_token.grants import VideoGrant

class TwilioTelemedicineBridge(TelemedicineBridge):
    def __init__(self, account_sid: str, auth_token: str, api_key: str, api_secret: str):
        self.client = Client(account_sid, auth_token)
        self.api_key = api_key
        self.api_secret = api_secret
    
    def find_available_doctor(self, district: str, state: str, specialization: Optional[str]) -> Optional[Doctor]:
        # Query database for available doctors
        query = """
            SELECT * FROM users
            WHERE role = 'doctor'
            AND status = 'online'
            AND (SELECT COUNT(*) FROM consultations WHERE doctor_id = users.id AND status = 'active') < 5
            ORDER BY
                CASE WHEN district = $1 THEN 10 ELSE 0 END +
                CASE WHEN state = $2 THEN 5 ELSE 0 END DESC,
                (SELECT COUNT(*) FROM consultations WHERE doctor_id = users.id AND status = 'active') ASC
            LIMIT 1
        """
        result = db.execute(query, district, state)
        return Doctor(**result) if result else None
    
    def create_video_room(self, consultation_id: str) -> VideoRoom:
        # Create Twilio video room
        room = self.client.video.rooms.create(
            unique_name=f"consultation-{consultation_id}",
            type="group",
            max_participants=3,  # ASHA, Doctor, optional supervisor
            record_participants_on_connect=True,
            status_callback=f"https://api.mediconnect.ai/webhooks/video-status"
        )
        
        return VideoRoom(
            room_id=room.sid,
            room_name=room.unique_name,
            status=room.status
        )
    
    def generate_access_token(self, user_id: str, room_name: str) -> str:
        # Generate JWT access token for video room
        token = AccessToken(
            self.client.account_sid,
            self.api_key,
            self.api_secret,
            identity=user_id
        )
        
        video_grant = VideoGrant(room=room_name)
        token.add_grant(video_grant)
        
        return token.to_jwt()
    
    async def generate_case_summary(self, assessment: Assessment, prediction: DiagnosisResult) -> str:
        # Use Groq Llama for case summary
        prompt = f"""Generate a concise case summary for a doctor:

Patient: {assessment.patient.age}yo {assessment.patient.gender}
Symptoms: {', '.join([f"{s.name} ({s.duration})" for s in assessment.symptoms])}
AI Diagnosis: {prediction.top_prediction.disease} ({prediction.top_prediction.confidence:.1%} confidence)
Risk Level: {prediction.risk_level}

Format as: Chief Complaint, History, AI Assessment, Recommended Actions"""
        
        response = await groq_client.chat.completions.create(
            model="llama-3.1-8b-instant",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.3,
            max_tokens=500
        )
        
        return response.choices[0].message.content
```

### Clinical Documentation Component

**Interface:**
```python
class ClinicalDocumentationSystem:
    def generate_soap_notes(self, transcription: str) -> SOAPNotes:
        """Generate structured SOAP notes from consultation transcription"""
        pass
    
    def extract_icd10_codes(self, diagnosis: str) -> List[str]:
        """Map diagnosis to ICD-10 codes"""
        pass
    
    def extract_prescription(self, transcription: str) -> Prescription:
        """Extract medications, dosages, frequency from transcription"""
        pass
    
    def check_drug_interactions(self, medications: List[Medication]) -> List[Interaction]:
        """Check for dangerous drug-drug interactions"""
        pass
```

**Implementation:**
```python
class GroqClinicalDocumentationSystem(ClinicalDocumentationSystem):
    def __init__(self, groq_api_key: str):
        self.groq_client = Groq(api_key=groq_api_key)
        self.icd10_db = self._load_icd10_database()
        self.drug_db = self._load_drug_database()
    
    async def generate_soap_notes(self, transcription: str) -> SOAPNotes:
        prompt = f"""You are a medical documentation assistant. Generate structured SOAP notes from this consultation transcript:

{transcription}

Format:
SUBJECTIVE: Patient's reported symptoms and history
OBJECTIVE: Observable findings, vital signs, examination results
ASSESSMENT: Diagnosis and clinical reasoning
PLAN: Treatment plan, medications, follow-up"""
        
        response = await self.groq_client.chat.completions.create(
            model="llama-3.1-8b-instant",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.3,
            max_tokens=2048
        )
        
        # Parse response into structured format
        content = response.choices[0].message.content
        sections = self._parse_soap_sections(content)
        
        return SOAPNotes(
            subjective=sections["SUBJECTIVE"],
            objective=sections["OBJECTIVE"],
            assessment=sections["ASSESSMENT"],
            plan=sections["PLAN"]
        )
    
    def extract_icd10_codes(self, diagnosis: str) -> List[str]:
        # Fuzzy match diagnosis to ICD-10 codes
        diagnosis_lower = diagnosis.lower()
        matches = []
        
        for code, description in self.icd10_db.items():
            if diagnosis_lower in description.lower():
                matches.append((code, self._similarity_score(diagnosis_lower, description.lower())))
        
        # Return top 3 matches
        matches.sort(key=lambda x: x[1], reverse=True)
        return [code for code, score in matches[:3]]
    
    def check_drug_interactions(self, medications: List[Medication]) -> List[Interaction]:
        interactions = []
        
        for i, med1 in enumerate(medications):
            for med2 in medications[i+1:]:
                interaction = self.drug_db.get_interaction(med1.name, med2.name)
                if interaction and interaction.severity in ["moderate", "severe"]:
                    interactions.append(interaction)
        
        return interactions
```

### Offline Sync Component

**Interface:**
```python
class OfflineSyncManager:
    def queue_assessment(self, assessment: Assessment) -> None:
        """Queue assessment for later sync"""
        pass
    
    def sync_queued_data(self) -> SyncResult:
        """Sync all queued data when online"""
        pass
    
    def detect_connectivity(self) -> bool:
        """Check if internet is available"""
        pass
```

**Implementation:**
```python
class IndexedDBSyncManager(OfflineSyncManager):
    def __init__(self, sqs_queue_url: str):
        self.sqs_client = boto3.client('sqs')
        self.queue_url = sqs_queue_url
        self.db_name = "mediconnect_offline"
    
    async def queue_assessment(self, assessment: Assessment) -> None:
        # Store in IndexedDB
        await self._store_in_indexeddb(assessment)
        
        # Try immediate sync if online
        if await self.detect_connectivity():
            await self.sync_queued_data()
    
    async def sync_queued_data(self) -> SyncResult:
        # Get all unsynced assessments from IndexedDB
        queued_assessments = await self._get_queued_assessments()
        
        synced_count = 0
        failed_count = 0
        
        for assessment in queued_assessments:
            try:
                # Send to SQS queue
                self.sqs_client.send_message(
                    QueueUrl=self.queue_url,
                    MessageBody=json.dumps(assessment.dict())
                )
                
                # Mark as synced in IndexedDB
                await self._mark_synced(assessment.id)
                synced_count += 1
                
            except Exception as e:
                logger.error(f"Failed to sync assessment {assessment.id}: {e}")
                failed_count += 1
        
        return SyncResult(
            synced=synced_count,
            failed=failed_count,
            total=len(queued_assessments)
        )
    
    async def detect_connectivity(self) -> bool:
        try:
            response = await httpx.get("https://api.mediconnect.ai/health", timeout=2)
            return response.status_code == 200
        except:
            return False
```

## Data Models

### Core Data Models

```python
from pydantic import BaseModel, Field
from typing import List, Optional
from enum import Enum
from datetime import datetime

class RiskLevel(str, Enum):
    EMERGENCY = "EMERGENCY"
    URGENT = "URGENT"
    ROUTINE = "ROUTINE"

class UserRole(str, Enum):
    ASHA = "asha"
    DOCTOR = "doctor"
    HEALTH_OFFICER = "health_officer"
    ADMIN = "admin"

class User(BaseModel):
    id: str
    phone: str
    role: UserRole
    name: str
    language: str = "en"
    district: Optional[str]
    state: Optional[str]
    status: str = "offline"  # online, offline, busy
    created_at: datetime

class Patient(BaseModel):
    id: str
    age: int
    gender: str
    district: str
    state: str
    created_by: str  # ASHA user_id
    created_at: datetime

class Symptom(BaseModel):
    name: str
    duration: Optional[str]  # "3 days", "2 weeks"
    severity: Optional[str]  # "mild", "moderate", "severe"
    location: Optional[str]  # "chest", "abdomen"

class Assessment(BaseModel):
    id: str
    patient_id: str
    asha_id: str
    symptoms: List[Symptom]
    voice_note_url: Optional[str]
    image_url: Optional[str]
    transcription: Optional[str]
    language: str
    created_at: datetime
    is_offline: bool = False
    synced_at: Optional[datetime]

class Prediction(BaseModel):
    disease: str
    confidence: float  # 0.0 to 1.0
    model_breakdown: dict  # {"biobert": 0.85, "pubmedbert": 0.78, "setfit": 0.72}

class DiagnosisResult(BaseModel):
    top_prediction: Prediction
    top_3: List[Prediction]
    ensemble_confidence: float
    risk_level: RiskLevel
    treatment_plan: str
    nearest_hospitals: List[dict]

class SOAPNotes(BaseModel):
    subjective: str
    objective: str
    assessment: str
    plan: str

class Medication(BaseModel):
    name: str
    dosage: str  # "500mg"
    frequency: str  # "TID" (3 times daily)
    duration: str  # "5 days"
    instructions: Optional[str]

class Prescription(BaseModel):
    medications: List[Medication]
    doctor_id: str
    patient_id: str
    created_at: datetime

class Consultation(BaseModel):
    id: str
    assessment_id: str
    asha_id: str
    doctor_id: str
    patient_id: str
    video_room_id: str
    case_summary: str
    started_at: datetime
    ended_at: Optional[datetime]
    duration_seconds: Optional[int]
    status: str  # pending, active, completed, cancelled
```

Now I need to perform the prework analysis before writing correctness properties:


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Voice Processing Properties

**Property 1: Voice transcription completes within time bound**
*For any* valid audio file in supported languages (Hindi, English, Tamil, Telugu, Bengali, Marathi, Gujarati, Kannada, Malayalam, Punjabi), transcription should complete within 2 seconds and return non-empty text.
**Validates: Requirements 1.1**

**Property 2: Offline voice recordings are queued and synced**
*For any* voice recording made while offline, it should appear in the local queue, and when connectivity is restored, it should be synced to the server with zero data loss.
**Validates: Requirements 1.4, 4.5, 4.6, 27.5**

### AI Diagnosis Properties

**Property 3: Diagnosis completes within time bound**
*For any* valid symptom input, the AI diagnosis engine should generate predictions within 3 seconds.
**Validates: Requirements 2.1, 21.2**

**Property 4: Ensemble voting produces structured output**
*For any* symptom input, the diagnosis result should contain predictions from all three models (BioBERT, PubMed-BERT, SetFit), exactly 3 top predictions ranked by confidence, and all confidence scores should be between 0 and 1.
**Validates: Requirements 2.2, 2.4**

**Property 5: Diagnosis includes treatment information**
*For any* valid diagnosis result, the system should return non-empty treatment protocols including medication names, dosages, and home care instructions.
**Validates: Requirements 2.5, 2.6**

### Risk Stratification Properties

**Property 6: Risk level is always assigned**
*For any* symptom input, the risk stratifier should categorize the case as exactly one of EMERGENCY, URGENT, or ROUTINE.
**Validates: Requirements 3.1**

**Property 7: Emergency keywords trigger emergency classification**
*For any* symptom input containing emergency keywords (chest pain, difficulty breathing, severe bleeding, unconsciousness, seizure, stroke symptoms), the risk level must be EMERGENCY.
**Validates: Requirements 3.2**

**Property 8: Risk level determines response content**
*For any* diagnosis result, if risk level is EMERGENCY then response contains ambulance contacts and hospital locations, if URGENT then response contains PHC visit recommendation, if ROUTINE then response contains home care instructions.
**Validates: Requirements 3.4, 3.5, 3.6**

**Property 9: Multiple symptoms use highest risk level**
*For any* case with multiple symptoms where at least one symptom has EMERGENCY risk, the overall risk level should be EMERGENCY (highest risk wins).
**Validates: Requirements 3.7**

### Offline Mode Properties

**Property 10: Offline mode uses local inference**
*For any* diagnosis request when internet connectivity is unavailable, the system should perform inference using cached TinyBERT model without making network calls and return results.
**Validates: Requirements 4.3**

**Property 11: Offline assessments are queued**
*For any* assessment made while offline, it should be stored in the local queue with synced=false flag.
**Validates: Requirements 4.4**

**Property 12: Offline mode provides treatment for cached diseases**
*For any* disease in the top 50 cached diseases, the offline system should return treatment recommendations without network access.
**Validates: Requirements 4.7**

### Image Analysis Properties

**Property 13: Image analysis returns structured results**
*For any* valid patient image, the analysis should return results containing anemia confidence score and jaundice confidence score, both between 0 and 1, within 5 seconds.
**Validates: Requirements 5.1, 5.3, 5.6**

**Property 14: High confidence detections include severity and recommendations**
*For any* image where anemia confidence > 0.7, the result should include severity level (mild/moderate/severe), and for any image where jaundice confidence > 0.7, the result should include medical evaluation recommendation.
**Validates: Requirements 5.4, 5.5**

**Property 15: Low quality images trigger re-capture request**
*For any* image with quality score < 0.5, the system should return a request for clearer photo with guidance on lighting and angle.
**Validates: Requirements 5.7**

### Hospital Locator Properties

**Property 16: Hospital search respects radius constraint**
*For any* location query, all returned hospitals should be within 5km radius, or if no hospitals found within 5km, search should expand to 10km with user notification.
**Validates: Requirements 6.1, 6.4**

**Property 17: Hospital results contain required fields**
*For any* hospital in search results, it should contain facility name, distance, contact number, available services, and facility type (PHC/district hospital/private clinic).
**Validates: Requirements 6.2, 6.7**

**Property 18: Hospital results are sorted by distance**
*For any* hospital search with multiple results, hospitals should be sorted by distance in ascending order.
**Validates: Requirements 6.3**

**Property 19: Selected hospital provides navigation**
*For any* selected hospital, the response should include navigation data (coordinates, directions, or map URL).
**Validates: Requirements 6.6**

### Clinical Documentation Properties

**Property 20: Real-time transcription has bounded latency**
*For any* speech input during consultation, transcription should appear within 2 seconds.
**Validates: Requirements 8.1, 21.3**

**Property 21: SOAP notes contain all required sections**
*For any* generated SOAP notes, they should contain non-empty Subjective, Objective, Assessment, and Plan sections, generated within 10 seconds of transcription completion.
**Validates: Requirements 8.2, 8.4**

**Property 22: Multi-lingual input produces English output**
*For any* consultation transcription in Hindi, Hinglish, or other supported languages, the generated SOAP notes should be in English.
**Validates: Requirements 8.5**

### Medical Coding Properties

**Property 23: Diagnosis generates ICD-10 codes**
*For any* documented diagnosis, the system should return at least one valid ICD-10 code.
**Validates: Requirements 9.1**

**Property 24: Multiple conditions generate multiple codes**
*For any* diagnosis with N distinct conditions, the system should return N ICD-10 codes, with primary and secondary labels.
**Validates: Requirements 9.2, 9.3**

**Property 25: Low confidence diagnoses provide alternatives**
*For any* diagnosis with confidence < 0.7, the system should return 3 ICD-10 code suggestions with confidence scores.
**Validates: Requirements 9.4**

### Prescription Properties

**Property 26: Voice prescription produces structured output**
*For any* medication dictation, the system should return a structured Prescription object containing medications list with drug names, dosages, frequency, and duration.
**Validates: Requirements 10.1**

**Property 27: Drug interaction checking is performed**
*For any* prescription with 2 or more medications, the system should check for drug-drug interactions and return warnings for moderate or severe interactions.
**Validates: Requirements 10.2, 10.3**

**Property 28: Allergy contraindications are enforced**
*For any* prescription where a medication matches patient documented allergies, the system should reject the medication or return a severe warning.
**Validates: Requirements 10.4**

**Property 29: Dosage limits are validated**
*For any* medication with dosage exceeding the drug database max_safe_dosage, the system should return an alert with recommended dosage range.
**Validates: Requirements 10.5**

**Property 30: Prescription includes required metadata**
*For any* completed prescription, it should include doctor ID, patient ID, timestamp, and formatted document with doctor signature.
**Validates: Requirements 10.7**

### Telemedicine Properties

**Property 31: Emergency cases trigger doctor search**
*For any* case classified as EMERGENCY, the telemedicine bridge should automatically initiate doctor search.
**Validates: Requirements 13.1**

**Property 32: Doctor search prioritizes proximity**
*For any* doctor search result list with multiple available doctors, doctors from the same district should appear before doctors from the same state, who should appear before doctors from other states.
**Validates: Requirements 13.2**

**Property 33: Doctor connection has bounded latency**
*For any* successful doctor match, video room creation and connection should complete within 10 seconds.
**Validates: Requirements 13.3**

**Property 34: Unavailable doctors trigger queueing**
*For any* doctor search returning zero available doctors, the response should contain queue position and estimated wait time.
**Validates: Requirements 13.4**

### Epidemic Surveillance Properties

**Property 35: Assessments are aggregated with anonymization**
*For any* completed assessment, it should appear in aggregated surveillance data within 5 minutes, with all PII removed (no name, phone, exact address) but retaining symptom type, disease prediction, district, state, and timestamp.
**Validates: Requirements 17.1, 17.2, 17.3, 17.4**

**Property 36: Aggregated data is accessible via API**
*For any* district and date range, the surveillance API should return aggregated case counts by disease.
**Validates: Requirements 17.5**

**Property 37: Data access creates audit logs**
*For any* access to patient records, surveillance data, or sensitive endpoints, an audit log entry should be created containing user ID, resource accessed, timestamp, and action.
**Validates: Requirements 17.7, 24.7, 25.3**

**Property 38: Case threshold triggers outbreak alerts**
*For any* district where 12 or more cases of the same disease are reported within 48 hours, an outbreak alert should be created and district health officers notified within 5 minutes.
**Validates: Requirements 18.2, 18.3**

**Property 39: Outbreak detection uses historical baselines**
*For any* outbreak detection analysis, the system should query historical data for the same season (same month in previous years) for baseline comparison.
**Validates: Requirements 18.5**

**Property 40: Multi-district outbreaks escalate to state level**
*For any* outbreak affecting 3 or more districts in the same state, state-level alerts should be created in addition to district alerts.
**Validates: Requirements 18.6**

**Property 41: Outbreak alerts include confidence scores**
*For any* outbreak alert, it should contain a confidence score between 0 and 1 indicating prediction certainty.
**Validates: Requirements 18.7**

### Performance Properties

**Property 42: System operations meet latency requirements**
*For any* API request, it should complete within 200ms (95th percentile), voice transcription within 2s, AI prediction within 3s, image analysis within 5s, and simple database queries within 100ms.
**Validates: Requirements 21.1, 21.2, 21.3, 21.4, 21.5**

### Authentication and Authorization Properties

**Property 43: Login generates JWT with expiration**
*For any* successful login, the system should return a JWT token with exp claim set to 24 hours from issuance time.
**Validates: Requirements 24.1**

**Property 44: JWT contains role information**
*For any* authenticated user, the JWT token should contain a role claim with value asha, doctor, health_officer, or admin.
**Validates: Requirements 24.2**

**Property 45: Role-based access control is enforced**
*For any* API request, ASHA users should only access diagnosis/telemedicine endpoints, doctors should access clinical documentation/telemedicine/patient records, health officers should only access surveillance dashboards, and admins should access all endpoints. Unauthorized access should return 401/403 and create audit log.
**Validates: Requirements 24.3, 24.4, 24.5, 24.6, 24.7**

### Security Properties

**Property 46: Stored data is encrypted**
*For any* data written to database or object storage, it should be encrypted with AES-256, and reading without decryption should return unreadable ciphertext.
**Validates: Requirements 25.1**

**Property 47: Passwords are hashed with bcrypt**
*For any* stored password, it should be bcrypt hashed with cost factor >= 12, and the stored value should not match the plaintext password.
**Validates: Requirements 25.6**

**Property 48: Rate limiting prevents brute force**
*For any* endpoint, after 5 failed authentication attempts within 15 minutes, subsequent requests from the same IP should be rate limited (429 response).
**Validates: Requirements 25.7**

### Offline Sync Properties

**Property 49: Connectivity restoration triggers auto-sync**
*For any* transition from offline to online state, the system should automatically detect connectivity and initiate sync within 5 seconds.
**Validates: Requirements 27.1**

**Property 50: Sync status is exposed to users**
*For any* ongoing sync operation, the API should return sync status including items_pending, items_synced, and sync_progress percentage.
**Validates: Requirements 27.3**

**Property 51: Sync conflicts use server-side resolution**
*For any* sync conflict (same resource modified offline and online), the server version should be preserved and a conflict notification should be generated for the user.
**Validates: Requirements 27.4**

**Property 52: Failed sync retries with exponential backoff**
*For any* failed sync attempt, subsequent retries should have delays following exponential backoff pattern (1s, 2s, 4s, 8s, 16s, max 60s).
**Validates: Requirements 27.6**

**Property 53: Sync completion updates local cache**
*For any* completed sync operation, the local cache should be updated with latest server data and a completion notification should be sent to the user.
**Validates: Requirements 27.7**

## Error Handling

### Error Categories

**1. User Input Errors (4xx)**
- Invalid symptom text (empty, too long, non-medical content)
- Invalid audio format or corrupted file
- Invalid image format or size
- Missing required fields
- Response: 400 Bad Request with descriptive error message

**2. Authentication/Authorization Errors (401/403)**
- Missing or expired JWT token
- Invalid credentials
- Insufficient permissions for requested resource
- Response: 401 Unauthorized or 403 Forbidden with error details

**3. Resource Not Found Errors (404)**
- Patient ID not found
- Assessment ID not found
- Doctor not available
- Hospital not found in radius
- Response: 404 Not Found with suggestions

**4. Rate Limiting Errors (429)**
- Too many requests from same user/IP
- API quota exceeded (Groq free tier limit)
- Response: 429 Too Many Requests with retry-after header

**5. External Service Errors (502/503)**
- Groq API unavailable
- AWS services down
- Twilio API errors
- Google Places API errors
- Response: 502 Bad Gateway or 503 Service Unavailable with fallback options

**6. Internal Server Errors (500)**
- Model inference failures
- Database connection errors
- Unexpected exceptions
- Response: 500 Internal Server Error with error ID for tracking

### Error Handling Strategies

**Graceful Degradation:**
- If Groq Whisper fails → Fallback to AWS Transcribe Medical
- If AWS Bedrock fails → Fallback to local ResNet model
- If primary database fails → Failover to read replica
- If real-time sync fails → Queue for later retry

**Retry Logic:**
- Transient errors: Retry 3 times with exponential backoff
- Network errors: Retry with jitter to avoid thundering herd
- Rate limit errors: Wait for retry-after duration

**Circuit Breaker:**
- If external service fails 5 times in 60 seconds → Open circuit for 5 minutes
- During open circuit → Use cached data or fallback service
- After 5 minutes → Half-open state, try one request
- If successful → Close circuit, resume normal operation

**Logging and Monitoring:**
- All errors logged to CloudWatch with context (user_id, request_id, stack trace)
- Critical errors trigger alerts (PagerDuty, Slack)
- Error rates tracked in dashboard
- Errors categorized by type for analysis

### Specific Error Scenarios

**Scenario 1: Voice Transcription Fails**
```python
try:
    transcription = await groq_whisper.transcribe(audio_file, language)
except GroqAPIError as e:
    logger.warning(f"Groq Whisper failed: {e}, falling back to AWS Transcribe")
    transcription = await aws_transcribe.transcribe(audio_file, language)
except Exception as e:
    logger.error(f"All transcription services failed: {e}")
    return ErrorResponse(
        status_code=503,
        message="Voice transcription temporarily unavailable. Please try again or type symptoms.",
        error_id=generate_error_id()
    )
```

**Scenario 2: No Doctors Available for Emergency**
```python
available_doctor = await telemedicine.find_available_doctor(district, state)
if not available_doctor:
    # Queue the request
    queue_position = await telemedicine.queue_request(consultation_id)
    
    # Send urgent notification to all doctors in state
    await notification_service.broadcast_emergency_alert(state, case_summary)
    
    return {
        "status": "queued",
        "queue_position": queue_position,
        "estimated_wait_minutes": queue_position * 5,
        "message": "No doctors currently available. Your request has been queued and doctors have been notified."
    }
```

**Scenario 3: Offline Sync Conflict**
```python
async def sync_assessment(local_assessment, server_assessment):
    if local_assessment.updated_at > server_assessment.updated_at:
        # Conflict: both modified
        logger.info(f"Sync conflict for assessment {local_assessment.id}")
        
        # Server wins
        await local_db.update(server_assessment)
        
        # Notify user
        await notification_service.send(
            user_id=local_assessment.asha_id,
            message=f"Assessment {local_assessment.id} was modified on another device. Server version has been kept.",
            type="conflict_resolution"
        )
        
        return SyncResult(status="conflict_resolved", action="server_wins")
```

## Testing Strategy

### Dual Testing Approach

MediConnect AI requires both unit testing and property-based testing for comprehensive coverage:

**Unit Tests:**
- Specific examples demonstrating correct behavior
- Edge cases (empty input, boundary values, special characters)
- Error conditions (invalid input, service failures, timeouts)
- Integration points between components
- Mock external services (Groq, AWS, Twilio)

**Property-Based Tests:**
- Universal properties holding for all inputs
- Randomized input generation (symptoms, audio files, images)
- Comprehensive coverage through 100+ iterations per property
- Catch edge cases not anticipated in unit tests
- Validate correctness properties from design document

### Property-Based Testing Configuration

**Framework Selection:**
- Python: Hypothesis (mature, well-documented)
- JavaScript/TypeScript: fast-check (QuickCheck port)

**Test Configuration:**
```python
from hypothesis import given, settings, strategies as st

@settings(max_examples=100, deadline=5000)  # 100 iterations, 5s timeout
@given(
    symptoms=st.lists(
        st.builds(Symptom,
            name=st.sampled_from(["fever", "cough", "headache", "nausea"]),
            duration=st.sampled_from(["1 day", "3 days", "1 week"]),
            severity=st.sampled_from(["mild", "moderate", "severe"])
        ),
        min_size=1,
        max_size=5
    )
)
def test_property_6_risk_level_always_assigned(symptoms):
    """
    Feature: mediconnect-ai, Property 6: Risk level is always assigned
    For any symptom input, risk stratifier should categorize as EMERGENCY, URGENT, or ROUTINE
    """
    result = diagnosis_engine.predict(symptoms, PatientContext(age=30, gender="M"))
    assert result.risk_level in [RiskLevel.EMERGENCY, RiskLevel.URGENT, RiskLevel.ROUTINE]
```

**Property Test Tags:**
Each property test must include a comment referencing the design document:
```python
# Feature: mediconnect-ai, Property 6: Risk level is always assigned
```

### Test Coverage Requirements

**Unit Test Coverage:**
- Minimum 80% code coverage
- 100% coverage for critical paths (emergency detection, prescription safety)
- All error handling branches tested

**Property Test Coverage:**
- All 53 correctness properties implemented as property tests
- Minimum 100 iterations per property
- Properties tested against both online and offline modes

### Testing Pyramid

```
        /\
       /  \
      / E2E \          10 tests - Critical user journeys
     /______\
    /        \
   / Integration\     50 tests - Component interactions
  /____________\
 /              \
/   Unit Tests   \   200 tests - Individual functions
/________________\

/  Property Tests \   53 tests - Universal properties (100 iterations each)
/__________________\
```

### Critical Test Scenarios

**1. ASHA Diagnosis Flow (E2E)**
```
1. ASHA sends voice note in Hindi via WhatsApp
2. System transcribes to English
3. AI generates diagnosis with 87% confidence
4. Risk classified as ROUTINE
5. Treatment recommendations sent via WhatsApp
6. Assessment stored in database
7. Data aggregated for surveillance (anonymized)
```

**2. Emergency Telemedicine Flow (E2E)**
```
1. ASHA inputs "chest pain, difficulty breathing"
2. System classifies as EMERGENCY
3. Automatically searches for available doctor
4. Creates Twilio video room
5. Sends case summary to doctor
6. Doctor joins video call
7. Real-time transcription during consultation
8. SOAP notes generated automatically
9. Prescription sent to ASHA via WhatsApp
```

**3. Offline Mode Flow (E2E)**
```
1. ASHA goes offline (airplane mode)
2. Performs 3 assessments using cached models
3. Assessments queued in IndexedDB
4. ASHA comes back online
5. System detects connectivity
6. Syncs all 3 assessments to server
7. Surveillance data updated
8. Local cache refreshed
```

### Performance Testing

**Load Testing:**
- Simulate 10,000 concurrent users
- 100,000 assessments per day
- Measure p50, p95, p99 latencies
- Identify bottlenecks

**Stress Testing:**
- Gradually increase load until system breaks
- Identify breaking point
- Verify graceful degradation

**Spike Testing:**
- Sudden traffic spike (disease outbreak scenario)
- Verify auto-scaling works
- Measure recovery time

### Security Testing

**Penetration Testing:**
- SQL injection attempts
- XSS attacks
- CSRF attacks
- JWT token manipulation
- Rate limit bypass attempts

**Compliance Testing:**
- DPDP Act 2023 requirements
- ABDM standards
- HIPAA-aligned practices
- Data anonymization verification

### Monitoring and Observability

**Metrics to Track:**
- API latency (p50, p95, p99)
- Error rates by endpoint
- Model inference time
- Database query time
- External service latency (Groq, AWS, Twilio)
- Active users (ASHA, doctors)
- Assessments per hour
- Telemedicine sessions per hour
- Offline sync queue length

**Alerts:**
- API latency > 500ms (p95)
- Error rate > 1%
- Model accuracy < 85%
- Emergency detection sensitivity < 99%
- Database connection pool exhausted
- External service failures
- Disk space > 80%
- Memory usage > 85%

**Dashboards:**
- Executive dashboard (users, assessments, revenue)
- Operations dashboard (latency, errors, uptime)
- Clinical dashboard (accuracy, emergency detection, telemedicine)
- Surveillance dashboard (disease trends, outbreaks, forecasts)
