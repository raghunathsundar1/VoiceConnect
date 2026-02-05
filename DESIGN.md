# VoiceConnect - System Design Document

## 1. Design Overview

### 1.1 Design Philosophy

VoiceConnect is designed with the following core principles:

1. **Accessibility First**: Every design decision prioritizes users with limited technology access
2. **Privacy by Design**: Minimal data collection, maximum user control
3. **Offline-First**: Core functionality works without internet connectivity
4. **Scalable Architecture**: Cloud-native design for horizontal scaling
5. **Modular Components**: Loosely coupled services for maintainability

### 1.2 Design Goals

- **Simplicity**: 3 steps or less to find any resource
- **Speed**: Sub-2-second response times
- **Reliability**: 99.9% uptime
- **Inclusivity**: Support for 20+ languages and all literacy levels
- **Maintainability**: Clean code, comprehensive tests, clear documentation

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     User Interfaces                          │
├──────────────┬──────────────┬──────────────┬────────────────┤
│ Phone (IVR)  │  SMS Gateway │  PWA (Web)   │  Mobile App    │
│  - Twilio    │  - Twilio    │  - React     │  - React       │
│  - Voice     │  - SMS       │  - Offline   │    Native      │
└──────┬───────┴──────┬───────┴──────┬───────┴────────┬───────┘
       │              │              │                │
       └──────────────┴──────────────┴────────────────┘
                      │
       ┌──────────────▼──────────────┐
       │     API Gateway (FastAPI)    │
       │  - Rate Limiting             │
       │  - Authentication            │
       │  - Load Balancing            │
       │  - Request Routing           │
       └──────────────┬──────────────┘
                      │
       ┌──────────────▼──────────────────────────────┐
       │          Core Services Layer                 │
       ├──────────────┬──────────────┬───────────────┤
       │ Voice Engine │ AI Engine    │ Integration   │
       │ - Whisper    │ - GPT-4      │ - Gov APIs    │
       │ - ElevenLabs │ - LangChain  │ - Community   │
       │ - WebSocket  │ - RAG        │ - Translation │
       │              │ - pgvector   │               │
       └──────┬───────┴──────┬───────┴───────┬───────┘
              │              │               │
       ┌──────▼──────────────▼───────────────▼───────┐
       │          Data Layer                          │
       ├──────────────┬──────────────┬────────────────┤
       │ PostgreSQL   │ Redis Cache  │ Object Storage │
       │ + PostGIS    │ - Sessions   │ - Audio Files  │
       │ + pgvector   │ - Rate Limit │ - Documents    │
       │ - Resources  │ - Queue      │ - Backups      │
       └──────────────┴──────────────┴────────────────┘
```

### 2.2 Component Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Frontend Layer                            │
├─────────────────────────────────────────────────────────────┤
│  React PWA                                                   │
│  ├── Voice Interface Component                              │
│  ├── Resource Search Component                              │
│  ├── Eligibility Checker Component                          │
│  ├── Map Component (Leaflet)                                │
│  ├── Service Worker (Offline Support)                       │
│  └── Redux Store (State Management)                         │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    Backend Layer                             │
├─────────────────────────────────────────────────────────────┤
│  FastAPI Application                                         │
│  ├── Voice Router                                            │
│  │   ├── WebSocket Handler                                  │
│  │   ├── Audio Stream Processor                             │
│  │   └── Session Manager                                    │
│  ├── Resource Router                                         │
│  │   ├── Search Endpoint                                    │
│  │   ├── Details Endpoint                                   │
│  │   └── Category Endpoint                                  │
│  ├── Eligibility Router                                      │
│  │   ├── Check Endpoint                                     │
│  │   ├── Programs Endpoint                                  │
│  │   └── Guidance Endpoint                                  │
│  └── User Router                                             │
│      ├── Profile Endpoint                                   │
│      ├── Saved Resources Endpoint                           │
│      └── History Endpoint                                   │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    Service Layer                             │
├─────────────────────────────────────────────────────────────┤
│  Voice Engine Service                                        │
│  ├── Speech-to-Text (Whisper)                               │
│  ├── Text-to-Speech (ElevenLabs)                            │
│  ├── Audio Processing                                        │
│  └── Language Detection                                      │
│                                                              │
│  AI Engine Service                                           │
│  ├── Intent Recognition (GPT-4)                              │
│  ├── Response Generation (GPT-4)                             │
│  ├── RAG System (LangChain + pgvector)                      │
│  ├── Conversation Memory                                     │
│  └── Context Management                                      │
│                                                              │
│  Resource Matcher Service                                    │
│  ├── Geospatial Search (PostGIS)                            │
│  ├── Filtering Engine                                        │
│  ├── Ranking Algorithm                                       │
│  └── Distance Calculation                                    │
│                                                              │
│  Eligibility Checker Service                                 │
│  ├── Rules Engine                                            │
│  ├── Calculation Logic                                       │
│  ├── Government API Integration                              │
│  └── Document Generator                                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Detailed Component Design

### 3.1 Voice Engine

#### Purpose
Handle all voice-related processing including speech recognition, synthesis, and audio streaming.

#### Components

**Speech-to-Text Module**
- **Technology**: OpenAI Whisper (medium model)
- **Input**: Audio stream (16kHz, mono, PCM)
- **Output**: Transcribed text with confidence scores
- **Features**:
  - Real-time streaming transcription
  - Automatic language detection
  - Noise reduction
  - Accent adaptation

**Text-to-Speech Module**
- **Technology**: ElevenLabs Multilingual v2
- **Input**: Text + language code
- **Output**: Audio stream (MP3, 24kHz)
- **Features**:
  - Natural prosody
  - Emotion control
  - Voice selection by language
  - Speed adjustment

**Audio Processing**
- **Buffer Management**: Circular buffer for streaming
- **Format Conversion**: PCM ↔ MP3 ↔ WAV
- **Quality Enhancement**: Noise reduction, normalization
- **Compression**: Adaptive bitrate for bandwidth

#### Data Flow

```
User Speech → Microphone → WebSocket → Audio Buffer
    ↓
Whisper API → Transcription → Intent Recognition
    ↓
GPT-4 → Response Text → ElevenLabs → Audio
    ↓
WebSocket → Speaker → User Hears Response
```

#### API Design

```python
class VoiceEngine:
    async def create_session(user_id: str, language: str) -> str
    async def transcribe_chunk(audio: bytes, session_id: str) -> str
    async def text_to_speech(text: str, language: str) -> bytes
    async def end_session(session_id: str) -> None
```

### 3.2 AI Engine

#### Purpose
Understand user intent, generate contextual responses, and provide intelligent resource recommendations.

#### Components

**Intent Recognition**
- **Technology**: GPT-4 with function calling
- **Input**: User query + conversation history
- **Output**: Intent classification + extracted entities
- **Intents**:
  - `find_resources`: User wants to find services
  - `check_eligibility`: User wants eligibility info
  - `get_guidance`: User needs application help
  - `general_query`: General conversation

**RAG System**
- **Vector Database**: pgvector extension in PostgreSQL
- **Embeddings**: OpenAI text-embedding-ada-002
- **Retrieval**: Semantic search with cosine similarity
- **Context Window**: Top 5 relevant documents
- **Reranking**: Relevance scoring with cross-encoder

**Response Generation**
- **Technology**: GPT-4 Turbo with streaming
- **Prompt Engineering**: System prompts for each intent
- **Context Injection**: User profile + conversation + retrieved docs
- **Output Formatting**: Structured responses with actions

#### Architecture

```
User Query
    ↓
[Intent Classifier] → Intent + Entities
    ↓
[Context Builder]
    ├── Conversation History
    ├── User Profile
    └── Vector Search (RAG)
    ↓
[Response Generator (GPT-4)]
    ├── System Prompt
    ├── Context
    └── User Query
    ↓
Structured Response
    ├── Text
    ├── Resources
    └── Suggested Actions
```

#### Prompt Templates

```python
INTENT_PROMPT = """
You are a community resource assistant. Classify the user's intent:
- find_resources: Looking for services/programs
- check_eligibility: Wants to know if they qualify
- get_guidance: Needs help with applications
- general_query: Other questions

User: {query}
Intent:
"""

RESPONSE_PROMPT = """
You are a helpful community resource assistant speaking in {language}.
User needs: {intent}
Context: {context}
Resources found: {resources}

Provide a helpful, conversational response that:
1. Acknowledges their need
2. Summarizes relevant resources
3. Explains next steps
4. Asks if they need more help

Response:
"""
```

### 3.3 Resource Matcher

#### Purpose
Find and rank community resources based on location, needs, and eligibility.

#### Geospatial Search Algorithm

```sql
-- Dynamic radius search with service area matching
SELECT 
    r.id,
    r.name,
    r.description,
    r.category,
    ST_Distance(
        r.location::geography,
        ST_SetSRID(ST_MakePoint(:lon, :lat), 4326)::geography
    ) / 1000 as distance_km
FROM resources r
JOIN service_areas sa ON r.id = sa.resource_id
WHERE 
    -- User is within service area
    ST_DWithin(
        r.location::geography,
        ST_SetSRID(ST_MakePoint(:lon, :lat), 4326)::geography,
        sa.radius_km * 1000
    )
    -- Language support
    AND (:language = ANY(r.languages))
    -- Category match
    AND r.category = :category
    -- Active resources only
    AND r.is_active = true
    -- Text search
    AND to_tsvector('english', r.name || ' ' || r.description) 
        @@ plainto_tsquery('english', :query)
ORDER BY distance_km ASC
LIMIT :limit;
```

#### Ranking Algorithm

```python
def rank_resources(resources, user_context):
    """
    Multi-factor ranking algorithm
    """
    for resource in resources:
        score = 0
        
        # Distance factor (closer is better)
        score += (1 / (1 + resource.distance_km)) * 40
        
        # Language match (exact match bonus)
        if user_context.language in resource.languages:
            score += 20
        
        # Hours (open now bonus)
        if is_open_now(resource.hours):
            score += 15
        
        # User ratings
        score += (resource.avg_rating / 5) * 15
        
        # Eligibility match
        if matches_eligibility(resource, user_context):
            score += 10
        
        resource.score = score
    
    return sorted(resources, key=lambda r: r.score, reverse=True)
```

### 3.4 Eligibility Checker

#### Purpose
Determine user eligibility for government programs and benefits.

#### Rules Engine

```python
class EligibilityRules:
    """
    Declarative rules for program eligibility
    """
    
    SNAP = {
        "max_income": {
            1: 18954,  # Household size: income limit
            2: 25520,
            3: 32086,
            4: 38652,
            # ... more sizes
        },
        "citizenship": ["citizen", "permanent_resident", "refugee"],
        "residency_required": True,
    }
    
    MEDICAID = {
        "max_income": {
            1: 20783,
            2: 28207,
            # ... more sizes
        },
        "citizenship": ["citizen", "permanent_resident", "refugee", "asylee"],
        "residency_required": True,
    }
    
    # ... more programs

def check_eligibility(user_info, program):
    """
    Check if user meets program requirements
    """
    rules = EligibilityRules[program]
    result = {
        "eligible": True,
        "reasons": [],
        "missing_info": []
    }
    
    # Income check
    if "max_income" in rules:
        household_size = user_info.get("household_size")
        income = user_info.get("annual_income")
        
        if income is None:
            result["missing_info"].append("annual_income")
        elif income > rules["max_income"][household_size]:
            result["eligible"] = False
            result["reasons"].append("Income exceeds limit")
    
    # Citizenship check
    if "citizenship" in rules:
        status = user_info.get("citizenship_status")
        if status not in rules["citizenship"]:
            result["eligible"] = False
            result["reasons"].append("Citizenship requirement not met")
    
    return result
```

---

## 4. Data Models

### 4.1 Database Schema

#### Resources Table
```sql
CREATE TABLE resources (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    category VARCHAR(100) NOT NULL,
    
    -- Contact
    phone VARCHAR(20),
    email VARCHAR(255),
    website VARCHAR(500),
    
    -- Location (PostGIS)
    location GEOMETRY(POINT, 4326) NOT NULL,
    address VARCHAR(500),
    city VARCHAR(100),
    state VARCHAR(50),
    zip_code VARCHAR(10),
    
    -- Multilingual
    languages TEXT[] DEFAULT ARRAY['en'],
    
    -- Operational
    hours JSONB,
    is_active BOOLEAN DEFAULT true,
    
    -- Eligibility
    eligibility_criteria JSONB,
    required_documents TEXT[],
    application_process TEXT,
    
    -- Metadata
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    verified_at TIMESTAMP,
    
    -- Indexes
    INDEX idx_location USING GIST(location),
    INDEX idx_category (category),
    INDEX idx_search USING GIN(to_tsvector('english', name || ' ' || description))
);
```

#### Service Areas Table
```sql
CREATE TABLE service_areas (
    id SERIAL PRIMARY KEY,
    resource_id INTEGER REFERENCES resources(id),
    radius_km FLOAT NOT NULL
);
```

#### Users Table
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    phone_number VARCHAR(20) UNIQUE,
    email VARCHAR(255) UNIQUE,
    
    -- Demographics (optional)
    age INTEGER,
    household_size INTEGER,
    annual_income FLOAT,
    citizenship_status VARCHAR(50),
    is_resident BOOLEAN,
    
    -- Preferences
    preferred_language VARCHAR(10) DEFAULT 'en',
    location GEOMETRY(POINT, 4326),
    
    -- Privacy
    data_sharing_consent BOOLEAN DEFAULT false,
    
    -- Metadata
    created_at TIMESTAMP DEFAULT NOW(),
    last_active TIMESTAMP DEFAULT NOW()
);
```

### 4.2 API Data Models

#### Resource Response
```typescript
interface Resource {
    id: number;
    name: string;
    description: string;
    category: string;
    phone: string;
    website: string;
    address: string;
    languages: string[];
    hours: {
        [day: string]: string;
    };
    distance_km?: number;
    eligibility_criteria: object;
    required_documents: string[];
}
```

#### Eligibility Result
```typescript
interface EligibilityResult {
    program: string;
    status: 'eligible' | 'not_eligible' | 'need_more_info';
    reasons: string[];
    missing_info: string[];
    next_steps: string[];
}
```

---

## 5. API Design

### 5.1 REST Endpoints

#### Voice API
```
POST   /api/v1/voice/session          # Create voice session
DELETE /api/v1/voice/session/:id      # End voice session
WS     /ws/voice                       # WebSocket for audio streaming
```

#### Resources API
```
GET    /api/v1/resources              # Search resources
GET    /api/v1/resources/:id          # Get resource details
GET    /api/v1/resources/categories   # List categories
POST   /api/v1/resources/:id/save     # Save resource
```

#### Eligibility API
```
POST   /api/v1/eligibility/check      # Check eligibility
GET    /api/v1/eligibility/programs   # List programs
GET    /api/v1/eligibility/:program   # Program details
```

### 5.2 WebSocket Protocol

#### Voice Streaming
```json
// Client → Server (Initialize)
{
    "type": "init",
    "language": "en",
    "user_id": "optional"
}

// Client → Server (Audio chunk)
{
    "type": "audio",
    "data": "<base64_encoded_audio>"
}

// Server → Client (Transcription)
{
    "type": "transcription",
    "text": "I need help finding food"
}

// Server → Client (AI Response)
{
    "type": "response",
    "text": "I can help you find food assistance...",
    "resources": [...],
    "actions": ["view_details", "get_directions"]
}

// Server → Client (Audio response)
{
    "type": "audio",
    "data": "<base64_encoded_audio>"
}
```

---

## 6. Security Design

### 6.1 Authentication Flow

```
User Request
    ↓
[API Gateway]
    ├── Check JWT Token
    ├── Validate Signature
    └── Check Expiration
    ↓
[Rate Limiter]
    ├── Check Request Count
    └── Apply Throttling
    ↓
[Authorization]
    ├── Check User Permissions
    └── Validate Resource Access
    ↓
[Backend Service]
```

### 6.2 Data Encryption

- **In Transit**: TLS 1.3 for all connections
- **At Rest**: AES-256 encryption for sensitive data
- **Passwords**: bcrypt with salt (cost factor 12)
- **API Keys**: Encrypted in database, never logged

### 6.3 Privacy Controls

```python
class PrivacySettings:
    """User privacy controls"""
    
    # Data collection levels
    MINIMAL = "minimal"      # Only essential data
    STANDARD = "standard"    # + usage analytics
    FULL = "full"           # + personalization data
    
    # Data retention
    RETENTION_DAYS = {
        "voice_recordings": 0,      # Never stored
        "transcriptions": 30,       # 30 days
        "search_history": 90,       # 90 days
        "user_profile": 365,        # 1 year
    }
    
    # User rights
    def export_data(user_id):
        """GDPR: Right to data portability"""
        pass
    
    def delete_data(user_id):
        """GDPR: Right to be forgotten"""
        pass
```

---

## 7. Deployment Architecture

### 7.1 Infrastructure

```
┌─────────────────────────────────────────────────────────────┐
│                    Load Balancer (AWS ALB)                   │
└─────────────────────────────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Backend     │  │  Backend     │  │  Backend     │
│  Instance 1  │  │  Instance 2  │  │  Instance 3  │
│  (ECS/K8s)   │  │  (ECS/K8s)   │  │  (ECS/K8s)   │
└──────────────┘  └──────────────┘  └──────────────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  PostgreSQL  │  │  Redis       │  │  S3 Storage  │
│  (RDS)       │  │  (ElastiCache│  │  (Objects)   │
│  + PostGIS   │  │  )           │  │              │
└──────────────┘  └──────────────┘  └──────────────┘
```

### 7.2 Scaling Strategy

**Horizontal Scaling**
- Auto-scaling groups (2-10 instances)
- Scale on CPU > 70% or Memory > 80%
- Scale down after 5 minutes below threshold

**Database Scaling**
- Read replicas for queries
- Connection pooling (PgBouncer)
- Query caching in Redis

**Caching Strategy**
- Redis for session data (TTL: 30 min)
- CDN for static assets (CloudFront)
- Browser caching for PWA resources

---

## 8. Monitoring & Observability

### 8.1 Metrics

**Application Metrics**
- Request rate (requests/second)
- Response time (p50, p95, p99)
- Error rate (%)
- Active users (concurrent)

**Business Metrics**
- Resources searched
- Eligibility checks performed
- Successful connections
- User satisfaction (NPS)

### 8.2 Logging

```python
# Structured logging format
{
    "timestamp": "2026-02-04T10:30:00Z",
    "level": "INFO",
    "service": "voice-engine",
    "user_id": "hashed_user_id",
    "session_id": "session_123",
    "event": "transcription_complete",
    "duration_ms": 1234,
    "language": "es",
    "metadata": {
        "audio_length_sec": 5.2,
        "confidence": 0.95
    }
}
```

### 8.3 Alerting

**Critical Alerts** (PagerDuty)
- API error rate > 5%
- Response time > 5s
- Database connection failures
- Service downtime

**Warning Alerts** (Slack)
- API error rate > 2%
- Response time > 3s
- High memory usage (> 85%)
- Low disk space (< 20%)

---

## 9. Testing Strategy

### 9.1 Test Pyramid

```
        ┌─────────────┐
        │   E2E Tests │  (10%)
        │   - Selenium│
        │   - Cypress │
        └─────────────┘
      ┌─────────────────┐
      │ Integration Tests│  (30%)
      │ - API Tests      │
      │ - DB Tests       │
      └─────────────────┘
    ┌───────────────────────┐
    │    Unit Tests         │  (60%)
    │    - Functions        │
    │    - Components       │
    │    - Services         │
    └───────────────────────┘
```

### 9.2 Test Coverage Goals

- **Unit Tests**: 80%+ coverage
- **Integration Tests**: All API endpoints
- **E2E Tests**: Critical user flows
- **Performance Tests**: Load testing to 2x expected traffic

---

## 10. Future Enhancements

### 10.1 Technical Roadmap

**Q2 2026**
- Native mobile apps (iOS/Android)
- Video call support
- Advanced analytics dashboard

**Q3 2026**
- Machine learning for resource recommendations
- Predictive eligibility scoring
- Multi-tenant architecture

**Q4 2026**
- Blockchain for verified credentials
- Federated learning for privacy
- Edge computing for offline AI

---

**Document Version**: 1.0  
**Last Updated**: February 4, 2026  
**Status**: Approved for Implementation  
**Next Review**: March 2026
