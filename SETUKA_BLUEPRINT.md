# 🛡️ SETUKA BLUEPRINT — Complete Architecture & Tech Stack

> **Setuka** — IoT + AI-Driven Tourist Safety Ecosystem for Northeast India
> 
> *A comprehensive guide to the three-layer architecture, tech stack decisions, and engineering methodology.*

---

## 📋 TABLE OF CONTENTS

1. [Executive Summary](#executive-summary)
2. [System Architecture](#system-architecture)
3. [Tech Stack Overview](#tech-stack-overview)
4. [Module Breakdown](#module-breakdown)
5. [Data Flow Architecture](#data-flow-architecture)
6. [Key Technologies & Justification](#key-technologies--justification)
7. [Database Schema](#database-schema)
8. [API Architecture](#api-architecture)
9. [Development Methodology](#development-methodology)
10. [Deployment & Infrastructure](#deployment--infrastructure)

---

## EXECUTIVE SUMMARY

### Problem Statement
Adventure tourism is growing at **19% CAGR**, but safety infrastructure in mountain corridors remains stagnant, creating three critical gaps:

| Gap | Impact | Setuka's Solution |
|-----|--------|------------------|
| **Connectivity Void** | 40% of corridors are digital dark zones; satellite SOS (Garmin) is illegal in Indian border regions | Ground-based LoRa mesh networking with P2P relay capability |
| **Information Vacuum** | Authorities operate in silos with zero real-time crowd data | AI-powered authority dashboard with live clustering & crowd analytics |
| **Response Lag** | Emergency response delayed 15–30 min without coordinates/vitals, turning injuries fatal | Integrated device data + emergency SOS with triple-mode transmission |

### Setuka's Value Proposition
A **rental-model safety ecosystem** (₹50/day) offering:
- ✅ **End-to-end coverage** — from tourist planning to on-ground emergency response
- ✅ **Offline resilience** — works in low-connectivity zones via LoRa mesh
- ✅ **Legal compliance** — ground-based LoRa instead of banned satellite systems
- ✅ **Military-grade security** — blockchain-backed digital ID with offline verification

---

## SYSTEM ARCHITECTURE

### High-Level 3-Layer Ecosystem

```
┌─────────────────────────────────────────────────────────────────┐
│                        LAYER 1: TOURIST                         │
│                   Progressive Web App (PWA)                     │
│          Next.js 14 | Installable on Android & iOS             │
├─────────────────────────────────────────────────────────────────┤
│  • Live Safety Score       • Itinerary Planner                  │
│  • Blockchain Digital ID   • Dynamic Geofencing                 │
│  • Emergency SOS Shield    • Background Location Tracking       │
└────────────────┬──────────────────────────────────┬─────────────┘
                 │ HTTPS + LoRa Mesh              │
                 │ Location Data + SOS            │
                 │ Blockchain Transactions        │
                 ▼                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                  LAYER 2: IoT WEARABLE DEVICE                   │
│        Custom-Built Hardware | 9-Phase Development             │
├─────────────────────────────────────────────────────────────────┤
│  • Sensors                     • Triple-Mode SOS                │
│    - LoRa SX1262               1. Direct Cloud (Internet)       │
│    - GPS Quectel EC200U        2. SMS (GSM Partial Signal)      │
│    - SpO₂ / Pulse Oximeter     3. LoRa P2P Mesh (No Net)        │
│    - GNSS Module             • Auto-SOS Detection              │
│    - OLED Display              on Critical Health Anomalies     │
└────────────────┬──────────────────────────────────┬─────────────┘
                 │ FastAPI + WebSockets            │
                 │ Real-Time Location + Vitals     │
                 │ Anomaly + Alert Events          │
                 ▼                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│              LAYER 3: AUTHORITY COMMAND CENTER                  │
│         Streamlit Dashboard | Real-Time Decision Support       │
├─────────────────────────────────────────────────────────────────┤
│  • Live Tourist Map            • Anomaly Detection              │
│  • Crowd Safety Scoring        • Police Reallocation AI         │
│    (DBSCAN Clustering)         • Inactivity Flagging            │
│                                • Route Deviation Alerts         │
└─────────────────────────────────────────────────────────────────┘
```

---

### Detailed Layer Architecture

#### **LAYER 1: Tourist App (PWA)**

```
┌───────────────────────────────────────────────────────────┐
│                  FRONTEND (Next.js 14)                    │
├───────────────────────────────────────────────────────────┤
│  Pages:                                                   │
│  ├─ Landing (Onboarding & Sign-up)                       │
│  ├─ Dashboard (Safety Score, Map, Itinerary)             │
│  ├─ Trip Planner (Route Selection & Ranking)             │
│  ├─ Profile & Digital ID Verification                    │
│  ├─ Emergency Alerts Popup                               │
│  └─ Offline Indicator & Sync Manager                     │
│                                                           │
│  Components:                                              │
│  ├─ Interactive Map (Leaflet + Mapbox GL)                │
│  ├─ QR Code Display & Scanner                            │
│  ├─ Real-Time Chart Updates (Recharts, Plotly)           │
│  ├─ PWA Install Prompt                                   │
│  └─ Theme Toggle (Light/Dark Mode)                       │
└────────────┬────────────────────────────────┬────────────┘
             │                                │
    ┌────────▼────────┐            ┌──────────▼────────┐
    │  REACT CONTEXT  │            │  SERVICE WORKER   │
    ├─────────────────┤            ├───────────────────┤
    │ • Auth Context  │            │ • Background      │
    │ • Location      │            │   Location Sync   │
    │   Context       │            │ • Periodic Sync   │
    │ • Session       │            │ • Offline Cache   │
    └─────────────────┘            └───────────────────┘
             │                                │
             └────────────┬───────────────────┘
                          │
              ┌───────────▼────────────┐
              │   NEXT.JS API ROUTES   │
              ├────────────────────────┤
              │ /api/auth/login        │
              │ /api/location/track    │
              │ /api/safety-score      │
              │ /api/blockchain/tx     │
              └─────────┬──────────────┘
                        │
                        ▼
              ┌──────────────────────┐
              │  EXTERNAL SERVICES   │
              ├──────────────────────┤
              │ Safety Score API     │
              │ Groq LLM (Narratives)│
              │ Polygon Blockchain   │
              │ MongoDB Atlas        │
              │ Mapbox/Leaflet       │
              └──────────────────────┘
```

#### **LAYER 2: IoT Wearable Architecture**

```
┌─────────────────────────────────────────────┐
│        WEARABLE HARDWARE STACK              │
├─────────────────────────────────────────────┤
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │    MAIN CONTROLLER: STM32L476      │   │
│  │    (ARM Cortex-M4, 80MHz)          │   │
│  └────────────┬────────────────────────┘   │
│               │                            │
│   ┌───────────┼───────────┐                │
│   │           │           │                │
│   ▼           ▼           ▼                │
│  ┌─────┐   ┌─────┐   ┌────────┐           │
│  │LoRa │   │ GPS │   │ Health │           │
│  │SX1262   │EC200U    │Sensors │           │
│  └─────┘   └─────┘   └────────┘           │
│   │           │           │                │
│   │   ┌───────┼───────┐   │                │
│   │   │       │       │   │                │
│   ▼   ▼       ▼       ▼   ▼                │
│  ┌────────────────────────────────────┐   │
│  │    TRIPLE-MODE SOS ENGINE          │   │
│  ├────────────────────────────────────┤   │
│  │ Mode 1: Direct Cloud (MQTT/HTTPS)  │   │
│  │ Mode 2: SMS via GSM                │   │
│  │ Mode 3: LoRa P2P Mesh Relay        │   │
│  └────────────────────────────────────┘   │
│                                             │
│  ┌────────────────────────────────────┐   │
│  │    OLED DISPLAY (128x64)           │   │
│  │    Status + Alerts + Navigation    │   │
│  └────────────────────────────────────┘   │
│                                             │
│  Battery: 3.7V LiPo, 2000mAh               │
│  Runtime: ~24 hours (with 30s updates)     │
└─────────────────────────────────────────────┘
```

#### **LAYER 3: Authority Dashboard**

```
┌─────────────────────────────────────────────┐
│    STREAMLIT DASHBOARD (Command Center)     │
├─────────────────────────────────────────────┤
│                                             │
│  HOME PAGE (main.py):                       │
│  ├─ Live Tourist Map (Folium)               │
│  └─ Real-Time Statistics Panel              │
│                                             │
│  PAGE 1: Crowd Safety Scoring               │
│  ├─ DBSCAN Clustering                       │
│  ├─ Risk Tier Visualization                 │
│  ├─ Heatmap Layers                          │
│  └─ Zone-wise Analytics                     │
│                                             │
│  PAGE 2: AI Recommendations                 │
│  ├─ LangChain Agent Chain                   │
│  ├─ LLM Insights (Groq/Gemini)              │
│  ├─ Patrol Reallocation Suggestions         │
│  └─ Crowd Redistribution Plans              │
│                                             │
│  PAGE 3: Tourist Anomaly Detection          │
│  ├─ Isolation Forest Pipeline               │
│  ├─ Route Deviation Flagging                │
│  ├─ Inactivity Detection                    │
│  └─ Real-Time Alerts                        │
│                                             │
│  BACKEND (api.py):                          │
│  ├─ FastAPI Endpoints                       │
│  ├─ WebSocket for Real-Time Updates         │
│  └─ Data Persistence (JSON)                 │
│                                             │
│  UTILS:                                     │
│  ├─ Clustering Logic (DBSCAN)               │
│  ├─ AI Agent Integration                    │
│  ├─ Map Layer Rendering (Folium)            │
│  ├─ Data Preprocessing                      │
│  └─ Geocoding & Reverse Geocoding           │
│                                             │
└─────────────────────────────────────────────┘
```

---

## TECH STACK OVERVIEW

### Frontend Stack

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| **Runtime** | Node.js | Latest | Package management & build tools |
| **Framework** | Next.js | 14+ | Server-side rendering, API routes, PWA support |
| **Language** | TypeScript | 5+ | Type-safe component development |
| **Styling** | Tailwind CSS | 4.1.9 | Utility-first CSS framework |
| **Component Lib** | Radix UI | Latest | Accessible, unstyled component primitives |
| **State Mgmt** | React Context + Hooks | 19.2.4 | Light-weight state for auth, location, session |
| **Mapping** | Leaflet + Mapbox GL | 1.9.4 & 3.15.0 | Interactive maps, real-time layers |
| **Forms** | React Hook Form + Zod | 7.60.0 & 3.25.67 | Form validation & submission |
| **Charts** | Recharts, Plotly.js | 2.15.4 & 3.1.0 | Real-time data visualization |
| **QR Codes** | qrcode, jsqr | 1.5.4 & 1.4.0 | QR generation & scanning |
| **Offline** | next-pwa, Workbox | 5.6.0 & 7.3.0 | Service Worker, offline caching |
| **Auth** | JWT + bcryptjs | 9.0.2 & 3.0.2 | Secure authentication |
| **Database** | MongoDB | 6.19.0 | NoSQL persistence layer |

### Backend Stack

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| **Framework** | FastAPI | Latest | RESTful API, WebSocket support |
| **Server** | Uvicorn | Latest | ASGI server for FastAPI |
| **Data Handling** | Pandas, NumPy | 2.1.3 & 1.26.2 | DataFrames, numerical computing |
| **ML/Clustering** | scikit-learn, XGBoost | 1.3.2 & 2.0.3 | DBSCAN, Isolation Forest models |
| **Geospatial** | GeoPandas, Shapely | 0.14.0 & 2.0.2 | GIS operations, spatial queries |
| **Mapping** | Folium, Streamlit-Folium | 0.15.0 & 0.15.0 | Server-side map rendering |
| **Dashboard** | Streamlit | 1.28.0 | Interactive dashboard UI |
| **Plotting** | Plotly | 5.17.0 | Interactive data visualizations |
| **LLM Integration** | LangChain, Groq SDK | 0.1.16+ | AI agent chains, LLM calls |
| **AI Models** | Google Gemini | 0.3.2 | Generative AI for narratives |
| **Data Science** | joblib | 1.3.2 | Model persistence & caching |

### Blockchain Stack

| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| **Chain** | Polygon Amoy | Testnet | Layer 2 scaling, low-cost deployments |
| **Library** | ethers.js, viem | 5.8.0 & 2.47.0 | Web3 contract interactions |
| **Smart Contracts** | Solidity | ^0.8.0 | Digital ID contract logic |
| **Deployment** | Hardhat | Latest | Contract compilation & deployment |

### Infrastructure & DevOps

| Tool | Purpose |
|------|---------|
| **Package Managers** | npm, pip |
| **Version Control** | Git |
| **Environment Mgmt** | python-dotenv (Python), .env files |
| **API Validation** | Pydantic |
| **HTTP Client** | requests (Python), fetch (JS) |
| **Task Queue** | None (currently sync-only) |
| **Caching** | In-memory (JSON files) |

---

## MODULE BREAKDOWN

### Module 1: Tourist App (tourist-app/)

#### Directory Structure
```
tourist-app/
├── app/
│   ├── layout.tsx              # PWA manifest, metadata, global setup
│   ├── page.tsx                # Root redirect
│   ├── globals.css             # Global styles
│   ├── landing/                # Onboarding & landing page
│   ├── dashboard/              # Main safety dashboard
│   ├── auth/                   # Login/signup pages
│   ├── verify/                 # Blockchain ID verification
│   └── api/                    # Next.js API routes
├── components/
│   ├── Interactive components (map, charts, forms)
│   ├── Emergency SOS popup
│   ├── Geofence manager
│   ├── Digital ID card display
│   ├── PWA installer
│   ├── Location tracker
│   ├── Auth forms
│   └── UI primitives (Radix-based)
├── lib/
│   ├── api-client.ts           # API wrapper functions
│   ├── auth-context.tsx        # Auth state management
│   ├── location-context.tsx    # Location tracking state
│   ├── session-context.tsx     # Session management
│   ├── blockchain-service.ts   # Web3 integration
│   ├── digital-id.ts           # Digital ID logic
│   ├── mongodb.ts              # db connection
│   └── utils.ts                # Helper functions
├── hooks/
│   ├── use-mobile.ts           # Mobile detection
│   ├── use-geofence.ts         # Geofencing logic
│   ├── use-pwa-install.ts      # PWA install detection
│   └── use-toast.ts            # Toast notifications
├── blockchain/
│   ├── TouristID.sol           # Smart contract
│   ├── blockchain-config.ts    # Polygon config
│   ├── blockchain-service.ts   # Web3 calls
│   └── deployment scripts
├── public/
│   ├── manifest.json           # PWA manifest
│   ├── site icons
│   └── source docs (data sources)
└── tsconfig.json, next.config.mjs, package.json
```

#### Key Features
1. **Live Safety Dashboard**
   - Real-time safety score (0–100)
   - Crime, accident, road-condition indexes
   - AI-generated safety narrative from Groq LLM

2. **Trip Planning & Route Optimization**
   - Route selector with safety rankings
   - Itinerary builder
   - Multi-point navigation

3. **Blockchain Digital ID**
   - QR-encoded ID card
   - Offline-accessible medical info
   - Stored on Polygon Amoy testnet

4. **Emergency SOS**
   - One-tap alert
   - Live GPS transmission
   - Push to authority dashboard

5. **Background Location Tracking**
   - Service Worker-based tracking
   - Continues in background
   - Batched uploads (3–100 points)
   - Retry logic with exponential backoff

6. **Geofencing**
   - Dynamic zone detection
   - Real-time alerts on entry/exit
   - Configurable radius

7. **PWA Installation**
   - Offline-first architecture
   - Installable on Android/iOS
   - ServiceWorker caching strategies

---

### Module 2: Authority Dashboard (authority-dashboard/)

#### Directory Structure
```
authority-dashboard/
├── app/
│   ├── main.py                 # Streamlit home page
│   ├── models/
│   └── pages/
│       ├── 1_Crowd_Safety_Scoring.py
│       ├── 2_AI_Recommendations.py
│       └── 3_Tourist_Anomaly_Detection.py
├── isolation_module/
│   ├── anomaly detection pipeline
│   ├── datasets
│   └── visualizations
├── utils/
│   ├── clustering.py           # DBSCAN algorithm
│   ├── ai_agents.py            # Gemini integration
│   ├── langchain_agents.py     # LangChain chains
│   ├── map_layers.py           # Folium rendering
│   ├── preprocessing.py        # Data cleaning
│   └── geocoding_utils.py      # Reverse geocoding
├── api.py                      # FastAPI backend
├── data.py                     # Dataset loading
├── run_dashboard.py            # Entry point
└── live_location.json          # Real-time data store
```

#### Key Features
1. **Live Tourist Map**
   - Real-time GPS clusters
   - Tourist count & distribution
   - Folium-based interactive rendering

2. **Crowd Safety Scoring (Page 1)**
   - DBSCAN clustering of GPS points
   - Risk tier assignment (Low/Medium/High/Critical)
   - Heatmap visualization
   - Zone correlation analysis

3. **AI Recommendations (Page 2)**
   - LangChain agent chains
   - Groq LLM (Llama 3) for insights
   - Google Gemini for secondary analysis
   - Patrol reallocation suggestions
   - Crowd redistribution strategies

4. **Tourist Anomaly Detection (Page 3)**
   - Isolation Forest model
   - Route deviation detection
   - Inactivity flagging
   - Health anomaly alerts
   - Auto-SOS triggering

5. **FastAPI Backend**
   - `/location/track` — receives tourist GPS
   - `/location/query` — retrieves live locations
   - `/anomalies` — returns detected anomalies
   - WebSocket support for real-time updates

---

### Module 3: IoT Wearable Hardware

#### Components & BOM
```
MCU:
├─ STM32L476 ARM Cortex-M4 @ 80MHz
├─ 128KB RAM, 256KB Flash
└─ Ultra-low power (5µA standby)

Connectivity:
├─ LoRa SX1262 (915MHz, 22dBm) — up to 15km range
├─ Quectel EC200U (GPS+GSM) — quad-band 2G/3G
└─ GPS Module + GNSS receiver

Sensors:
├─ MAX30100 SpO₂ / Heart Rate sensor
├─ GPS coordinates + altitude
├─ Temperature/Humidity sensor
└─ 6-axis IMU (accelerometer + gyro)

Output:
├─ OLED Display (128x64)
└─ Status LEDs (Power, Alert, GPS fix)

Power:
├─ 3.7V 2000mAh LiPo battery
├─ Li-ion charging module
└─ Battery monitoring IC

Integration:
├─ 9-phase development/ build process documented
├─ Soldering, PCB layout, firmware integration
└─ Hardware debugging (LCD, sensor calibration, etc.)
```

#### Firmware Stack
- **Language:** C/Embedded C
- **RTOS:** None (simple scheduler)
- **Protocols:** LoRa, GPS NMEA, I2C, SPI, UART
- **SOS Modes:**
  1. HTTPS/MQTT to cloud (when 4G/WiFi available)
  2. SMS via GSM (when signal partial)
  3. LoRa P2P mesh relay (zero connectivity)

---

### Module 4: Safety Score Regression Engine

#### Data Sources
1. **Kolkata (250 locations)**
   - KMC ward road surveys
   - Kolkata Police crime statistics (2023–2024)
   - NCRB FIR counts (2021–2022)
   - Google Street View + OSM surface tags
   - Traffic Police intersection data

2. **Darjeeling (286 locations)**
   - PWD West Bengal mountain road assessments
   - Darjeeling Police crime statistics
   - BRO mountain incident reports
   - Tea estate road condition surveys
   - SaveLIFE Foundation black spot analysis

#### Dual Model Architecture
```
Model A: Kolkata (Coupled IDW)
├─ Inverse Distance Weighting (k=5, power=2)
├─ Road→Crime regression coupling
│  crime_coupled = −0.7281 × road + 8.3410
├─ Sparse-zone blending (alpha factor)
└─ Validation MAE: 0.474

Model B: Darjeeling (IDW + Geographic Gradient)
├─ Geographic gradient planes (OLS)
│  crime(lat, lon)    = −4.328×lat + 1.835×lon − 43.035
│  accident(lat, lon) = −4.352×lat + 2.565×lon − 104.823
│  road(lat, lon)     = −1.755×lat + 1.960×lon − 121.370
├─ Adaptive gradient blending (15%–50%)
└─ Validation MAE: 0.369

Output:
├─ Safety Score (0–100)
├─ Confidence Tier (HIGH/MEDIUM/LOW/VERY LOW)
└─ Component Breakdown (crime, accident, road)
```

#### API Deployment
- **Endpoint:** `safetyscore-regression.onrender.com`
- **Method:** POST /score
- **Input:** { lat, lon }
- **Output:** { score, crime, accident, road, confidence }

---

## DATA FLOW ARCHITECTURE

### Tourist Registration & Login Flow

```
┌──────────────┐
│ Mobile User  │
└──────┬───────┘
       │
       ▼ Sign-up form
┌────────────────────────────────────┐
│ TOURIST APP (Next.js Frontend)    │
├────────────────────────────────────┤
│ Collect: name, email, phone,      │
│          medical info, emergency   │
│          contacts, trip details    │
└────────┬─────────────────────────┘
         │
         ▼ POST /api/auth/register
┌────────────────────────────────────┐
│ NEXT.JS API ROUTE                  │
├────────────────────────────────────┤
│ 1. Hash password (bcryptjs)        │
│ 2. Validate input (Zod)            │
│ 3. Create MongoDB document         │
│ 4. Generate JWT token              │
└────────┬─────────────────────────┘
         │
         ▼ Store(email, hash, contact)
┌────────────────────────────────────┐
│ MONGODB ATLAS                      │
├────────────────────────────────────┤
│ Database: setuka                   │
│ Collection: users                  │
│ Document: {                        │
│   _id, email, passwordHash,        │
│   name, phone, medicalInfo,        │
│   emergencyContacts, createdAt     │
│ }                                  │
└────────────────────────────────────┘
         │
         ◀─ JWT token
┌────────────────────────────────────┐
│ TOURIST APP (LocalStorage)         │
├────────────────────────────────────┤
│ Store: JWT token + user profile    │
│ Attach to all subsequent requests  │
└────────────────────────────────────┘
```

### Real-Time Location Tracking Flow

```
┌──────────────┐
│ Tourist GPS  │
│  (Device)    │
└──────┬───────┘
       │ Every 20–30 seconds
       ▼
┌────────────────────────────────────┐
│ SERVICE WORKER                     │
│ (Background Location Tracking)     │
├────────────────────────────────────┤
│ • Read last location               │
│ • Fetch current GPS                │
│ • Buffer location in IndexedDB      │
│ • Batch when: ≥3 points OR ≥2 min  │
└────────┬──────────────────┬────────┘
         │                  │
         ▼                  ▼
    [ONLINE]           [OFFLINE]
         │                  │
         ▼                  │
┌──────────────────────┐    │
│ POST /api/track      │    │
│ + Authorization:     │    │
│   Bearer JWT         │    │
│ + Body: [           │    │
│   { lat, lon,       │    │
│     timestamp,       │    │
│     accuracy },      │    │
│   ...                │    │
│ ]                    │    │
└────────┬─────────────┘    │
         │                  │
         ▼                  ▼ Queue in SW
┌────────────────────────┐   until online
│ NEXT.JS API ROUTE      │
├────────────────────────┤
│ 1. Verify JWT token    │
│ 2. Validate coords     │
│ 3. Store each          │
│    location point      │
│ 4. Update last-seen    │
│ 5. Check geofences     │
│ 6. Trigger any alerts  │
└────────┬───────────────┘
         │
         ▼ Persist (user_id, coords)
┌────────────────────────────────────┐
│ MONGODB ATLAS                      │
├────────────────────────────────────┤
│ Collection: locations              │
│ Doc: { userId, lat, lon,           │
│        timestamp, accuracy }        │
│                                    │
│ Indexed on: userId, timestamp      │
└────────────────────────────────────┘
         │
         │ Also pushed via WebSocket
         ▼
┌────────────────────────────────────┐
│ AUTHORITY DASHBOARD LIVE FEED      │
├────────────────────────────────────┤
│ Display on map                     │
│ Update cluster layer               │
│ Check anomalies                    │
└────────────────────────────────────┘
```

### Safety Score Computation Flow

```
┌────────────────────────────┐
│ Tourist at GPS: (lat, lon) │
└────────┬───────────────────┘
         │
         ▼ GET /safety-score
┌────────────────────────────────────┐
│ NEXT.JS API ROUTE                  │
├────────────────────────────────────┤
│ Determine region:                  │
│  if lat/lon in [Kolkata bounds]    │
│    => Use Model A                  │
│  else if lat/lon in [Darjeeling]   │
│    => Use Model B                  │
│  else => return "outside coverage" │
└────────┬────────────────────┬──────┘
         │                    │
         ▼ POST to            ▼ POST to
   [Model A API]         [Model B API]
┌──────────────────┐  ┌──────────────────┐
│  IDW+Coupling    │  │  IDW+Gradient    │
│  (Kolkata)       │  │  (Darjeeling)    │
└────────┬─────────┘  └────────┬─────────┘
         │                     │
         └──────────┬──────────┘
                    │
                    ▼ Response: {
                      score: 62,
                      crime: 7,
                      accident: 5,
                      road: 3,
                      confidence: "HIGH"
                    }
┌────────────────────────────────────┐
│ TOURIST APP (Frontend Display)     │
├────────────────────────────────────┤
│ Show: Score meter (0–100)          │
│ Show: Component breakdown          │
│ Show: Confidence indicator         │
│ Call: LLM for narrative            │
└────────────────────────────────────┘
         │
         ▼ GET /ai-narrative
┌────────────────────────────────────┐
│ NEXT.JS API ROUTE → Groq LLM       │
├────────────────────────────────────┤
│ Prompt: "Safety score is 62 at     │
│          Kolkata, Khidderpore.     │
│          Crime: 7/10, Road: 3/10.  │
│          Generate a safety brief." │
└────────┬────────────────────────────┘
         │
         ▼ "This area has moderate
        crime. Avoid unlit streets
        after 9 PM. Road quality
        is poor; watch for potholes."
┌────────────────────────────────────┐
│ TOURIST APP (Display Narrative)    │
└────────────────────────────────────┘
```

### Emergency SOS Flow

```
┌────────────────┐
│ Tourist Taps   │
│ "SOS" Button   │
└────────┬───────┘
         │
         ▼
┌────────────────────────────────────┐
│ TOURIST APP (Frontend)             │
├────────────────────────────────────┤
│ 1. Get current GPS + timestamp     │
│ 2. Get health vitals from wearable │
│ 3. Prepare SOS payload:            │
│    {                               │
│      userId, lat, lon,            │
│      timestamp,                    │
│      health: { hr, spo2 },        │
│      message: "Emergency!"         │
│    }                               │
└────────┬──────────────┬────────────┘
         │              │
         ▼ HTTPS        ▼ LoRa
    [ONLINE]        [MESH/OFFLINE]
         │              │
         ▼              ▼
┌──────────────────┐ ┌────────────┐
│POST /api/sos     │ │ Wearable   │
│+ JWT Bearer      │ │ broadcasts │
│+ SOS Payload     │ │ via LoRa   │
│                  │ │ P2P mesh   │
└────────┬─────────┘ └────┬───────┘
         │                │
         ▼                │
┌──────────────────────┐  │
│ NEXT.JS API ROUTE    │  │
├──────────────────────┤  │
│ 1. Verify JWT        │  │
│ 2. Log SOS event     │  │
│ 3. Push alert to:    │  │
│    • authority.db    │  │
│    • SMS via Twilio  │  │
│ 4. WebSocket notify  │  │
└────────┬─────────────┘  │
         │                │ until reaches
         ▼                │ receiver/gateway
┌────────────────────────┐│
│ AUTHORITY DASHBOARD    ││
├────────────────────┬───┘│
│ Alert Popup:       │    │
│ • Tourist: John    │    │
│ • Location: GPS    │    │
│ • HR: 148 bpm ⚠️   │    │
│ • SpO₂: 92% ⚠️      │    │
│ [Dispatch] [Map]   │    │
└────────────────────┘    │
                          │
                          ▼
                    ┌──────────────┐
                    │ LoRa Gateway │
                    │ Relays to    │
                    │ Cloud via    │
                    │ MQTT/IoT Hub │
                    └──────────────┘
```

### Crowd Safety Scoring & Anomaly Detection Flow

```
┌────────────────────────────────────┐
│ Live Tourist Locations (Real-time) │
│ Collection: [lat, lon, timestamp]  │
└────────┬───────────────────────────┘
         │ Every 60 seconds (Streamlit reruns)
         ▼
┌────────────────────────────────────┐
│ AUTHORITY DASHBOARD (Streamlit)    │
├────────────────────────────────────┤
│ Step 1: Fetch latest locations     │
│ Step 2: Apply Preprocessing        │
│   • Remove stale points (>30 min)  │
│   • Handle missing coordinates     │
│   • Normalize timestamps           │
└────────┬───────────────────────────┘
         │
         ├─────────────────┬─────────────────┐
         │                 │                 │
         ▼                 ▼                 ▼
    [BRANCH A]         [BRANCH B]       [BRANCH C]
  Clustering       Anomaly Detection   Reverse
                                      Geocoding
         │                 │                 │
┌────────▼──────────┐ ┌────▼──────────┐ ┌──▼───────────┐
│ DBSCAN Algorithm │ │Isolation Forest│ │OpenCage API  │
│ (eps=50m,        │ │                │ │              │
│  min_samples=3)  │ │ Detects:       │ │ Returns:     │
│                  │ │ • Route deviations           │ │street, area,
│ Output:          │ │ • Inactivity  │ │ locality     │
│ • Cluster labels │ │ • Health     │ │              │
│ • Core points    │ │   anomalies   │ │              │
│ • Outliers       │ │               │ │              │
└────────┬─────────┘ └────┬──────────┘ └──┬───────────┘
         │                │                │
         └────────────────┼────────────────┘
                          │
                          ▼
                ┌──────────────────────┐
                │ Risk Tier Assignment │
                ├──────────────────────┤
                │ Cluster density +    │
                │ anomaly count +      │
                │ past incidents       │
                │ => Risk level        │
                │ (Low/Med/High/Crit)  │
                └──────┬───────────────┘
                       │
         ┌─────────────┼─────────────┐
         │             │             │
         ▼             ▼             ▼
    [Display]    [AI Analysis]  [Alerts]
         │             │             │
┌────────▼──────────┐  ▼             ▼
│ Heatmap Layer     │┌──────────────────────┐
│ Cluster Markers   ││ LangChain Agent      │
│ Risk Color Code   ││                      │
│ Zone Stats        ││ Prompt: "Analyze     │
└──────────────────┘│ hotspots & suggest   │
                    │ patrol deployment"   │
                    │                      │
                    │ LLM Response:        │
                    │ "Increase patrols    │
                    │  near Khidderpore,  │
                    │  redistribute from   │
                    │  quiet zones"        │
                    └──┬───────────────────┘
                       │
                    ┌──▼──────────────┐
                    │ Flag Alerts:    │
                    │ • SMS to police │
                    │ • In-app alert  │
                    │ • Log to DB     │
                    └─────────────────┘
```

---

## KEY TECHNOLOGIES & JUSTIFICATION

### 1. Next.js 14 for Tourist App
**Why:**
- ✅ Built-in PWA support via `next-pwa`
- ✅ Server-side rendering for fast initial load
- ✅ API routes eliminate need for separate backend
- ✅ Static export for offline-mode pre-caching
- ✅ Type-safe with TypeScript out-of-the-box

### 2. Streamlit for Authority Dashboard
**Why:**
- ✅ Rapid prototyping of data UIs
- ✅ No frontend boilerplate — focus on logic
- ✅ Built-in Plotly/Folium integration
- ✅ Session management with `@st.cache_data`
- ✅ Quick iteration for PoC→MVP

### 3. DBSCAN for Crowd Clustering
**Why:**
- ✅ Finds clusters of arbitrary shape (realistic crowd distributions)
- ✅ Automatically detects outliers (isolated tourists)
- ✅ No need to pre-specify number of clusters (unlike K-means)
- ✅ Scales well for real-time analysis with Scikit-learn

### 4. Isolation Forest for Anomaly Detection
**Why:**
- ✅ Unsupervised — no need for labeled anomaly data
- ✅ Detects multi-dimensional anomalies (route + time + health)
- ✅ Works on sparse data (mountain regions)
- ✅ Fast inference time (~ms per point)

### 5. MongoDB for Data Persistence
**Why:**
- ✅ Flexible schema for evolving features (tourist data, sensor readings)
- ✅ Geospatial indexing (2dsphere) for fast location queries
- ✅ Real-time change streams for trigger-based logic
- ✅ Atlas deployment handles scaling + backups

### 6. LangChain + Groq for LLM Integration
**Why:**
- ✅ LangChain abstracts LLM provider switching
- ✅ Groq is fastest LLM API (*<50ms inference*)
- ✅ Chain abstraction for multi-step reasoning
- ✅ Easy integration with vector stores (future embeddings)

### 7. Polygon Blockchain for Digital ID
**Why:**
- ✅ Layer 2 scaling (low tx fees, ~0.1¢)
- ✅ EVM-compatible smart contracts
- ✅ Testnet (Amoy) for safe development
- ✅ Fast finality (~2 seconds)
- ✅ Decentralized storage of tamper-proof data

### 8. LoRa for Wearable Connectivity
**Why:**
- ✅ 915MHz unlicensed spectrum (legal in India)
- ✅ 15+ km range — covers mountain corridors
- ✅ Low power — 24-hour battery on wearable
- ✅ P2P mesh capability for zero-connectivity zones
- ✅ Non-cellular alternative to banned satellite systems

---

## DATABASE SCHEMA

### MongoDB Collections

#### users
```javascript
{
  _id: ObjectId,
  email: String (indexed, unique),
  passwordHash: String (bcrypt),
  name: String,
  phone: String (E.164 format),
  dateOfBirth: Date,
  gender: String,
  medicalInfo: {
    bloodGroup: String,
    allergies: [String],
    medications: [String],
    conditions: [String],
    insuranceProvider: String,
    policyNumber: String
  },
  emergencyContacts: [{
    name: String,
    phone: String,
    relationship: String
  }],
  preferences: {
    notificationsEnabled: Boolean,
    shareLocationWith: [String], // user IDs
    privacyLevel: String (public|private|protected)
  },
  blockchainAddress: String (Ethereum address),
  createdAt: Date,
  updatedAt: Date,
  lastActiveAt: Date
}
```

#### locations
```javascript
{
  _id: ObjectId,
  userId: ObjectId (foreign key → users),
  coordinates: {
    type: "Point",
    coordinates: [lon, lat] (GeoJSON)
  },
  accuracy: Number (meters),
  timestamp: Date (indexed),
  deviceType: String (phone|wearable),
  source: String (gps|network),
  altitude: Number,
  bearing: Number,
  speed: Number,
  batteryLevel: Number
}
```
**Indexes:**
- `{ userId: 1, timestamp: -1 }` — Recent locations for a tourist
- `{ "coordinates": "2dsphere" }` — Geospatial queries

#### sos_alerts
```javascript
{
  _id: ObjectId,
  userId: ObjectId,
  timestamp: Date,
  location: {
    type: "Point",
    coordinates: [lon, lat]
  },
  reason: String (user_triggered|auto_health|inactivity),
  health: {
    heartRate: Number,
    spo2: Number,
    temperature: Number
  },
  message: String,
  status: String (pending|confirmed|resolved),
  respondedBy: ObjectId, // officer ID
  respondedAt: Date,
  resolution: String
}
```

#### geofences
```javascript
{
  _id: ObjectId,
  name: String,
  description: String,
  polygon: {
    type: "Polygon",
    coordinates: [[[lon, lat], ...]]
  },
  riskLevel: String (low|medium|high|critical),
  reason: String,
  createdBy: ObjectId (officer),
  alerts: [{
    userId: ObjectId,
    entryTime: Date,
    exitTime: Date
  }]
}
```

#### anomalies
```javascript
{
  _id: ObjectId,
  userId: ObjectId,
  timestamp: Date,
  type: String (route_deviation|inactivity|health_spike),
  severity: String (low|medium|high),
  location: {
    type: "Point",
    coordinates: [lon, lat]
  },
  data: {
    expectedRoute: [[lon, lat], ...],
    actualLocation: [lon, lat],
    deviationDistance: Number,
    inactivityDuration: Number,
    healthValue: Number
  },
  flaggedAt: Date,
  acknowledgedBy: ObjectId,
  acknowledgedAt: Date
}
```

---

## API ARCHITECTURE

### Next.js API Routes

#### Authentication Endpoints
```
POST /api/auth/register
  Body: { email, password, name, phone, medicalInfo, emergencyContacts }
  Response: { token, user: { id, email, name } }
  Status: 201 Created

POST /api/auth/login
  Body: { email, password }
  Response: { token, user }
  Status: 200 OK

POST /api/auth/logout
  Header: Authorization: Bearer <token>
  Response: { success: true }
  Status: 200 OK
```

#### Location Endpoints
```
POST /api/location/track
  Header: Authorization: Bearer <token>
  Body: [
    { lat, lon, timestamp, accuracy },
    ...
  ]
  Response: { tracked: 5, anomalies: [] }
  Status: 201 Created

GET /api/location/current
  Header: Authorization: Bearer <token>
  Response: { lat, lon, timestamp, accuracy, address }
  Status: 200 OK

GET /api/location/history
  Header: Authorization: Bearer <token>
  Query: { startTime, endTime, limit }
  Response: [{ lat, lon, timestamp }, ...]
  Status: 200 OK
```

#### Safety Score Endpoints
```
GET /api/safety-score
  Query: { lat, lon }
  Response: {
    score: 62,
    components: {
      crime: 7,
      accident: 5,
      road: 3
    },
    confidence: "HIGH",
    narrative: "..."
  }
  Status: 200 OK

GET /api/safety-score/nearby
  Query: { lat, lon, radius }
  Response: [{ lat, lon, score, riskLevel }, ...]
  Status: 200 OK
```

#### SOS Endpoints
```
POST /api/sos
  Header: Authorization: Bearer <token>
  Body: { lat, lon, reason, health: { hr, spo2 }, message }
  Response: { alertId, status, respondingOfficer }
  Status: 201 Created

GET /api/sos/:alertId
  Header: Authorization: Bearer <token>
  Response: { status, respondedBy, respondedAt, resolution }
  Status: 200 OK
```

#### Geofence Endpoints
```
GET /api/geofences
  Query: { lat, lon, radius }
  Response: [{ id, name, riskLevel, description }, ...]
  Status: 200 OK

POST /api/geofences/subscribe
  Header: Authorization: Bearer <token>
  Body: { geofenceId }
  Response: { subscribed: true }
  Status: 201 Created
```

#### Blockchain Endpoints
```
POST /api/blockchain/deploy-id
  Header: Authorization: Bearer <token>
  Body: { name, bloodGroup, insuranceInfo }
  Response: { txHash, contractAddress, qrCode }
  Status: 201 Created

GET /api/blockchain/verify-id
  Query: { qrCodeData }
  Response: { valid: true, owner, medicalInfo }
  Status: 200 OK
```

### FastAPI Backend (Authority Dashboard)

#### Location Ingestion
```
POST /location/track
  Body: [{ userId, lat, lon, timestamp }, ...]
  Response: { stored: 5, failed: 0 }
  Status: 200 OK
```

#### Live Query
```
GET /location/live
  Query: { region }
  Response: {
    tourists: [{ id, lat, lon, lastUpdate }, ...],
    clusters: [{ center, count, riskLevel }, ...],
    heatmap: [...]
  }
  Status: 200 OK
```

#### Anomalies
```
GET /anomalies
  Response: [
    {
      userId, type, severity, timestamp, location,
      details: {}
    },
    ...
  ]
  Status: 200 OK
```

#### WebSocket
```
WS /ws/live-feed
  Connect → Receive real-time location & alert updates
  Messages: {
    type: "location_update|anomaly|sos",
    data: {}
  }
```

---

## DEVELOPMENT METHODOLOGY

### 1. **Architecture-First Design**
- Define layer boundaries (tourist app / wearable / dashboard) upfront
- Build modular, independently deployable components
- Each layer has single responsibility

### 2. **Data-Driven Feature Prioritization**
- Phases 1–3: Core MVP (user registration, basic tracking, safety score)
- Phases 4–6: Advanced analytics (anomaly detection, crowd scoring)
- Phases 7–9: AI/LLM narratives + blockchain ID

### 3. **Dual-City Regression Model Training**
- Separate models due to geographic/data structure differences
- Kolkata: traffic-density-driven interpolation
- Darjeeling: terrain-gradient-driven interpolation
- Leave-one-out cross-validation for MAE estimation

### 4. **Iterative Hardware Development (9 Phases)**
- Phase 1–2: PCB layout + component selection
- Phase 3–5: Soldering + sensor integration
- Phase 6–8: Firmware + communication stack
- Phase 9: Final calibration + field testing

### 5. **Real-Time System Design**
- WebSocket for low-latency dashboard updates
- Service Worker for background tracking on mobile
- Streamlit's `@st.cache_resource` for persistent state
- Batch processing (location uploads in groups of 3–10)

### 6. **Privacy & Security by Design**
- End-to-end encryption for sensitive data (health, ID)
- JWT tokens for API authentication
- Blockchain for tamper-proof digital ID
- User consent for location sharing

### 7. **Testing Strategy**
- Unit tests for ML models (isolation forest, DBSCAN)
- Integration tests for API endpoints
- End-to-end tests for critical flows (SOS, geofencing)
- Load testing for real-time dashboard (100+ concurrent tourists)

### 8. **Documentation-Heavy Approach**
- Detailed challenge documentation (CHALLENGES.md)
- Source attribution for all datasets (source_kol.md, source_darj.md)
- Hardware build logs (9-phase video documentation)
- API contract diagrams (this blueprint)

---

## DEPLOYMENT & INFRASTRUCTURE

### Frontend Hosting
- **Platform:** Vercel
- **Region:** Auto-scaled globally
- **Features:** 
  - Edge caching for static assets
  - Automatic HTTPS
  - Preview deployments for PRs
  - Analytics & monitoring

### Backend APIs
- **Platform:** Render / Railway / AWS EC2
- **Services:**
  - Next.js server (API routes)
  - FastAPI server (authority dashboard backend)
  - MongoDB Atlas (database)
  - Streamlit server (dashboard UI)

**Configuration:**
```yaml
Services:
  - tourist-app: Vercel (Next.js)
  - api-backend: Render (FastAPI + Uvicorn)
  - dashboard: Render (Streamlit)
  - database: MongoDB Atlas (paid tier)

Domains:
  - Frontend: setuka.app
  - API: api.setuka.app
  - Dashboard: dashboard.setuka.app
```

### Blockchain Infrastructure
- **Network:** Polygon Amoy (testnet)
- **RPC Provider:** Polygon RPC (https://rpc-amoy.polygon.technology/)
- **Smart Contracts:** Deployed via Hardhat
- **Wallet:** MetaMask for user interactions

### CI/CD Pipeline
```
Trigger: Git push to main branch

┌──────────────┐
│ GitHub Push  │
└──────┬───────┘
       │
       ▼
┌──────────────────────┐
│ GitHub Actions       │
├──────────────────────┤
│ Run Tests            │
│ Lint Code            │
│ Build Docker Images  │
└──────┬───────────────┘
       │
       ├─ Success → Deploy to staging
       │
       └─ Failure → Notify team

Staging (preview.setuka.app):
├─ Run integration tests
├─ Smoke tests (API calls)
└─ Manual QA approval

Production (setuka.app):
├─ Deploy to Vercel
├─ Deploy API to Render
├─ Update database indexes
└─ Smoke tests + monitoring
```

### Monitoring & Logging
- **Application Monitoring:** Sentry (error tracking)
- **Infrastructure:** Datadog / New Relic
- **Logs:** CloudWatch / Loki
- **Metrics:** Prometheus + Grafana
- **Alerting:** PagerDuty (on-call)

**Key Metrics:**
- API latency (target: <100ms p95)
- Location update throughput (target: 1000 loc/sec)
- Anomaly detection accuracy (target: >85%)
- Dashboard refresh rate (target: <2 sec)
- Wearable connectivity (target: >95% uptime)

---

## SECURITY ARCHITECTURE

### Data Protection
- **Encryption in Transit:** TLS 1.3 for all HTTPS
- **Encryption at Rest:** 
  - MongoDB: database-level encryption
  - Health records: field-level encryption before MongoDB
- **API Authentication:** JWT with RS256 signing
- **Session Management:** Refresh tokens (24h), access tokens (1h)

### Access Control
- **Role-Based:** Tourist, Officer, Admin roles
- **Geofence Authorization:** Officers can only see tourists in their jurisdiction
- **Health Data:** Only accessible to authenticated user + emergency contacts
- **Blockchain ID:** QR code requires user consent to display

### Threat Mitigation
| Threat | Mitigation |
|--------|-----------|
| **SOS spoofing** | JWT + IP verification + rate limiting |
| **Location tracking abuse** | Privacy levels (public/private/protected) |
| **Blockchain ID theft** | QR code + PIN, offline signature verification |
| **Data breach** | Hashed passwords, encrypted health fields, audit logging |
| **DDoS** | Cloudflare DDoS protection, rate limiting on API routes |

---

## PERFORMANCE TARGETS

| Metric | Target | Current |
|--------|--------|---------|
| **Tourist App Load Time** | <2s | TBD |
| **Safety Score Latency** | <100ms | ~50ms |
| **Location Tracking** | <5s from device to dashboard | TBD |
| **SOS Alert** | <2s notification to officer | TBD |
| **Dashboard Rerender** | <1s (Streamlit) | ~2s |
| **Anomaly Detection** | <500ms per batch | TBD |
| **Wearable Battery Life** | >18 hours | ~24h (initial) |

---

## FUTURE ROADMAP

### Phase 2 (Post-MVP)
- ✅ Multi-language support (Bengali, Hindi, English)
- ✅ Offline-first data sync with IndexedDB
- ✅ Police dispatch integration (SMS, mobile app)
- ✅ Real-time video streaming from wearable camera

### Phase 3 (Scaling)
- ✅ Mobile app (React Native) for officers
- ✅ Predictive routing (ML model for safest paths)
- ✅ Crowd simulation + capacity planning
- ✅ Tourism board API (integrate with govt systems)

### Phase 4 (Enterprise)
- ✅ Multi-city deployment (Trekking routes nationwide)
- ✅ Insurance integration (real-time claim processing)
- ✅ Hardware manufacturing partnership
- ✅ API marketplace for 3rd-party integrations

---

## KEY TAKEAWAYS

1. **Three-Layer Architecture:** Tourist app (PWA) → Wearable (IoT) → Authority Dashboard separates concerns
2. **Offline-First Design:** LoRa mesh + Service Worker ensure safety even in zero-connectivity zones
3. **Data-Driven Intelligence:** Dual regression models + DBSCAN + Isolation Forest provide real-time insights
4. **Legal Compliance:** Ground-based LoRa avoids satellite SOS bans in Indian border regions
5. **User-Centric Security:** Blockchain digital ID accessible offline via QR code — no internet required for emergencies
6. **Agile Methodology:** 9-phase hardware development + iterative feature rollout
7. **Real-Time Operations:** WebSocket-driven dashboard with <2s refresh for command-center decision-making

---

## CONTRIBUTORS

**Author:** [Your Name]  
**Last Updated:** March 23, 2026  
**Repository:** [https://github.com/your-org/setuka](https://github.com/your-org/setuka)

---

## REFERENCES

- [Full README](README.md)
- [Challenges & Solutions](CHALLENGES.md)
- [Hardware Build Log](hardware-Progress.txt)
- [Tourist App Repository](tourist-app/)
- [Authority Dashboard Repository](authority-dashboard/)
