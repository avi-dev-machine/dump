# 🏋️ FITVISION — ARCHITECTURE DIAGRAMS

> Visual representations of FitVision's system architecture, real-time pipeline, exercise state machines, AI integrations, and deployment topology.
> Smart India Hackathon 2025 Finalist | FastAPI · YOLO11n-Pose · OpenCV · Next.js · Gemini · Groq

---

## 1. HIGH-LEVEL SYSTEM ARCHITECTURE

### 3-Layer Ecosystem Overview

```mermaid
graph TB
    subgraph Layer1["🖥️ LAYER 1: FRONTEND (Next.js — Vercel)"]
        F1["Next.js 16 App Router<br/>React 19 + CSS Modules"]
        F2["useExerciseSocket Hook<br/>WebSocket + Flow Control"]
        F3["Camera API<br/>getUserMedia 320x240 @ 15 FPS"]
        F4["REST API Client<br/>lib/api.js — Session CRUD"]
    end

    subgraph Layer2["⚙️ LAYER 2: BACKEND (FastAPI — HF Spaces Docker)"]
        subgraph SessionMgr["Session Management"]
            S1["active_sessions Dict<br/>In-Memory Store"]
            S2["ExerciseSession Object<br/>Per-User Isolated State"]
        end
        subgraph CVPipeline["Computer Vision Pipeline"]
            CV1["PoseCalibrator<br/>YOLO11n-pose Inference"]
            CV2["17 Keypoints (COCO)<br/>Confidence Filtering"]
            CV3["Joint Angle Engine<br/>Vector Dot Product"]
            CV4["Height Calibration<br/>px to cm Conversion"]
        end
        subgraph ExerciseEngine["Exercise Processing"]
            EX1["8 State Machines<br/>Pushup / Squat / Situp / SitnReach<br/>Skipping / JJ / VJump / BJump"]
            EX2["PerformanceMetrics<br/>Per-Rep Data Accumulation"]
            EX3["3-Frame Moving Average<br/>Angle Smoothing"]
        end
    end

    subgraph Layer3["🤖 LAYER 3: AI SERVICES"]
        AI1["Gemini 2.5 Flash<br/>LangChain — AI Coach"]
        AI2["Groq LLaMA 3.3-70B<br/>LangChain — Dietary Assistant"]
        AI3["Metrics JSON<br/>results/ on disk"]
    end

    subgraph Infra["☁️ INFRASTRUCTURE"]
        I1["Hugging Face Spaces<br/>Docker SDK — Port 7860"]
        I2["Vercel<br/>Frontend Hosting"]
        I3["Git LFS<br/>YOLO .pt weights ~12 MB"]
    end

    F4 -->|"POST /session/create"| S1
    F4 -->|"GET /session/id/metrics"| S2
    F4 -->|"DELETE /session/id"| S1

    F2 -->|"WSS: binary JPEG / 35% quality / 15-20 KB"| CV1
    CV1 -->|"JSON + base64 frame / metrics"| F2
    F3 -->|"Canvas toBlob"| F2

    CV1 --> CV2 --> CV3 --> CV4
    CV3 --> EX1
    CV4 --> EX1
    EX1 --> EX2
    EX3 --> EX1

    S1 --> S2
    S2 --> CV1
    S2 --> EX2

    EX2 -->|"metrics JSON saved"| AI3
    AI3 -->|"POST /analyze"| AI1
    AI1 -->|"Athletic profile + Sport recs + Score/100"| F1
    F1 -->|"POST /api/diet/chat"| AI2
    AI2 -->|"Nutrition advice < 1s"| F1

    Layer2 -.->|"Deployed on"| I1
    Layer1 -.->|"Deployed on"| I2
    I3 -.->|"weights pulled at build"| CV1

    style Layer1 fill:#e1f5ff,stroke:#01579b,stroke-width:3px
    style Layer2 fill:#e8f5e9,stroke:#1b5e20,stroke-width:3px
    style Layer3 fill:#fff3e0,stroke:#e65100,stroke-width:3px
    style Infra fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    style SessionMgr fill:#c8e6c9,stroke:#2e7d32,stroke-width:1px
    style CVPipeline fill:#b3e5fc,stroke:#0277bd,stroke-width:1px
    style ExerciseEngine fill:#ffe0b2,stroke:#e65100,stroke-width:1px
```

---

## 2. WEBSOCKET STREAMING PIPELINE (Frame-by-Frame)

### Client to Server Real-Time Loop with Flow Control

```mermaid
graph TD
    subgraph Client["🖥️ CLIENT — useExerciseSocket Hook"]
        C1["getUserMedia<br/>320x240 @ 15 FPS"]
        C2["requestAnimationFrame Loop<br/>capture every frame"]
        C3["drawImage to Canvas<br/>offscreen 320x240"]
        C4["canvas.toBlob<br/>JPEG 35% quality / 15-20 KB"]
        C5{"pendingFrameRef = false?"}
        C6["ws.send blob<br/>pendingFrameRef = true"]
        C7["Receive JSON response<br/>pendingFrameRef = false"]
        C8["Render annotated frame<br/>update counter / stage / feedback"]
        C9["Update FPS counter<br/>responses per second"]
    end

    subgraph Server["⚙️ SERVER — FastAPI WebSocket Endpoint"]
        S1["await websocket.receive<br/>timeout = 30s"]
        S2["np.frombuffer bytes<br/>cv2.imdecode to numpy array"]
        S3["YOLO11n-pose inference<br/>results.keypoints.data.cpu.numpy"]
        S4["17 keypoints extracted<br/>x, y, confidence per joint"]
        S5["_calibrate_height<br/>first 30 frames"]
        S6["process exercise<br/>angles to state machine"]
        S7["PerformanceMetrics.update<br/>record rep data"]
        S8["cv2.imencode JPEG 40%<br/>base64 encode frame"]
        S9["websocket.send_json<br/>frame + counter + stage + feedback"]
        S10["asyncio.TimeoutError<br/>send keepalive ping"]
    end

    C1 --> C2 --> C3 --> C4 --> C5
    C5 -->|"YES — send"| C6
    C5 -->|"NO — skip frame"| C2
    C6 -->|"Binary over WSS"| S1
    S1 --> S2 --> S3 --> S4
    S4 --> S5
    S5 -->|"calibration_complete"| S6
    S6 --> S7 --> S8 --> S9
    S1 -->|"30s no frame"| S10
    S10 -.->|"keepalive"| C7
    S9 -->|"JSON response"| C7
    C7 --> C8 --> C9
    C9 --> C2

    style Client fill:#e1f5ff,stroke:#01579b,stroke-width:2px
    style Server fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px
    style C5 fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style S10 fill:#ffe0b2,stroke:#e65100,stroke-width:2px
```

---

## 3. PIXEL-TO-CM CALIBRATION ALGORITHM

### Height Calibration to Real-World Measurements

```mermaid
graph TD
    A["User enters height_cm<br/>e.g. 173 cm"] --> B["POST /session/create<br/>height_cm stored in ExerciseSession"]
    B --> C["WebSocket streaming begins"]
    C --> D{"calibration_complete?"}
    D -->|"NO"| E["_calibrate_height keypoints"]
    E --> F["Extract: nose kp-0 + best ankle kp-15 or 16"]
    F --> G{"head conf > 0.3<br/>ankle conf > 0.3?"}
    G -->|"NO"| C
    G -->|"YES"| H["H_p = sqrt headX-ankleX sq + headY-ankleY sq<br/>Euclidean pixel height"]
    H --> I{"H_p >= 40 pixels?"}
    I -->|"NO — person too far"| C
    I -->|"YES"| J["calibration_frames.append H_p"]
    J --> K{"len frames >= 30?"}
    K -->|"NO — accumulate"| C
    K -->|"YES — lock in"| L["avg_H_p = mean calibration_frames<br/>cm_per_pixel = user_height_cm / avg_H_p"]
    L --> M["calibration_complete = True<br/>metrics.cm_per_pixel = ratio"]
    D -->|"YES"| N["Apply ratio to all measurements"]
    N --> O["jump_height_cm = height_px x cm_per_pixel"]
    N --> P["reach_dist_cm = reach_px x cm_per_pixel"]
    N --> Q["broad_jump_cm = distance_px x cm_per_pixel"]

    style A fill:#e1f5ff,stroke:#01579b
    style L fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style M fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style O fill:#fff9c4,stroke:#f57f17
    style P fill:#fff9c4,stroke:#f57f17
    style Q fill:#fff9c4,stroke:#f57f17
```

---

## 4. ALL 8 EXERCISE STATE MACHINES

### Rep Counting via Joint Angle Geometry

```mermaid
graph TB
    subgraph PushUp["PUSH-UP — Elbow Angle"]
        PU1["elbow > 140 deg — stage = UP"]
        PU2["elbow < 90 deg — stage = DOWN"]
        PU3{"DOWN to UP?"}
        PU4["counter += 1"]
        PU5{"hip < 110 deg?"}
        PU6["feedback = Fix Back!"]
        PU7["feedback = Good Form"]
        PU1 --> PU3
        PU2 --> PU3
        PU3 -->|"YES"| PU4
        PU4 --> PU5
        PU5 -->|"YES"| PU6
        PU5 -->|"NO"| PU7
    end

    subgraph Squat["SQUAT — Knee Angle"]
        SQ1["knee > 155 deg — UP"]
        SQ2["knee < 135 deg — DOWN"]
        SQ3{"UP to DOWN?"}
        SQ4["counter += 1<br/>record squat_depth"]
        SQ5{"knee < 90 deg?"}
        SQ6["Good rep / Great Depth!"]
        SQ7["Bad rep / Go Lower"]
        SQ1 --> SQ3
        SQ2 --> SQ3
        SQ3 -->|"YES"| SQ4
        SQ4 --> SQ5
        SQ5 -->|"YES"| SQ6
        SQ5 -->|"NO"| SQ7
    end

    subgraph SitUp["SIT-UP — Torso Inclination"]
        ST1["torso <= 20 deg — rest / DOWN"]
        ST2["torso >= 70 deg — peak / UP"]
        ST3{"rest to peak?"}
        ST4["counter += 1<br/>record concentric time"]
        ST1 --> ST3
        ST2 --> ST3
        ST3 -->|"YES"| ST4
    end

    subgraph SitnReach["SIT-AND-REACH — Max Reach"]
        SR1["reach_px = wrist_x minus ankle_x"]
        SR2["max_reach = max all readings"]
        SR3["counter = int max_reach_px"]
        SR4{"knee < 165 deg?"}
        SR5["feedback = Straighten Legs!"]
        SR1 --> SR2 --> SR3 --> SR4
        SR4 -->|"YES"| SR5
    end

    subgraph VJump["VERTICAL JUMP — Peak Ankle Y"]
        VJ1["ground_baseline_y locked at standing"]
        VJ2{"ankle_y < ground minus threshold?"}
        VJ3["track min_ankle_y during airborne"]
        VJ4["height_px = ground_y minus min_ankle_y<br/>height_cm = px x cm_per_pixel"]
        VJ5["vjump_jump_count += 1"]
        VJ2 -->|"YES — airborne"| VJ3
        VJ3 -->|"ankle returns to ground"| VJ4
        VJ4 --> VJ5
    end

    subgraph BJump["BROAD JUMP — Horizontal Distance"]
        BJ1["takeoff_x = ankle_x at jump start"]
        BJ2["landing_x = ankle_x on landing"]
        BJ3["dist_px = landing_x minus takeoff_x<br/>dist_cm = px x cm_per_pixel"]
        BJ4["bjump_jump_count += 1<br/>update max_distance"]
        BJ1 --> BJ2 --> BJ3 --> BJ4
    end

    subgraph Skipping["SKIPPING — Ankle Y Position"]
        SK1["ground_baseline = avg ankle Y at rest"]
        SK2{"ankle_y < ground minus threshold?"}
        SK3["state = airborne"]
        SK4["state = landing<br/>jump_count += 1 / record airtime"]
        SK1 --> SK2
        SK2 -->|"YES"| SK3
        SK3 -->|"ankle returns"| SK4
        SK4 --> SK1
    end

    subgraph JJacks["JUMPING JACKS — Dual State"]
        JJ1{"arms open > 150 deg<br/>AND legs open > 150 deg?"}
        JJ2["state = OPEN"]
        JJ3{"OPEN to CLOSED?"}
        JJ4["jj_rep_count += 1"]
        JJ1 -->|"YES"| JJ2
        JJ2 --> JJ3
        JJ3 -->|"YES"| JJ4
    end

    style PushUp fill:#e1f5ff,stroke:#01579b,stroke-width:2px
    style Squat fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px
    style SitUp fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style SitnReach fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    style Skipping fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style VJump fill:#ffe0b2,stroke:#e65100,stroke-width:2px
    style BJump fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style JJacks fill:#ffccbc,stroke:#bf360c,stroke-width:2px
```

---

## 5. SESSION LIFECYCLE AND IN-MEMORY MANAGEMENT

```mermaid
graph TD
    A["POST /session/create<br/>?exercise=squat and height_cm=173"] --> B["Generate session_id<br/>session_YYYYMMDD_HHMMSS_N"]
    B --> C["Instantiate ExerciseSession<br/>PoseCalibrator + PerformanceMetrics<br/>load YOLO11n-pose.pt once"]
    C --> D["active_sessions session_id = session<br/>return session_id to client"]
    D --> E["Client opens WS /ws/session_id"]
    E --> F["session.is_active = True<br/>send connected handshake"]
    F --> G["Frame processing loop<br/>YOLO → angles → state machine → metrics"]
    G --> H{"User sends stop / WS closes"}
    H -->|"WS disconnect"| I["session.is_active = False<br/>session STAYS in active_sessions"]
    I --> J["GET /session/id/metrics<br/>compile final report JSON"]
    J --> K["Save to results/metrics_id.json<br/>return to frontend"]
    K --> L["POST /analyze<br/>Gemini reads metrics file"]
    L --> M["DELETE /session/id<br/>del active_sessions session_id"]

    subgraph SessionState["ExerciseSession Fields (Per-User Isolation)"]
        SS1["calibrator — PoseCalibrator YOLO instance"]
        SS2["metrics — PerformanceMetrics object"]
        SS3["cm_per_pixel — calibration ratio"]
        SS4["counter, stage, feedback"]
        SS5["last_knee_angles — 3-frame smoothing buffer"]
        SS6["calibration_frames — 30 frame list"]
        SS7["thresholds — per-exercise angle dict"]
    end

    C -.->|"contains"| SessionState

    style A fill:#e1f5ff,stroke:#01579b
    style C fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px
    style I fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style M fill:#ffccbc,stroke:#bf360c,stroke-width:2px
    style SessionState fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
```

---

## 6. AI INTEGRATIONS — GEMINI COACH AND GROQ DIET ASSISTANT

```mermaid
graph TB
    A["Session ends<br/>User clicks Stop"] --> B["GET /session/id/metrics<br/>PerformanceMetrics compile report"]
    B --> C["Structured JSON compiled<br/>reps, depths, times, form flags"]
    C --> D["Save: results/metrics_id.json"]

    D --> E["POST /analyze — AI Coach"]
    D --> F["POST /api/diet/chat — Nutrition"]

    subgraph GeminiPath["GEMINI 2.5 FLASH — AI COACH"]
        E --> G["analyze_exercise_metrics<br/>open metrics file"]
        G --> H["ChatGoogleGenerativeAI<br/>temperature=0.3 deterministic"]
        H --> I["SAI Coach Prompt<br/>6-section structured analysis"]
        I --> J["Response in ~2s"]
        J --> K1["Body Condition Assessment<br/>flexibility / power / endurance"]
        J --> K2["Athletic Profile Type<br/>power / endurance / agility / mixed"]
        J --> K3["Top 5 Recommended Sports<br/>with WHY rationale per sport"]
        J --> K4["3 Key Development Areas<br/>specific drills per area"]
        J --> K5["Overall Fitness Score X/100"]
    end

    subgraph GroqPath["GROQ LLaMA 3.3-70B — DIETARY ASSISTANT"]
        F --> L["get_dietary_assistant singleton"]
        L --> M["set_user_profile<br/>name / age / weight / activity / diet / goals"]
        L --> N["set_exercise_history<br/>recent reps / score / streak"]
        M --> O["Build system prompt<br/>user_context + exercise_data injected"]
        N --> O
        O --> P["conversation_history last 10 msgs<br/>sliding context window — 5 turns"]
        P --> Q["ChatGroq llama-3.3-70b-versatile<br/>temperature=0.7 / max_tokens=1024"]
        Q --> R["Response in < 1s<br/>Groq LPU hardware ~400 tok/s"]
        R --> S["Indian dietary preferences aware<br/>science-backed actionable advice"]
    end

    style GeminiPath fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    style GroqPath fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style K5 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style R fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
```

---

## 7. PERFORMANCEMETRICS ENGINE — DATA ACCUMULATION

```mermaid
graph LR
    subgraph Input["PER-FRAME INPUTS"]
        I1["knee_angle smoothed"]
        I2["torso_angle from vertical"]
        I3["shin_angle from vertical"]
        I4["current_time timestamp"]
        I5["keypoints 17x3 array"]
    end

    subgraph Accumulators["PER-SESSION ACCUMULATORS"]
        A1["squat_depths list<br/>knee angle at rep bottom"]
        A2["concentric_times list<br/>time to ascend"]
        A3["eccentric_times list<br/>time to descend"]
        A4["torso_angles deque<br/>rolling 500 frames"]
        A5["shin_angles deque<br/>rolling 500 frames"]
        A6["sticking_points list<br/>angle at min velocity"]
        A7["good_reps / bad_reps int"]
        A8["rep_angles list<br/>max/min per rep"]
    end

    subgraph SpatialMetrics["SPATIAL (calibration required)"]
        SP1["vjump_jump_heights list<br/>ground_y minus min_ankle_y px"]
        SP2["bjump_max_distance px<br/>landing_x minus takeoff_x"]
        SP3["reach_distances list<br/>wrist_x minus ankle_x px"]
        SP4["All x cm_per_pixel to cm"]
    end

    subgraph FinalReport["FINAL REPORT JSON"]
        R1["total_reps / good_reps / bad_reps"]
        R2["avg_depth_angle"]
        R3["avg_concentric_time"]
        R4["avg_eccentric_time"]
        R5["sticking_point_angle"]
        R6["session_id / timestamp"]
    end

    I1 --> A1
    I1 --> A6
    I2 --> A4
    I3 --> A5
    I4 --> A2
    I4 --> A3
    I5 --> SP1
    I5 --> SP2
    I5 --> SP3

    A1 --> R2
    A2 --> R3
    A3 --> R4
    A6 --> R5
    A7 --> R1
    SP1 --> SP4
    SP2 --> SP4
    SP3 --> SP4

    style Input fill:#e1f5ff,stroke:#01579b,stroke-width:2px
    style Accumulators fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px
    style SpatialMetrics fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style FinalReport fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
```

---

## 8. DEPLOYMENT ARCHITECTURE

### HF Spaces Backend + Vercel Frontend Production Topology

```mermaid
graph TD
    subgraph Dev["DEVELOPMENT"]
        D1["Local codebase fitvision-api/"]
        D2["YOLO weights via Git LFS<br/>*.pt filter tracked"]
        D3["Dockerfile<br/>python:3.11-slim base"]
    end

    subgraph HFSpaces["HUGGING FACE SPACES — Backend"]
        HF1["HF detects Dockerfile<br/>SDK: docker in README frontmatter"]
        HF2["Docker build<br/>apt-get: libgl1 libgomp1 ffmpeg"]
        HF3["pip install requirements.txt<br/>ultralytics fastapi uvicorn langchain"]
        HF4["RUN as uid=1000 user<br/>HF Spaces sandbox security policy"]
        HF5["uvicorn fast:app<br/>host 0.0.0.0 port 7860"]
        HF6["results/ directory<br/>metrics JSON files on disk"]
        HF1 --> HF2 --> HF3 --> HF4 --> HF5
        HF5 --> HF6
    end

    subgraph Vercel["VERCEL — Frontend"]
        V1["Next.js 16 build<br/>npm run build"]
        V2["NEXT_PUBLIC_API_URL env var<br/>= https://hf-spaces-url"]
        V3["vercel.json rewrites<br/>WS upgrade proxy config"]
        V4["Edge Network<br/>HTTPS + WSS termination"]
    end

    subgraph ExternalAPIs["EXTERNAL API CALLS — Server-Side Only"]
        E1["Google Gemini 2.5 Flash<br/>GOOGLE_API_KEY env var"]
        E2["Groq LLaMA 3.3-70b<br/>GROQ_API_KEY env var"]
    end

    subgraph Browser["BROWSER — End User"]
        B1["HTTPS to Vercel Edge"]
        B2["WSS to Vercel proxied to HF Spaces"]
        B3["Camera API getUserMedia<br/>320x240 @ 15 FPS"]
    end

    D1 -->|"git push"| HFSpaces
    D2 -->|"LFS objects"| HF2
    D1 -->|"git push"| Vercel
    D3 --> HF1
    V2 --> V3 --> V4
    HF5 <-->|"HTTPS outbound — API keys never in browser"| E1
    HF5 <-->|"HTTPS outbound — API keys never in browser"| E2
    B1 --> V4
    B2 --> V4
    V4 -->|"proxied WSS"| HF5
    B3 --> B2

    style Dev fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    style HFSpaces fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px
    style Vercel fill:#e1f5ff,stroke:#01579b,stroke-width:2px
    style ExternalAPIs fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style Browser fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style HF4 fill:#ffccbc,stroke:#bf360c,stroke-width:2px
```

---

## 9. FULL REST + WEBSOCKET API SURFACE

```mermaid
graph LR
    subgraph Client["CLIENT — Next.js Frontend"]
        C1["Session flow"]
        C2["Live exercise"]
        C3["AI features"]
    end

    subgraph REST["REST ENDPOINTS"]
        R1["GET / — API info"]
        R2["GET /health — active session count"]
        R3["GET /exercises — list 8 exercises"]
        R4["POST /session/create — ?exercise= and height_cm="]
        R5["GET /session/id — session info"]
        R6["GET /session/id/metrics — final report JSON"]
        R7["DELETE /session/id — cleanup"]
        R8["POST /analyze — Gemini AI coach"]
        R9["POST /api/diet/chat — Groq nutrition chat"]
        R10["GET POST /api/diet/profile — user profile"]
        R11["POST /api/diet/history — sync exercise data"]
        R12["GET /api/diet/onboarding — setup questions"]
        R13["DELETE /api/diet/clear — reset chat"]
    end

    subgraph WS["WEBSOCKET ENDPOINTS"]
        W1["WS /ws/session_id<br/>Primary: client sends binary JPEG frames"]
        W2["WS /ws/webcam/session_id<br/>Server webcam for local testing"]
    end

    subgraph Responses["KEY RESPONSE PAYLOADS"]
        RS1["session_id string"]
        RS2["metrics JSON per exercise"]
        RS3["AI report string from Gemini"]
        RS4["nutrition advice from Groq < 1s"]
        RS5["JSON with base64 annotated frame<br/>+ counter, stage, feedback, cm_per_pixel"]
    end

    C1 --> R4 --> RS1
    C1 --> R6 --> RS2
    C3 --> R8 --> RS3
    C3 --> R9 --> RS4
    C2 --> W1 --> RS5

    style REST fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px
    style WS fill:#e1f5ff,stroke:#01579b,stroke-width:2px
    style Responses fill:#fff3e0,stroke:#e65100,stroke-width:2px
```

---

## Diagram Index

| # | Diagram | What It Shows |
|---|---|---|
| 1 | **High-Level System Architecture** | 3-layer ecosystem: Frontend to Backend CV pipeline to AI services + Infra |
| 2 | **WebSocket Streaming Pipeline** | Frame-by-frame loop with flow control, encoding, keepalive |
| 3 | **Pixel-to-cm Calibration Algorithm** | 30-frame averaging, quality gates, ratio application |
| 4 | **8 Exercise State Machines** | Angle thresholds, state transitions, rep counting logic per exercise |
| 5 | **Session Lifecycle** | Create → Stream → Disconnect → Metrics → Analyze → Delete |
| 6 | **AI Integrations** | Gemini coach pipeline + Groq dietary assistant with sliding context window |
| 7 | **PerformanceMetrics Engine** | Per-frame inputs → per-session accumulators → final report JSON |
| 8 | **Deployment Architecture** | HF Spaces Docker build + Vercel + Git LFS + external API calls |
| 9 | **API Surface** | All REST + WebSocket endpoints with response types |

---

> **Mermaid syntax rule followed:** `<br/>` is only used inside node labels `["text<br/>line2"]` — never inside edge link text `|text|`.
> Paste any diagram block into [Mermaid Live Editor](https://mermaid.live/) to visualize interactively.
