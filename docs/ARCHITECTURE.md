# 🏛️ Architecture Documentation
## RockBees - System Architecture & Design

---

## Overview

RockBees uses a modern, scalable microservices-inspired architecture designed for:
- **Real-time processing** of bee colony detection
- **Asynchronous operations** for ML inference
- **Geospatial data management** with PostGIS
- **High availability** and horizontal scaling

---

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     CLIENT LAYER                                 │
├──────────────┬──────────────────┬──────────────┬────────────────┤
│ Web Dashboard│  Mobile App      │ Consumer QR  │  Admin Portal  │
│ (React 18)   │ (React Native)   │  App         │  (React)       │
└────────┬─────┴────────┬─────────┴──────────┬────┴────────┬─────┘
         │              │                    │             │
         └──────────────┼────────────────────┼─────────────┘
                        │
        ┌───────────────▼───────────────┐
        │  API Gateway & Load Balancer   │
        │  (FastAPI + WebSocket)         │
        │  - Authentication (JWT/OAuth)  │
        │  - Request routing             │
        │  - Rate limiting               │
        └───────────────┬───────────────┘
                        │
    ┌───────────────────┼───────────────────┐
    │                   │                   │
    ▼                   ▼                   ▼
┌────────────────┐ ┌──────────────┐ ┌────────────────┐
│   Detection    │ │    Honey     │ │   Monitoring   │
│   Microservice │ │   Analysis   │ │   Microservice │
│  (YOLOv8)      │ │  Service     │ │   (Sensors)    │
└────────┬───────┘ │  (OpenCV)    │ └────────┬───────┘
         │         └──────┬───────┘          │
         │                │                  │
         └────────────────┼──────────────────┘
                          │
         ┌────────────────▼────────────────┐
         │ ML Inference Pipeline           │
         │ - Image preprocessing           │
         │ - Model inference (TensorFlow)  │
         │ - Result post-processing        │
         │ - Background jobs (Celery)      │
         └────────────────┬────────────────┘
                          │
    ┌─────────────────────┼─────────────────────┐
    │                     │                     │
    ▼                     ▼                     ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  PostgreSQL  │  │    Redis     │  │   S3 Storage │
│  + PostGIS   │  │   (Cache)    │  │  (Media)     │
│ (Primary DB) │  │  (Session)   │  │              │
└──────────────┘  └──────────────┘  └──────────────┘
```

---

## Component Details

### 1. Client Layer

#### Web Dashboard (React 18)
- **Purpose**: Beekeeper colony management and monitoring
- **Features**: 
  - Real-time map view with GPS locations
  - Health status dashboard
  - Alert management
  - Historical analytics
- **Technology**: React 18, TypeScript, Tailwind CSS, Leaflet Maps

#### Mobile App (React Native)
- **Purpose**: Field data collection and consumer verification
- **Features**:
  - Camera integration for colony images
  - QR code scanning for honey verification
  - Offline capability
  - Location tracking
- **Technology**: React Native, Expo, React Navigation

#### Consumer QR App
- **Purpose**: Instant honey authenticity verification
- **Features**:
  - QR code scanning
  - Purity report display
  - Supply chain information
- **Technology**: React Native, Camera API

---

### 2. API Gateway Layer

**Technology**: FastAPI + Uvicorn

```python
Key Responsibilities:
- Route requests to appropriate microservices
- Handle authentication/authorization
- Rate limiting and request throttling
- WebSocket management for real-time updates
- Request/response logging
- Error handling and status codes
```

**Endpoints Structure**:
```
/api/v1/
├── /colonies          # Colony detection & management
├── /honey            # Honey analysis & verification
├── /monitoring       # Real-time monitoring (WebSocket)
├── /users            # User management & auth
└── /health           # System health check
```

---

### 3. Microservices

#### Detection Service
```
Input: Image/Video
Process:
  1. Image preprocessing (normalization, resizing)
  2. YOLOv8 inference
  3. Post-processing (NMS, confidence filtering)
  4. Behavior classification (CNN)
  5. GPS tagging
Output: Detected colonies with metadata
```

#### Honey Analysis Service
```
Input: Honey sample image
Process:
  1. Spectral analysis (RGB extraction)
  2. Feature engineering (color, texture, crystallinity)
  3. ML classifier inference
  4. Purity scoring and grading
  5. QR code generation
Output: Purity report with confidence score
```

#### Monitoring Service
```
Input: Sensor data streams
Process:
  1. Data collection (temperature, humidity, etc.)
  2. Anomaly detection
  3. Alert generation
  4. Real-time broadcasting (WebSocket)
Output: Live status updates
```

---

### 4. ML Inference Pipeline

**Technology**: TensorFlow, PyTorch

**Models Deployed**:
1. **YOLOv8** - Object detection for bee colonies
2. **Custom CNN** - Health status classification
3. **Spectral Classifier** - Honey purity analysis
4. **LSTM** - Temporal behavior analysis

**Processing Flow**:
```
Raw Input → Preprocessing → Model Inference → Post-processing → Output
```

**Optimization Techniques**:
- Model quantization for faster inference
- Batch processing for efficiency
- Caching of frequent predictions
- GPU acceleration when available

---

### 5. Database Layer

#### PostgreSQL with PostGIS
```sql
Tables:
- users               # User accounts
- colonies            # Colony records with GPS locations
- detection_records   # Detection history
- honey_reports       # Purity analysis results
- sensor_readings     # Environmental sensor data
- alerts              # Alert notifications
```

**PostGIS Features**:
- Geospatial queries (distance, boundaries)
- GPS coordinate management
- Location-based filtering
- Spatial indexing for performance

#### Redis Cache
```
Usage:
- Session storage
- Real-time monitoring data
- Cached API responses
- Task queue (Celery)
```

#### S3 Storage
```
Storage:
- Original images
- Detection results (images with boxes)
- Purity analysis images
- Report PDFs
```

---

## Data Flow Architecture

### Colony Detection Data Flow

```
Field Image/Video
    │
    ▼
┌─────────────────────────┐
│ API: /colonies/detect   │
│ (multipart/form-data)   │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│ Image Preprocessing     │
│ - Normalize (0-1)       │
│ - Resize to 640×640     │
│ - Enhance contrast      │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│ YOLOv8 Detection        │
│ - Inference (45ms)      │
│ - Get bounding boxes    │
│ - Confidence scores     │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│ Post-processing         │
│ - NMS (remove overlaps) │
│ - Filter low confidence │
│ - Extract regions       │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│ Behavior Classifier     │
│ - CNN inference         │
│ - Health status         │
│ - Strength estimation   │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│ GPS Tagging             │
│ - Add coordinates       │
│ - Timestamp             │
│ - Metadata              │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│ Database Storage        │
│ - Save detection record │
│ - Update colony status  │
│ - Store images (S3)     │
└──────────┬──────────────┘
           │
           ▼
        API Response
      + WebSocket Update
```

### Honey Purity Analysis Flow

```
Honey Sample Image
    │
    ▼
┌──────────────────────┐
│ API: /honey/analyze  │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Spectral Analysis    │
│ - RGB extraction     │
│ - Color space conv   │
│ - Histogram equal    │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Feature Extraction   │
│ - Color features     │
│ - Texture (GLCM)     │
│ - Crystallinity      │
│ - Edge detection     │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ ML Classification    │
│ - Purity score (%)   │
│ - Adulteration prob  │
│ - Confidence level   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Grading System       │
│ - Grade A (90-100%)  │
│ - Grade B (70-89%)   │
│ - Grade C (<70%)     │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Report Generation    │
│ - PDF creation       │
│ - QR code encoding   │
│ - Database storage   │
└──────────┬───────────┘
           │
           ▼
        API Response
      + QR Code Image
```

---

## Deployment Architecture

### Development Environment
```yaml
# docker-compose.yml
Services:
  - PostgreSQL (db)
  - Redis (cache)
  - FastAPI (api)
  - React (frontend)
  - Celery Worker (background)
```

### Production Environment
```
Kubernetes Cluster:
├── API Deployment (3+ replicas)
├── Worker Deployment (Celery)
├── Ingress (load balancing)
├── StatefulSet (databases)
├── ConfigMaps (configuration)
└── Secrets (credentials)
```

### Cloud Architecture
```
AWS/GCP:
├── ECS/GKE (container orchestration)
├── RDS (managed PostgreSQL)
├── ElastiCache (managed Redis)
├── S3/Cloud Storage (object storage)
├── CloudFront/CDN (content delivery)
└── CloudWatch/Stackdriver (monitoring)
```

---

## Scalability Considerations

### Horizontal Scaling
- **Stateless API**: Multiple instances behind load balancer
- **Database**: Read replicas for query load
- **Cache**: Distributed Redis cluster
- **Storage**: S3 with infinite scalability

### Performance Optimization
- Image caching (Redis)
- API response caching
- Database query optimization
- Connection pooling
- Batch processing for ML

### Monitoring & Observability
- Prometheus metrics collection
- Grafana dashboards
- ELK stack for logging
- Distributed tracing
- Alerting system

---

## Security Architecture

### Authentication & Authorization
- JWT tokens for API access
- OAuth 2.0 integration
- Role-based access control (RBAC)
- Session management

### Data Security
- Encryption in transit (HTTPS/TLS)
- Encryption at rest (database, S3)
- Input validation and sanitization
- SQL injection prevention (ORM)
- CORS policy enforcement

### Infrastructure Security
- VPC isolation
- Security groups/network policies
- API rate limiting
- DDoS protection
- Regular security audits

---

## Technology Decisions & Rationale

| Component | Technology | Why |
|-----------|-----------|-----|
| **Detection** | YOLOv8 | State-of-art accuracy, real-time performance |
| **Analysis** | OpenCV | Proven spectral analysis capabilities |
| **Backend** | FastAPI | High performance, async, auto docs |
| **Database** | PostgreSQL + PostGIS | ACID compliance, geospatial queries |
| **Cache** | Redis | Sub-millisecond responses, session store |
| **ML Framework** | TensorFlow | Production-ready, ecosystem support |
| **Frontend** | React | Large community, component reusability |
| **Containerization** | Docker | Consistent environments, easy deployment |
| **Orchestration** | Kubernetes | Industry standard, auto-scaling |
| **Task Queue** | Celery | Distributed task processing, scheduling |

---

## Future Architecture Enhancements

### Phase 2
- Microservices mesh (Istio)
- Service discovery improvements
- Advanced caching strategies
- GraphQL API layer

### Phase 3
- Blockchain integration (supply chain)
- Edge computing (on-device ML)
- Real-time analytics (Kafka)
- Multi-region deployment

### Phase 4
- Machine learning model serving (TensorFlow Serving)
- Advanced monitoring (OpenTelemetry)
- Federated learning capabilities
- Hybrid cloud deployment

---

**Last Updated**: December 2025  
**Version**: 1.0.0  
**Status**: Active Development