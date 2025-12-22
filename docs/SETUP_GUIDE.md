# 🛠️ Setup Guide
## RockBees - Complete Installation Instructions

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [System Requirements](#system-requirements)
3. [Local Development Setup](#local-development-setup)
4. [Docker Setup (Recommended)](#docker-setup-recommended)
5. [Database Configuration](#database-configuration)
6. [ML Model Download](#ml-model-download)
7. [Environment Variables](#environment-variables)
8. [Testing the Setup](#testing-the-setup)
9. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Software
- **Python 3.9+** - [Download](https://www.python.org/downloads/)
- **Node.js 16+** - [Download](https://nodejs.org/)
- **Git** - [Download](https://git-scm.com/)
- **PostgreSQL 12+** - [Download](https://www.postgresql.org/download/)
- **Redis 6+** - [Download](https://redis.io/download/)

### Optional (Recommended)
- **Docker & Docker Compose** - [Download](https://www.docker.com/products/docker-desktop/)
- **Visual Studio Code** - [Download](https://code.visualstudio.com/)
- **Postman** - [Download](https://www.postman.com/downloads/)

---

## System Requirements

### Minimum Specifications
```
CPU:        4 cores (Intel/AMD)
RAM:        8 GB
Disk:       20 GB free space
OS:         Windows 10+, macOS 10.14+, or Linux (Ubuntu 18.04+)
GPU:        Optional (NVIDIA with CUDA for faster ML inference)
```

### Recommended Specifications
```
CPU:        8+ cores
RAM:        16+ GB
Disk:       50 GB+ SSD
GPU:        NVIDIA RTX 3060 or better (with CUDA 11.8)
```

---

## Local Development Setup

### Step 1: Clone Repository

```bash
# Clone the repository
git clone https://github.com/SolvyrEryx/rockbees-colony-monitoring-and-honey-ai.git
cd RockBees

# Create main directory structure
mkdir -p backend frontend/web frontend/mobile ml tests scripts
```

### Step 2: Backend Setup

#### Create Python Virtual Environment
```bash
# Create venv
python -m venv venv

# Activate venv
# On Linux/macOS:
source venv/bin/activate

# On Windows:
venv\Scripts\activate
```

#### Install Backend Dependencies
```bash
cd backend

# Install required packages
pip install -r requirements.txt

# If requirements.txt doesn't exist, install key packages:
pip install fastapi==0.104.1
pip install uvicorn==0.24.0
pip install sqlalchemy==2.0.23
pip install psycopg2-binary==2.9.9
pip install python-dotenv==1.0.0
pip install pydantic==2.4.2
pip install pydantic-settings==2.0.3
pip install opencv-python==4.8.1.78
pip install tensorflow==2.13.0
pip install torch==2.0.1
pip install ultralytics==8.0.201
pip install scikit-image==0.22.0
pip install numpy==1.24.3
pip install pillow==10.0.1
pip install python-multipart==0.0.6
pip install aiofiles==23.2.1
pip install celery==5.3.4
pip install redis==5.0.0
pip install pytest==7.4.3
pip install pytest-asyncio==0.21.1
```

#### Configure Environment
```bash
# Copy example env file
cp .env.example .env

# Edit .env with your configuration (see Environment Variables section)
nano .env  # or use your preferred editor
```

#### Initialize Database
```bash
# Create database user (in PostgreSQL)
# Run this in PostgreSQL terminal:
CREATE USER rockbees_user WITH PASSWORD 'your_secure_password';
CREATE DATABASE rockbees_db OWNER rockbees_user;
CREATE EXTENSION postgis;

# From Python:
python scripts/setup_db.py
```

#### Start Backend Server
```bash
# From backend directory
uvicorn main:app --reload --host 0.0.0.0 --port 8000

# You should see:
# INFO:     Uvicorn running on http://0.0.0.0:8000
# INFO:     Application startup complete
```

Visit API docs: http://localhost:8000/docs

### Step 3: Frontend Setup (Web Dashboard)

```bash
cd ../frontend/web

# Install dependencies
npm install

# Copy environment variables
cp .env.example .env

# Edit .env as needed
nano .env

# Start development server
npm start

# App opens at http://localhost:3000
```

### Step 4: Start Redis (Required)

```bash
# On macOS (using Homebrew)
brew services start redis

# On Linux
sudo systemctl start redis-server

# On Windows (WSL)
wsl sudo service redis-server start

# Or using Docker
docker run -d -p 6379:6379 redis:latest
```

---

## Docker Setup (Recommended)

### Using Docker Compose (All-in-One)

```bash
# From project root
cd RockBees

# Build all services
docker-compose build

# Start all services
docker-compose up -d

# View logs
docker-compose logs -f

# Useful commands:
docker-compose ps                  # Show running containers
docker-compose logs api            # View API logs
docker-compose exec api bash       # Access container shell
docker-compose down               # Stop all services
```

### Docker Compose File Structure

```yaml
version: '3.8'

services:
  postgres:
    image: postgis/postgis:15-3.3
    environment:
      POSTGRES_USER: rockbees_user
      POSTGRES_PASSWORD: secure_password
      POSTGRES_DB: rockbees_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  api:
    build: ./backend
    command: uvicorn main:app --reload --host 0.0.0.0 --port 8000
    environment:
      DATABASE_URL: postgresql://rockbees_user:secure_password@postgres:5432/rockbees_db
      REDIS_URL: redis://redis:6379
    ports:
      - "8000:8000"
    depends_on:
      - postgres
      - redis
    volumes:
      - ./backend:/app

  frontend:
    build: ./frontend/web
    ports:
      - "3000:3000"
    environment:
      REACT_APP_API_URL: http://localhost:8000
    depends_on:
      - api

volumes:
  postgres_data:
```

### Access Services After Docker Compose Up

```
PostgreSQL:  localhost:5432
Redis:       localhost:6379
API:         http://localhost:8000
API Docs:    http://localhost:8000/docs
Frontend:    http://localhost:3000
```

---

## Database Configuration

### PostgreSQL Setup

#### Create Database (Manual)
```sql
-- Connect as superuser
psql -U postgres

-- Create user
CREATE USER rockbees_user WITH PASSWORD 'your_secure_password';

-- Create database
CREATE DATABASE rockbees_db OWNER rockbees_user;

-- Connect to database
\c rockbees_db

-- Enable PostGIS extension
CREATE EXTENSION postgis;

-- Create initial tables (from SQLAlchemy models)
-- This happens automatically on first API run

-- Grant permissions
GRANT ALL PRIVILEGES ON DATABASE rockbees_db TO rockbees_user;
GRANT ALL PRIVILEGES ON SCHEMA public TO rockbees_user;
```

#### Connection String
```
postgresql://rockbees_user:your_secure_password@localhost:5432/rockbees_db
```

#### Update PostgreSQL for PostGIS

If you encounter PostGIS issues:

```bash
# Install PostGIS (macOS with Homebrew)
brew install postgis

# Install PostGIS (Ubuntu/Debian)
sudo apt-get install postgresql-15-postgis-3

# Create extension in database
psql -U postgres -d rockbees_db -c "CREATE EXTENSION postgis;"
```

---

## ML Model Download

### Download Pre-trained Models

```bash
cd ml/models

# Download YOLOv8 model (first run auto-downloads)
# Or manually download from Ultralytics:
wget https://github.com/ultralytics/assets/releases/download/v8.0.0/yolov8m.pt

# Download TensorFlow models for honey analysis
# Create models directory if needed
mkdir -p honey_classifier
cd honey_classifier

# Download model weights (example)
# Place in: ml/models/honey_classifier/model.h5
```

### Model Paths in Config

```python
# backend/config/settings.py

YOLO_MODEL_PATH = "ml/models/yolov8m.pt"
HONEY_MODEL_PATH = "ml/models/honey_classifier/model.h5"
BEHAVIOR_MODEL_PATH = "ml/models/behavior_classifier/model.h5"
```

---

## Environment Variables

### Backend (.env file)

```bash
# Database
DATABASE_URL=postgresql://rockbees_user:your_secure_password@localhost:5432/rockbees_db

# Redis
REDIS_URL=redis://localhost:6379/0

# API Configuration
API_HOST=0.0.0.0
API_PORT=8000
DEBUG=True
SECRET_KEY=your-super-secret-key-change-this-in-production

# JWT Configuration
JWT_SECRET=your-jwt-secret-key
JWT_ALGORITHM=HS256
JWT_EXPIRATION=3600

# AWS (for S3 storage)
AWS_ACCESS_KEY_ID=your_aws_key
AWS_SECRET_ACCESS_KEY=your_aws_secret
AWS_S3_BUCKET=rockbees-bucket
AWS_REGION=us-east-1

# ML Models
YOLO_CONFIDENCE_THRESHOLD=0.5
YOLO_IOU_THRESHOLD=0.45
MAX_IMAGE_SIZE=2048
INFERENCE_BATCH_SIZE=1

# Email Configuration
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your-email@gmail.com
SMTP_PASSWORD=your-app-password

# Celery
CELERY_BROKER_URL=redis://localhost:6379/1
CELERY_RESULT_BACKEND=redis://localhost:6379/2
```

### Frontend (.env file)

```bash
# API Configuration
REACT_APP_API_URL=http://localhost:8000
REACT_APP_API_TIMEOUT=30000

# Maps
REACT_APP_MAPBOX_TOKEN=your_mapbox_token

# Analytics (Optional)
REACT_APP_GOOGLE_ANALYTICS_ID=your_ga_id

# Environment
REACT_APP_ENV=development
```

---

## Testing the Setup

### Test Backend API

```bash
# Test API is running
curl http://localhost:8000/health

# Expected response:
# {"status":"healthy","version":"1.0.0"}

# Test with Postman
# 1. Open Postman
# 2. Import API collection from docs/postman_collection.json
# 3. Run requests
```

### Test Database Connection

```bash
# From backend directory
python -c "
from backend.database.connection import engine
from sqlalchemy import text
with engine.connect() as conn:
    result = conn.execute(text('SELECT 1'))
    print('Database connected successfully')
"
```

### Test ML Models

```bash
# Download test image
wget https://example.com/test_colony.jpg

# Test detection
python backend/models/detection_model.py test_colony.jpg

# Expected output:
# Detected 3 colonies with 89% confidence
```

### Run Unit Tests

```bash
# Backend tests
cd backend
pytest tests/unit/ -v

# With coverage
pytest tests/unit/ --cov=backend --cov-report=html
open htmlcov/index.html

# Frontend tests
cd ../frontend/web
npm test
```

---

## Troubleshooting

### Issue: PostgreSQL Connection Failed

**Symptom**: `psycopg2.OperationalError: could not connect to server`

**Solution**:
```bash
# Check if PostgreSQL is running
# macOS
brew services list | grep postgres

# Ubuntu
sudo systemctl status postgresql

# Start PostgreSQL
# macOS
brew services start postgresql

# Ubuntu
sudo systemctl start postgresql

# Check connection string in .env
DATABASE_URL=postgresql://username:password@localhost:5432/dbname
```

### Issue: Redis Connection Failed

**Symptom**: `ConnectionError: Error 111 connecting to localhost:6379`

**Solution**:
```bash
# Check if Redis is running
redis-cli ping
# Expected: PONG

# Start Redis
# macOS
brew services start redis

# Ubuntu
sudo systemctl start redis-server

# Docker
docker run -d -p 6379:6379 redis:latest
```

### Issue: ML Models Not Found

**Symptom**: `FileNotFoundError: model.pt not found`

**Solution**:
```bash
# Download models
cd ml/models
pip install ultralytics
python -c "from ultralytics import YOLO; YOLO('yolov8m.pt')"

# Check paths in settings.py match actual locations
ls -la ml/models/
```

### Issue: GPU Not Detected

**Symptom**: Using CPU instead of GPU despite having NVIDIA card

**Solution**:
```bash
# Install CUDA
# Check CUDA installation
nvidia-smi

# Install CUDA-compatible TensorFlow
pip install tensorflow[and-cuda]

# Or for PyTorch
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
```

### Issue: Port Already in Use

**Symptom**: `Address already in use: ('0.0.0.0', 8000)`

**Solution**:
```bash
# Find process using port 8000
# macOS/Linux
lsof -i :8000
kill -9 <PID>

# Windows
netstat -ano | findstr :8000
taskkill /PID <PID> /F

# Or use different port
uvicorn main:app --port 8001
```

### Issue: CORS Errors

**Symptom**: `Access-Control-Allow-Origin` errors in browser console

**Solution**:
```python
# backend/main.py
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:3001"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Issue: OutOfMemory Error During ML Inference

**Symptom**: `cuda out of memory` or `RuntimeError: CUDA out of memory`

**Solution**:
```bash
# Reduce batch size in settings.py
INFERENCE_BATCH_SIZE=1  # Reduce from default

# Reduce image size
MAX_IMAGE_SIZE=1024  # From 2048

# Or disable GPU
export CUDA_VISIBLE_DEVICES=""
```

---

## Next Steps

After successful setup:

1. **Read API Documentation**: See [API_DOCUMENTATION.md](API_DOCUMENTATION.md)
2. **Explore Architecture**: See [ARCHITECTURE.md](ARCHITECTURE.md)
3. **Run Examples**: Check `backend/examples/` directory
4. **Train Models**: See [TRAINING_GUIDE.md](TRAINING_GUIDE.md)
5. **Deploy**: See [DEPLOYMENT.md](DEPLOYMENT.md)

---

## Support

**Documentation**: [docs/](./docs/)  
**Issues**: [GitHub Issues](https://github.com/SolvyrEryx/rockbees-colony-monitoring-and-honey-ai/issues)  
**Email**: [rahuljaiprakashden@gmail.com](mailto:rahuljaiprakashden@gmail.com)  

---

**Last Updated**: December 2025  
**Version**: 1.0.0  
**Status**: Active