
---
> [BACK TO INDEX](INDEX.md)
---
# NeuroFit Backend — Technical Requirements Document (TRD)

Last updated: 2026-05-20

File: NeuroFit-Backend-TRD.md

Purpose: Enterprise-grade backend architecture blueprint for NeuroFit, a real-time AI neuroscience platform. This TRD is production-ready, implementation-focused, beginner-friendly, and scalable.

---

# Contents

- 1 Executive Summary
- 2 System Architecture
- 3 Technology Decision Matrix
- 4 Backend Folder Architecture
- 5 Database Design
- 6 Authentication & Authorization (Kinde)
- 7 API Design Specification
- 8 EEG Processing Pipeline
- 9 Machine Learning Architecture
- 10 ML Training + Testing Pipeline
- 11 ML API Integration
- 12 WebSocket Architecture
- 13 Background Jobs (Celery vs Dramatiq)
- 14 Testing Strategy
- 15 Security Architecture
- 16 Deployment Guide (FREE)
- 17 CI/CD Pipeline
- 18 Monitoring & Logging
- 19 Development Roadmap
- 20 Cost Estimation
- 21 Appendix & Glossary

---

## 1. Executive Summary

**Project goals**
- Build NeuroFit backend: a production-grade FastAPI application supporting real-time EEG streaming, ML inference, and multi-tenant user management.
- Handle 50k+ users with multi-modal analytics (focus, stress, sleep, training).
- Support wearable integration via BLE gateways and mobile apps.
- Deliver low-latency WebSocket streams (<250ms) for real-time neurofeedback.
- Enable scalable ML inference with model versioning and A/B testing.

**Business vision**
- Democratize EEG analytics and neurofeedback via accessible APIs and UX.
- Monetize via subscriptions, device bundles, and enterprise licensing.
- Build defensible ML moats through proprietary signal processing and personalization.

**Technical vision**
- Modular, domain-driven backend ready for microservices migration.
- Resilient, observable architecture with comprehensive logging and monitoring.
- Production-grade security with encryption, rate limiting, and RBAC.
- Free and cost-effective deployment with option to scale globally.

**User personas** (backend perspective)
- Consumer users: real-time dashboards, notifications, recommendations.
- Enterprise (clinicians): HIPAA compliance, SSO, audit logs, export APIs.
- Researchers: raw data export, reproducible pipelines, API access.
- Developers: well-documented APIs, SDKs, webhooks.

**System objectives**
- Ingest and process 100+ sessions/day across 50k users.
- Serve <200ms inference latency for realtime scoring.
- Guarantee 99.5% uptime with graceful degradation.
- Maintain <100ms WebSocket latency for live EEG.

**Engineering goals**
- Type-safe, well-tested codebase using Python + FastAPI + SQLAlchemy.
- Automated CI/CD with GitHub Actions.
- Comprehensive observability: logs, metrics, traces.
- Clear separation of concerns: services, repositories, schemas.

---

## 2. System Architecture

**High-level architecture diagram explanation**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  Mobile App (BLE) ──┐    Web Dashboard ──┐    Devices (USB/BLE) ──┐   │
│                    │                     │                        │   │
│                    └──> API Gateway (Nginx/Traefik) <─────────────┘   │
│                                  │                                    │
│                    ┌─────────────┼─────────────┐                      │
│                    │             │             │                      │
│            ┌──────────────┐  ┌─────────┐  ┌──────────────┐           │
│            │  FastAPI     │  │WebSocket│  │ Background  │            │
│            │  API Server  │  │ Gateway │  │ Worker      │            │
│            └──────────────┘  └─────────┘  └──────────────┘           │
│                    │             │             │                      │
│           ┌────────┴─────────────┼─────────────┴─────────┐           │
│           │                      │                       │            │
│      ┌──────────────┐    ┌─────────────────┐  ┌─────────────────┐   │
│      │PostgreSQL DB │    │Redis (broker)   │  │ML Model Service │   │
│      │+ TimescaleDB │    │+ cache          │  │(TensorFlow)     │   │
│      └──────────────┘    └─────────────────┘  └─────────────────┘   │
│           │                      │                       │            │
│      ┌────────────────────────────────────────────────────┐           │
│      │                 Observability                      │           │
│      │  (Prometheus + Grafana + Sentry + Logs)           │           │
│      └────────────────────────────────────────────────────┘           │
│                                                                         │
│      ┌────────────────────────────────────────────────────┐           │
│      │    Storage: R2 (raw EEG blobs) / Supabase         │           │
│      └────────────────────────────────────────────────────┘           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Request lifecycle**
1. Mobile/web client authenticates via Kinde OIDC.
2. Request hits API Gateway (rate limiting, SSL termination).
3. FastAPI app validates JWT, routes to service layer.
4. Service queries DB or calls external APIs.
5. Response returned with cache headers.
6. Real-time streams use WebSocket for low-latency.

**Backend flow**
- Incoming request → middleware (auth, logging) → route handler → service layer → repository → ORM → DB/cache.

**Data pipeline flow**
- Device → Mobile (BLE) → WebSocket to `/ws/sessions/{id}` → binary frame parsing → signal preprocessing → feature extraction → storage → background job queue.

**Realtime EEG pipeline**
- WebSocket receives binary frames (header + channels) → validate sequence → store to time-series table → compute band powers → publish to Redis → broadcast to connected clients.

**ML inference pipeline**
- Incoming feature batch → load model from registry → run inference → store predictions → publish events → notify clients.

**Service communication**
- Services call each other via dependency injection or message queue (async jobs).
- Example: `eeg_service` calls `analytics_service.compute_bands()` synchronously; fires async job to `ml_service.predict()`.

**Monolith vs Modular vs Microservices**

Monolith (single FastAPI app)
- Pros: simpler deployment, shared DB transaction, faster development.
- Cons: harder to scale independently, single point of failure, coupled dependencies.

Modular Monolith (single codebase, multiple services as classes)
- Pros: good separation, shared infra, easier to split later.
- Cons: still single deployment unit.

Microservices (separate services per domain)
- Pros: independent scaling, resilience, polyglot.
- Cons: operational complexity, data consistency, network latency.

**Recommendation for NeuroFit**
- Start with **Modular Monolith** (one FastAPI app with clear service layers). Deploy as single container initially.
- Design to enable future migration to microservices (event-driven architecture, clear service boundaries, API contracts).
- Use background workers (Celery) for async tasks as the bridge to eventual microservices.

---

## 3. Technology Decision Matrix

| Decision | Option | Pros | Cons | Recommendation |
|----------|--------|------|------|-----------------|
| **Framework** | FastAPI | Type-safe, async, auto docs, modern | Younger than Django | ✅ FastAPI |
| | Django | Mature, batteries-included | Overkill, slower | ❌ |
| | Flask | Minimal, lightweight | Less batteries | ❌ |
| **ORM** | SQLAlchemy 2.0 | Powerful, flexible, type hints | Steeper learning curve | ✅ SQLAlchemy |
| | Prisma-like | Simpler API | Limited Python tooling | ❌ |
| | Raw SQL | Full control | Error-prone, verbose | ❌ |
| **Auth** | Kinde | Free tier, OIDC, built-in RBAC | Vendor lock-in (mitigated) | ✅ Kinde |
| | Auth0 | Mature | More expensive | ❌ |
| | Supabase Auth | Free, integrated | Less RBAC features | ❌ |
| **Cache** | Redis | Fast, pub/sub, versatile | Memory overhead | ✅ Redis |
| | Memcached | Simple, fast | No pub/sub | ❌ |
| | SQLite | Single file | Not distributed | ❌ |
| **Task Queue** | Celery | Mature, feature-rich, scalable | Complex, verbose config | ✅ Celery (with caveats) |
| | Dramatiq | Simple, fast, less config | Smaller ecosystem | ✅ Dramatiq (recommended) |
| | APScheduler | Simple scheduling | No distributed queue | ❌ |
| **ML Framework** | TensorFlow | Production-grade, large ecosystem | Heavy, slower iteration | ✅ TensorFlow |
| | PyTorch | Research-friendly, faster | Less prod tooling out-of-box | ✅ PyTorch (alternative) |
| | Scikit-learn | Lightweight, EEG-friendly | Limited deep learning | ✅ Scikit-learn (MVP) |
| **Storage** | Cloudflare R2 | S3-compatible, cheap, reliable | Less data residency control | ✅ R2 |
| | Supabase Storage | Integrated, free tier | Limited bandwidth | ✅ Supabase (alternative) |
| | AWS S3 | Mature, global | More expensive | ❌ |
| **Monitoring** | Prometheus + Grafana | Open-source, powerful, free | Requires setup | ✅ Prometheus + Grafana |
| | BetterStack | Integrated logs + metrics | Proprietary | ✅ BetterStack (alternative) |
| | Datadog | All-in-one | Expensive | ❌ |

**Recommended stack**
- Backend: **FastAPI** (type-safe, modern, great for APIs).
- DB: **PostgreSQL** with **SQLAlchemy 2.0** (production-grade, flexible).
- Cache: **Redis** (pub/sub for WebSocket, caching).
- Auth: **Kinde** (free tier with OIDC + RBAC).
- Task Queue: **Dramatiq** (simpler than Celery, good enough for MVP).
- ML: **TensorFlow** (production-grade) or **Scikit-learn** (MVP-friendly).
- Storage: **Cloudflare R2** (cheap, reliable, S3-compatible).
- Monitoring: **Prometheus + Grafana** (open-source, free) or **BetterStack** (all-in-one).

---

## 4. Backend Folder Architecture

**Professional FastAPI structure**

```
neurofit-backend/
├── app/
│   ├── main.py                 # FastAPI app initialization
│   ├── config.py               # configuration (env vars, settings)
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── v1/                 # API versioning
│   │   │   ├── __init__.py
│   │   │   ├── endpoints/      # route handlers
│   │   │   │   ├── auth.py     # /auth/*
│   │   │   │   ├── eeg.py      # /eeg/*, /sessions/*
│   │   │   │   ├── analytics.py # /analytics/*
│   │   │   │   ├── ai.py       # /ai/predict, /ai/recommendations
│   │   │   │   ├── sleep.py    # /sleep/*
│   │   │   │   ├── training.py # /training/*
│   │   │   │   ├── devices.py  # /devices/*
│   │   │   │   ├── community.py # /community/*
│   │   │   │   └── admin.py    # /admin/*
│   │   │   └── router.py       # APIRouter aggregation
│   │   │
│   │   └── websocket/          # WebSocket handlers
│   │       ├── __init__.py
│   │       └── manager.py      # connection management
│   │
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py           # settings, env
│   │   ├── security.py         # JWT, encryption, hashing
│   │   ├── constants.py        # app-wide constants
│   │   └── exceptions.py       # custom exceptions
│   │
│   ├── middleware/
│   │   ├── __init__.py
│   │   ├── auth.py             # JWT verification middleware
│   │   ├── logging.py          # structured logging
│   │   ├── error_handler.py    # global error handling
│   │   └── rate_limit.py       # rate limiting
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py             # SQLAlchemy User model
│   │   ├── session.py          # EEG Session model
│   │   ├── brainwave.py        # Brainwave Metrics model
│   │   ├── device.py           # Device model
│   │   ├── prediction.py       # AI Prediction model
│   │   ├── sleep.py            # Sleep Session model
│   │   ├── notification.py     # Notification model
│   │   ├── community.py        # Post, Comment, Like models
│   │   ├── achievement.py      # Achievement model
│   │   └── base.py             # shared base for all models
│   │
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── user.py             # Pydantic User schemas
│   │   ├── session.py
│   │   ├── brainwave.py
│   │   ├── prediction.py
│   │   └── base.py             # shared base schemas
│   │
│   ├── services/               # business logic
│   │   ├── __init__.py
│   │   ├── auth_service.py     # user identity, tokens
│   │   ├── eeg_service.py      # session lifecycle, ingestion
│   │   ├── analytics_service.py # band computations, features
│   │   ├── ai_service.py       # model inference, versioning
│   │   ├── sleep_service.py    # sleep analysis
│   │   ├── ml_service.py       # ML model loading/caching
│   │   ├── notification_service.py
│   │   ├── community_service.py
│   │   └── base_service.py     # shared service base
│   │
│   ├── repositories/           # data access layer
│   │   ├── __init__.py
│   │   ├── user_repo.py        # User CRUD
│   │   ├── session_repo.py     # Session queries
│   │   ├── brainwave_repo.py   # Brainwave time-series queries
│   │   ├── device_repo.py
│   │   └── base_repo.py        # shared repository base
│   │
│   ├── websocket/
│   │   ├── __init__.py
│   │   ├── manager.py          # WebSocket connection management
│   │   ├── handlers.py         # WebSocket message handlers
│   │   └── events.py           # event definitions
│   │
│   ├── ml/
│   │   ├── __init__.py
│   │   ├── models.py           # model registry, loading
│   │   ├── preprocessing.py    # EEG signal preprocessing
│   │   ├── features.py         # feature extraction (band power, etc.)
│   │   ├── inference.py        # inference runner
│   │   └── utils.py            # ML utility functions
│   │
│   ├── analytics/
│   │   ├── __init__.py
│   │   ├── fft.py              # FFT computations
│   │   ├── connectivity.py     # coherence, PLV
│   │   ├── sleep.py            # sleep stage detection logic
│   │   └── utils.py
│   │
│   ├── tasks/                  # Celery/Dramatiq tasks
│   │   ├── __init__.py
│   │   ├── eeg_tasks.py        # async EEG processing
│   │   ├── ml_tasks.py         # async model training
│   │   ├── notification_tasks.py
│   │   └── schedule.py         # scheduled tasks
│   │
│   ├── database/
│   │   ├── __init__.py
│   │   ├── db.py               # SQLAlchemy session factory
│   │   ├── init_db.py          # DB initialization
│   │   └── migrations/         # Alembic migrations
│   │       ├── versions/
│   │       └── env.py
│   │
│   ├── dependencies.py         # FastAPI Depends() callables
│   ├── utils/
│   │   ├── __init__.py
│   │   ├── time.py
│   │   ├── math_utils.py
│   │   └── validators.py
│   │
│   └── integrations/
│       ├── __init__.py
│       ├── kinde.py            # Kinde integration
│       ├── notification.py     # push notifications (APNs, FCM)
│       └── storage.py          # R2/Supabase integration
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py             # pytest fixtures
│   ├── unit/                   # unit tests
│   │   ├── test_auth.py
│   │   ├── test_eeg.py
│   │   └── test_ml.py
│   ├── integration/            # integration tests
│   │   ├── test_api.py
│   │   └── test_websocket.py
│   └── fixtures/               # test data
│       └── sample_eeg.py
│
├── workers/                    # worker entry points
│   ├── celery_worker.py        # OR dramatiq_worker.py
│   └── scheduler.py
│
├── scripts/
│   ├── init_db.py
│   ├── seed_data.py
│   ├── train_model.py
│   └── export_data.py
│
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
│
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
│
├── requirements.txt
├── pyproject.toml
├── .env.example
├── README.md
└── docker-compose.yml
```

**Folder purpose guide**

- `app/main.py`: FastAPI app factory; starts server and initializes middleware.
- `app/api/`: all HTTP route handlers organized by domain and version.
- `app/core/`: configuration, security helpers, exceptions, constants.
- `app/middleware/`: cross-cutting concerns (auth, logging, rate limiting).
- `app/models/`: SQLAlchemy ORM models (DB schemas).
- `app/schemas/`: Pydantic validation schemas (request/response DTOs).
- `app/services/`: business logic; dependencies injected.
- `app/repositories/`: data access layer; abstracts DB queries.
- `app/websocket/`: WebSocket connection lifecycle and message routing.
- `app/ml/`: ML model loading, feature extraction, inference.
- `app/analytics/`: signal processing (FFT, connectivity, sleep staging).
- `app/tasks/`: background jobs (Celery/Dramatiq tasks).
- `app/database/`: ORM setup, session factory, migrations.
- `app/integrations/`: external services (Kinde, storage, push).
- `tests/`: pytest suite (unit, integration, fixtures).
- `workers/`: worker entry points for task queue.
- `scripts/`: CLI utilities (DB init, training, exports).
- `docker/`: containerization files.
- `.github/workflows/`: CI/CD pipelines.

---

## 5. Database Design

**ER Diagram explanation**

```
Users (1) ───────────────────────────────────── (N) EEG_Sessions
  |                                                    |
  |                                                    └─── (N) Brainwave_Metrics
  |                                                    └─── (N) AI_Predictions
  |
  ├─ (1) ───────────────────────── (N) Device_Pairings ─── (1) Devices
  |
  ├─ (1) ───────────────────────── (N) Sleep_Sessions
  |
  ├─ (1) ───────────────────────── (N) Training_Sessions
  |
  ├─ (1) ───────────────────────── (N) Subscriptions
  |
  ├─ (1) ───────────────────────── (N) Notifications
  |
  ├─ (1) ───────────────────────── (N) Community_Posts
  |                                    |
  |                                    └─ (N) Comments
  |                                    └─ (N) Likes
  |
  └─ (1) ───────────────────────── (N) Achievements
```

**Example tables**

```sql
-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255),
    kinde_user_id VARCHAR(255) UNIQUE,
    role VARCHAR(50) DEFAULT 'user',  -- user, researcher, clinician, admin
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    deleted_at TIMESTAMPTZ  -- soft delete
);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_kinde_user_id ON users(kinde_user_id);

-- Devices table
CREATE TABLE devices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    device_type VARCHAR(50),  -- headband, ear-eeg
    serial_number VARCHAR(255) UNIQUE,
    firmware_version VARCHAR(50),
    last_seen TIMESTAMPTZ,
    battery_level INT,
    metadata JSONB DEFAULT '{}',  -- flexible fields
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_devices_user_id ON devices(user_id);

-- EEG_Sessions table
CREATE TABLE eeg_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    device_id UUID REFERENCES devices(id),
    started_at TIMESTAMPTZ NOT NULL,
    ended_at TIMESTAMPTZ,
    duration_seconds INT,
    sampling_rate INT,  -- e.g. 256 Hz
    num_channels INT,
    status VARCHAR(50) DEFAULT 'active',  -- active, completed, failed
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_eeg_sessions_user_id ON eeg_sessions(user_id);
CREATE INDEX idx_eeg_sessions_started_at ON eeg_sessions(started_at);

-- Brainwave_Metrics table (time-series, partitioned)
CREATE TABLE brainwave_metrics (
    id BIGSERIAL PRIMARY KEY,
    session_id UUID NOT NULL REFERENCES eeg_sessions(id),
    timestamp TIMESTAMPTZ NOT NULL,
    channel_1 FLOAT8,
    channel_2 FLOAT8,
    -- ... up to 8 channels
    band_delta FLOAT8,      -- power in 0.5-4 Hz
    band_theta FLOAT8,      -- 4-8 Hz
    band_alpha FLOAT8,      -- 8-13 Hz
    band_beta FLOAT8,       -- 13-30 Hz
    band_gamma FLOAT8,      -- 30-100 Hz
    features JSONB          -- extra features (PSD, connectivity)
);
CREATE INDEX idx_brainwave_session_ts ON brainwave_metrics(session_id, timestamp);
-- Partition by month: PARTITION BY RANGE (EXTRACT(MONTH FROM timestamp))

-- AI_Predictions table
CREATE TABLE ai_predictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES eeg_sessions(id),
    model_version VARCHAR(50),  -- e.g. v1.0.2
    prediction_type VARCHAR(50),  -- focus, stress, fatigue
    value FLOAT8,           -- score 0-100
    confidence FLOAT8,      -- 0-1
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_ai_pred_session ON ai_predictions(session_id);

-- Sleep_Sessions table
CREATE TABLE sleep_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ,
    duration_minutes INT,
    sleep_score INT,  -- 0-100
    stage_summary JSONB,  -- {rem: %, light: %, deep: %}
    quality_metrics JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_sleep_user_date ON sleep_sessions(user_id, start_time);

-- Device_Pairings table
CREATE TABLE device_pairings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    device_id UUID NOT NULL REFERENCES devices(id),
    paired_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(user_id, device_id)
);

-- Subscriptions table
CREATE TABLE subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    plan VARCHAR(50),  -- free, pro, enterprise
    status VARCHAR(50),  -- active, cancelled, paused
    billing_period_start TIMESTAMPTZ,
    billing_period_end TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_subscriptions_user ON subscriptions(user_id);

-- Notifications table
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    notification_type VARCHAR(50),  -- focus_alert, recovery_ready, ...
    title VARCHAR(255),
    body TEXT,
    payload JSONB DEFAULT '{}',
    read_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_notifications_user_read ON notifications(user_id, read_at);

-- Community_Posts table
CREATE TABLE community_posts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    content TEXT NOT NULL,
    media_url VARCHAR(512),
    likes_count INT DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_posts_user ON community_posts(user_id);
CREATE INDEX idx_posts_created ON community_posts(created_at DESC);

-- Comments table
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    post_id UUID NOT NULL REFERENCES community_posts(id),
    user_id UUID NOT NULL REFERENCES users(id),
    content TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_comments_post ON comments(post_id);

-- Achievements table
CREATE TABLE achievements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    badge_name VARCHAR(100),  -- focus_streak_7, focus_streak_30, ...
    unlocked_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(user_id, badge_name)
);
```

**SQLAlchemy models strategy**

Use declarative models with type hints:

```python
from sqlalchemy import Column, DateTime, String, UUID, Integer, Float, ForeignKey, Index
from sqlalchemy.orm import relationship
from sqlalchemy.dialects.postgresql import JSONB
from datetime import datetime
import uuid

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    
    id: UUID = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    email: str = Column(String(255), unique=True, nullable=False)
    name: str = Column(String(255))
    kinde_user_id: str = Column(String(255), unique=True)
    role: str = Column(String(50), default="user")
    created_at: datetime = Column(DateTime(timezone=True), default=datetime.utcnow)
    updated_at: datetime = Column(DateTime(timezone=True), onupdate=datetime.utcnow)
    deleted_at: datetime = Column(DateTime(timezone=True), nullable=True)
    
    sessions = relationship("EEGSession", back_populates="user")
    devices = relationship("Device", back_populates="user")

class EEGSession(Base):
    __tablename__ = "eeg_sessions"
    __table_args__ = (Index("idx_user_started", "user_id", "started_at"),)
    
    id: UUID = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    user_id: UUID = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False)
    device_id: UUID = Column(UUID(as_uuid=True), ForeignKey("devices.id"))
    started_at: datetime = Column(DateTime(timezone=True), nullable=False)
    ended_at: datetime = Column(DateTime(timezone=True))
    duration_seconds: int = Column(Integer)
    sampling_rate: int = Column(Integer)
    num_channels: int = Column(Integer)
    status: str = Column(String(50), default="active")
    metadata: dict = Column(JSONB, default={})
    
    user = relationship("User", back_populates="sessions")
    metrics = relationship("BrainwaveMetric", back_populates="session")
```

**Alembic migration workflow**

1. Create migration: `alembic revision --autogenerate -m "add column"`
2. Review and edit `migrations/versions/*.py`
3. Test: `alembic upgrade head` (on local dev)
4. Commit: git commit
5. On deploy: CI runs `alembic upgrade head` before deploying new code.

**Partitioning recommendations**
- Time-series tables (brainwave_metrics): partition by date (monthly or daily).
- Use TimescaleDB hypertable if heavy analytics: `SELECT create_hypertable('brainwave_metrics', 'timestamp');`

**Performance optimization**
- Index frequently queried columns (user_id, session_id, created_at).
- Use connection pooling (SQLAlchemy supports this out-of-box).
- Cache popular queries in Redis.
- Read replicas for analytical queries.

---

## 6. Authentication & Authorization (Kinde)

**Kinde integration with FastAPI**

Kinde is an OIDC provider with built-in RBAC. NeuroFit uses Kinde for:
- User signup/login
- JWT generation
- Refresh token flow
- Role assignment

**Signup flow**
1. User clicks "Sign up" on frontend.
2. Frontend redirects to Kinde login (OpenID Connect Authorization Code flow with PKCE).
3. User authenticates (email, social, MFA).
4. Kinde redirects to callback URL with authorization code.
5. Frontend exchanges code for tokens.
6. Backend verifies JWT using Kinde's JWKS.

**Login flow**
- Similar to signup; Kinde recognizes returning user.

**JWT verification**

```python
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer, HTTPAuthCredentials
import jwt
from jwt import PyJWTError
import requests

KINDE_DOMAIN = "https://your-org.kinde.com"
JWKS_CLIENT = None

async def get_jwks():
    global JWKS_CLIENT
    if not JWKS_CLIENT:
        resp = requests.get(f"{KINDE_DOMAIN}/.well-known/jwks.json")
        JWKS_CLIENT = resp.json()
    return JWKS_CLIENT

async def verify_token(credentials: HTTPAuthCredentials = Depends(HTTPBearer())) -> dict:
    try:
        token = credentials.credentials
        jwks = await get_jwks()
        
        # Decode without verification first to get kid
        unverified = jwt.get_unverified_header(token)
        kid = unverified.get("kid")
        
        # Find public key
        key = None
        for k in jwks["keys"]:
            if k["kid"] == kid:
                key = k
                break
        
        if not key:
            raise HTTPException(status_code=401, detail="Invalid token")
        
        # Verify
        decoded = jwt.decode(
            token,
            jwt.algorithms.RSAAlgorithm.from_jwk(json.dumps(key)),
            algorithms=["RS256"],
            audience="your-api",
            issuer=KINDE_DOMAIN
        )
        
        return decoded
    except PyJWTError:
        raise HTTPException(status_code=401, detail="Invalid token")
```

**Refresh token flow**

```python
@router.post("/auth/refresh")
async def refresh_token(request: RefreshTokenRequest):
    # Client sends refresh token
    # Use Kinde SDK or direct HTTP call to refresh
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            f"{KINDE_DOMAIN}/oauth/token",
            data={
                "grant_type": "refresh_token",
                "refresh_token": request.refresh_token,
                "client_id": CLIENT_ID,
                "client_secret": CLIENT_SECRET,
            }
        )
        tokens = resp.json()
        return {"access_token": tokens["access_token"], ...}
```

**RBAC (Role-Based Access Control)**

Kinde natively supports roles:
- `user` (default consumer)
- `researcher` (data export, annotations)
- `clinician` (HIPAA, audit logs)
- `admin` (full access)

In FastAPI:

```python
async def require_role(required_roles: List[str]):
    async def check_role(current_user: dict = Depends(verify_token)):
        user_roles = current_user.get("roles", [])
        if not any(role in user_roles for role in required_roles):
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return current_user
    return check_role

# Usage
@router.post("/admin/users/export")
async def export_users(_=Depends(require_role(["admin"]))):
    ...
```

**Auth middleware**

```python
from starlette.middleware.base import BaseHTTPMiddleware

class AuthMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        # Extract token
        auth = request.headers.get("Authorization", "")
        if auth.startswith("Bearer "):
            token = auth[7:]
            try:
                decoded = jwt.decode(...)
                request.state.user = decoded
            except:
                pass
        
        response = await call_next(request)
        return response

app.add_middleware(AuthMiddleware)
```

**Token lifecycle diagram**

```
┌──────────────┐
│   Kinde      │
│ Authorization│
│   Server     │
└──────────────┘
       │
       │ returns access_token (15m)
       │           + refresh_token (7d)
       │
       v
┌──────────────────┐        ┌─────────────────┐
│  Client App      │───────→│  Backend API    │
│ (stores tokens)  │        │ (validates JWT) │
└──────────────────┘        └─────────────────┘
       │                           
       │ (after 15m)  ┌──────────────┐
       │ refresh      │   Kinde      │
       └─────────────→│  Token EP    │
                      └──────────────┘
                           │
                           │ new access_token
                           v
                      ┌──────────────┐
                      │ Client (new) │
                      └──────────────┘
```

---

## 7. API Design Specification

**Complete API architecture**

NeuroFit exposes REST endpoints + WebSocket for realtime.

**Auth APIs**

```
POST /api/v1/auth/kinde-callback
  - Receives authorization code from Kinde
  - Returns access_token, refresh_token

POST /api/v1/auth/refresh
  - Body: { refresh_token: "..." }
  - Returns: { access_token: "...", refresh_token: "..." }

GET /api/v1/auth/me
  - Returns current user info
  - Requires: Authorization: Bearer <token>

POST /api/v1/auth/logout
  - Invalidates refresh token (optional)
```

**Session/EEG APIs**

```
POST /api/v1/sessions
  - Start a new EEG session
  - Body: { device_id: uuid, session_type: "focus|sleep|training" }
  - Returns: { session_id: uuid, websocket_url: "wss://..." }

GET /api/v1/sessions/{id}
  - Get session metadata
  - Returns: { id, user_id, started_at, ended_at, duration, sampling_rate, ... }

POST /api/v1/sessions/{id}/stop
  - End session
  - Returns: { session_id, ended_at, duration }

GET /api/v1/sessions?user_id=...&from=2024-01-01&to=2024-01-31
  - List user sessions with filters
  - Pagination: cursor-based
  - Returns: { items: [...], next_cursor: "..." }

GET /api/v1/analytics/session/{id}/bands
  - Get band power summary for session
  - Returns: { session_id, bands: { delta, theta, alpha, beta, gamma }, ... }
```

**AI/ML APIs**

```
POST /api/v1/ai/predict
  - Predict on batch of features
  - Body: { features: [[...], [...]], model: "focus_v2" }
  - Returns: { predictions: [{ value, confidence }], model_version }

GET /api/v1/ai/recommendations
  - Get personalized AI recommendations
  - Returns: { recommendations: [{ type, priority, action_url }] }

GET /api/v1/ai/models
  - List available models and versions
  - Returns: { models: [{ name, version, created_at, status }] }
```

**Sleep APIs**

```
POST /api/v1/sleep/sessions
  - Create sleep session
  - Body: { start_time, end_time }
  - Returns: { session_id, score, stages }

GET /api/v1/sleep/sessions/{id}
  - Get sleep analysis
  - Returns: { sleep_score, rem_pct, light_pct, deep_pct, hypnogram }

GET /api/v1/sleep/sessions?user_id=...&date=2024-01-15
  - List sleep sessions for date
```

**Device APIs**

```
POST /api/v1/devices/pair
  - Request device pairing
  - Body: { device_type, serial_number, pairing_token }
  - Returns: { device_id, status }

GET /api/v1/devices
  - List user's paired devices
  - Returns: { devices: [...] }

POST /api/v1/devices/{id}/firmware/update
  - Trigger OTA firmware update
  - Returns: { device_id, update_status, progress }
```

**Community APIs**

```
POST /api/v1/community/posts
  - Create post
  - Body: { content, media_url }
  - Returns: { post_id, created_at }

GET /api/v1/community/posts?limit=20&cursor=...
  - Get feed (paginated)
  - Returns: { posts: [...], next_cursor }

POST /api/v1/community/posts/{id}/like
  - Like a post
  - Returns: { liked: true }
```

**Admin APIs**

```
GET /api/v1/admin/users
  - List all users (admin only)
  - Pagination: limit, offset
  - Returns: { users: [...], total }

POST /api/v1/admin/users/{id}/role
  - Change user role
  - Body: { role: "researcher" }
  - Returns: { user_id, role }

GET /api/v1/admin/audit-logs
  - List audit logs
  - Returns: { logs: [{user, action, timestamp}], ... }
```

**Response standardization**

```json
{
  "status": "success|error",
  "data": { ... },
  "error": { "code": "INVALID_REQUEST", "message": "..." },
  "meta": { "timestamp", "request_id" }
}
```

**Pagination strategy**

Cursor-based for scalability (avoids offset issues):

```python
class CursorPaginationQuery(BaseModel):
    cursor: Optional[str] = None
    limit: int = 20

# Response
class PaginatedResponse(BaseModel):
    items: List[T]
    next_cursor: Optional[str]
```

**OpenAPI strategy**

FastAPI auto-generates OpenAPI docs (`/docs`). Ensure:
- Schemas use Pydantic models with descriptions.
- All endpoints have docstrings.
- Tag endpoints by domain.
- Include examples.

```python
@router.post("/sessions", tags=["sessions"], responses={201: {"model": SessionResponse}})
async def create_session(body: CreateSessionRequest):
    """Create a new EEG session."""
    ...
```

---

## 8. EEG Processing Pipeline

**Raw EEG acquisition**

Device streams raw ADC samples via BLE or USB. Each frame contains:
- Timestamp
- Sequence number
- Channel samples (interleaved)
- Optional: device status, battery

**Signal filtering**

```python
from scipy import signal
import numpy as np

def preprocess_eeg(raw_signal, sample_rate=256, channels=2):
    """
    1. Bandpass filter (0.5-40 Hz)
    2. Notch filter (50/60 Hz depending on region)
    3. Re-reference (common average)
    """
    
    # Bandpass 0.5-40 Hz
    sos = signal.butter(4, [0.5, 40], 'bandpass', fs=sample_rate, output='sos')
    filtered = signal.sosfilt(sos, raw_signal, axis=0)
    
    # Notch filter 60 Hz (USA) or 50 Hz (Europe)
    notch_freq = 60
    Q = 30
    sos_notch = signal.iirnotch(notch_freq, Q, fs=sample_rate, output='sos')
    filtered = signal.sosfilt(sos_notch, filtered, axis=0)
    
    # Common average re-reference
    if channels > 1:
        filtered = filtered - np.mean(filtered, axis=1, keepdims=True)
    
    return filtered
```

**FFT & Band Power**

```python
def compute_band_power(signal, sample_rate, window_size=256):
    """
    Compute power in classic EEG bands:
    - Delta (0.5-4 Hz)
    - Theta (4-8 Hz)
    - Alpha (8-13 Hz)
    - Beta (13-30 Hz)
    - Gamma (30-100 Hz)
    """
    
    # Apply Hann window and compute PSD
    freqs, psd = signal.welch(signal, fs=sample_rate, 
                               nperseg=window_size, 
                               noverlap=window_size//2)
    
    # Integrate power in frequency bands
    bands = {
        'delta': integrate(psd, freqs, 0.5, 4),
        'theta': integrate(psd, freqs, 4, 8),
        'alpha': integrate(psd, freqs, 8, 13),
        'beta': integrate(psd, freqs, 13, 30),
        'gamma': integrate(psd, freqs, 30, 100),
    }
    
    return bands

def integrate(psd, freqs, freq_min, freq_max):
    mask = (freqs >= freq_min) & (freqs <= freq_max)
    return np.trapz(psd[mask], freqs[mask])
```

**Feature extraction**

```python
def extract_features(eeg_signal, sample_rate):
    """
    Extract classical EEG features for ML:
    - Band power ratios (alpha/beta, theta/beta)
    - Spectral entropy
    - Approximate entropy
    - Cross-frequency coupling
    """
    
    bands = compute_band_power(eeg_signal, sample_rate)
    
    features = {
        'band_power': bands,
        'alpha_beta_ratio': bands['alpha'] / (bands['beta'] + 1e-8),
        'theta_beta_ratio': bands['theta'] / (bands['beta'] + 1e-8),
        'spectral_entropy': entropy(eeg_signal),
        'approximate_entropy': approx_entropy(eeg_signal),
    }
    
    return features
```

**Artifact removal**

```python
def remove_artifacts(eeg_signal, threshold=3.0):
    """
    Remove EOG/EMG artifacts using:
    1. Z-score thresholding
    2. Optional: Independent Component Analysis (ICA)
    """
    
    # Z-score method (simple)
    z_scores = np.abs(signal.detrend(eeg_signal))
    artifacts = z_scores > threshold * np.std(z_scores)
    
    # Interpolate or zero out
    clean_signal = eeg_signal.copy()
    clean_signal[artifacts] = np.nan
    clean_signal = pd.Series(clean_signal).interpolate().values
    
    return clean_signal
```

**MNE-Python workflow**

```python
import mne

def load_and_preprocess_eeg(raw_data, sample_rate, channel_names):
    """
    Use MNE for professional EEG handling.
    """
    
    # Create MNE Raw object
    info = mne.create_info(
        ch_names=channel_names,
        sfreq=sample_rate,
        ch_types=['eeg'] * len(channel_names)
    )
    raw = mne.io.RawArray(raw_data, info)
    
    # Filter
    raw.filter(0.5, 40)
    raw.notch_filter([60])  # USA
    
    # Re-reference to average
    raw.set_eeg_reference('average')
    
    # Remove artifacts (optional: ICA)
    ica = mne.preprocessing.ICA(n_components=15)
    ica.fit(raw)
    ica.exclude = [0, 1]  # exclude bad components
    raw = ica.apply(raw)
    
    return raw
```

**Performance considerations**

- Use numpy/scipy operations (vectorized, fast).
- For real-time: precompute on 2-4 sec windows; update incrementally.
- Offload heavy processing (ICA) to background workers.
- Cache preprocessed signals in Redis for replay.

---

## 9. Machine Learning Architecture

**Model objectives**

Build models for:
- Focus prediction (attention classifier)
- Stress detection (binary/ternary classifier)
- Cognitive scoring (regression)
- Fatigue prediction
- Sleep quality

**Data collection strategy**

1. Collect EEG sessions with user labels (mood, task, outcome).
2. Extract features (band power, connectivity).
3. Store in pandas DataFrame or HDF5 for training.

```python
# Example dataset structure
df = pd.DataFrame({
    'session_id': [...],
    'user_id': [...],
    'timestamp': [...],
    'band_delta': [...],
    'band_theta': [...],
    'band_alpha': [...],
    'band_beta': [...],
    'band_gamma': [...],
    'label_focus': [0, 1, 1, 0, 1],  # 0=low, 1=high
})
```

**Preprocessing**

```python
# Normalize features
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()
X_scaled = scaler.fit_transform(df[['band_delta', 'band_theta', ...]])

# Save scaler for inference
import joblib
joblib.dump(scaler, 'models/focus_scaler.pkl')
```

**Feature engineering**

```python
def engineer_features(df):
    """Create domain-specific features."""
    
    df['alpha_beta_ratio'] = df['band_alpha'] / (df['band_beta'] + 1e-8)
    df['theta_beta_ratio'] = df['band_theta'] / (df['band_beta'] + 1e-8)
    df['alpha_peak'] = (df['band_alpha'] + df['band_beta']) / 2
    
    # Temporal features (rolling window)
    df['focus_trend'] = df['label_focus'].rolling(10).mean()
    
    return df
```

**Training (Scikit-learn example)**

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split, cross_val_score

# Load and prepare data
X = df[['band_delta', 'band_theta', 'alpha_beta_ratio', ...]].values
y = df['label_focus'].values

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Train
model = RandomForestClassifier(n_estimators=100, max_depth=10, random_state=42)
model.fit(X_train, y_train)

# Evaluate
accuracy = model.score(X_test, y_test)
cv_scores = cross_val_score(model, X_train, y_train, cv=5)
print(f"Accuracy: {accuracy:.3f}, CV: {cv_scores.mean():.3f}")

# Save
joblib.dump(model, 'models/focus_v1.0.pkl')
```

**Training (TensorFlow example)**

```python
import tensorflow as tf
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import Dense, Dropout

# Build model
model = Sequential([
    Dense(64, activation='relu', input_shape=(X_train.shape[1],)),
    Dropout(0.3),
    Dense(32, activation='relu'),
    Dropout(0.3),
    Dense(1, activation='sigmoid')  # binary classification
])

model.compile(optimizer='adam', loss='binary_crossentropy', metrics=['accuracy'])

# Train
history = model.fit(X_train, y_train, 
                   validation_split=0.2, 
                   epochs=50, 
                   batch_size=32)

# Evaluate
loss, accuracy = model.evaluate(X_test, y_test)

# Save
model.save('models/focus_v1.0.h5')
```

**Model versioning**

```python
# Version naming: {model_name}_v{major}.{minor}.{patch}_{timestamp}
model_path = f"models/focus_v1.0.0_{datetime.now().strftime('%Y%m%d_%H%M%S')}.pkl"

# Model registry (JSON metadata)
{
  "model_id": "focus_v1.0.0",
  "created_at": "2024-05-20T10:00:00Z",
  "algorithm": "RandomForest",
  "features": ["band_delta", "band_theta", "alpha_beta_ratio"],
  "train_accuracy": 0.87,
  "cv_accuracy": 0.85,
  "test_accuracy": 0.84,
  "production": true,
  "path": "models/focus_v1.0.0_20240520_100000.pkl"
}
```

---

## 10. ML Training + Testing Pipeline

**Step-by-step ML workflow**

```bash
# 1. Collect EEG dataset (from backend)
python scripts/export_training_data.py \
  --from 2024-01-01 --to 2024-05-20 \
  --output data/eeg_raw.csv

# 2. Prepare dataset
python scripts/prepare_dataset.py \
  --input data/eeg_raw.csv \
  --output data/eeg_prepared.csv \
  --train-split 0.7

# 3. Clean and validate
python scripts/validate_data.py --data data/eeg_prepared.csv

# 4. Train model
python train.py \
  --data data/eeg_prepared.csv \
  --model-type focus \
  --output models/focus_v1.0.pkl

# 5. Evaluate
python evaluate.py --model models/focus_v1.0.pkl

# 6. Test inference
python test_inference.py --model models/focus_v1.0.pkl

# 7. Export to ONNX (for mobile)
python export_onnx.py --model models/focus_v1.0.pkl \
  --output models/focus_v1.0.onnx
```

**Confusion matrix, metrics**

```python
from sklearn.metrics import confusion_matrix, classification_report, roc_auc_score

# Predictions
y_pred = model.predict(X_test)

# Confusion matrix
cm = confusion_matrix(y_test, y_pred)
# [[TN, FP],
#  [FN, TP]]

# Metrics
print(classification_report(y_test, y_pred))
# Precision: TP / (TP + FP)
# Recall: TP / (TP + FN)
# F1: 2 * (Precision * Recall) / (Precision + Recall)
# ROC-AUC: area under ROC curve

auc = roc_auc_score(y_test, y_pred_proba)
```

**train.py (example)**

```python
import argparse
import pandas as pd
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import StandardScaler
import joblib
import json
from datetime import datetime

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--data', required=True)
    parser.add_argument('--model-type', required=True)
    parser.add_argument('--output', required=True)
    args = parser.parse_args()
    
    # Load data
    df = pd.read_csv(args.data)
    X = df.drop(columns=['label']).values
    y = df['label'].values
    
    # Train
    model = RandomForestClassifier(n_estimators=100, max_depth=10)
    model.fit(X, y)
    
    # Save
    joblib.dump(model, args.output)
    
    # Metadata
    metadata = {
        'model_type': args.model_type,
        'version': '1.0.0',
        'created_at': datetime.utcnow().isoformat(),
        'features': df.drop(columns=['label']).columns.tolist(),
        'samples': len(df),
    }
    
    with open(args.output.replace('.pkl', '.json'), 'w') as f:
        json.dump(metadata, f)

if __name__ == '__main__':
    main()
```

---

## 11. ML API Integration

**Expose models through FastAPI**

```python
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np

router = APIRouter(prefix="/api/v1/ai", tags=["AI"])

# Load model at startup
model = None
scaler = None

@router.on_event("startup")
async def load_model():
    global model, scaler
    model = joblib.load("models/focus_v1.0.pkl")
    scaler = joblib.load("models/focus_scaler.pkl")

class PredictRequest(BaseModel):
    features: list[list[float]]  # [[band_delta, band_theta, ...], ...]
    model_version: str = "focus_v1.0"

class PredictionResponse(BaseModel):
    predictions: list[float]
    confidence: list[float]
    model_version: str

@router.post("/predict", response_model=PredictionResponse)
async def predict(request: PredictRequest):
    if model is None:
        raise HTTPException(status_code=503, detail="Model not loaded")
    
    try:
        X = np.array(request.features)
        X_scaled = scaler.transform(X)
        predictions = model.predict(X_scaled)
        confidence = model.predict_proba(X_scaled).max(axis=1)
        
        return PredictionResponse(
            predictions=predictions.tolist(),
            confidence=confidence.tolist(),
            model_version=request.model_version
        )
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.post("/predict/stress")
async def predict_stress(request: PredictRequest):
    """Predict stress level."""
    return await predict(request)

@router.post("/predict/focus")
async def predict_focus(request: PredictRequest):
    """Predict focus level."""
    return await predict(request)

@router.get("/models")
async def list_models():
    """List available models."""
    return {
        "models": [
            {"name": "focus", "version": "1.0.0", "created_at": "2024-05-20"},
            {"name": "stress", "version": "1.0.1", "created_at": "2024-05-19"},
        ]
    }
```

**ONNX optimization**

Export sklearn/TensorFlow models to ONNX for cross-platform inference:

```python
from skl2onnx import convert_sklearn
from skl2onnx.common.data_types import FloatTensorType

initial_type = [("float_input", FloatTensorType([None, 5]))]
onnx_model = convert_sklearn(model, initial_types=initial_type)

with open("models/focus_v1.0.onnx", "wb") as f:
    f.write(onnx_model.SerializeToString())
```

Load and run ONNX on mobile or server:

```python
import onnxruntime as ort

sess = ort.InferenceSession("models/focus_v1.0.onnx")
input_name = sess.get_inputs()[0].name
output_name = sess.get_outputs()[0].name

# Predict
pred = sess.run([output_name], {input_name: X_scaled})
```

---

## 12. WebSocket Architecture

**WebSocket lifecycle**

```
1. Client connects to ws://api.neurofit.ai/ws/sessions/{session_id}
2. Include JWT: ws://...?token=<access_token>
3. Server validates token + rate limits
4. Add connection to in-memory connection manager
5. Client sends/receives frames
6. On disconnect: clean up connection
```

**WebSocket manager**

```python
from fastapi import WebSocket, WebSocketDisconnect
from typing import Set, Dict

class ConnectionManager:
    def __init__(self):
        self.active_connections: Dict[str, Set[WebSocket]] = {}
    
    async def connect(self, session_id: str, websocket: WebSocket):
        await websocket.accept()
        if session_id not in self.active_connections:
            self.active_connections[session_id] = set()
        self.active_connections[session_id].add(websocket)
    
    def disconnect(self, session_id: str, websocket: WebSocket):
        self.active_connections[session_id].remove(websocket)
    
    async def broadcast(self, session_id: str, message: dict):
        if session_id in self.active_connections:
            for connection in self.active_connections[session_id]:
                try:
                    await connection.send_json(message)
                except:
                    pass  # connection closed

manager = ConnectionManager()

@router.websocket("/ws/sessions/{session_id}")
async def websocket_endpoint(websocket: WebSocket, session_id: str):
    # Authenticate
    token = websocket.query_params.get("token")
    user = await verify_token_ws(token)
    
    await manager.connect(session_id, websocket)
    
    try:
        while True:
            data = await websocket.receive_bytes()  # binary frames
            
            # Process frame
            frame = parse_binary_frame(data)
            
            # Broadcast to other viewers
            await manager.broadcast(session_id, {
                "type": "frame",
                "data": frame_to_json(frame)
            })
    
    except WebSocketDisconnect:
        manager.disconnect(session_id, websocket)
```

**Message schema**

Binary frame format (optimized):
```
[header (12 bytes)]
  - session_id (4 bytes)
  - sequence (4 bytes)
  - timestamp (4 bytes)
[channels (8 * 4 bytes = 32 bytes)]
[checksum (4 bytes)]
```

JSON control messages:
```json
{
  "type": "metrics",
  "data": {
    "band_delta": 1.2,
    "band_theta": 2.3,
    "alpha_beta_ratio": 0.8
  }
}
```

**Reconnect strategy**

```python
# Client-side (pseudocode)
const MAX_RETRIES = 5
const BASE_DELAY = 1000  // 1s

let retries = 0
let delay = BASE_DELAY

function connect():
    try:
        ws = new WebSocket(...)
    catch:
        retries++
        if retries < MAX_RETRIES:
            setTimeout(() => connect(), delay)
            delay = min(delay * 2, 30000)  // exponential backoff, capped at 30s

// Server-side: client sends last_seq; server replays missing frames
```

**Scaling WebSocket**

With Redis pub/sub:
```python
import aioredis

redis = await aioredis.create_redis_pool('redis://localhost')

# When frame received on instance 1
await redis.publish(f"session:{session_id}", frame_json)

# All instances subscribe
async def listen_redis(session_id):
    ch = await redis.subscribe(f"session:{session_id}")
    async for msg in ch.iter():
        await manager.broadcast(session_id, json.loads(msg))

# Start listener per session
asyncio.create_task(listen_redis(session_id))
```

---

## 13. Background Jobs (Celery vs Dramatiq)

**Celery pros/cons**

Pros:
- Mature, production-proven
- Rich monitoring tools
- Task routing, retries, schedules

Cons:
- Complex configuration
- Heavy dependencies
- Learning curve

**Dramatiq pros/cons**

Pros:
- Simple, lightweight
- Less config
- Good enough for MVP

Cons:
- Smaller ecosystem
- Fewer features than Celery

**Recommendation**: Use **Dramatiq** for MVP; migrate to **Celery** if scaling requires advanced features.

**Dramatiq setup**

```python
import dramatiq
from dramatiq.brokers.redis import RedisBroker

broker = RedisBroker(host="localhost", port=6379)
dramatiq.set_broker(broker)

@dramatiq.actor(time_limit=600000)  # 10 min timeout
def process_eeg_session(session_id: str):
    """Background job: process EEG session."""
    session = SessionRepo().get(session_id)
    
    # Preprocess signals
    raw_data = load_raw_eeg(session_id)
    clean_data = preprocess_eeg(raw_data)
    
    # Compute analytics
    bands = compute_band_power(clean_data)
    
    # Run ML inference
    features = extract_features(clean_data)
    prediction = ml_service.predict_focus(features)
    
    # Store results
    store_analytics(session_id, bands, prediction)

@dramatiq.actor
def send_notification(user_id: str, title: str, body: str):
    """Send push notification."""
    push_service.send(user_id, title, body)

# Queue tasks
process_eeg_session.send(session_id)
send_notification.send(user_id, "Session complete", "Your session is ready!")
```

**Scheduled tasks**

```python
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger

scheduler = AsyncIOScheduler()

@scheduler.scheduled_job(CronTrigger(hour=2, minute=0))  # 2 AM daily
def nightly_analytics():
    """Generate daily analytics for all users."""
    users = UserRepo().get_all()
    for user in users:
        generate_user_insights.send(user.id)

scheduler.start()
```

---

## 14. Testing Strategy

**Unit testing**

```python
# tests/unit/test_eeg_service.py
import pytest
from app.services.eeg_service import EEGService
from app.repositories.session_repo import SessionRepo

@pytest.fixture
def eeg_service():
    return EEGService(repo=SessionRepo())

def test_create_session(eeg_service):
    session = eeg_service.create_session(
        user_id="user123",
        device_id="device123"
    )
    assert session.id is not None
    assert session.user_id == "user123"
    assert session.status == "active"

def test_stop_session(eeg_service):
    session = eeg_service.create_session("user123", "device123")
    eeg_service.stop_session(session.id)
    
    updated = eeg_service.get_session(session.id)
    assert updated.status == "completed"
    assert updated.ended_at is not None
```

**Integration testing**

```python
# tests/integration/test_api.py
import pytest
from fastapi.testclient import TestClient
from app.main import app

@pytest.fixture
def client():
    return TestClient(app)

def test_create_session_api(client):
    response = client.post(
        "/api/v1/sessions",
        json={"device_id": "device123", "session_type": "focus"},
        headers={"Authorization": "Bearer <test_token>"}
    )
    assert response.status_code == 201
    data = response.json()
    assert data["status"] == "success"
    assert "session_id" in data["data"]
```

**WebSocket testing**

```python
# tests/integration/test_websocket.py
from fastapi.testclient import TestClient
import json

def test_websocket_connection(client):
    with client.websocket_connect(
        "/ws/sessions/session123?token=<test_token>"
    ) as websocket:
        # Send frame
        frame = bytes([0, 1, 2, 3, ...])
        websocket.send_bytes(frame)
        
        # Receive response
        data = websocket.receive_json()
        assert data["type"] == "frame"
```

**ML testing**

```python
# tests/unit/test_ml.py
import numpy as np

def test_focus_model_prediction():
    from app.ml.inference import predict_focus
    
    # Synthetic features
    features = np.array([[1.2, 2.3, 0.8, ...]])
    
    prediction = predict_focus(features)
    assert 0 <= prediction <= 1
    assert isinstance(prediction, float)
```

**pytest.ini**

```ini
[pytest]
testpaths = tests
python_files = test_*.py
addopts = --cov=app --cov-report=html
```

---

## 15. Security Architecture

**JWT security**

- Use RS256 (asymmetric) for token signing (Kinde handles this).
- Short-lived access tokens (15m).
- Refresh tokens stored securely (httpOnly cookies, Keychain).
- No sensitive data in JWT payload.

**Rate limiting**

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()
app.state.limiter = limiter

@router.post("/sessions")
@limiter.limit("100/hour")
async def create_session(request: Request):
    ...
```

**SQL injection prevention**

Use SQLAlchemy ORM (parameterized queries):
```python
# Safe
user = db.query(User).filter(User.email == email).first()

# Unsafe (don't do this)
query = f"SELECT * FROM users WHERE email = '{email}'"
```

**RBAC**

```python
async def require_role(required_roles: List[str]):
    async def checker(user = Depends(verify_token)):
        if user.get("role") not in required_roles:
            raise HTTPException(status_code=403)
        return user
    return checker

@router.post("/admin/export")
async def export(_=Depends(require_role(["admin"]))):
    ...
```

**Encryption**

- In transit: HTTPS/TLS (enforced by API Gateway).
- At rest: database encryption (PostgreSQL pgcrypto) or application-level encryption for PII.

```python
from cryptography.fernet import Fernet

key = Fernet.generate_key()
cipher = Fernet(key)

encrypted_pii = cipher.encrypt(b"user@example.com")
```

**OWASP Mobile Security checklist**

- Input validation
- Output encoding
- Authentication (strong tokens)
- Authorization (role-based)
- Cryptography (encrypt PII)
- Secure communication (TLS)
- Logging and monitoring
- Error handling (no stack traces in prod)

**Security headers**

```python
from starlette.middleware.cors import CORSMiddleware
from starlette.middleware.hsts import HSTSMiddleware

app.add_middleware(HSTSMiddleware, max_age=31536000)  # 1 year
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://neurofit.ai"],
    allow_methods=["GET", "POST"],
)

# Add headers
@app.middleware("http")
async def add_security_headers(request, call_next):
    response = await call_next(request)
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["X-XSS-Protection"] = "1; mode=block"
    return response
```

---

## 16. Deployment Guide (FREE)

**Free deployment platforms comparison**

| Platform | Free Tier | Postgres | Redis | Notes |
|----------|-----------|----------|-------|-------|
| Render | $7/mo | $15/mo managed | $7/mo | Easy deploys, free tier tight |
| Railway | $5 credit | $15/mo managed | via Redis Cloud | Good UX |
| Fly.io | Up to 3 shared-cpu VMs | Managed + free tier | via Upstash | Edge deployment |
| Koyeb | Limited | No built-in | No built-in | Serverless containers |
| Northflank | Free tier | $15/mo | No | Good monitoring |

**Recommended for MVP**: **Fly.io** (edge deployment, good for WebSocket; free tier generous).

**Fly.io deployment**

1. Sign up: https://fly.io
2. Install CLI: `curl -L https://fly.io/install.sh | sh`
3. Create app: `flyctl apps create neurofit-api`
4. Set secrets: `flyctl secrets set DATABASE_URL=postgres://...`
5. Deploy: `flyctl deploy`

**Fly.io config (fly.toml)**

```toml
app = "neurofit-api"

[build]
  builder = "paketobuildpacks"

[[services]]
  internal_port = 8000
  protocol = "tcp"

  [services.concurrency]
    type = "requests"
    hard_limit = 1000
    soft_limit = 800

[env]
  PYTHONUNBUFFERED = "true"
```

**PostgreSQL (free tier)**

Use **Supabase** (Postgres + Auth + Storage):
```bash
# Create free database on supabase.com
# Get CONNECTION_STRING
flyctl secrets set DATABASE_URL="postgresql://user:pass@db.supabase.co:5432/postgres"
```

Or **Railway** (simpler):
```bash
# Create Postgres on railway.app
# Get connection string from UI
flyctl secrets set DATABASE_URL="..."
```

**Redis (free tier)**

Use **Upstash** (serverless Redis):
```bash
# Create on upstash.com
# Get REDIS_URL
flyctl secrets set REDIS_URL="redis://default:pass@host:port"
```

**Object storage (free)**

Use **Cloudflare R2** (free tier = 10 GB):
```python
import boto3

s3 = boto3.client(
    's3',
    endpoint_url='https://[account_id].r2.cloudflarestorage.com',
    aws_access_key_id=R2_ACCESS_KEY,
    aws_secret_access_key=R2_SECRET_KEY
)

s3.put_object(Bucket='eeg-sessions', Key='session_123.parquet', Body=data)
```

**Domain + SSL**

- Buy domain: Namecheap, GoDaddy
- Point DNS to Fly.io
- Fly.io auto-provisions Let's Encrypt certificate

---

## 17. CI/CD Pipeline

**GitHub Actions workflow (.github/workflows/ci.yml)**

```yaml
name: CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov
      
      - name: Lint
        run: |
          pip install flake8
          flake8 app --max-line-length=120
      
      - name: Type check
        run: |
          pip install mypy
          mypy app
      
      - name: Test
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
          REDIS_URL: redis://localhost:6379
        run: pytest tests --cov=app --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.xml

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Fly
        uses: superfly/flyctl-actions@master
        with:
          args: deploy
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```

---

## 18. Monitoring & Logging

**Logging setup (Loguru)**

```python
from loguru import logger

logger.add("logs/app.log", rotation="500 MB", retention="7 days")

# Usage
logger.info("Session created", session_id=session_id)
logger.error("Prediction failed", error=str(e), model=model_name)
```

**Prometheus metrics**

```python
from prometheus_client import Counter, Histogram, start_http_server

request_count = Counter('requests_total', 'Total requests', ['method', 'endpoint'])
request_latency = Histogram('request_latency_seconds', 'Request latency')

@app.middleware("http")
async def track_metrics(request, call_next):
    start = time.time()
    response = await call_next(request)
    duration = time.time() - start
    
    request_count.labels(method=request.method, endpoint=request.url.path).inc()
    request_latency.observe(duration)
    
    return response

# Expose metrics on /metrics
from prometheus_client import make_wsgi_app
from starlette.middleware.wsgi import WSGIMiddleware
app.add_route("/metrics", WSGIMiddleware(make_wsgi_app()))
```

**Sentry (error tracking)**

```python
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration

sentry_sdk.init(
    dsn="https://key@sentry.io/project_id",
    integrations=[FastApiIntegration()],
    traces_sample_rate=0.1,
)
```

**BetterStack (all-in-one, free tier)**

```python
import logging
from pythonjsonlogger import jsonlogger

# Structured JSON logs
logHandler = logging.StreamHandler()
formatter = jsonlogger.JsonFormatter()
logHandler.setFormatter(formatter)

logger = logging.getLogger()
logger.addHandler(logHandler)

# BetterStack receives logs via syslog or HTTP
```

---

## 19. Development Roadmap

**MVP (0–6 months)**
- Basic EEG streaming (1–2 channels)
- Session recording and export
- Band power analytics
- Simple focus classifier
- Mobile app pairing and live session
- Web dashboard with live viewer and session history
- Kinde auth
- PostgreSQL DB

**V1 (6–12 months)**
- Multi-channel support (8 channels)
- Sleep analytics (staging, quality)
- Neurofeedback training flows
- AI recommendations engine
- Subscription billing
- Community features
- Mobile notification improvements

**V2 (12–24 months)**
- Clinician/researcher features (HIPAA, annotations)
- Enterprise multi-tenancy
- Model personalization and federated learning
- Regulatory certification path
- Admin dashboards and audit logs

**Enterprise (24+ months)**
- EHR integrations
- On-prem deployment options
- Advanced analytics dashboards

---

## 20. Cost Estimation

**Free MVP**
- Fly.io: $0–5/mo (shared CPU)
- PostgreSQL (Supabase): $0 (free tier)
- Redis (Upstash): $0 (free tier)
- R2 storage: $0 (free tier)
- Domain: ~$10/year
- **Total: ~$5–10/mo**

**Startup scale (50k MAU)**
- Fly.io: $50–200/mo (dedicated CPU)
- PostgreSQL: $50–200/mo (managed)
- Redis: $10–50/mo (Upstash)
- R2: $50–100/mo (storage egress)
- Monitoring/Logs: $50–100/mo
- **Total: ~$250–750/mo**

**Enterprise scale**
- Multi-region deployment: $2k–10k/mo
- Compliance/security: +$500–2k/mo
- GPU inference: +$500–5k/mo (depending on volume)

---

## 21. Appendix & Glossary

**Glossary**

- ASGI: Asynchronous Server Gateway Interface (async web servers like Uvicorn)
- ORM: Object-Relational Mapping (SQLAlchemy)
- JWT: JSON Web Token (signed auth token)
- OIDC: OpenID Connect (authentication protocol; Kinde uses this)
- EEG: Electroencephalogram (brain electrical activity)
- PSD: Power Spectral Density
- FFT: Fast Fourier Transform
- Band power: integrated power in frequency range
- Artifact: noise/contamination in EEG
- ICA: Independent Component Analysis
- RBAC: Role-Based Access Control
- Latency: time delay (ms)
- Throughput: data rate (samples/sec)

**Best practices**

1. Type-safe code: use Pydantic + SQLAlchemy type hints.
2. Logging: use structured logging (JSON) for observability.
3. Testing: write unit + integration tests; aim for >80% coverage.
4. Security: validate input, sanitize output, use HTTPS, rotate secrets.
5. Monitoring: instrument critical paths; alert on anomalies.
6. Documentation: keep OpenAPI docs updated; use docstrings.
7. Scalability: design for horizontal scaling; avoid single points of failure.

**Learning resources**

- FastAPI: https://fastapi.tiangolo.com
- SQLAlchemy: https://docs.sqlalchemy.org
- Pydantic: https://pydantic-settings.helpmanual.io
- EEG processing: MNE-Python docs (https://mne.tools)
- ML: scikit-learn docs, TensorFlow tutorials
- Testing: pytest docs

**Architecture decision record (ADR)**

Example ADR: "Why we chose FastAPI over Django"

```
# ADR-001: Use FastAPI for backend

## Context
Need lightweight, async API for real-time WebSocket + REST endpoints.

## Decision
Use FastAPI (async, type-safe, auto-docs).

## Consequences
- Faster iteration, cleaner code
- Smaller ecosystem than Django (mitigated by active community)
- Good for MVP and scalable to enterprise

## Alternatives considered
- Django (too heavy, overkill)
- Flask (too minimal)
```

---

End of Backend TRD document.
