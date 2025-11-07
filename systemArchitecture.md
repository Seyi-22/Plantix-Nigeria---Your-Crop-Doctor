# System Architecture - Plantix Nigeria

## System Overview

Plantix Nigeria is a mobile-first agricultural application with offline-capable AI disease diagnosis, voice-first interface, and real-time community disease tracking. The architecture is designed to work seamlessly in low-connectivity rural environments while providing real-time features when online.

**Core Technical Requirements:**
- Offline disease diagnosis without internet
- Voice input/output in 4 Nigerian languages
- Real-time geolocation for agro-dealer network
- Community disease outbreak tracking
- Scalable to support 100,000+ concurrent users

---

## Technology Stack

### Frontend - Mobile Application

**Platform:** React Native  
**Why:** Cross-platform development (iOS & Android from single codebase), native performance, large community support

**Core Libraries:**
- `react-native-paper` - Material Design UI components
- `@react-navigation/native` - App navigation
- `redux-toolkit` - State management
- `react-native-camera` - Crop photo capture
- `react-native-maps` - Dealer location mapping
- `react-native-voice` - Voice input integration
- `react-native-tts` - Text-to-speech for voice output
- `react-native-fs` - File system access for offline storage

**Offline Storage:**
- `@react-native-async-storage/async-storage` - Key-value storage
- `react-native-sqlite-storage` - Local database for disease information

**AI/ML Integration:**
- `@tensorflow/tfjs-react-native` - TensorFlow Lite integration
- Custom TFLite models for on-device inference

---

### Backend - Server Infrastructure

**Runtime Environment:** Node.js v20 LTS  
**Framework:** Express.js v4.18+

**API Architecture:**
- RESTful API design
- JWT-based authentication
- Rate limiting for API protection
- CORS enabled for mobile app access

**Core Packages:**
```
- express - Web framework
- jsonwebtoken - Authentication tokens
- bcrypt - Password hashing
- helmet - Security middleware
- morgan - Request logging
- express-validator - Input validation
- multer - File upload handling (crop images)
- socket.io - Real-time outbreak alerts
- node-cron - Scheduled tasks (price updates)
```

**External API Integrations:**
- **Google Cloud Speech-to-Text:** Voice recognition with custom Nigerian language models
- **Google Cloud Text-to-Speech:** Voice output
- **Google Maps Geocoding API:** Location services
- **Twilio SMS Gateway:** Backup alert system for offline farmers
- **Paystack Payment API:** Premium subscription processing

---

### Database Architecture

#### **Primary Database: PostgreSQL v15**

**Why PostgreSQL:**
- Excellent geospatial support (PostGIS extension)
- ACID compliance for data integrity
- Robust JSON support for flexible schema
- Strong performance for complex queries

**Schema Design:**

**Users Table:**
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  phone_number VARCHAR(15) UNIQUE NOT NULL,
  full_name VARCHAR(100),
  preferred_language VARCHAR(10) DEFAULT 'en',
  state VARCHAR(50),
  lga VARCHAR(50),
  location GEOGRAPHY(POINT),
  is_premium BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT NOW(),
  last_active TIMESTAMP
);
```

**Crops Table:**
```sql
CREATE TABLE crops (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50) NOT NULL,
  scientific_name VARCHAR(100),
  local_name_yoruba VARCHAR(50),
  local_name_igbo VARCHAR(50),
  local_name_hausa VARCHAR(50),
  growing_season VARCHAR(20)
);
```

**Diseases Table:**
```sql
CREATE TABLE diseases (
  id SERIAL PRIMARY KEY,
  crop_id INTEGER REFERENCES crops(id),
  disease_name VARCHAR(100) NOT NULL,
  scientific_name VARCHAR(100),
  symptoms TEXT,
  severity_level VARCHAR(20),
  treatment_chemical TEXT,
  treatment_organic TEXT,
  prevention_tips TEXT
);
```

**Agro_Dealers Table:**
```sql
CREATE TABLE agro_dealers (
  id SERIAL PRIMARY KEY,
  business_name VARCHAR(100) NOT NULL,
  owner_name VARCHAR(100),
  phone_number VARCHAR(15),
  address TEXT,
  location GEOGRAPHY(POINT),
  state VARCHAR(50),
  lga VARCHAR(50),
  inventory JSONB,
  verified BOOLEAN DEFAULT FALSE,
  last_price_update TIMESTAMP
);
```

**Diagnosis_Reports Table:**
```sql
CREATE TABLE diagnosis_reports (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  crop_id INTEGER REFERENCES crops(id),
  disease_id INTEGER REFERENCES diseases(id),
  image_url TEXT,
  location GEOGRAPHY(POINT),
  confidence_score DECIMAL(5,2),
  treatment_applied VARCHAR(200),
  outcome VARCHAR(20),
  reported_at TIMESTAMP DEFAULT NOW(),
  is_public BOOLEAN DEFAULT TRUE
);
```

**Outbreak_Alerts Table:**
```sql
CREATE TABLE outbreak_alerts (
  id SERIAL PRIMARY KEY,
  disease_id INTEGER REFERENCES diseases(id),
  center_location GEOGRAPHY(POINT),
  radius_km INTEGER,
  affected_count INTEGER,
  severity VARCHAR(20),
  alert_sent_at TIMESTAMP DEFAULT NOW(),
  resolved BOOLEAN DEFAULT FALSE
);
```

#### **Mobile Database: SQLite (Offline Storage)**

Stored locally on each device for offline functionality:

**Tables:**
- `crops_local` - Copy of all supported crops
- `diseases_local` - Disease database with symptoms and treatments
- `user_diagnoses` - User's personal diagnosis history
- `cached_dealers` - Nearby dealers (synced when online)
- `ml_models` - TensorFlow Lite model metadata

**Sync Strategy:**
- Download full disease database on first install
- Sync user data when online
- Update dealer inventory weekly
- Upload diagnoses when connectivity restored

---

## AI/ML Architecture

### Disease Detection Model

**Framework:** TensorFlow → TensorFlow Lite (for mobile deployment)

**Model Architecture:**
- Base: MobileNetV3 (optimized for mobile devices)
- Custom classification head for Nigerian crops
- Input: 224x224 RGB images
- Output: 50 disease classes + confidence scores

**Training Dataset:**
- 50,000+ images of 10 Nigerian crops
- 5 diseases per crop (including healthy state)
- Augmented with variations (lighting, angles, backgrounds)
- Validated by Nigerian agronomists

**Model Performance:**
- Accuracy: 87% on validation set
- Inference time: <2 seconds on mid-range devices
- Model size: 45MB (optimized for mobile)

**On-Device Inference Pipeline:**
```
1. User captures crop image
2. Image preprocessed (resize, normalize)
3. TFLite model inference runs locally
4. Top 3 predictions with confidence scores
5. Results mapped to treatment recommendations
```

---

## Component Communication Architecture

### System Flow Diagram

```
┌─────────────────────────────────────────────┐
│         Mobile App (React Native)          │
│                                             │
│  ┌─────────────┐  ┌──────────────────┐    │
│  │   Voice I/O │  │  Camera Module   │    │
│  └──────┬──────┘  └────────┬─────────┘    │
│         │                   │               │
│  ┌──────▼───────────────────▼──────┐       │
│  │    TensorFlow Lite Engine       │       │
│  │  (Offline Disease Detection)    │       │
│  └──────┬───────────────────────────┘       │
│         │                                    │
│  ┌──────▼───────────────────────────┐       │
│  │    SQLite (Local Database)       │       │
│  └──────┬───────────────────────────┘       │
└─────────┼─────────────────────────────────┘
          │
          │ HTTPS/REST API (when online)
          │
┌─────────▼─────────────────────────────────┐
│       Backend API (Node.js/Express)       │
│                                             │
│  ┌──────────────┐  ┌──────────────────┐   │
│  │ Auth Service │  │  Dealer Service  │   │
│  └──────┬───────┘  └────────┬─────────┘   │
│         │                    │              │
│  ┌──────▼────────────────────▼─────────┐  │
│  │   PostgreSQL Database (PostGIS)     │  │
│  └──────┬──────────────────────────────┘  │
│         │                                   │
│  ┌──────▼──────────────────────────────┐  │
│  │   Socket.io (Real-time Alerts)     │  │
│  └─────────────────────────────────────┘  │
└───────────────────────────────────────────┘
          │
          │ External APIs
          │
┌─────────▼─────────────────────────────────┐
│          External Services                 │
│  ┌────────────────┐  ┌──────────────────┐ │
│  │ Google Speech  │  │  Google Maps     │ │
│  │    API         │  │     API          │ │
│  └────────────────┘  └──────────────────┘ │
│  ┌────────────────┐  ┌──────────────────┐ │
│  │   Twilio SMS   │  │    Paystack      │ │
│  └────────────────┘  └──────────────────┘ │
└───────────────────────────────────────────┘
```

### API Endpoints

**Authentication:**
```
POST /api/auth/register - User registration
POST /api/auth/login - User login
POST /api/auth/verify-otp - Phone verification
POST /api/auth/refresh-token - Token refresh
```

**Diagnosis:**
```
POST /api/diagnosis/submit - Submit new diagnosis
GET /api/diagnosis/history - User diagnosis history
PUT /api/diagnosis/:id/outcome - Update treatment outcome
```

**Dealers:**
```
GET /api/dealers/nearby?lat={lat}&lng={lng} - Find nearby dealers
GET /api/dealers/:id/inventory - Check dealer inventory
GET /api/dealers/:id/prices - Get current prices
```

**Outbreaks:**
```
GET /api/outbreaks/alerts - Get active outbreak alerts
POST /api/outbreaks/report - Report disease occurrence
GET /api/outbreaks/map - Get outbreak heatmap data
```

**Premium Features:**
```
POST /api/premium/subscribe - Start subscription
POST /api/premium/consultation/book - Book agronomist call
POST /api/premium/soil-test/request - Request soil testing
```

### Communication Protocols

**Mobile ↔ Backend:**
- Protocol: HTTPS (TLS 1.3)
- Format: JSON
- Authentication: JWT Bearer tokens
- Timeout: 30 seconds
- Retry logic: 3 attempts with exponential backoff

**Real-time Features:**
- Protocol: WebSocket (Socket.io)
- Used for: Outbreak alerts, live chat support
- Fallback: HTTP long-polling

**Offline Sync:**
- Queue system stores actions when offline
- Automatic sync when connection restored
- Conflict resolution: Last-write-wins

---

## Security Architecture

**Authentication & Authorization:**
- JWT tokens with 24-hour expiry
- Refresh tokens for extended sessions
- Phone number-based authentication (OTP via SMS)
- Role-based access control (User, Dealer, Admin, Agronomist)

**Data Protection:**
- All API communications over HTTPS
- Database encryption at rest (AES-256)
- Passwords hashed with bcrypt (12 rounds)
- PII data anonymized for analytics

**API Security:**
- Rate limiting: 100 requests/minute per user
- Input validation on all endpoints
- SQL injection prevention (parameterized queries)
- XSS protection via input sanitization
- CORS restricted to mobile app domains

---

## Scalability & Performance

**Backend Scaling:**
- Horizontal scaling via load balancer (AWS ELB)
- Containerized deployment (Docker)
- Auto-scaling based on CPU/memory metrics
- Database read replicas for high traffic

**CDN & Caching:**
- CloudFront CDN for static assets
- Redis caching for:
  - Dealer inventory (5-minute TTL)
  - Disease information (24-hour TTL)
  - User sessions
- Browser caching for mobile assets

**Database Optimization:**
- Indexed columns: location, disease_id, user_id, created_at
- Partitioning on diagnosis_reports by date
- Query optimization with EXPLAIN ANALYZE
- Connection pooling (50 max connections)

**Performance Targets:**
- API response time: <200ms (95th percentile)
- Image upload: <3 seconds
- Offline diagnosis: <2 seconds
- App startup time: <1.5 seconds

---

## Deployment Architecture

**Hosting:** AWS (Amazon Web Services)

**Infrastructure:**
- **Compute:** EC2 instances (t3.medium)
- **Database:** RDS PostgreSQL (Multi-AZ deployment)
- **Storage:** S3 for crop images
- **CDN:** CloudFront
- **Load Balancer:** Application Load Balancer
- **Monitoring:** CloudWatch + Sentry for error tracking

**CI/CD Pipeline:**
```
GitHub → GitHub Actions → Docker Build → AWS ECR → ECS Deployment
```

**Environments:**
- Development: dev.api.plantixnigeria.com
- Staging: staging.api.plantixnigeria.com
- Production: api.plantixnigeria.com


## Technical Feasibility Justification

### Why This Architecture Works

**1. Offline-First Design**
- TensorFlow Lite enables on-device AI without internet
- SQLite stores all critical data locally
- Proven in similar apps (Google Translate, Maps offline mode)

**2. Voice-First Interface**
- Google Speech API supports Nigerian languages
- Low bandwidth requirement (audio compression)
- Fallback to text input if voice fails

**3. Scalability**
- React Native handles 100,000+ DAU in similar apps
- PostgreSQL with PostGIS proven for geospatial apps (Uber, DoorDash)
- AWS infrastructure scales automatically

**4. Cost Effectiveness**
- React Native = single codebase for both platforms (50% dev time saved)
- TensorFlow Lite = no server costs for ML inference
- PostgreSQL = free to start, predictable scaling costs

**5. Nigerian Market Fit**
- Mobile-first (99% smartphone penetration among target users)
- Offline capability (addresses connectivity issues)
- Voice interface (addresses literacy barriers)
- Local language support (cultural relevance)


## Development Roadmap

**Phase 1 (Months 1-3):** MVP
- Basic disease detection (10 crops, 50 diseases)
- Voice interface (English + 1 local language)
- Offline diagnosis
- Simple dealer locator

**Phase 2 (Months 4-6):** Enhanced Features
- All 4 languages supported
- Community outbreak tracking
- Premium features
- Payment integration

**Phase 3 (Months 7-12):** Scale
- Expand to 20 crops
- Video consultations
- Government partnerships
- Analytics dashboard


## Risk Mitigation

**Technical Risks:**
- Model accuracy issues → Continuous retraining with user feedback
- Offline sync conflicts → Last-write-wins with manual override
- Device fragmentation → Testing on 20+ device models

**Operational Risks:**
- Dealer data accuracy → Weekly automated verification calls
- User adoption → Partner with extension workers for training
- Scalability → Load testing before major campaigns


## Monitoring & Analytics

**Application Monitoring:**
- Crash reporting (Sentry)
- Performance monitoring (New Relic)
- User analytics (Mixpanel)
- API metrics (CloudWatch)

**Key Metrics:**
- Daily Active Users (DAU)
- Diagnosis accuracy rate
- Average response time
- Conversion to premium
- Dealer engagement rate



**Document Version:** 1.0  
**Last Updated:** November 2025  
**Maintained By:** Plantix Nigeria Engineering Team
