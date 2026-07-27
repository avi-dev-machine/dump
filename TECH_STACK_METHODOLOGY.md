# 💡 SETUKA TECH STACK & METHODOLOGY GUIDE

> Detailed justifications, best practices, and implementation patterns for every technology decision in Setuka

---

## TABLE OF CONTENTS

1. [Tech Stack Summary](#tech-stack-summary)
2. [Frontend Technologies](#frontend-technologies)
3. [Backend Technologies](#backend-technologies)
4. [ML & Data Science Stack](#ml--data-science-stack)
5. [Infrastructure & DevOps](#infrastructure--devops)
6. [Development Methodology](#development-methodology)
7. [Best Practices & Patterns](#best-practices--patterns)
8. [Performance Optimization](#performance-optimization)
9. [Scalability Considerations](#scalability-considerations)

---

## TECH STACK SUMMARY

### Bird's Eye View

```
┌─────────────────────────────────────────────────────────────┐
│ FRONTEND TIER (Tourist App)                                 │
├─────────────────────────────────────────────────────────────┤
│ • Next.js 14 (React 19, TypeScript)                         │
│ • Tailwind CSS + Radix UI Components                        │
│ • Leaflet + Mapbox GL (Interactive Maps)                    │
│ • RHF + Zod (Form Validation)                               │
│ • Service Worker + IndexedDB (Offline Support)              │
│ • JWT Authentication + bcryptjs                             │
├─────────────────────────────────────────────────────────────┤
│ BACKEND TIER (API + Dashboard)                              │
├─────────────────────────────────────────────────────────────┤
│ • FastAPI (Async REST + WebSocket)                          │
│ • Uvicorn (ASGI Server)                                     │
│ • MongoDB Atlas (NoSQL Persistence)                         │
│ • Streamlit (Dashboard UI)                                  │
├─────────────────────────────────────────────────────────────┤
│ ML/DATA SCIENCE TIER (Analysis)                             │
├─────────────────────────────────────────────────────────────┤
│ • Pandas + NumPy (Data Manipulation)                        │
│ • scikit-learn (DBSCAN, Isolation Forest)                   │
│ • GeoPandas + Shapely (Geospatial)                          │
│ • Plotly + Folium (Visualization)                           │
├─────────────────────────────────────────────────────────────┤
│ AI/LLM TIER (Intelligence)                                  │
├─────────────────────────────────────────────────────────────┤
│ • LangChain (LLM Orchestration)                             │
│ • Groq API (Llama 3 - Fast LLM)                             │
│ • Google Gemini (Secondary AI)                              │
├─────────────────────────────────────────────────────────────┤
│ BLOCKCHAIN TIER (Security)                                  │
├─────────────────────────────────────────────────────────────┤
│ • Polygon Amoy (Layer 2 Network)                            │
│ • ethers.js + viem (Web3 Integration)                       │
│ • Solidity Smart Contracts                                  │
├─────────────────────────────────────────────────────────────┤
│ IoT TIER (Hardware)                                         │
├─────────────────────────────────────────────────────────────┤
│ • STM32L476 (ARM Cortex-M4 MCU)                             │
│ • LoRa SX1262 (Long-Range Radio)                            │
│ • Quectel EC200U (GPS + GSM)                                │
│ • MAX30100 (Health Sensors)                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## FRONTEND TECHNOLOGIES

### 1. **Next.js 14 — Why It's Perfect for Setuka**

#### Pain Points Solved
| Problem | Solution |
|---------|----------|
| **Separate frontend/backend repos** | Built-in API routes (`/app/api/*`) |
| **Cold starts on API calls** | Edge Functions + ISR (Incremental Static Regeneration) |
| **PWA setup complexity** | `next-pwa` plugin — auto generates manifest + SW |
| **TypeScript overhead** | Native TS support, no babel configuration needed |
| **SEO for marketing** | Server-side rendering + open graph meta tags |
| **Offline support** | Static export + Service Worker integration |

#### Key Features We Leverage
```typescript
// 1. API Routes (replaces Express backend)
// pages/api/location/track.ts
export async function POST(req: NextRequest) {
  const locations = await req.json();
  // Validate & store in MongoDB
  return NextResponse.json({ tracked: 5 });
}

// 2. Middleware (Auth verification on all requests)
// middleware.ts
export function middleware(request: NextRequest) {
  const token = request.headers.get('authorization');
  if (!token) return NextResponse.redirect('/login');
  return NextResponse.next();
}

// 3. Server Components (Render on edge)
// app/dashboard/page.tsx
export default async function Dashboard() {
  const safetyScore = await fetchSafetyScore();
  return <ScoreDisplay score={safetyScore} />;
}

// 4. Service Worker Registration
// next.config.mjs
import withPWA from 'next-pwa';
export default withPWA({
  dest: 'public',
  skipWaiting: true,
  clientsClaim: true,
});
```

#### Why NOT Alternatives?
| Alternative | Why Not | Setuka Uses |
|-------------|--------|------------|
| **Vite + Express** | PWA setup manual, more boilerplate | Next.js (batteries included) |
| **Create React App** | Ejected, no PWA by default, slow build | Next.js (optimized) |
| **Svelte/Vue** | Smaller ecosystem for our tech stack | React (larger ecosystem) |

---

### 2. **Tailwind CSS + Radix UI — Utility-First + Unstyled**

#### Decision Rationale
```css
/* Tailwind: Utility-first approach */
<button className="px-4 py-2 bg-blue-500 hover:bg-blue-600 rounded-lg text-white">
  SOS
</button>

/* Why this matters for Setuka: */
✅ Consistency across 50+ emergency alert components
✅ Dark mode support (theme-provider.tsx)
✅ Mobile-responsive (tailwind breakpoints)
✅ Accessibility built-in (focus states, aria labels)
✅ No CSS naming conflicts (map components, dashboard, app)

/* Radix UI provides: */
✅ Unstyled accessibility primitives (Dialog, Popover, etc.)
✅ No style duplication battles
✅ Works seamlessly with Tailwind
✅ Small bundle size (~5KB gzipped)
```

#### Component Pattern
```typescript
// components/SOS-button.tsx
import * as AlertDialog from "@radix-ui/react-alert-dialog";
import { Button } from "@/components/ui/button";

export function SOSButton() {
  return (
    <AlertDialog.Root>
      <AlertDialog.Trigger asChild>
        <Button className="bg-red-500 hover:bg-red-600 w-full py-3">
          🚨 EMERGENCY SOS
        </Button>
      </AlertDialog.Trigger>
      <AlertDialog.Content>
        <AlertDialog.Title>Confirm SOS Alert</AlertDialog.Title>
        <AlertDialog.Cancel>Cancel</AlertDialog.Cancel>
        <AlertDialog.Action onClick={handleSOS}>Send Alert</AlertDialog.Action>
      </AlertDialog.Content>
    </AlertDialog.Root>
  );
}
```

---

### 3. **Leaflet + Mapbox GL — Lightweight + Production Maps**

#### Architecture Decision
```typescript
// Why two different mapping libraries?

// Leaflet: Simple, lightweight, perfect for tourist app
// ✅ ~40KB gzipped (small bundle impact)
// ✅ Built-in tile layer support (OpenStreetMap, Mapbox)
// ✅ Marker clusters (tourists on map)
// ✅ Geofence polygon rendering

import { MapContainer, TileLayer, Marker, Popup } from "react-leaflet";

export function TouristMap({ tourists }) {
  return (
    <MapContainer center={[28, 77]} zoom={11} style={{ height: "500px" }}>
      <TileLayer url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png" />
      {tourists.map(t => (
        <Marker key={t.id} position={[t.lat, t.lon]}>
          <Popup>{t.name}</Popup>
        </Marker>
      ))}
    </MapContainer>
  );
}

// Mapbox GL: Heavy-duty vector tiles, custom styling
// ✅ 3D terrain rendering (Darjeeling mountains!)
// ✅ Custom layer styling (safety zones in red)
// ✅ High-resolution raster tiles
// ✅ Better performance at zoom levels

import mapboxgl from 'mapbox-gl';

mapboxgl.accessToken = process.env.NEXT_PUBLIC_MAPBOX_TOKEN;
const map = new mapboxgl.Map({
  container: 'map',
  style: 'mapbox://styles/mapbox/streets-v11',
  center: [77.2, 28.6],
  zoom: 11,
  terrain: { source: 'mapbox-dem', exaggeration: 1.5 }
});
```

---

### 4. **Service Worker + IndexedDB — Offline Superpowers**

#### Why Service Worker for Setuka?

**The Problem:** When tourist's app is backgrounded, React stops running. GPS tracking stops. SOS won't transmit.

**The Solution:** Service Worker continues running even when app is closed.

```javascript
// public/sw-background-location.js
const CACHE_NAME = 'setuka-v1';
const LOCATIONS_DB = 'setuka-locations';

self.addEventListener('install', (event) => {
  event.waitUntil(self.skipWaiting());
});

// Periodic sync (if supported)
self.addEventListener('periodicsync', (event) => {
  if (event.tag === 'sync-locations') {
    event.waitUntil(syncLocations());
  }
});

// Main tracking loop
async function startTracking() {
  setInterval(async () => {
    const location = await getGPS(); // Native location API
    
    // Store in IndexedDB (persistent)
    const db = await openDB(LOCATIONS_DB);
    await db.add('locations', {
      lat: location.latitude,
      lon: location.longitude,
      timestamp: Date.now(),
    });
    
    // Try to upload
    uploadLocationsBatch();
  }, 30000); // Every 30 seconds
}

// Upload when batch is ready OR online
async function uploadLocationsBatch() {
  const db = await openDB(LOCATIONS_DB);
  const allLocations = await db.getAll('locations');
  
  if (allLocations.length >= 3) {
    try {
      const response = await fetch('/api/location/track', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${await getStoredToken()}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(allLocations)
      });
      
      if (response.ok) {
        await db.clear('locations'); // Clear after upload
      }
    } catch (err) {
      console.log('Offline - will retry later');
      // IndexedDB keeps locations, retry on next network connect
    }
  }
}

self.addEventListener('message', (event) => {
  if (event.data.type === 'START_TRACKING') {
    startTracking();
  }
});
```

#### IndexedDB Schema
```javascript
// Stores location data offline (doesn't require network)
{
  "setuka-locations": {
    keyPath: "id",
    indexes: [
      { name: "timestamp", keyPath: "timestamp" },
      { name: "userId", keyPath: "userId" }
    ]
  },
  "sos-alerts": {
    keyPath: "id",
    indexes: [{ name: "status", keyPath: "status" }]
  }
}
```

---

### 5. **React Hook Form + Zod — Type-Safe Forms**

#### Why This Combo?

```typescript
import { useForm } from 'react-hook-form';
import { z } from 'zod';
import { zodResolver } from '@hookform/resolvers/zod';

// 1. Define schema with Zod (type-safe validation)
const emergencyContactSchema = z.object({
  name: z.string().min(2, "Name too short"),
  phone: z.string().regex(/^\+91\d{10}$/, "Invalid Indian phone"),
  relationship: z.enum(["parent", "spouse", "sibling", "friend"]),
});

type EmergencyContact = z.infer<typeof emergencyContactSchema>;

// 2. Use in form with RHF (no extra validation boilerplate)
export function EmergencyContactForm() {
  const { register, formState: { errors }, handleSubmit } = useForm<EmergencyContact>({
    resolver: zodResolver(emergencyContactSchema),
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input 
        {...register('name')} 
        placeholder="Full Name"
      />
      {errors.name && <span>{errors.name.message}</span>}
      
      <input 
        {...register('phone')} 
        placeholder="+91 XXXXXXXXXX"
      />
      {errors.phone && <span>{errors.phone.message}</span>}
      
      <button type="submit">Save</button>
    </form>
  );
}

// Benefits:
// ✅ Single source of truth (schema definition)
// ✅ Type inference (TypeScript knows EmergencyContact.phone is string)
// ✅ Backend validation matches frontend (same schema)
// ✅ Error messages centralized
// ✅ Small bundle (~3KB)
```

---

## BACKEND TECHNOLOGIES

### 1. **FastAPI — Async, Validated, Fast**

#### Why FastAPI Over Django?

| Feature | FastAPI | Django |
|---------|---------|--------|
| **Request Speed** | ~3000 req/s (async) | ~500 req/s (sync) |
| **JSON Validation** | Built-in (Pydantic) | Manual |
| **WebSocket Support** | Native | Requires Channels |
| **Auto API Docs** | Yes (Swagger, ReDoc) | Manual setup |
| **Startup Time** | ~50ms | ~1000ms |
| **Deployment** | ~100MB Docker image | ~500MB Docker image |

#### FastAPI Architecture for Setuka

```python
# main.py (api.py in dashboard)
from fastapi import FastAPI, WebSocket, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel

app = FastAPI(title="Setuka Authority API")

# CORS for cross-origin requests (Next.js frontend)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://setuka.app", "http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 1. Request validation via Pydantic
class LocationUpdate(BaseModel):
    userId: str
    lat: float
    lon: float
    timestamp: int
    accuracy: float

class LocationsResponse(BaseModel):
    tracked: int
    failed: int
    anomalies: list

# 2. Async endpoint with dependency injection
@app.post("/location/track", response_model=LocationsResponse)
async def track_locations(
    locations: list[LocationUpdate],
    current_user = Depends(verify_jwt_token)
):
    """
    - Validates input schema automatically
    - Returns schema-conforming response
    - Generates Swagger docs automatically
    """
    stored = []
    for loc in locations:
        result = await db.locations.insert_one({
            "userId": current_user.id,
            "lat": loc.lat,
            "lon": loc.lon,
            "timestamp": loc.timestamp,
        })
        stored.append(result.inserted_id)
    
    return LocationsResponse(
        tracked=len(stored),
        failed=len(locations) - len(stored),
        anomalies=[]
    )

# 3. WebSocket for real-time updates
@app.websocket("/ws/live-feed")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    while True:
        # Broadcast location updates to all connected officers
        live_data = await get_live_locations()
        await websocket.send_json(live_data)
        await asyncio.sleep(1)  # Update every second
```

---

### 2. **MongoDB + Geospatial Indexing**

#### Why MongoDB for Setuka?

```javascript
// 1. Flexible schema for evolving features
db.users.insertOne({
  _id: ObjectId(...),
  email: "tourist@example.com",
  name: "Raj Kumar",
  medicalInfo: {
    bloodGroup: "O+",
    allergies: ["Penicillin"],
    insuranceProvider: "Bajaj Allianz"
  },
  // Can add new fields without migration
  wearableDeviceId: "D12345",  // Added later
  preferredLanguage: "Hindi"   // Added later
});

// 2. Geospatial indexing for fast location queries
db.locations.createIndex({ "coordinates": "2dsphere" });

// Find all tourists within 500m of Darjeeling
db.locations.find({
  coordinates: {
    $near: {
      $geometry: {
        type: "Point",
        coordinates: [88.2681, 27.0410]
      },
      $maxDistance: 500  // meters
    }
  }
});

// 3. Time-Series Collections (new in MongoDB 5.0)
// Perfect for rapid location updates
db.createCollection("locations", {
  timeseries: {
    timeField: "timestamp",
    metaField: "metadata",
    granularity: "minutes"
  }
});

// 4. Change Streams (real-time triggers)
const changeStream = db.sos_alerts.watch([
  { $match: { operationType: "insert" } }
]);

changeStream.on('change', (change) => {
  console.log("New SOS Alert:", change.fullDocument);
  // Broadcast to WebSocket clients
  broadcast({ type: 'new_sos', data: change.fullDocument });
});
```

#### MongoDB Indexing Strategy

```javascript
// Query patterns for Setuka:

// 1. Get recent locations for a user
db.locations.createIndex({ "userId": 1, "timestamp": -1 });

// 2. Anomaly queries (find by type & severity)
db.anomalies.createIndex({ "userId": 1, "severity": 1 });

// 3. Geofence lookups
db.geofences.createIndex({ "polygon": "2dsphere" });

// 4. Time-range queries (last 24 hours)
db.locations.createIndex({ "timestamp": 1 }, { expireAfterSeconds: 86400 });
```

---

### 3. **Streamlit — Rapid Dashboard Development**

#### Why Streamlit for Authority Dashboard?

```python
# Unlike traditional web frameworks (Flask, Django)
# Streamlit is SCRIPT-FIRST, not ROUTE-FIRST

# Traditional web framework
@app.route('/dashboard', methods=['GET'])
def dashboard():
    tourists = query_db("SELECT * FROM tourists WHERE active=1")
    return render_template('dashboard.html', tourists=tourists)

# Streamlit approach
import streamlit as st

st.title("🛡️ Authority Dashboard")

# 1. Data loading (cached)
@st.cache_data
def load_tourist_data():
    return query_db("SELECT * FROM tourists WHERE active=1")

tourists = load_tourist_data()

# 2. Display (automatic UI generation)
st.metric("Active Tourists", len(tourists))

# 3. Visualization (one-liner)
st.map(tourists[['latitude', 'longitude']])

# 4. Form
with st.form("filter_form"):
    risk_level = st.selectbox("Filter by risk", ["Low", "Medium", "High"])
    submitted = st.form_submit_button("Apply")
    if submitted:
        st.write(f"Filtering by {risk_level}")

# Benefits:
# ✅ No HTML/CSS/JavaScript needed
# ✅ Data scientist can build dashboard
# ✅ Hot reload (changes reflect immediately)
# ✅ Public deployment (Streamlit Cloud)
# ✅ Built-in state management
```

#### Multi-Page Streamlit Structure

```
authority-dashboard/
├── app/
│   ├── main.py           # Home + live map
│   └── pages/
│       ├── 1_Crowd_Safety_Scoring.py
│       ├── 2_AI_Recommendations.py
│       └── 3_Tourist_Anomaly_Detection.py
```

```python
# Run streamlit app/ → auto-creates sidebar navigation
# Each page is a separate script
```

---

## ML & DATA SCIENCE STACK

### 1. **scikit-learn: DBSCAN for Crowd Clustering**

#### Why DBSCAN Over K-Means?

```python
# DBSCAN: Finds clusters of ANY shape, auto-detects outliers
from sklearn.cluster import DBSCAN
import numpy as np

# Tourist GPS points
X = np.array([
    [28.60, 77.20],  # Kolkata location
    [28.61, 77.21],
    [28.62, 77.22],  # Cluster 1
    [27.04, 88.27],  # Darjeeling location (50 km away)
    [27.05, 88.28],  # Cluster 2
])

# Fit DBSCAN
db = DBSCAN(eps=0.1, min_samples=2)  # eps in degrees ≈ 11 km
labels = db.fit_predict(X)

# Result:
# labels = [0, 0, 0, 1, 1]
# Correctly identifies TWO separate clusters despite distance
# K-Means would have created unnecessary clusters in between!

# Benefits for tourists:
# ✅ Realistic crowd formations (not forced into k groups)
# ✅ Outliers flagged automatically
# ✅ No need to guess optimal clusters
# ✅ <1ms inference per point
```

---

### 2. **Isolation Forest for Anomaly Detection**

#### Why Isolation Forest for Tourist Route Tracking?

```python
from sklearn.ensemble import IsolationForest

# Features for each location point
X_features = np.array([
    [speed, bearing_change, inactivity_duration, distance_from_route],
    [5.2, 15, 30, 50],    # Normal
    [8.1, 12, 25, 45],    # Normal
    [120, 180, 300, 2000], # ANOMALY! (stopped for 5+ min, far from route)
])

# Train on unlabeled data (unsupervised)
iso_forest = IsolationForest(contamination=0.1)
anomaly_labels = iso_forest.fit_predict(X_features)

# Result:
# -1 = anomaly (flag to officer)
# +1 = normal (continue)

# Why Isolation Forest?
# ✅ No labeled training data needed
# ✅ Detects multi-dimensional anomalies
# ✅ Works on sparse data (mountain regions)
# ✅ ~ms inference time
# ✅ Built-in outlier scoring (0–1 probability)

# Output
anomaly_scores = iso_forest.score_samples(X_features)
# [+0.2, +0.15, -0.95]  # Last one is 95% likely anomaly
```

#### Feature Engineering for Anomaly Detection

```python
def compute_features(prev_location, current_location, trip_route):
    """
    Extract features for anomaly detection
    """
    # 1. Speed anomaly
    distance = haversine_distance(prev_location, current_location)
    time_diff = current_location['timestamp'] - prev_location['timestamp']
    speed = distance / time_diff if time_diff > 0 else 0
    
    # 2. Direction anomaly
    bearing = calculate_bearing(prev_location, current_location)
    route_bearing = calculate_bearing(trip_route[i], trip_route[i+1])
    bearing_change = abs(bearing - route_bearing)
    
    # 3. Inactivity
    inactivity_duration = current_location['timestamp'] - last_movement_time
    
    # 4. Route deviation
    distance_from_route = min_distance_to_route(current_location, trip_route)
    
    return [speed, bearing_change, inactivity_duration, distance_from_route]
```

---

### 3. **GeoPandas + Shapely — Geospatial Operations**

#### Why These Libraries?

```python
import geopandas as gpd
from shapely.geometry import Point, Polygon

# 1. Geofence as GeoDataFrame
geofences = gpd.GeoDataFrame({
    'name': ['High Crime Zone', 'Mountain Path'],
    'risk_level': ['high', 'medium'],
    'geometry': [
        Polygon([[77.2, 28.6], [77.3, 28.6], [77.3, 28.7], [77.2, 28.7]]),
        Polygon([[88.2, 27.0], [88.3, 27.0], [88.3, 27.1], [88.2, 27.1]])
    ]
})

# 2. Tourist location
tourist_point = Point(77.25, 28.65)

# 3. Fast intersection check
geofences['contains_tourist'] = geofences.geometry.contains(tourist_point)

# Result: tourist_point is in "High Crime Zone"

# Benefits:
# ✅ Efficient spatial indexing
# ✅ One-liner polygon operations
# ✅ GeoJSON export for maps
# ✅ CRS (coordinate reference system) handling
```

---

### 4. **Plotly + Folium — Interactive Visualizations**

#### Comparison

```python
import plotly.graph_objects as go
import folium

# PLOTLY: Time-series, aggregated analytics
fig = go.Figure()
fig.add_trace(go.Scatter(
    x=["12:00", "12:30", "13:00"],
    y=[45, 67, 52],
    name="Tourists"
))
fig.add_trace(go.Scatter(
    x=["12:00", "12:30", "13:00"],
    y=[2, 5, 3],
    name="SOS Alerts"
))
fig.show()  # Interactive chart

# FOLIUM: Geographic data, heatmaps
m = folium.Map(location=[28.6, 77.2], zoom_start=11)
folium.HeatMap(
    [[28.6, 77.2], [28.61, 77.21], [28.62, 77.22]],
    radius=20
).add_to(m)
m.save('map.html')

# Use both:
# ✅ Plotly for top-N cities, time trends
# ✅ Folium for geographic heatmaps
```

---

## INFRASTRUCTURE & DEVOPS

### 1. **MongoDB Atlas — Managed Database**

#### Why Atlas vs Self-Hosted?

| Aspect | MongoD Atlas | Self-Hosted |
|--------|---------------|----|
| **Backup** | Automatic daily | Manual scripts |
| **Scaling** | Automatic sharding | Manual |
| **Uptime SLA** | 99.95% guaranteed | Your responsibility |
| **Security** | Encryption, whitelisting | DIY |
| **Cost** | $10–100/month | $50+ for EC2 + ops |

#### Atlas Setup for Setuka

```javascript
// Connection string from Atlas dashboard
const mongoURI = "mongodb+srv://<user>:<pass>@setuka-cluster.mongodb.net/setuka?retryWrites=true&w=majority";

// Python
from motor.motor_asyncio import AsyncClient
client = AsyncClient(mongoURI)
db = client["setuka"]

// Node.js
import { MongoClient } from "mongodb";
const client = new MongoClient(mongoURI);
await client.connect();
const db = client.db("setuka");
```

---

### 2. **Vercel — Next.js Deployment**

#### Why Vercel for Tourist App?

```
✅ Native Next.js support (built by creators)
✅ Edge functions (global latency <50ms)
✅ Automatic HTTPS + CDN
✅ Preview deployments for PRs
✅ Analytics dashboard
✅ Serverless auto-scaling
✅ Free tier: great for MVP
✅ Custom domain: $10/month for pro features
```

#### Deployment Flow

```yaml
Git Push → GitHub → Vercel Webhook
  ↓
Vercel Build
  - npm install
  - npm run build (next build)
  - npm run lint
  ↓
Build Artifacts
  - .next/ (optimized app)
  - public/ (static files)
  ↓
Deploy to Edge + Serverless
  - API routes → AWS Lambda
  - Static → CloudFront CDN
  ↓
Smoke Tests
  ✅ Health check
  ✅ API endpoints
  ↓
✅ LIVE at https://setuka.app
```

---

### 3. **Render — FastAPI Deployment**

```yaml
# render.yaml
services:
  - type: web
    name: setuka-api
    env: python
    buildCommand: "pip install -r requirements.txt"
    startCommand: "uvicorn api:app --host 0.0.0.0 --port $PORT"
    envVars:
      - key: MONGODB_URI
        fromDatabase:
          name: setuka-db
          property: connectionString
    disk:
      name: data_disk
      mountPath: /data
      sizeGB: 10
```

---

## DEVELOPMENT METHODOLOGY

### 1. **Agile Sprints with Hardware Validation**

The 9-phase development process:

```
Phase 1-2: PCB Design (2 weeks)
├─ Component selection
├─ Schematic design
└─ PCB layout

Phase 3-4: Soldering & Assembly (3 weeks)
├─ Solder STM32, LoRa, GPS modules
├─ Power distribution testing
└─ Initial power-on tests

Phase 5-6: Sensor Integration (3 weeks)
├─ Gyro/Accelerometer calibration
├─ SpO₂ sensor tuning
├─ LCD debugging (often stuck here!)
└─ OLED software integration

Phase 7-8: Firmware & Communication (3 weeks)
├─ UART drivers (GPS, GSM)
├─ LoRa radio protocol stack
├─ Triple-mode SOS logic
└─ Over-the-air firmware update

Phase 9: Field Testing (2 weeks)
├─ Range testing (15km target)
├─ Battery endurance (24h target)
├─ Real-world scenario validation
└─ Production readiness
```

**Key: Hardware + Software iterate in parallel, not sequentially.**

---

### 2. **Data-Driven ML Model Development**

#### Safety Score Model Training Pipeline

```python
# Phase 1: Data Collection (1 week)
# Manually research 536 locations across 2 cities
kolkata_data = pd.read_csv('kolkata_536_locations.csv')
darjeeling_data = pd.read_csv('darjeeling_286_locations.csv')

# Phase 2: Exploratory Data Analysis (1 week)
# Check correlations
kolkata_corr = kolkata_data.corr()
# Result: road & crime = -0.83 (strong negative correlation!)

# Phase 3: Model Development (2 weeks)
# Kolkata: IDW + coupling
# Darjeeling: IDW + gradient

# Phase 4: Validation (1 week)
# Leave-one-out cross-validation
# Kolkata MAE: 0.474 ✅
# Darjeeling MAE: 0.369 ✅

# Phase 5: Production Deployment (rest of sprint)
# Deploy to: safetyscore-regression.onrender.com
```

---

### 3. **Testing Strategy**

#### Testing Pyramid for Setuka

```
        ▲
        │     E2E Tests (5%)
        │     • Full SOS flow
        │     • Geofence triggering
        │    ╱════════════╲
        │  Integration Tests (25%)
        │  • API + Database
        │  • DBSCAN clustering
        │  • LLM prompt chains
        │ ╱═════════════════════╲
        │Unit Tests (70%)
        │• Location validation
        │• JWT token verification
        │• Anomaly scoring
        │╱═════════════════════════╲
        └─────────────────────────────
```

#### Example Unit Test: Isolation Forest

```python
import pytest
from isolation_module.detect_anomalies import score_anomaly

def test_anomaly_detection_normal():
    """Normal tourist movement should not be flagged"""
    features = [5.2, 15, 30, 50]  # Normal speed, direction, inactivity, deviation
    score = score_anomaly(features)
    assert score < 0.5, "Normal movement flagged as anomaly"

def test_anomaly_detection_stopped():
    """Tourist stopped for 5+ min far from route = ANOMALY"""
    features = [0.1, 180, 300, 2000]  # Stopped, direction flip, long inactivity, far
    score = score_anomaly(features)
    assert score > 0.8, "Stopped tourist not flagged as anomaly"

def test_anomaly_batch_performance():
    """Batch processing should stay <500ms for 1000 points"""
    import time
    features_batch = [[5, 15, 30, 50] for _ in range(1000)]
    
    start = time.time()
    scores = [score_anomaly(f) for f in features_batch]
    elapsed = time.time() - start
    
    assert elapsed < 0.5, f"Batch processing too slow: {elapsed}s"
```

---

### 4. **Documentation-First Approach**

#### Three Types of Docs:

1. **Challenge Documentation** (CHALLENGES.md)
   - Real problems + how we solved them
   - For future developers & stakeholders

2. **API Documentation** (Auto-generated)
   ```
   FastAPI + Swagger: /docs
   Streamlit + reStructuredText: built-in
   ```

3. **Architecture & Runbooks**
   - This blueprint
   - Deployment guides
   - Troubleshooting guides

---

## BEST PRACTICES & PATTERNS

### 1. **Error Handling Pattern**

#### Frontend (Next.js)

```typescript
// lib/api-client.ts
export async function fetchSafetyScore(lat: number, lon: number) {
  try {
    const response = await fetch(`/api/safety-score?lat=${lat}&lon=${lon}`);
    
    if (!response.ok) {
      const error = await response.json();
      throw new AppError(error.detail, response.status);
    }
    
    return await response.json();
  } catch (err) {
    if (err instanceof AppError) {
      toast.error(err.message);  // User-facing error
      logger.warn(`API error: ${err.code}`);  // Debug logging
    } else {
      toast.error("Connection failed. Retrying...");
      return retry(() => fetchSafetyScore(lat, lon), 3);  // Auto-retry
    }
  }
}
```

#### Backend (FastAPI)

```python
from fastapi import HTTPException

@app.post("/location/track")
async def track_locations(locations: list[LocationUpdate]):
    if not locations:
        raise HTTPException(
            status_code=400,
            detail="Empty location batch"
        )
    
    try:
        results = await db.locations.insert_many([...])
    except pymongo.errors.DuplicateKeyError:
        raise HTTPException(
            status_code=409,
            detail="Duplicate location entry"
        )
    
    return { "tracked": len(results) }
```

---

### 2. **Caching Strategy**

#### Three Layers:

```
┌─────────────────────────────┐
│ Browser Cache (ETags)        │
│ Setuka app store static maps │
└─────────┬───────────────────┘
          │
┌─────────▼──────────────────────┐
│ Redis Cache (5–60 min TTL)      │
│ Safety scores, cluster results  │
└─────────┬──────────────────────┘
          │
┌─────────▼─────────────────────────┐
│ MongoDB Primary Cache (indexes)    │
│ Location & user queries            │
└───────────────────────────────────┘
```

```python
# Streamlit caching (authority dashboard)
@st.cache_data(ttl=300)  # 5-minute cache
def load_live_locations():
    return list(db.locations.find(
        {"timestamp": {"$gte": time.time() - 1800}}  # Last 30 min
    ))

# FastAPI caching
from functools import lru_cache

@lru_cache(maxsize=1000)
def get_safety_score_cached(lat: float, lon: float):
    return call_regression_api(lat, lon)

# Manual cache invalidation
@app.post("/safety-score/invalidate")
async def invalidate_cache():
    get_safety_score_cached.cache_clear()
```

---

### 3. **Logging & Monitoring**

#### Structured Logging

```python
import logging
import json

logger = logging.getLogger(__name__)

# Log with context
logger.info(
    "Location tracked",
    extra={
        "userId": user_id,
        "lat": lat,
        "lon": lon,
        "accuracy": accuracy,
        "source": "wearable",
        "timestamp": dt.isoformat()
    }
)

# Output (JSON format for log aggregation)
{
  "timestamp": "2026-03-23T10:30:45Z",
  "level": "INFO",
  "message": "Location tracked",
  "userId": "user123",
  "accuracy": 5.2,
  "source": "wearable"
}
```

#### Key Metrics to Monitor

```
Tourist App:
├─ Page Load Time (target: <2s)
├─ API Latency (target: <100ms p95)
├─ PWA Installation Rate
└─ Background Sync Success Rate

Authority Dashboard:
├─ Rerender Latency (target: <2s)
├─ Anomaly Detection Accuracy
├─ DBSCAN Clustering Performance
└─ LLM Response Time

Wearable:
├─ GPS Accuracy
├─ LoRa Range Performance
├─ Battery Drain Rate
└─ SOS Transmission Success Rate
```

---

## PERFORMANCE OPTIMIZATION

### 1. **Frontend Optimization**

```typescript
// Code splitting
import dynamic from 'next/dynamic';

// Load map only when needed
const InteractiveMap = dynamic(
  () => import('@/components/interactive-map'),
  { loading: () => <Skeleton /> }
);

// Image optimization
<Image 
  src="/logo.png"
  alt="Setuka"
  width={200}
  height={200}
  priority  // Load above the fold
  quality={75}  // JPEG quality
/>

// Bundle analysis
// npm run analyze
// Check which libraries bloat the bundle
```

### 2. **API Optimization**

```python
# Pagination (avoid loading 100k tourists at once)
@app.get("/locations")
async def get_locations(skip: int = 0, limit: int = 100):
    return list(db.locations.find().skip(skip).limit(limit))

# Projection (return only needed fields)
db.locations.find(
    {"userId": "user123"},
    {"lat": 1, "lon": 1, "timestamp": 1}  # Only these fields
)

# Batch processing
# Instead of: 1000 individual INSERT queries
# Do: 1 INSERT with 1000 documents
db.locations.insert_many(locations)

# Connection pooling
# FastAPI + Motor handles this automatically
```

---

## SCALABILITY CONSIDERATIONS

### 1. **From MVP to 10,000 Tourists**

```
Current (MVP):
├─ Single FastAPI server
├─ Single MongoDB instance
└─ Handles: 100 tourists, ~1000 locations/min

10K Tourists:
├─ API server + replica set
├─ MongoDB sharding by userId
├─ Redis for session cache
├─ Handles: 10,000 tourists, ~100K locations/min
└─ Infrastructure cost: ~$200/month
```

### 2. **Database Sharding Strategy**

```python
# Shard by userId to distribute load
# Shards: 0–25%, 25–50%, 50–75%, 75–100%

# Query stays on single shard (fast)
db.locations.find({"userId": "user123"})

# But aggregation requires merging shards
# Use aggregation pipeline carefully
```

---

**End of Tech Stack & Methodology Guide**
