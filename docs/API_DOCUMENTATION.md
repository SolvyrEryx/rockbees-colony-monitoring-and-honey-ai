# 🔌 API Documentation
## RockBees - Complete API Reference

---

## Base URL

```
Production: https://api.rockbees.app/api/v1
Development: http://localhost:8000/api/v1
```

---

## Authentication

### JWT Token Flow

**1. Get Access Token**
```bash
POST /auth/login
Content-Type: application/json

{
  "username": "beekeeper@example.com",
  "password": "secure_password"
}

Response (200):
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer",
  "expires_in": 3600
}
```

**2. Use Token in Requests**
```bash
Authorization: Bearer <access_token>
```

**3. Refresh Token**
```bash
POST /auth/refresh
Authorization: Bearer <refresh_token>

Response (200):
{
  "access_token": "new_token_here",
  "expires_in": 3600
}
```

---

## Endpoints

### 1. Colony Detection

#### POST /colonies/detect

**Detect bee colonies from image**

```bash
POST /api/v1/colonies/detect
Content-Type: multipart/form-data
Authorization: Bearer <token>

Parameters:
  - file (File): Image file (JPEG, PNG, MP4)
  - latitude (float, optional): GPS latitude
  - longitude (float, optional): GPS longitude
  - location_name (string, optional): Location label

Request Example:
curl -X POST "http://localhost:8000/api/v1/colonies/detect" \
  -H "Authorization: Bearer token" \
  -F "file=@colony_image.jpg" \
  -F "latitude=19.0176" \
  -F "longitude=73.0197"
```

**Response (200 OK)**
```json
{
  "detection_id": "det_1234567890",
  "timestamp": "2025-12-22T11:21:00Z",
  "status": "success",
  "colonies_detected": 3,
  "detections": [
    {
      "colony_id": "col_001",
      "bbox": [100, 150, 250, 280],
      "confidence": 0.892,
      "health_status": "healthy",
      "estimated_strength": 45000,
      "swarm_probability": 0.05
    },
    {
      "colony_id": "col_002",
      "bbox": [300, 100, 420, 220],
      "confidence": 0.856,
      "health_status": "weak",
      "estimated_strength": 15000,
      "swarm_probability": 0.15
    },
    {
      "colony_id": "col_003",
      "bbox": [450, 200, 550, 340],
      "confidence": 0.823,
      "health_status": "swarming",
      "estimated_strength": 30000,
      "swarm_probability": 0.85
    }
  ],
  "location": {
    "latitude": 19.0176,
    "longitude": 73.0197,
    "accuracy_meters": 5,
    "address": "Navi Mumbai, Maharashtra, India"
  },
  "image_url": "https://s3.amazonaws.com/rockbees/...",
  "processing_time_ms": 450
}
```

**Error Response (400)**
```json
{
  "status": "error",
  "error_code": "INVALID_IMAGE",
  "message": "Uploaded file is not a valid image format",
  "details": "Only JPEG, PNG, and MP4 files are supported"
}
```

---

#### GET /colonies/{colony_id}

**Fetch colony details**

```bash
GET /api/v1/colonies/col_001
Authorization: Bearer <token>

Response (200):
{
  "colony_id": "col_001",
  "location": {
    "latitude": 19.0176,
    "longitude": 73.0197,
    "address": "Navi Mumbai, Maharashtra, India"
  },
  "discovered_date": "2025-12-15",
  "last_seen": "2025-12-22T11:21:00Z",
  "health_status": "healthy",
  "strength": 45000,
  "detected_count": 5,
  "detection_history": [
    {
      "detection_id": "det_1001",
      "timestamp": "2025-12-22T11:21:00Z",
      "confidence": 0.892,
      "status": "healthy"
    }
  ],
  "alerts": [
    {
      "alert_id": "alr_001",
      "type": "temperature_high",
      "severity": "medium",
      "created_at": "2025-12-22T10:15:00Z",
      "message": "Temperature exceeded 35°C"
    }
  ],
  "environmental_data": {
    "current_temp": 32.5,
    "humidity": 65,
    "wind_speed": 10,
    "nearby_flowers": ["Mango", "Neem", "Coconut"]
  }
}
```

---

#### GET /colonies/map

**Get all colonies as GeoJSON (for map visualization)**

```bash
GET /api/v1/colonies/map?bounds=-180,-90,180,90&zoom=10
Authorization: Bearer <token>

Query Parameters:
  - bounds (string): minLon,minLat,maxLon,maxLat
  - zoom (int): Zoom level (1-20)
  - health_filter (string): healthy,weak,swarming,sick (optional)
  - days_ago (int): Show detections from last N days (optional)

Response (200):
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "Point",
        "coordinates": [73.0197, 19.0176]
      },
      "properties": {
        "colony_id": "col_001",
        "health_status": "healthy",
        "strength": 45000,
        "last_detected": "2025-12-22T11:21:00Z",
        "confidence": 0.892,
        "color": "#00ff00"
      }
    }
  ]
}
```

---

### 2. Honey Analysis

#### POST /honey/analyze

**Analyze honey sample for purity**

```bash
POST /api/v1/honey/analyze
Content-Type: multipart/form-data
Authorization: Bearer <token>

Parameters:
  - file (File): Honey sample image (JPEG, PNG)
  - batch_id (string, optional): Batch tracking ID
  - producer_id (string, optional): Producer identifier

Request Example:
curl -X POST "http://localhost:8000/api/v1/honey/analyze" \
  -H "Authorization: Bearer token" \
  -F "file=@honey_sample.jpg" \
  -F "batch_id=BATCH_001_2025"
```

**Response (200 OK)**
```json
{
  "report_id": "hon_8876543210",
  "timestamp": "2025-12-22T11:21:00Z",
  "batch_id": "BATCH_001_2025",
  "status": "success",
  
  "purity_analysis": {
    "purity_score": 87,
    "confidence_percentage": 94.2,
    "grade": "A",
    "quality_recommendation": "Premium Grade - Pure Honey"
  },
  
  "composition_breakdown": {
    "pure_honey_probability": 0.942,
    "sugar_syrup_probability": 0.031,
    "high_fructose_syrup_probability": 0.015,
    "glucose_probability": 0.012
  },
  
  "detailed_analysis": {
    "color_index": 42,
    "crystallinity_ratio": 0.15,
    "viscosity_estimate": "medium",
    "moisture_content": "17.2%",
    "detected_markers": [
      "natural_glucose",
      "pollen_presence",
      "enzyme_activity"
    ]
  },
  
  "supply_chain": {
    "qr_code": "data:image/png;base64,iVBORw0KG...",
    "tracking_url": "https://verify.rockbees.app/hon_8876543210",
    "producer_verified": true,
    "producer_name": "Sustainable Apiary Co.",
    "production_date": "2025-11-15",
    "batch_notes": "Acacia honey, single origin"
  },
  
  "health_safety": {
    "contamination_risk": "low",
    "pesticide_detected": false,
    "heavy_metals_detected": false,
    "pathogen_risk": "none_detected"
  },
  
  "image_url": "https://s3.amazonaws.com/rockbees/...",
  "processing_time_ms": 850
}
```

**Error Response (400)**
```json
{
  "status": "error",
  "error_code": "LOW_QUALITY_IMAGE",
  "message": "Image quality too low for accurate analysis",
  "details": "Please provide a clear, well-lit image of the honey sample"
}
```

---

#### GET /honey/{report_id}

**Fetch honey analysis report**

```bash
GET /api/v1/honey/hon_8876543210
Authorization: Bearer <token>

Response (200):
{
  "report_id": "hon_8876543210",
  "timestamp": "2025-12-22T11:21:00Z",
  "purity_score": 87,
  "grade": "A",
  "confidence_percentage": 94.2,
  "status": "verified",
  "qr_code": "data:image/png;base64,...",
  "tracking_url": "https://verify.rockbees.app/hon_8876543210"
}
```

---

#### POST /honey/{report_id}/verify-qr

**Consumer verification via QR code**

```bash
POST /api/v1/honey/hon_8876543210/verify-qr
Authorization: Bearer <token>

Response (200):
{
  "verified": true,
  "purity_grade": "A",
  "purity_percentage": 87,
  "producer": "Sustainable Apiary Co.",
  "production_date": "2025-11-15",
  "message": "✓ Authentic honey verified"
}
```

---

### 3. Monitoring & Alerts

#### GET /monitoring/dashboard

**Real-time monitoring dashboard data**

```bash
GET /api/v1/monitoring/dashboard
Authorization: Bearer <token>

Response (200):
{
  "timestamp": "2025-12-22T11:21:00Z",
  "total_colonies": 156,
  "health_summary": {
    "healthy": 120,
    "weak": 25,
    "swarming": 8,
    "sick": 3
  },
  "alerts": {
    "total": 5,
    "critical": 1,
    "high": 2,
    "medium": 2
  },
  "recent_detections": 23,
  "average_confidence": 0.887,
  "system_uptime": 99.92
}
```

---

#### WebSocket /ws/monitor/{colony_id}

**Real-time colony monitoring via WebSocket**

```javascript
const ws = new WebSocket(
  'ws://localhost:8000/api/v1/monitoring/ws/col_001?token=<token>'
);

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log(data);
  // {
  //   "timestamp": "2025-12-22T11:21:00Z",
  //   "colony_id": "col_001",
  //   "status": "healthy",
  //   "temperature": 32.5,
  //   "humidity": 65,
  //   "strength": 45000,
  //   "alert": null
  // }
};
```

---

#### GET /alerts

**Fetch alert history**

```bash
GET /api/v1/alerts?severity=high&limit=10&offset=0
Authorization: Bearer <token>

Query Parameters:
  - severity (string): critical,high,medium,low (optional)
  - status (string): unresolved,resolved (optional)
  - limit (int): Results per page (default: 20)
  - offset (int): Pagination offset (default: 0)

Response (200):
{
  "total": 45,
  "alerts": [
    {
      "alert_id": "alr_001",
      "colony_id": "col_001",
      "type": "swarming_risk",
      "severity": "high",
      "message": "High swarm probability detected",
      "created_at": "2025-12-22T10:15:00Z",
      "resolved": false,
      "action_required": true
    }
  ]
}
```

---

### 4. User Management

#### POST /users/register

**Create new user account**

```bash
POST /api/v1/users/register
Content-Type: application/json

{
  "username": "beekeeper@example.com",
  "password": "secure_password_123",
  "full_name": "John Beekeeper",
  "user_type": "beekeeper",
  "location": "Navi Mumbai, India"
}

Response (201):
{
  "user_id": "usr_001",
  "username": "beekeeper@example.com",
  "user_type": "beekeeper",
  "created_at": "2025-12-22T11:21:00Z",
  "message": "Account created successfully"
}
```

---

#### GET /users/profile

**Get current user profile**

```bash
GET /api/v1/users/profile
Authorization: Bearer <token>

Response (200):
{
  "user_id": "usr_001",
  "username": "beekeeper@example.com",
  "full_name": "John Beekeeper",
  "user_type": "beekeeper",
  "location": "Navi Mumbai, India",
  "colonies_count": 15,
  "detections_count": 87,
  "honey_reports_count": 23,
  "created_at": "2025-12-01T00:00:00Z",
  "last_login": "2025-12-22T11:21:00Z"
}
```

---

### 5. Health Check

#### GET /health

**System health status**

```bash
GET /api/v1/health

Response (200):
{
  "status": "healthy",
  "timestamp": "2025-12-22T11:21:00Z",
  "services": {
    "api": "operational",
    "database": "operational",
    "cache": "operational",
    "ml_models": "operational"
  },
  "uptime_hours": 720,
  "version": "1.0.0"
}
```

---

## Rate Limiting

All endpoints have rate limits:

```
Free Tier:
  - 100 requests/hour
  - 10 colony detections/day
  - 5 honey analyses/day

Premium Tier:
  - 10,000 requests/hour
  - Unlimited detections
  - Unlimited analyses
```

**Rate Limit Headers**:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1703251260
```

---

## Error Codes

| Code | Status | Meaning |
|------|--------|---------|
| 200 | OK | Successful request |
| 201 | Created | Resource created |
| 400 | Bad Request | Invalid input |
| 401 | Unauthorized | Missing/invalid token |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource not found |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server error |
| 503 | Service Unavailable | Service down |

---

## SDKs & Client Libraries

### Python
```python
from rockbees_sdk import RockBeesClient

client = RockBeesClient(api_key="your_api_key")
detection = client.colonies.detect(image_path="colony.jpg")
print(detection.colonies_detected)
```

### JavaScript/TypeScript
```javascript
import { RockBeesAPI } from 'rockbees-js';

const client = new RockBeesAPI({ apiKey: 'your_api_key' });
const detection = await client.colonies.detect(imageFile);
console.log(detection.coloniesDetected);
```

---

## Webhook Events

Subscribe to events:

```bash
POST /api/v1/webhooks
Authorization: Bearer <token>

{
  "event_type": "detection_complete",
  "url": "https://yourdomain.com/webhooks/detection"
}

# Webhook payload:
POST https://yourdomain.com/webhooks/detection
{
  "event": "detection_complete",
  "data": {
    "detection_id": "det_123",
    "colonies_detected": 3
  }
}
```

---

**Last Updated**: December 2025  
**Version**: 1.0.0  
**Status**: Active