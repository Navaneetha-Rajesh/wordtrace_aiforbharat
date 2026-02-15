# Design Document: WordTrace

## Overview

WordTrace is an AI-assisted writing transparency platform built on a modular, event-driven architecture. The system consists of a React-based frontend providing a rich text editing experience, a FastAPI backend orchestrating AI agents, and a PostgreSQL database for persistent storage. Four specialized AI agents collaborate to track writing activity, validate citations, analyze authenticity signals, and generate evidence reports.

The architecture prioritizes privacy, transparency, and scalability. Writing events are captured as structured metadata (not raw keystrokes), processed asynchronously through an event queue, and stored with encryption. The system uses lightweight pre-trained NLP models (sentence-transformers) for similarity analysis, avoiding heavy model training while maintaining fast response times.

Key design principles:
- **Privacy-first**: No keystroke surveillance, only structured activity metadata
- **Transparency**: All AI insights are explainable and non-punitive
- **Modularity**: Agents operate independently through event-driven communication
- **Scalability**: Asynchronous processing and database optimization for concurrent users
- **Hackathon-ready**: Lightweight dependencies, container-based deployment, minimal infrastructure

## Architecture

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Frontend Layer                          │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐ │
│  │  Rich Text       │  │  Timeline        │  │  Dashboard   │ │
│  │  Editor          │  │  Replay UI       │  │  (Educator)  │ │
│  └──────────────────┘  └──────────────────┘  └──────────────┘ │
│                    React + TypeScript                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ REST API / WebSocket
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Backend Layer                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    FastAPI Server                        │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────────┐ │  │
│  │  │  Auth      │  │  Document  │  │  Report            │ │  │
│  │  │  Service   │  │  Service   │  │  Service           │ │  │
│  │  └────────────┘  └────────────┘  └────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              │ Event Queue                      │
│                              ▼                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    AI Agent Layer                        │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │  │
│  │  │ Observer │  │ Citation │  │ Reasoning│  │Evidence │ │  │
│  │  │ Agent    │  │ Agent    │  │ Agent    │  │ Agent   │ │  │
│  │  └──────────┘  └──────────┘  └──────────┘  └─────────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
│                    Python + sentence-transformers               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ SQL Queries
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Data Layer                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    PostgreSQL Database                   │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────────┐ │  │
│  │  │  Users     │  │  Documents │  │  Writing_Events    │ │  │
│  │  │  Sessions  │  │  Citations │  │  Reports           │ │  │
│  │  └────────────┘  └────────────┘  └────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
│                    Encrypted at Rest                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ External API Calls
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      External Services                          │
│  ┌──────────────────┐  ┌──────────────────┐                    │
│  │  Crossref API    │  │  Semantic Scholar│                    │
│  │  (DOI Lookup)    │  │  API             │                    │
│  └──────────────────┘  └──────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
```

### Event-Driven Agent Workflow

```
Student Types in Editor
         │
         ▼
Frontend Captures Event
         │
         ▼
POST /api/events ──────────────────┐
         │                         │
         ▼                         │
Observer_Agent Receives Event      │
         │                         │
         ▼                         │
Validate & Enrich Metadata         │
         │                         │
         ▼                         │
Publish to Event Queue ────────────┤
         │                         │
         ▼                         │
Store in Database                  │
                                    │
         ┌──────────────────────────┘
         │
         ├─────────────────────────────────────┐
         │                                     │
         ▼                                     ▼
Citation_Agent Subscribes          Reasoning_Agent Subscribes
(if citation event)                (for periodic analysis)
         │                                     │
         ▼                                     ▼
Query Scholarly APIs               Calculate Similarity Scores
         │                                     │
         ▼                                     ▼
Store Validation Result            Store Authenticity Signals
         │                                     │
         └─────────────┬───────────────────────┘
                       │
                       ▼
         Evidence_Agent Triggered
         (on report request)
                       │
                       ▼
         Aggregate All Data
                       │
                       ▼
         Generate Visualizations
                       │
                       ▼
         Compile PDF Report
                       │
                       ▼
         Return to Student/Educator
```

## Components and Interfaces

### Frontend Components

#### 1. Rich Text Editor Component
**Technology**: React + Draft.js or Slate.js

**Responsibilities**:
- Provide WYSIWYG editing interface
- Capture writing events (insert, delete, paste, format)
- Debounce event transmission (batch events every 2 seconds)
- Display inline citation validation status
- Handle auto-save functionality

**Key Interfaces**:
```typescript
interface WritingEvent {
  eventId: string;
  sessionId: string;
  eventType: 'insert' | 'delete' | 'paste' | 'format' | 'revision';
  timestamp: number;
  contentRange: { start: number; end: number };
  contentLength: number;
  contentSnapshot?: string;
}
```

#### 2. Timeline Replay Component
**Technology**: React + D3.js

**Responsibilities**:
- Fetch writing events from backend
- Render chronological content evolution
- Provide playback controls (play, pause, speed)
- Highlight additions and deletions with color coding

#### 3. Contribution Heatmap Component
**Technology**: React + react-calendar-heatmap

**Responsibilities**:
- Display activity intensity over calendar grid
- Show tooltips with session details
- Allow drill-down into specific dates

#### 4. Educator Dashboard Component
**Technology**: React + Material-UI

**Responsibilities**:
- List student submissions
- Display summary statistics
- Filter and search functionality
- Navigate to detailed reports

### Backend Services

#### 1. Document Service
**Technology**: FastAPI + SQLAlchemy

**Key Endpoints**:
```python
POST   /api/documents                    # Create new document
GET    /api/documents/{id}               # Get document content
PUT    /api/documents/{id}               # Update document
DELETE /api/documents/{id}               # Delete document
POST   /api/documents/{id}/sessions      # Start new session
PUT    /api/documents/{id}/sessions/{sid} # End session
POST   /api/events                       # Ingest writing events (batch)
GET    /api/documents/{id}/events        # Get all events for document
GET    /api/documents/{id}/heatmap       # Get aggregated heatmap data
```

#### 2. Report Service
**Technology**: FastAPI + ReportLab

**Key Endpoints**:
```python
POST /api/reports/generate/{document_id}  # Trigger report generation
GET  /api/reports/{report_id}            # Get report status
GET  /api/reports/{report_id}/download   # Download PDF
GET  /api/reports/{report_id}/export     # Download ZIP bundle
```

#### 3. Auth Service
**Technology**: FastAPI + JWT

**Key Endpoints**:
```python
POST /api/auth/register  # Register new user
POST /api/auth/login     # Login and get JWT
POST /api/auth/refresh   # Refresh JWT token
GET  /api/auth/me        # Get current user info
```

### AI Agent Layer

#### 1. Observer Agent
**Technology**: Python + asyncio

**Responsibilities**:
- Receive writing events from Document Service
- Validate event structure and metadata
- Enrich events with derived metrics (words added, words deleted)
- Publish to event queue for other agents
- Persist events to database

#### 2. Citation Agent
**Technology**: Python + httpx (async HTTP client)

**Responsibilities**:
- Subscribe to citation-related events
- Extract citation metadata (title, author, DOI, URL)
- Query external scholarly APIs (Crossref, Semantic Scholar)
- Store validation results with confidence scores
- Retry failed validations with exponential backoff

**External API Integration**:
- **Crossref API**: Free, no authentication required, rate limit 50 req/sec
- **Semantic Scholar API**: Free, requires API key, rate limit 100 req/sec
- **Fallback**: If both fail, mark as "unverified" with reason

#### 3. Reasoning Agent
**Technology**: Python + sentence-transformers

**Responsibilities**:
- Analyze writing patterns for authenticity signals
- Calculate similarity scores between consecutive revisions
- Detect sudden large content additions (potential paste from external source)
- Identify gradual evolution patterns (positive signal)
- Aggregate signals into confidence summary
- Generate explainable insights (no binary judgments)

**NLP Model**:
- Model: `sentence-transformers/all-MiniLM-L6-v2`
- Size: ~80MB, fast inference (~50ms for 500 words)
- Output: 384-dimensional embeddings
- Similarity: Cosine similarity between embeddings

**Authenticity Signals**:
1. **Gradual Evolution**: High similarity (>0.8) between consecutive revisions indicates incremental writing
2. **Sudden Addition**: Large paste (>200 words) with low similarity to previous content flags potential external source
3. **Revision Frequency**: Multiple revisions to same section indicates thoughtful editing
4. **Session Distribution**: Writing spread across multiple sessions indicates sustained effort

#### 4. Evidence Agent
**Technology**: Python + ReportLab + Matplotlib

**Responsibilities**:
- Coordinate report generation
- Query all relevant data (events, citations, signals)
- Generate timeline visualization
- Generate heatmap visualization
- Compile PDF report with all sections
- Calculate summary statistics

**Report Structure**:
1. **Cover Page**: Document title, student name, submission date
2. **Summary Statistics**: Total time, session count, word count, revision count
3. **Timeline Visualization**: Key revision points with timestamps
4. **Contribution Heatmap**: Activity calendar with intensity levels
5. **Citation Validation**: List of citations with validation status
6. **Authenticity Insights**: AI-generated signals with explanations
7. **Appendix**: Raw activity metadata (JSON)

## Data Models

### Database Schema

```sql
-- Users table
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL CHECK (role IN ('student', 'educator')),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Documents table
CREATE TABLE documents (
    document_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    title VARCHAR(500) NOT NULL,
    content TEXT NOT NULL,
    word_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    submitted_at TIMESTAMP NULL
);

-- Writing sessions table
CREATE TABLE writing_sessions (
    session_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL REFERENCES documents(document_id) ON DELETE CASCADE,
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP NULL,
    duration_minutes INTEGER NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Writing events table
CREATE TABLE writing_events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES writing_sessions(session_id) ON DELETE CASCADE,
    document_id UUID NOT NULL REFERENCES documents(document_id) ON DELETE CASCADE,
    event_type VARCHAR(50) NOT NULL CHECK (event_type IN ('insert', 'delete', 'paste', 'format', 'revision')),
    timestamp TIMESTAMP NOT NULL,
    content_range_start INTEGER NOT NULL,
    content_range_end INTEGER NOT NULL,
    content_length INTEGER NOT NULL,
    content_snapshot TEXT NULL,
    metadata JSONB NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Citations table
CREATE TABLE citations (
    citation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL REFERENCES documents(document_id) ON DELETE CASCADE,
    citation_text TEXT NOT NULL,
    citation_type VARCHAR(50) NULL CHECK (citation_type IN ('paper', 'book', 'web', 'unknown')),
    title VARCHAR(500) NULL,
    author VARCHAR(500) NULL,
    year INTEGER NULL,
    doi VARCHAR(255) NULL,
    url TEXT NULL,
    validation_status VARCHAR(50) NOT NULL CHECK (validation_status IN ('validated', 'unverified', 'failed', 'pending')),
    validation_reason TEXT NULL,
    validated_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Authenticity signals table
CREATE TABLE authenticity_signals (
    signal_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL REFERENCES documents(document_id) ON DELETE CASCADE,
    signal_type VARCHAR(100) NOT NULL,
    signal_value FLOAT NOT NULL,
    signal_description TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Reports table
CREATE TABLE reports (
    report_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL REFERENCES documents(document_id) ON DELETE CASCADE,
    report_status VARCHAR(50) NOT NULL CHECK (report_status IN ('pending', 'generating', 'completed', 'failed')),
    pdf_path VARCHAR(500) NULL,
    generated_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for performance
CREATE INDEX idx_documents_user_id ON documents(user_id);
CREATE INDEX idx_writing_sessions_document_id ON writing_sessions(document_id);
CREATE INDEX idx_writing_events_session_id ON writing_events(session_id);
CREATE INDEX idx_writing_events_document_id ON writing_events(document_id);
CREATE INDEX idx_writing_events_timestamp ON writing_events(timestamp);
CREATE INDEX idx_citations_document_id ON citations(document_id);
CREATE INDEX idx_authenticity_signals_document_id ON authenticity_signals(document_id);
CREATE INDEX idx_reports_document_id ON reports(document_id);
```

## Technology Stack Justification

### Frontend: React + TypeScript
- Component-based architecture for modular UI
- TypeScript adds type safety, reducing runtime errors
- Large ecosystem of libraries (Draft.js, D3.js, Material-UI)
- Excellent developer experience with hot reloading

### Backend: FastAPI (Python)
- High performance with native async/await support
- Automatic API documentation (OpenAPI/Swagger)
- Python ecosystem ideal for AI/NLP integration
- Easy integration with sentence-transformers

### Database: PostgreSQL
- Robust ACID compliance for data integrity
- Excellent JSON support (JSONB) for flexible metadata storage
- Strong indexing capabilities for time-series queries
- Free and open-source

### NLP: sentence-transformers
- Pre-trained models (no training required)
- Lightweight and fast inference (<100ms)
- High-quality semantic similarity
- Easy Python integration

### PDF Generation: ReportLab
- Pure Python library (easy integration)
- Powerful layout control
- Supports images, charts, tables

### Containerization: Docker
- Consistent deployment across environments
- Easy dependency management
- Wide cloud platform support

## Privacy Considerations

### Data Minimization
- NO keystroke logging or raw keystroke data
- Only structured event metadata (type, timestamp, range, length)
- Content snapshots only at revision points, not every keystroke
- No tracking of mouse movements or cursor position

### Access Control
- Role-based access control (RBAC) enforced at API level
- Students can only access their own data
- Educators can only access assigned students
- Database-level foreign key constraints with CASCADE DELETE

### Data Retention
- Students can delete documents and all associated data
- Cascade deletes remove all related records
- Optional soft deletes for audit trail (configurable)

### Encryption
- Encryption at rest for database
- TLS/HTTPS for all API communication
- JWT tokens for authentication (short expiration)
- Secure password hashing (bcrypt)

### Third-Party Data Sharing
- NO sharing of student data with third parties
- Scholarly APIs only receive citation metadata (no student info)
- No analytics or tracking services

## Testing Approach (MVP)

The prototype will be validated through:
- Unit testing of core APIs and agents
- End-to-end workflow testing (write → analyze → generate report)
- Sample classroom-scale load testing
- Manual validation of timeline replay and citation verification

The focus is functional validation rather than exhaustive testing.

## MVP Implementation Plan

During the hackathon, the system will implement:

1. **Writing editor with event capture**
   - Rich text editor component
   - Event batching and transmission
   - Session management

2. **Observer Agent storing writing events**
   - Event validation and enrichment
   - Database persistence
   - Event queue publishing

3. **Basic timeline replay visualization**
   - Chronological event display
   - Playback controls
   - Addition/deletion highlighting

4. **Citation validation using public APIs**
   - Citation detection and parsing
   - Crossref and Semantic Scholar integration
   - Validation status display

5. **Proof-of-Effort report generation**
   - PDF compilation with ReportLab
   - Summary statistics calculation
   - Timeline and heatmap visualizations

6. **Lightweight similarity analysis using pre-trained NLP models**
   - sentence-transformers integration
   - Similarity score calculation
   - Authenticity signal generation

Advanced dashboards, scaling features, and integrations are deferred beyond the prototype.

## Deployment (Prototype)

The system will run using Docker Compose with:
- React frontend (development server)
- FastAPI backend
- PostgreSQL database

This enables quick local deployment for demonstration and testing.

**docker-compose.yml structure**:
```yaml
services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
  
  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/wordtrace
  
  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=wordtrace
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

This design provides a solid foundation for the WordTrace MVP, balancing functionality, performance, privacy, and hackathon constraints.
