# 🏗️ SETUKA ARCHITECTURE DIAGRAMS

> Visual representations of Setuka's system architecture, data flows, and component interactions

---

## 1. HIGH-LEVEL SYSTEM ARCHITECTURE

### 3-Layer Ecosystem Overview (Complete Data Flow)

```mermaid
graph TB
    subgraph Layer1["🎯 LAYER 1: TOURIST APP (PWA)"]
        SubA["Next.js 14 Frontend<br/>React + TypeScript"]
        SubB["Service Worker<br/>Background Tracking"]
        SubC["PWA Installation<br/>Offline-Ready"]
    end

    subgraph Layer2["📡 LAYER 2: IoT WEARABLE"]
        SubD["STM32L476 MCU<br/>Low-Power Core"]
        SubE["LoRa SX1262<br/>GPS, Health Sensors"]
        SubF["Triple-Mode SOS<br/>Cloud/SMS/Mesh"]
    end

    subgraph Layer3["🎛️ LAYER 3: AUTHORITY DASHBOARD"]
        SubG["Streamlit UI<br/>Command Center"]
        SubH["FastAPI Backend<br/>Real-Time Analytics"]
        SubI["ML Pipeline<br/>DBSCAN, Isolation Forest"]
    end

    subgraph Services["☁️ EXTERNAL SERVICES"]
        SubJ["MongoDB Atlas<br/>Data Persistence"]
        SubK["Groq LLM<br/>AI Narratives"]
        SubL["Polygon Blockchain<br/>Digital ID"]
        SubM["Safety Score API<br/>Regression Engine"]
    end

    %% LAYER 1 ↔ LAYER 3 (HTTPS/WebSocket)
    Layer1 -->|Location + SOS / HTTPS/WebSocket| SubH
    SubH -->|Results + Alerts / WebSocket Push| Layer1
    
    %% LAYER 2 ↔ LAYER 3 (Triple-mode)
    Layer2 -->|GPS + Vitals + Critical SOS / MQTT/4G/SMS| SubH
    SubH -->|Acknowledgments + Commands / MQTT/4G| Layer2
    Layer2 -.->|Fallback P2P Mesh / LoRa Relay| Layer3
    
    %% LAYER 2 ↔ LAYER 1 (Emergency fallback)
    Layer2 -.->|Emergency SOS / LoRa Direct| Layer1
    Layer1 -.->|Relay to Backend / When online| SubH
    
    %% LAYER 3 (Internal)
    SubG <-->|Dashboard UI| SubH
    SubH <-->|Push Analytics| SubG
    SubH <-->|Run ML Models| SubI
    SubH -->|Query/Store| SubJ
    SubI -->|Anomaly Scores| SubH
    
    %% ALL TO SERVICES
    SubH <-->|Health Data + Vectors| SubJ
    Layer1 <-->|Auth + Digital ID| SubL
    Layer1 <-->|Safety Scores| SubM
    SubH <-->|LLM Insights| SubK
    
    %% STYLING
    style Layer1 fill:#e1f5ff,stroke:#01579b,stroke-width:3px
    style Layer2 fill:#f3e5f5,stroke:#4a148c,stroke-width:3px
    style Layer3 fill:#e8f5e9,stroke:#1b5e20,stroke-width:3px
    style Services fill:#fff3e0,stroke:#e65100,stroke-width:2px
    
    %% HIGHLIGHT CRITICAL CONNECTIONS
    linkStyle 2,3,4,5 stroke:#ff0000,stroke-width:2px
```

---

## 2. COMPLETE LAYER-TO-LAYER DATA FLOWS

### Wearable ↔ Backend ↔ Dashboard Communication Mesh

```mermaid
graph TB
    subgraph Tourist["LAYER 1<br/>Tourist App"]
        T1["Location Tracking<br/>Service Worker"]
        T2["SOS Emergency<br/>Button"]
        T3["Geofence Alerts<br/>Push Notifications"]
    end
    
    subgraph Wearable["LAYER 2<br/>Wearable Device"]
        W1["GPS + Health Sensors<br/>SpO₂, HR, Temp"]
        W2["Triple-Mode SOS<br/>Cloud/SMS/LoRa"]
        W3["OLED Display<br/>Status + Alerts"]
    end
    
    subgraph Backend["LAYER 3 CORE<br/>FastAPI Backend"]
        B1["Location Ingestion<br/>Endpoint"]
        B2["SOS Alert Handler<br/>Multi-channel"]
        B3["WebSocket Manager<br/>Real-time Push"]
        B4["Anomaly Scorer<br/>Isolation Forest"]
    end
    
    subgraph Dashboard["LAYER 3 UI<br/>Streamlit Dashboard"]
        D1["Live Tourist Map<br/>DBSCAN Clusters"]
        D2["Crowd Safety View<br/>Risk Heatmap"]
        D3["Anomaly Alerts<br/>Officer Panel"]
        D4["AI Recommendations<br/>LLM Insights"]
    end
    
    %% ===== TOURIST APP → BACKEND =====
    T1 -->|POST /api/location/track / Batch: lat,lon,ts / JWT Auth| B1
    T2 -->|POST /api/sos / GPS + health vitals / Emergency flag| B2
    
    %% ===== WEARABLE → BACKEND (Primary) =====
    W1 -->|MQTT: device/GPS/telemetry / 4G/3G when available / Every 20-30s| B1
    W2 -->|SOS Trigger / Critical health anomaly / Auto-detection| B2
    
    %% ===== WEARABLE → BACKEND (Fallback) =====
    W2 -.->|SMS via Twilio / When 4G unavailable / GPS + Alert ID| B2
    W2 -.->|LoRa P2P Relay / To nearby wearables / Until internet| B3
    
    %% ===== BACKEND → DASHBOARD (Real-time) =====
    B1 -->|WebSocket: live_locations / Streaming / Every 1-2s| D1
    B2 -->|WebSocket: critical_alerts / SOS events + severity / Instant| D3
    B3 -->|WebSocket: updates / Cluster changes + anomalies / Push to UI| D2
    B4 -->|Isolation Forest Scores / Route deviations + inactivity / Flagged| D3
    
    %% ===== DASHBOARD ANALYSIS → BACKEND =====
    D2 -->|Trigger LLM Analysis / Crowd density + risk data / Patrol suggestions| B4
    D3 -->|Officer Actions / Acknowledge anomalies / Mark SOS resolved| B2
    
    %% ===== BACKEND → TOURIST APP (Push) =====
    B2 -->|WebSocket: sos_confirmed / Officer dispatched / Provides ETA| T3
    B3 -->|WebSocket: geofence_alert / Entering high-risk zone / Warning| T3
    B4 -->|WebSocket: anomaly_check / Inactivity detected / Request status| T2
    
    %% ===== BACKEND → WEARABLE =====
    B2 -->|MQTT ACK / SOS received / Vibration confirm| W3
    B3 -->|MQTT command / Geofence entry alert / Update OLED| W3
    B2 -->|SMS Response / Officer details / If SMS path used| W3
    
    %% ===== DASHBOARD → WEARABLE (Indirect) =====
    D3 -->|Officer sends alert via backend| B3
    B3 -->|MQTT Push / Officer approaching / Update OLED| W3
    
    %% Styling
    style Tourist fill:#e1f5ff,stroke:#01579b,stroke-width:2px
    style Wearable fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    style Backend fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px
    style Dashboard fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    
    %% Highlight critical emergency path
    linkStyle 4,5,6,7,8 stroke:#ff0000,stroke-width:2.5px
```

---

## 3. TOURIST APP: COMPLETE FEATURE STACK

### All Features Combined: Location Tracking + Safety Scoring + Geofencing + Digital ID

```mermaid
graph TB
    subgraph Browser["🎯 BROWSER LAYER (Next.js Frontend)"]
        subgraph UIPages["📱 UI Pages"]
            P1["🔐 Auth Page<br/>Login/Signup"]
            P2["📊 Dashboard<br/>Safety Score + Map"]
            P3["👤 Profile Page<br/>Digital ID + Contacts"]
            P4["🗺️ Trip Planner<br/>Route Selection"]
        end
        
        subgraph ReactState["⚡ React State & Hooks"]
            R1["AuthContext<br/>JWT Token"]
            R2["LocationContext<br/>Current GPS"]
            R3["SessionContext<br/>User Data"]
        end
        
        subgraph UIComponents["🎨 UI Components"]
            C1["Interactive Map<br/>Leaflet + Markers"]
            C2["Safety Score Card<br/>0–100 gauge"]
            C3["Geofence Alert Popup<br/>Entry/Exit warnings"]
            C4["QR Code Scanner<br/>Digital ID verify"]
            C5["SOS Emergency Button<br/>Red modal"]
        end
    end
    
    subgraph ServiceWorker["⚙️ SERVICE WORKER & STORAGE"]
        subgraph BackgroundOps["📡 Persistent Tracking"]
            SW1["Background tracking<br/>Even when closed"]
            SW2["Fetch GPS 20–30s"]
            SW3["Buffer to IndexedDB<br/>Local persistence"]
        end
        
        subgraph BatchUpload["📦 Smart Batching"]
            SW4["Check: ≥3 pts<br/>OR ≥2 min?"]
            SW5["POST /api/location/track<br/>JWT + Batch"]
        end
        
        subgraph OfflineCache["🔴 Offline-First"]
            SW6["No network?<br/>Store locally"]
            SW7["Retry 3x backoff"]
            SW8["Auto-sync online"]
        end
    end
    
    subgraph SafetyFeatures["🛡️ SAFETY SCORING ENGINE"]
        subgraph LocationFetch["📍 Location Processing"]
            ST1["Get GPS: lat, lon"]
            ST2["Determine Region<br/>Kolkata or Darjeeling?"]
        end
        
        subgraph DualModels["🧮 Dual Regression Models"]
            ST3A["Model A: Kolkata<br/>IDW + Crime coupling<br/>MAE: 0.474"]
            ST3B["Model B: Darjeeling<br/>IDW + Gradient<br/>MAE: 0.369"]
        end
        
        subgraph ScoreOutput["📊 Score Rendering"]
            ST4["Safety Response<br/>score, crime,<br/>accident, road"]
            ST5["Confidence Tier<br/>HIGH→VERY LOW"]
            ST6["Display Meter<br/>+ Components"]
            ST7["LLM Narrative<br/>Groq Llama 3<br/>Safety briefing"]
        end
    end
    
    subgraph GeofencingFeatures["🗺️ GEOFENCE SYSTEM"]
        subgraph GeoMonitor["👁️ Real-Time Monitoring"]
            GF1["Service Worker<br/>tracks GPS"]
            GF2["POST geofence check<br/>To API endpoint"]
        end
        
        subgraph GeoDetection["🔍 Zone Detection"]
            GF3["Query: Point in polygon?"]
            GF4["Matched zones?<br/>Entry/Exit/InZone"]
        end
        
        subgraph GeoAlerts["🚨 Alert Dispatch"]
            GF5["Entry detected<br/>Toast + marker"]
            GF6["Exit detected<br/>Clear warning"]
            GF7["Return: zoneName<br/>+ riskLevel"]
        end
    end
    
    subgraph BlockchainFeatures["⛓️ DIGITAL ID SYSTEM"]
        subgraph IDGeneration["🆔 ID Creation"]
            BC1["User requests<br/>Digital ID"]
            BC2["Collect: Name<br/>Blood Group<br/>Insurance<br/>Emergency Contacts"]
        end
        
        subgraph IDBlockchain["📝 Blockchain Deploy"]
            BC3["Deploy Smart<br/>Contract to<br/>Polygon Amoy"]
            BC4["Get: TxHash<br/>ContractAddress"]
        end
        
        subgraph IDQRCode["📸 QR Encoding"]
            BC5["Encode:<br/>ContractAddress<br/>+ Signature Hash"]
            BC6["Generate QR Code<br/>Display to user"]
        end
        
        subgraph IDVerification["✅ Offline Verification"]
            BC7["Officer scans QR<br/>(no internet)"]
            BC8["Read ContractAddress<br/>+ Validate signature"]
            BC9["Display: Blood type<br/>Insurance<br/>Emergency contacts"]
        end
        
        subgraph IDOnlineVerify["🌐 Online Verification"]
            BC10["POST /api/blockchain<br/>/verify-id"]
            BC11["Read from<br/>Polygon contract"]
            BC12["Return verified info<br/>+ timestamp"]
        end
    end
    
    subgraph NextJSAPI["🔌 NEXT.JS API ROUTES"]
        N1["/api/auth/*<br/>Login/register JWT"]
        N2["/api/location/track<br/>Store + geofence check"]
        N3["/api/safety-score<br/>Call regression API"]
        N4["/api/sos<br/>Multi-channel dispatch"]
        N5["/api/blockchain/*<br/>Digital ID ops"]
        N6["/api/geofences<br/>Zone queries"]
    end
    
    subgraph Backend["☁️ BACKEND & SERVICES"]
        B1["FastAPI Backend<br/>WebSocket push"]
        B2["MongoDB Atlas<br/>users, locations"]
        B3["Safety Score API<br/>Regression engine"]
        B4["Groq LLM<br/>AI narratives"]
        B5["Polygon RPC<br/>Digital ID"]
        B6["Geofence DB<br/>Zone storage"]
    end
    
    %% PAGE ROUTING
    P1 --> R1
    P2 --> R2
    P3 --> BC1
    P4 --> ST1
    
    %% COMPONENTS
    C1 --> UIPages
    C2 --> SafetyFeatures
    C3 --> GeofencingFeatures
    C4 --> BlockchainFeatures
    C5 --> N4
    
    %% FOREGROUND TRACKING
    P2 -->|Real-time| C1
    
    %% SERVICE WORKER TO TRACKING
    SW1 --> SW2
    SW2 --> SW3
    SW3 --> SW4
    SW4 -->|Ready| SW5
    SW5 --> N2
    SW4 -->|Offline| SW6
    SW6 --> SW7
    SW7 -->|Online| SW5
    
    %% LOCATION TO GEOFENCING
    N2 -->|Location batch| GF2
    GF2 --> GF3
    GF3 -->|Match found| GF4
    GF4 -->|Entry| GF5
    GF4 -->|Exit| GF6
    GF5 --> C3
    GF6 --> C3
    
    %% LOCATION TO SAFETY SCORE
    N3 -->|lat, lon| ST1
    ST1 --> ST2
    ST2 -->|Kolkata| ST3A
    ST2 -->|Darjeeling| ST3B
    ST3A --> ST4
    ST3B --> ST4
    ST4 --> ST5
    ST5 --> ST6
    ST6 --> C2
    ST6 -->|Score| ST7
    ST7 --> C2
    
    %% DIGITAL ID FLOW
    BC1 --> BC2
    BC2 --> BC3
    BC3 --> BC4
    BC4 --> BC5
    BC5 --> BC6
    BC6 --> C4
    C4 -->|Scan| BC7
    BC7 --> BC8
    BC8 --> BC9
    BC9 -->|Also| BC10
    BC10 --> BC11
    BC11 --> BC12
    
    %% API DISTRIBUTION
    N1 --> B2
    N2 --> B2
    N3 --> B3
    N3 --> B4
    N4 --> B1
    N5 --> B5
    N6 --> B6
    
    %% BACKEND TO FRONTEND (WebSocket)
    B1 -.->|Live updates| C1
    B1 -.->|Alerts| C3
    B1 -.->|Score updates| C2
    
    %% STYLING
    style Browser fill:#e1f5ff,stroke:#01579b,stroke-width:2px
    style ServiceWorker fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    style SafetyFeatures fill:#c8e6c9,stroke:#1b5e20,stroke-width:2px
    style GeofencingFeatures fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style BlockchainFeatures fill:#ffe0b2,stroke:#e65100,stroke-width:2px
    style NextJSAPI fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    style Backend fill:#fff9c4,stroke:#f57f17,stroke-width:2px
```

**Feature Flows:**

**🟢 LOCATION TRACKING (Persistent):**
- Foreground: Real-time display on map (20s intervals)
- Background: Service Worker continues, buffers with IndexedDB
- Offline: Stores locally, auto-syncs when online

**🛡️ SAFETY SCORING (Live Computation):**
- Fetch GPS coordinates
- Route through region detection (Kolkata Model A vs Darjeeling Model B)
- Dual regression engines compute score (0–100)
- Display meter + AI narrative from Groq LLM

**🗺️ GEOFENCING (Real-Time Alerts):**
- Location batch → Check geofence zones
- Detect entry/exit via polygon containment
- Instant popup alert to user (no refresh needed)
- Risk level displayed on map

**⛓️ DIGITAL ID (Offline-Accessible):**
- Deploy smart contract to Polygon Amoy
- QR encode with contract address + signature
- Officer scans offline → Reads medical info directly from QR
- Online verification checks blockchain state

---

## 4. AUTHORITY DASHBOARD: COMPLETE REAL-TIME ANALYTICS PIPELINE

### Consolidated: Data Ingestion + Clustering + Anomaly Detection + LLM Insights + Dashboard UI + Officer Feedback

```mermaid
graph TB
    subgraph DataInput["📥 DATA INGESTION LAYER"]
        I1["Tourist App<br/>Location Batch Updates"]
        I2["Wearable Sensors<br/>GPS + Health Vitals"]
        I3["SOS Emergency Alerts<br/>Manual/Auto-triggered"]
        I4["Real-time WebSocket<br/>Streaming Feed"]
        
        I1 --> I4
        I2 --> I4
        I3 --> I4
    end
    
    subgraph Processing["⚙️ REAL-TIME PROCESSING"]
        P1["Data Validation<br/>Schema check, clean"]
        P2["Preprocessing<br/>Remove stale points"]
        P3["Geocoding<br/>Coords →Street names"]
        P4["Geo Indexing"]
        
        I4 --> P1 --> P2 --> P3 --> P4
    end
    
    subgraph ClusterAnalysis["🎯 CLUSTERING + RISK"]
        C1["DBSCAN eps=50m min=3"]
        C2["Risk Scoring"] 
        C3["Heatmap OUTPUT"]
        P4 --> C1 --> C2 --> C3
    end
    
    subgraph AnomalyOps["🚨 ANOMALY DETECTION"]
        F1["Feature Engineering"]
        AF1["Isolation Forest"]
        S1["Severity Classify"]
        AA1["Alert Dispatch"]
        P4 --> F1 --> AF1 --> S1 --> AA1
    end
    
    subgraph LLMAnalysis["🤖 AI INSIGHTS"]
        L1["LangChain + Groq LLM"]
        L2["Patrol Recommendations"]
        C3 --> L1 --> L2
        AA1 --> L1
    end
    
    subgraph DashboardUI["📊 STREAMLIT DASHBOARD"]
        M1["Live Map Heatmap"]
        SP1["Officer Panel Alerts"]
        RP1["AI Recommendations"]
        C3 --> M1
        AA1 --> SP1 --> RP1
        L1 --> RP1
    end
    
    subgraph Feedback["🔄 FEEDBACK LOOP"]
        FB1["Officer Acknowledges"]
        FB2["Zone Status Update"]
        FB3["Tourists Notified"]
        RP1 --> FB1 --> FB2 --> FB3
    end
    
    style DataInput fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style Processing fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style ClusterAnalysis fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style AnomalyOps fill:#ffe0b2,stroke:#e65100,stroke-width:2px
    style LLMAnalysis fill:#f1f8e9,stroke:#558b2f,stroke-width:2px
    style DashboardUI fill:#e0f2f1,stroke:#00796b,stroke-width:2px
    style Feedback fill:#fce4ec,stroke:#c2185b,stroke-width:2px
```

---

## 5. IoT WEARABLE HARDWARE STACK

### Triple-Mode SOS Transmission

```mermaid
graph TD
    A["Health/GPS Data on Wearable"] -->|Critical Threshold| B["Trigger SOS"]
    
    B -->|Internet Available| C1["Mode 1: Direct Cloud"]
    B -->|Internet Unavailable| C2["Mode 2: GSM SMS"]
    B -->|No Connectivity| C3["Mode 3: LoRa P2P Mesh"]
    
    subgraph M1["☁️ Mode 1: Direct Cloud"]
        C1 -->|MQTT| D1["IoT Hub"]
        D1 -->|Relay| E1["FastAPI Backend"]
        E1 -->|Notify| F1["Authority Dashboard"]
    end
    
    subgraph M2["📱 Mode 2: GSM SMS"]
        C2 -->|Twilio API| D2["SMS Gateway"]
        D2 -->|Police Number| E2["Officer Phone"]
        E2 -->|Manual Map| F2["Location Lookup"]
    end
    
    subgraph M3["📡 Mode 3: LoRa Mesh"]
        C3 -->|LoRa RF| D3["Nearby Node 1"]
        D3 -->|LoRa RF| E3["Nearby Node 2"]
        E3 -->|Eventually| F3["Gateway with Internet"]
        F3 -->|Cloud Upload| G3["Authority Dashboard"]
    end
    
    F1 --> H["Alert Displayed"]
    F2 --> H
    G3 --> H
    
    H --> I["Officer Responds<br/>Dispatch sent"]
    
    style M1 fill:#c8e6c9
    style M2 fill:#fff9c4
    style M3 fill:#ffe0b2
    style I fill:#ffccbc
```

---

## 6. EMERGENCY SOS TRIGGERING PROTOCOL

### Multi-Channel Alert Dispatch

```mermaid
graph TD
    A{SOS Triggered} -->|Manual| B1["User taps SOS"]
    A -->|Auto| B2["Health anomaly detected"]
    A -->|Inactivity| B3["No movement in 30min"]
    
    B1 --> C["Collect SOS Payload"]
    B2 --> C
    B3 --> C
    
    C --> D["Get current GPS<br/>Health vitals<br/>Timestamp"]
    
    D -->|Online| E1["POST /api/sos<br/>+ JWT Bearer"]
    D -->|Offline| E2["Store in IndexedDB<br/>Retry when online"]
    
    E1 -->|Success| F["FastAPI Acknowledges"]
    E2 -->|Eventually| F
    
    F --> G["Multi-Channel Dispatch"]
    
    G -->|Channel 1| H1["SMS to Police<br/>Location + Alert ID"]
    G -->|Channel 2| H2["Dashboard Alert Popup<br/>Officer's map"]
    G -->|Channel 3| H3["LoRa Broadcast<br/>Nearby receivers"]
    
    H1 --> I["Officer Notified"]
    H2 --> I
    H3 --> I
    
    I --> J["Officer Responds<br/>Mark acknowledged"]
    J --> K["Dispatch Sent<br/>Tourist Notified"]
    
    style B1 fill:#ffccbc
    style B2 fill:#ffccbc
    style B3 fill:#ffccbc
    style G fill:#ff9999
    style K fill:#c8e6c9
```

---



## 7. DATABASE SCHEMA RELATIONSHIPS

### MongoDB Collections & Indexing

```mermaid
graph LR
    subgraph Users["👥 users"]
        U1["_id (ObjectId)"]
        U2["email (unique)"]
        U3["passwordHash"]
        U4["medicalInfo"]
        U5["emergencyContacts"]
    end
    
    subgraph Locations["📍 locations"]
        L1["_id"]
        L2["userId (FK→users)"]
        L3["coordinates (2dsphere)"]
        L4["timestamp"]
        L5["accuracy"]
    end
    
    subgraph SOSAlerts["🚨 sos_alerts"]
        S1["_id"]
        S2["userId (FK→users)"]
        S3["location (Point)"]
        S4["health { hr, spo2 }"]
        S5["status"]
    end
    
    subgraph Geofences["🛡️ geofences"]
        G1["_id"]
        G2["polygon (Polygon)"]
        G3["riskLevel"]
        G4["alerts"]
    end
    
    subgraph Anomalies["⚠️ anomalies"]
        A1["_id"]
        A2["userId (FK→users)"]
        A3["type"]
        A4["severity"]
        A5["location (Point)"]
    end
    
    U2 -->|1:Many| L2
    U2 -->|1:Many| S2
    U2 -->|1:Many| A2
    
    Locations -->|Geospatial Index| Geofences
    
    L1 -.->|correlates| A5
    
    style Users fill:#bbdefb
    style Locations fill:#c8e6c9
    style SOSAlerts fill:#ffccbc
    style Geofences fill:#f0f4c3
    style Anomalies fill:#ffe0b2
```

---

## 8. CI/CD & DEPLOYMENT PIPELINE

### Automated Testing & Deploy Flow

```mermaid
graph TD
    A["Developer<br/>Pushes Code"] -->|Git Push| B["GitHub Events"]
    
    B -->|Trigger| C["GitHub Actions<br/>Workflow Start"]
    
    C --> D["Unit Tests<br/>Frontend + Backend"]
    D -->|❌ Fail| E["Notify Team<br/>Block Merge"]
    D -->|✅ Pass| F["Lint & Format<br/>ESLint, Prettier"]
    
    F -->|❌ Fail| E
    F -->|✅ Pass| G["Build Docker Images<br/>frontend, api, dashboard"]
    
    G -->|❌ Fail| E
    G -->|✅ Pass| H["Push to Registry<br/>Docker Hub"]
    
    H --> I["Deploy to Staging"]
    I --> J["Smoke Tests<br/>Health checks"]
    
    J -->|❌ Fail| E
    J -->|✅ Pass| K["Manual QA Approval<br/>Test on staging URL"]
    
    K -->|Approved| L["Deploy to Production"]
    L --> M["Update DNS<br/>Gradual rollout"]
    
    M --> N["Production Smoke Tests"]
    N -->|❌ Fail| O["Automatic Rollback<br/>Previous version"]
    N -->|✅ Pass| P["✅ Deployment Complete<br/>Notify Team"]
    
    E --> Q["Fix Issues<br/>Create new PR"]
    Q -.->|Re-trigger| C
    
    O -.->|Investigate| Q
    
    style A fill:#e3f2fd
    style C fill:#f3e5f5
    style D fill:#fff9c4
    style G fill:#fff9c4
    style I fill:#ffe0b2
    style L fill:#c8e6c9
    style P fill:#c8e6c9
    style E fill:#ffccbc
    style O fill:#ffccbc
```

---

## 9. SECURITY & ENCRYPTION ARCHITECTURE

### Data Protection Layers

```mermaid
graph TD
    subgraph Input["🔓 User Input"]
        I1["Tourist Data"]
        I2["Health Records"]
        I3["Location"]
    end
    
    subgraph Transit["🔐 In Transit"]
        T1["TLS 1.3 Encryption"]
        T2["All HTTPS endpoints"]
        T3["JWT Bearer tokens"]
    end
    
    subgraph Storage["🔒 At Rest"]
        S1["MongoDB Field-Level<br/>Encryption<br/>(health records)"]
        S2["Database-Level<br/>Encryption<br/>(all data)"]
        S3["Hashed Passwords<br/>bcrypt (10 rounds)"]
    end
    
    subgraph Auth["🛡️ Authentication"]
        A1["JWT RS256 Signing"]
        A2["Access Token (1h)"]
        A3["Refresh Token (24h)"]
        A4["Rate Limiting<br/>on Auth endpoints"]
    end
    
    subgraph Blockchain["⛓️ Blockchain"]
        B1["Digital ID Contract<br/>on Polygon"]
        B2["Immutable Record<br/>Tamper-proof"]
        B3["QR Code Encoding<br/>Offline Verification"]
    end
    
    Input --> Transit
    Transit --> Storage
    Storage --> Auth
    Auth --> Blockchain
    
    style I1 fill:#ffccbc
    style I2 fill:#ff9999
    style I3 fill:#fff9c4
    style T1 fill:#c8e6c9
    style S1 fill:#c8e6c9
    style S3 fill:#c8e6c9
    style A1 fill:#bbdefb
    style B1 fill:#f3e5f5
```

---

## 10. WEARABLE ↔ APP COMMUNICATION PROTOCOL

### Real-Time Data Sync Between Devices

```mermaid
graph TD
    subgraph Wearable["📱 Wearable Device"]
        W1["STM32 Main Loop"]
        W2["Sensor Reading<br/>GPS + Health"]
        W3["Data Buffering<br/>LoRa TX"]
    end
    
    subgraph Connection["📡 Wireless Link"]
        C1["LoRa RF (915 MHz)"]
        C2["Range: 5–15 km"]
        C3["Low Power: ~50mA TX"]
    end
    
    subgraph App["📲 Tourist App"]
        A1["WebBluetooth API<br/>(near future)"]
        A2["Service Worker"]
        A3["Display vitals real-time"]
    end
    
    subgraph Dashboard["🎛️ Authority Dashboard"]
        D1["Receive wearable data<br/>via API"]
        D2["Store health history"]
        D3["Trigger anomaly detection"]
    end
    
    W1 -->|Loop 20s| W2
    W2 -->|Every 5–10 reads| W3
    W3 -->|Transmit| C1
    
    C1 -->|Receive| A1
    A1 -->|Process| A2
    A2 -->|Display| A3
    
    A2 -->|Forward| D1
    D1 -->|Analyze| D3
    D3 -->|Flag anomaly| Dashboard
    
    style Connection fill:#ffe0b2
    style Wearable fill:#f3e5f5
    style App fill:#c8e6c9
    style Dashboard fill:#bbdefb
```

---

**Complete Diagram Index (10 Total):**

1. **High-Level System Architecture** — 3-layer ecosystem + external services + bidirectional data connections
2. **Complete Layer-to-Layer Data Flows** — Wearable ↔ backend ↔ dashboard communication mesh with all protocols
3. **Tourist App: Complete Feature Stack** — Browser UI pages + React context + Service Worker background tracking + Safety scoring (dual-region) + Geofencing (entry/exit alerts) + Blockchain Digital ID (offline-accessible QR) + all integrations
4. **Authority Dashboard: Complete Real-Time Analytics Pipeline** — Data ingestion (tourist app + wearables + SOS) → processing → DBSCAN clustering + risk scoring + heatmap → Isolation Forest anomalies + severity classification → LLM insights + recommendations → Streamlit UI + officer controls → feedback loop
5. **IoT Wearable Hardware** — Triple-mode SOS transmission (Cloud/SMS/LoRa mesh)
6. **Emergency SOS Protocol** — Multi-channel alert dispatch (SMS, dashboard, LoRa broadcast)
7. **Database Schema** — MongoDB collections, relationships, and geospatial indexing
8. **CI/CD Pipeline** — GitHub Actions workflow (tests → build → staging → production → monitoring)
9. **Security Architecture** — Encryption layers (transport, auth, storage, blockchain)
10. **Wearable ↔ App Communication** — LoRa/BLE data sync between wearable and app

---

> All diagrams are interactive Mermaid flowcharts. You can copy them into [Mermaid Live Editor](https://mermaid.live/) for visualization and editing.
>
> **Key Enhancements:** 
> - **Diagram #3:** Now consolidates 4 key features into ONE "Tourist App: Complete Feature Stack" showing:
>   - Browser UI pages (Auth, Dashboard, Trip Planner, Profile)
>   - React context & state management
>   - Foreground tracking (real-time, 20s) + Service Worker background tracking (continues when closed)
>   - Smart batching (≥3 points OR 2 min threshold) + offline-first IndexedDB buffer + 3x retry backoff
>   - Safety scoring (Kolkata Model A: IDW+crime coupling vs Darjeeling Model B: IDW+gradient) with LLM narratives
>   - Geofencing (real-time zone detection, entry/exit alerts, polygon containment)
>   - Digital ID (blockchain deployment on Polygon Amoy, QR encoding, offline verification)
>   - All UI components (interactive map, safety meter, SOS button, geofence alerts, QR scanner, vitals)
>   - Next.js API routes with JWT auth + backend service integration
> 
> - **Diagram #4:** Now consolidates Authority Dashboard pipeline (previously 3 separate diagrams) into "Complete Real-Time Analytics Pipeline":
>   - Data ingestion: Tourist app locations + wearable sensors (GPS/health vitals) + SOS emergencies via WebSocket
>   - Processing layer: Validation → preprocessing → geocoding → geospatial indexing
>   - Clustering: DBSCAN (eps=50m, min_samples=3) → crowd detection → risk scoring (using crime data + road quality + patrol density) → heatmap (GREEN < 30, YELLOW 30-50, ORANGE 50-75, RED > 75)
>   - Anomaly detection: Feature engineering → Isolation Forest model → severity classification (Normal/Minor/Medium/Critical) → alert dispatch to officer panel
>   - AI insights: LangChain agent chain + Groq LLM (Llama 3, <50ms response) → patrol recommendations + resource allocation
>   - Dashboard UI: Streamlit interface with Folium map layer + officer alert panel + AI suggestions + control actions
>   - Feedback loop: Officer acknowledgement → zone status update → tourist notification via WebSocket
> 
> - **Benefits:**
>   - Diagram #3: Eliminates 3 redundant diagrams by merging safety, blockchain, and geofencing into single comprehensive feature view
>   - Diagram #4: Eliminates 2 redundant diagrams by integrating DBSCAN clustering and Isolation Forest anomalies into unified pipeline
>   - Overall: Reduces documentation from 16 → 10 diagrams while maintaining complete system visibility
> 
> - **Additional Note:** Diagram #2 shows complete bidirectional communication between wearable and dashboard including primary MQTT/4G, SMS fallback, LoRa P2P relay, and WebSocket real-time updates
