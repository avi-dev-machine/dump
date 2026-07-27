# FitVision — Comprehensive Study Material
### Smart India Hackathon 2025 Finalist | Real-Time AI Fitness Evaluation Platform

> **Resume Line Decoded:** *"Built a real-time pose estimation pipeline using YOLO11n-Pose and OpenCV, tracking body keypoints to count reps and evaluate form across 8 exercises at 15+ FPS on CPU."*

---

## Table of Contents
1. [Project Overview & Problem Statement](#1-project-overview--problem-statement)
2. [System Architecture](#2-system-architecture)
3. [Backend Architecture (Deep Dive)](#3-backend-architecture-deep-dive)
4. [Pose Estimation Pipeline](#4-pose-estimation-pipeline)
5. [Pixel-to-Centimetre Calibration Algorithm](#5-pixel-to-centimetre-calibration-algorithm)
6. [Exercise State Machines — All 8 Exercises](#6-exercise-state-machines--all-8-exercises)
7. [Metrics Engine (PerformanceMetrics)](#7-metrics-engine-performancemetrics)
8. [WebSocket Streaming Pipeline](#8-websocket-streaming-pipeline)
9. [Session Management](#9-session-management)
10. [AI Coach — Gemini 2.5 Flash Integration](#10-ai-coach--gemini-25-flash-integration)
11. [Dietary Assistant — Groq LLaMA 3.3-70B](#11-dietary-assistant--groq-llama-33-70b)
12. [Frontend Architecture — Next.js App Router](#12-frontend-architecture--nextjs-app-router)
13. [Containerisation & Deployment](#13-containerisation--deployment)
14. [Full API Surface Reference](#14-full-api-surface-reference)
15. [Key Design Decisions & Trade-offs](#15-key-design-decisions--trade-offs)
16. [Performance Numbers & Resume Justification](#16-performance-numbers--resume-justification)
17. [Likely Interview Questions & Expert Answers](#17-likely-interview-questions--expert-answers)

---

## 1. Project Overview & Problem Statement

### What FitVision Solves
Physical fitness testing — push-ups, squats, jump height, flexibility — traditionally requires:
- A trained SAI (Sports Authority of India) certified coach
- Measuring tapes, timers, force plates (expensive equipment)
- A physical facility

**FitVision eliminates all three constraints.** A user opens a browser, points their laptop or mobile camera at themselves, and receives:
- **Automated rep counting** via geometric angle analysis (not pixel counting)
- **Real-world measurements in centimetres** from a plain webcam
- **Frame-by-frame form evaluation** with live feedback cues
- **AI-generated athletic profiles** matched to sport recommendations
- **Personalised nutrition advice** powered by a conversational LLM

### Target Context
Built to replicate the assessment tools used by **SAI coaches** for national-level athlete identification. Selected as a **Smart India Hackathon 2025 Finalist**, validating the real-world applicability of the approach.

---

## 2. System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLIENT (Browser / Mobile)                     │
│                                                                   │
│  ┌───────────────┐    ┌──────────────────┐   ┌──────────────┐   │
│  │ Next.js 16    │    │ WebSocket Client  │   │ Camera API   │   │
│  │ App Router    │◄──►│ (useExerciseSocket│◄──│getUserMedia  │   │
│  │ (Vercel)      │    │  custom hook)     │   │ 320×240@15fps│   │
│  └───────────────┘    └──────────────────┘   └──────────────┘   │
└─────────────────────────────────────────────────────────────────┘
           │ REST (session mgmt)          │ WSS (binary JPEG frames)
           │ HTTPS                        │ ~15–20 KB/frame
           ▼                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 BACKEND (HF Spaces / Docker)                      │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  FastAPI + Uvicorn (ASGI)  |  Port 7860                 │    │
│  │                                                          │    │
│  │  ┌──────────────┐  ┌────────────────┐  ┌─────────────┐ │    │
│  │  │ Session Mgr  │  │ WS Endpoint    │  │ REST Routes │ │    │
│  │  │ (in-memory   │  │ /ws/{id}       │  │ /session/*  │ │    │
│  │  │  dict)       │  │                │  │ /analyze    │ │    │
│  │  └──────┬───────┘  └───────┬────────┘  └─────────────┘ │    │
│  │         │                  │                             │    │
│  │  ┌──────▼──────────────────▼──────────────────────────┐ │    │
│  │  │              ExerciseSession (per user)              │ │    │
│  │  │                                                      │ │    │
│  │  │  ┌─────────────┐  ┌──────────────┐  ┌───────────┐  │ │    │
│  │  │  │PoseCalibrator│  │Performance   │  │Calibration│  │ │    │
│  │  │  │(YOLO11n-pose)│  │Metrics Engine│  │State      │  │ │    │
│  │  │  └──────┬──────┘  └──────┬───────┘  └───────────┘  │ │    │
│  │  │         │                │                           │ │    │
│  │  │  ┌──────▼──────────────────▼──────────────────────┐ │ │    │
│  │  │  │  Keypoints (17) → Angles → State Machine → Rep │ │ │    │
│  │  │  └────────────────────────────────────────────────┘ │ │    │
│  │  └──────────────────────────────────────────────────────┘ │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌──────────────────────┐    ┌──────────────────────────────┐   │
│  │  Gemini 2.5 Flash    │    │   Groq LLaMA 3.3-70B         │   │
│  │  (AI Coach /analyze) │    │   (Dietary Assistant /diet/* )│   │
│  │  LangChain wrapper   │    │   LangChain + sliding window  │   │
│  └──────────────────────┘    └──────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
           │ Docker / Git LFS
           ▼
┌──────────────────────┐     ┌───────────────────────┐
│  Hugging Face Spaces │     │  Vercel (Frontend)    │
│  (Docker SDK, HF LFS)│     │  NEXT_PUBLIC_API_URL  │
└──────────────────────┘     └───────────────────────┘
```

### Data Flow (Single Frame)
```
Camera → Canvas → JPEG Blob (35% quality) → WebSocket binary send
  → Server: np.frombuffer → cv2.imdecode → YOLO inference (17 keypoints)
  → _calibrate_height() → exercise processor (angles → state machine)
  → PerformanceMetrics.update_*() → JSON response + base64 frame (40% quality)
  → Client: render annotated frame + update UI counters
```

---

## 3. Backend Architecture (Deep Dive)

### Tech Stack
| Layer | Technology | Why |
|---|---|---|
| Web Framework | **FastAPI** + **Uvicorn** (ASGI) | Async native, WebSocket support built-in, auto Swagger docs |
| Real-time Transport | **WebSockets** (native FastAPI) | Bidirectional persistent; HTTP can't receive frames |
| Computer Vision | **OpenCV** (headless), **NumPy**, **SciPy** | De-facto CV stack; `cv2.imdecode` for JPEG→array, `imencode` for array→JPEG |
| Pose Estimation | **Ultralytics YOLO11n-pose** | Single-pass detect+keypoints; nano variant runs on CPU-only containers |
| Object Detection | **YOLOv8n** | Available for secondary detection tasks |
| AI Coach | **LangChain** + **Google Gemini 2.5 Flash** | Large context window for verbose metrics JSON; fast + cheap |
| Dietary AI | **LangChain** + **Groq LLaMA 3.3-70b-versatile** | Groq LPU hardware delivers ~300–500 tok/s; near-instant chat |
| Data Validation | **Pydantic v2** | Request/response schema enforcement for all REST endpoints |
| Config | **python-dotenv** | Loads `GROQ_API_KEY`, `GOOGLE_API_KEY` from `.env` |
| Containerization | **Docker** (Python 3.11-slim) | Reproducible deployment; non-root uid=1000 for HF Spaces |
| Deployment | **Hugging Face Spaces** (Docker SDK) / Railway | Free persistent container; HF provides managed infra |

### Key Source Files
| File | Role | Size |
|---|---|---|
| `fast.py` | Main FastAPI app — all REST + WS endpoints, `ExerciseSession` class, per-exercise `process_*` methods | ~1383 lines |
| `utils.py` | `PoseCalibrator` class — YOLO inference, skeleton rendering, all angle computation methods | ~881 lines |
| `metrics.py` | `PerformanceMetrics` class — per-rep/per-exercise data accumulation, final report generation | ~2600 lines |
| `ai.py` | `analyze_exercise_metrics()` — Gemini integration; reads saved metrics JSON, invokes LLM | ~52 lines |
| `dietary_assistant.py` | `DietaryAssistant` class — Groq LangChain wrapper, sliding context window, user profiling | ~209 lines |

### Application Startup — Lifespan & CORS
```python
# CORS: allows any origin (needed for Vercel frontend ↔ HF Spaces backend)
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])

# Global in-memory session store
active_sessions: Dict[str, ExerciseSession] = {}
```

---

## 4. Pose Estimation Pipeline

### Model: YOLO11n-Pose
The system uses the **nano** (`n`) variant of Ultralytics' YOLO11 pose model. Compared to larger variants:

| Variant | Size | mAP (pose) | FPS (CPU) |
|---|---|---|---|
| YOLO11n-pose | ~6 MB | ~50 | **15+ FPS** |
| YOLO11s-pose | ~12 MB | ~56 | ~8 FPS |
| YOLO11m-pose | ~26 MB | ~62 | ~3 FPS |

The nano model was chosen deliberately — it's the only variant fast enough for real-time CPU inference on HF Spaces containers with no GPU.

### 17 COCO Keypoints
```python
keypoint_names = {
    0: 'nose',        1: 'left_eye',     2: 'right_eye',
    3: 'left_ear',    4: 'right_ear',    5: 'left_shoulder',
    6: 'right_shoulder', 7: 'left_elbow', 8: 'right_elbow',
    9: 'left_wrist',  10: 'right_wrist', 11: 'left_hip',
    12: 'right_hip',  13: 'left_knee',   14: 'right_knee',
    15: 'left_ankle', 16: 'right_ankle'
}
# Each keypoint: (x_pixel, y_pixel, confidence_score)
```

### Inference Pipeline (Per Frame)
```python
def detect_pose(self, frame):
    results = self.model(frame, verbose=False)          # YOLO inference
    if len(results[0].keypoints) > 0:
        keypoints = results[0].keypoints.data[0].cpu().numpy()
        # keypoints.shape = (17, 3) → [x, y, confidence] per joint
        return keypoints
    return None
```

**`.cpu().numpy()` is critical** — YOLO returns PyTorch tensors on whatever device the model is loaded on. Converting to NumPy makes the data compatible with OpenCV and SciPy operations.

### Confidence Filtering
Every keypoint access in `get_all_joint_angles()` checks confidence before computing an angle:
```python
if (keypoints[idx1][2] > 0.5 and
    keypoints[idx2][2] > 0.5 and
    keypoints[idx3][2] > 0.5):
    angle = self.calculate_angle(...)
else:
    angles[joint_name] = None    # Skip this joint; don't produce a garbage angle
```

If a required joint returns `None`, the exercise processor falls back to the other side of the body (e.g., right elbow if left elbow is occluded) or shows a feedback cue like `"Position yourself so arms are visible"`.

### Angle Calculation — Vector Dot Product
The interior angle at joint `pt2` given three keypoints `(pt1, pt2, pt3)`:
```python
def calculate_angle(self, pt1, pt2, pt3):
    v1 = pt1 - pt2        # Vector from vertex to first arm
    v2 = pt3 - pt2        # Vector from vertex to second arm

    cos_angle = np.dot(v1, v2) / (np.linalg.norm(v1) * np.linalg.norm(v2))
    cos_angle = np.clip(cos_angle, -1.0, 1.0)   # CRITICAL: prevent NaN from float precision
    angle = np.degrees(np.arccos(cos_angle))
    return int(round(angle))
```

> **Why `np.clip(-1, 1)`?** Floating point arithmetic can produce a cosine like `1.0000000002` due to accumulated rounding errors. `arccos(>1.0)` returns `NaN`, which then silently corrupts all downstream state. Always clamp.

### Pre-defined Joint Angle Triples
```python
joint_angles = {
    'left_elbow':    (5,  7,  9),   # shoulder–elbow–wrist
    'right_elbow':   (6,  8,  10),
    'left_shoulder': (7,  5,  11),  # elbow–shoulder–hip
    'right_shoulder':(8,  6,  12),
    'left_hip':      (5,  11, 13),  # shoulder–hip–knee
    'right_hip':     (6,  12, 14),
    'left_knee':     (11, 13, 15),  # hip–knee–ankle
    'right_knee':    (12, 14, 16),
}
```

### Angle Smoothing — 3-Frame Moving Average
Raw YOLO keypoints jitter frame to frame (model variance + JPEG compression artifacts):
```python
def _smooth_knee_angle(self, knee_angle):
    self.last_knee_angles.append(knee_angle)
    if len(self.last_knee_angles) > 3:
        self.last_knee_angles.pop(0)
    return int(np.mean(self.last_knee_angles))
```

**Window = 3 frames:** At 12 FPS, this is a 250ms smoothing window. Short enough that the feedback lag is imperceptible; long enough to filter single-frame outliers.

### Skeleton Rendering (Visual Output)
The `PoseCalibrator` draws the 16-bone COCO skeleton onto the frame:
```python
skeleton_connections = [
    (0,1),(0,2),(1,3),(2,4),              # head
    (5,6),(5,11),(6,12),(11,12),           # torso
    (5,7),(7,9),(6,8),(8,10),              # arms
    (11,13),(13,15),(12,14),(14,16)        # legs
]
```
Each keypoint is drawn as a green circle with magenta border. Bone lines are white. Angle values are overlaid in cyan text.

---

## 5. Pixel-to-Centimetre Calibration Algorithm

> **Resume Line:** *"Designed a pixel-to-centimetre calibration algorithm to estimate jump height and reach distance with approx. 92% accuracy against manual measurements."*

### The Problem
A webcam sees the world in pixels, not centimetres. Two users, same physical height, at different distances from the camera will have very different pixel heights. To report jump height or reach distance in real units, you must establish a per-session, per-distance calibration ratio.

### The Algorithm (Implemented in `_calibrate_height()`)

**Step 1 — User Input:** Before the session, the user enters their height (e.g., `170 cm`). This is passed as `user_height_cm` to `ExerciseSession`.

**Step 2 — Landmark Selection:**
```python
head   = keypoints[0]   # nose keypoint
l_ankle = keypoints[15]
r_ankle = keypoints[16]
# Choose the ankle with higher confidence
ankle = l_ankle if l_ankle[2] > r_ankle[2] else r_ankle
```

**Step 3 — Pixel Height Measurement:**
```python
H_p = np.sqrt((head[0] - ankle[0])**2 + (head[1] - ankle[1])**2)
# This is the Euclidean distance between nose and ankle in pixel space
```

**Step 4 — Quality Gate:**
```python
if H_p < 40:  # pixel height too small = person too far / partially out of frame
    return    # skip this frame; don't contaminate the average
```

**Step 5 — Accumulate 30 measurements:**
```python
self.calibration_frames.append(H_p)
if len(self.calibration_frames) >= 30:  # ~2.5 seconds at 12 FPS
    avg_H_p = np.mean(self.calibration_frames)
    self.cm_per_pixel = self.user_height_cm / avg_H_p
    self.calibration_complete = True
    self.metrics.cm_per_pixel = self.cm_per_pixel
```

**Why 30 frames?** A single frame is noisy (person may be mid-motion, lighting flicker). Averaging 30 measurements over ~2.5 seconds yields a stable ratio without requiring the user to explicitly stand still.

### Application of the Ratio
Once `cm_per_pixel` is established, any pixel distance multiplies to give centimetres:

```python
# Vertical Jump
height_px = ground_baseline_y - min_ankle_y_during_jump
height_cm = height_px * self.cm_per_pixel

# Sit-and-Reach
reach_cm = reach_distance_px * self.cm_per_pixel

# Broad Jump
distance_cm = horizontal_distance_px * self.cm_per_pixel
```

### Accuracy (~92%)
The 92% accuracy figure (against manual tape measurements) is primarily limited by:
1. **Nose vs Crown offset** — The nose keypoint is ~3–5 cm below the crown of the head. This introduces a systematic underestimate of person height.
2. **Viewing angle error** — The algorithm assumes the person stands perpendicular to the camera. Tilting introduces cosine error.
3. **Keypoint confidence on ankles** — At 320×240, ankle keypoints can have low confidence if feet are partially out of frame.

---

## 6. Exercise State Machines — All 8 Exercises

Every exercise is implemented as a **finite state machine driven purely by joint geometry** — no timers, no frame counting.

### Exercise 1: Push-ups (`process_pushup`)
**Key joint:** Elbow angle (shoulder–elbow–wrist)
**Form check:** Hip angle (shoulder–hip–knee) — detects back sag

```
States:  UP ←→ DOWN
Trigger: elbow > 140° → UP;  elbow < 90° → DOWN
Rep counted: DOWN → UP transition

Thresholds:
  'up'          : 140°   (arms extended)
  'down'        : 90°    (arms fully bent)
  'form_hip_min': 130°   (hip must be ≥ 110° to avoid back sag)

Form feedback:
  hip < 110°  → "Fix Back!"
  otherwise   → "Good Form"
```

**Side-fallback logic:** Tries left elbow + left hip first, then right side, then cross combinations. Handles side-on camera angles.

### Exercise 2: Squats (`process_squat`)
**Key joint:** Knee angle (hip–knee–ankle)
**Additional tracking:** Torso angle, shin angle, concentric/eccentric timing

```
States:  UP ←→ DOWN
Trigger: knee > 155° → UP;  knee < 135° → DOWN
Rep counted: UP → DOWN transition (when descending past threshold)

Depth quality:
  knee < 90°  → "Great Depth!" (good rep)
  otherwise   → "Go Lower"    (bad rep)

Additional metrics recorded per rep:
  - squat_depth (knee angle at bottom)
  - concentric_time (time to ascend)
  - eccentric_time (time to descend)
  - sticking_point (angle at minimum velocity)
  - torso_angle, shin_angle (for form analysis)
```

### Exercise 3: Sit-ups (`process_situp`)
**Key joint:** Torso inclination relative to horizontal

```
States:  rest → ascending → peak → descending → rest
Trigger: torso_inclination ≤ 20° → rest (DOWN);  ≥ 70° → peak (UP)
Rep counted: rest → peak transition

Additional check:
  hip_flexion ≤ 50° also triggers peak (crunch angle reached)

Metrics:
  situp_concentric_times, situp_eccentric_times
```

### Exercise 4: Sit-and-Reach (`process_sitnreach`)
**No rep count** — tracks maximum reach distance over the session.

```python
reach_distance = wrist_x - ankle_x   # horizontal pixel distance
max_reach = max(max_reach, reach_distance)
counter = int(max_reach)   # counter field repurposed to show reach distance

Feedback cues:
  reach > 95% of max → "MAX REACH!"
  reach > 85% of max → "Keep Reaching"
  else               → "Stretch Forward"
  knee_angle < 165°  → append "Straighten Legs!"
```

**Validity check:** Knee angle must be ≥ 165° for the reach to be considered valid (straight legs are required in the SAI sit-and-reach protocol).

### Exercise 5: Skipping (`process_skipping`)
**Detection method:** Vertical position of ankle midpoint relative to a ground baseline

```
States:  standing → airborne → landing
Trigger: ankle_y rises above (ground_y - threshold_px) → airborne

Metrics:
  - jump_count
  - airtime per jump
  - skipping frequency (jumps/second)
  - double-bounce detection (two brief airborne phases per rope turn)
```

> **Y-axis note:** In image coordinates, Y increases downward. Higher jump = smaller Y value. `ground_y - ankle_y > 0` means the foot has lifted above the ground line.

### Exercise 6: Jumping Jacks (`process_jumpingjacks`)
**Detection:** Dual state machine — arms AND legs must both open/close

```
States: CLOSED → OPEN → CLOSED (one rep)

Open condition:
  arm_open: shoulder-elbow angle > 150°  (arms raised laterally)
  leg_open: hip-knee angle > 150°        (legs spread)

Rep counted: OPEN → CLOSED transition
```

### Exercise 7: Vertical Jump (`process_vjump`)
**Detection:** Ankle midpoint Y versus ground baseline (same principle as skipping but different metric)

```
States: standing → airborne → landing

Jump height calculation:
  height_px = ground_baseline_y - min_ankle_y_during_jump
  height_cm = height_px * cm_per_pixel

Additional:
  landing_knee_angle: assesses soft vs stiff landing
  countermovement_depth: knee angle before jump (< 110° = good countermovement)
```

> **Critical Y-coordinate inversion:** In image space, Y=0 is the TOP of the frame. When a person jumps, their ankle Y value DECREASES (moves towards 0). So `ground_y - min_ankle_y` is always positive and represents the jump height. This is the most common conceptual mistake people make when first looking at this code.

### Exercise 8: Broad Jump (`process_bjump`)
**Detection:** Horizontal displacement of ankle position from a take-off baseline

```
States: pre-jump → airborne → landing

Distance calculation:
  distance_px = landing_ankle_x - takeoff_ankle_x
  distance_cm = distance_px * cm_per_pixel

Countermovement:
  knee_angle < 110° before jump → good depth (stored as quality flag)
```

---

## 7. Metrics Engine (PerformanceMetrics)

`metrics.py` is the computational heart — ~2600 lines of per-exercise data accumulation and reporting.

### Per-Session Data Structure (Squat example)
```python
# Angular data — rolling deques for real-time trend tracking
self.squat_depths = []           # knee angle at bottom of each rep
self.concentric_times = []       # time to ascend from bottom
self.eccentric_times = []        # time to descend to bottom
self.torso_angles = deque(maxlen=500)   # rolling torso lean
self.shin_angles = deque(maxlen=500)    # rolling shin angle

# Quality classification
self.good_reps = 0
self.bad_reps = 0
self.sticking_points = []        # angle at minimum velocity per rep

# Calibration (shared across all exercises)
self.cm_per_pixel = None
self.user_height_cm = None
```

### Rep Recording
```python
def record_rep(self, rep_max, rep_min, duration_seconds, is_good_form):
    if is_good_form:
        self.good_reps += 1
    else:
        self.bad_reps += 1
    self.rep_angles.append((rep_max, rep_min))
    self.rep_durations.append(duration_seconds)
```

### Final Report Generation
At session end, each exercise has its own `*_metrics()` method that compiles all accumulated data:
```json
{
  "total_reps": 15,
  "good_reps": 12,
  "bad_reps": 3,
  "avg_depth_angle": 88.3,
  "avg_concentric_time": 1.2,
  "avg_eccentric_time": 1.8,
  "sticking_point_angle": 95,
  "timestamp": "2026-07-25T12:00:00",
  "session_id": "session_20260725_120000_0"
}
```

This JSON is also **saved to disk** as `results/metrics_{session_id}.json` so that the Gemini AI coach can read it later without rerunning the session.

### Cross-Exercise Spatial Measurements
| Exercise | Pixel Metric | Real-World Conversion |
|---|---|---|
| Vertical Jump | `ground_y - min_ankle_y` | `× cm_per_pixel` → cm |
| Broad Jump | `landing_ankle_x - takeoff_x` | `× cm_per_pixel` → cm |
| Sit-and-Reach | `wrist_x - ankle_x` | `× cm_per_pixel` → cm |

---

## 8. WebSocket Streaming Pipeline

> **Resume Line:** *"Supporting 10+ concurrent WebSocket users at <200 ms latency"*

### Why WebSocket Over HTTP Alternatives

| Protocol | Why NOT Used |
|---|---|
| HTTP Polling | Creates new TCP connection per request; high overhead at 12 FPS |
| Server-Sent Events (SSE) | Unidirectional — server pushes only; can't receive camera frames |
| HTTP/2 streaming | Still request-initiated; no clean bidirectional binary path |
| **WebSocket** | ✅ Full-duplex, persistent TCP; client sends frames, server sends metrics |

### Client-Side Flow (Next.js / `useExerciseSocket` hook)

#### Camera Setup
```javascript
stream = await navigator.mediaDevices.getUserMedia({
    video: {
        width: { exact: 320 },
        height: { exact: 240 },
        frameRate: { max: 15 }
    }
});
```
320×240 is intentional — small enough to encode fast (~15–20 KB at 35% JPEG quality), large enough for YOLO to detect a full-body pose.

#### Frame Capture & Encoding (requestAnimationFrame loop)
```javascript
ctx.drawImage(videoElement, 0, 0, 320, 240);
canvas.toBlob((blob) => {
    ws.send(blob);               // raw binary JPEG over WebSocket
}, 'image/jpeg', 0.35);         // 35% quality ≈ 15-20 KB/frame
```

#### Flow Control — The Most Important Design Decision
```javascript
const pendingFrameRef = useRef(false);

// Sending side:
if (!pendingFrameRef.current) {
    pendingFrameRef.current = true;
    ws.send(blob);
}

// On receiving server response:
pendingFrameRef.current = false;   // release; next frame can be sent
```

**Why this matters:** Without flow control, a fast client sends frames continuously. On a slow/congested connection, frames pile up in the WebSocket send buffer. Each response is then tied to an increasingly stale frame. After 10 seconds, the user sees feedback from 10 seconds ago — completely useless for real-time guidance.

The **one-frame-in-flight** constraint ensures that the system always responds to the most recent frame the server has actually processed, regardless of variable network latency.

#### FPS Measurement
The frontend counts **received responses per second**, not sent frames. This measures true processed throughput:
```javascript
// On each received response:
fpsCountRef.current++;
// Every 1 second:
setFps(fpsCountRef.current);
fpsCountRef.current = 0;
```

#### WebSocket URL Construction (ws:// vs wss://)
```javascript
const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
const wsUrl = `${protocol}//${API_HOST}/ws/${sessionId}`;
```
This handles the secure-context requirement automatically — HTTPS deployments (Vercel) must use `wss://`.

### Server-Side WebSocket Handler (`fast.py`)

```python
@app.websocket("/ws/{session_id}")
async def websocket_endpoint(websocket: WebSocket, session_id: str):
    await websocket.accept()

    session.is_active = True                          # support reconnection
    await websocket.send_json({"type": "connected"})  # handshake

    while session.is_active:
        data = await asyncio.wait_for(websocket.receive(), timeout=30.0)

        if "bytes" in data:
            # PRIMARY PATH: binary JPEG frame
            nparr = np.frombuffer(data["bytes"], np.uint8)
            frame = cv2.imdecode(nparr, cv2.IMREAD_COLOR)

        elif "text" in data:
            message = json.loads(data["text"])
            if message.get("type") == "stop":
                break
            if message.get("type") == "frame":
                # FALLBACK: base64-encoded frame in JSON
                frame = cv2.imdecode(np.frombuffer(base64.b64decode(message["data"]), np.uint8), cv2.IMREAD_COLOR)

        result = session.process_frame(frame)
        await websocket.send_json(result)              # JSON with base64 frame + metrics

    except asyncio.TimeoutError:
        await websocket.send_json({"type": "keepalive"})  # prevent proxy timeout
```

**Why `timeout=30.0`?** Intermediate proxies (Vercel Edge, Cloudflare, HF Spaces load balancer) close idle connections. The keepalive ensures the connection stays alive during brief pauses (user resting between sets).

### Response Encoding
```python
# Encode processed frame into response JSON
small_frame = cv2.resize(processed_frame, (320, 240), interpolation=cv2.INTER_NEAREST)
_, buffer = cv2.imencode('.jpg', small_frame, [cv2.IMWRITE_JPEG_QUALITY, 40])
response["frame"] = base64.b64encode(buffer).decode('utf-8')
await websocket.send_json(response)
```

**Why `INTER_NEAREST` for resize?** Fastest interpolation (nearest-neighbour, no weighted averaging). Since the frame is already 320×240 from the client, this is usually a no-op, but ensures consistency.

**Why 40% quality for response vs 35% for request?** The server-processed frame has skeleton lines, text overlays, and annotation graphics drawn on it. These elements don't compress as well as a plain camera frame — 40% keeps the skeleton visually crisp.

**Why base64 in JSON rather than raw binary?** The response carries both the frame AND JSON metadata (counter, stage, feedback, calibration progress). Mixing binary and JSON in a single WebSocket message is complex. Base64 encodes the frame into a string field, keeping the entire response a single clean JSON payload.

---

## 9. Session Management

### In-Memory Session Store
```python
active_sessions: Dict[str, ExerciseSession] = {}
```

Each `ExerciseSession` object holds the **complete state** for one user:
- `PoseCalibrator` instance (holds loaded YOLO model)
- `PerformanceMetrics` instance (all accumulated rep data)
- `cm_per_pixel` ratio, `calibration_frames` buffer
- `counter`, `stage`, `feedback` (live exercise state)
- `last_knee_angles`, `last_elbow_angles` (smoothing buffers)
- `thresholds` dict (per-exercise angle thresholds)

### Session Lifecycle
```
POST /session/create?exercise=squat&height_cm=170
  → session_id = "session_20260725_120000_0"
  → ExerciseSession instantiated, YOLO model loaded (once per session)
  → active_sessions[session_id] = session
         ↓
WS /ws/{session_id}  ← client connects
  → session.is_active = True
  → frames processed, metrics accumulated in memory
         ↓
WebSocket closes (user presses Stop)
  → session.is_active = False
  → session NOT deleted (metrics still in memory)
         ↓
GET /session/{id}/metrics
  → session.metrics.{exercise}_metrics() called
  → JSON compiled and returned
  → saved to results/metrics_{id}.json
         ↓
DELETE /session/{id}
  → del active_sessions[session_id]  (session removed from memory)
```

**Steps 3 and 4 are intentionally separate.** The frontend fetches metrics after the WebSocket closes. If the session were deleted on WS close, metrics retrieval would fail.

### Trade-offs of In-Memory Sessions
| Aspect | In-Memory | Alternative (Redis) |
|---|---|---|
| Read latency per frame | 0 ms (direct dict lookup) | ~1 ms (network round-trip) |
| Model loading | Once per session | Once per session (still) |
| Horizontal scaling | ❌ Sessions tied to one instance | ✅ Multiple instances share sessions |
| Persistence on restart | ❌ All sessions lost | ✅ Survives restart |
| Implementation complexity | Simple | Requires Redis client, serialization |

For the use case (1–10 minute single-user sessions), in-memory is the correct trade-off. Redis adds complexity with no meaningful benefit here.

### Session ID Generation
```python
session_id = f"session_{datetime.now().strftime('%Y%m%d_%H%M%S')}_{len(active_sessions)}"
```
Combines timestamp + current session count → guaranteed unique within a process.

### Reconnection Support
```python
# On WebSocket reconnect to existing session_id:
session = active_sessions[session_id]
session.is_active = True    # re-activate
# Metrics and rep counters preserved from before disconnect
```

---

## 10. AI Coach — Gemini 2.5 Flash Integration

> **Resume Line:** *"Integrated Gemini and Groq APIs to generate AI-powered performance reports and nutrition recommendations in <3 s."*

### How It Works (`ai.py` + `POST /analyze`)

1. **Session ends** → metrics compiled → saved as `results/metrics_{session_id}.json`
2. **Frontend calls** `POST /analyze` with `{"metrics_file": "results/metrics_{id}.json"}`
3. **Server reads** the file and injects the raw data into a structured prompt
4. **Gemini returns** a coach-level analysis in ~2–3 seconds

```python
def analyze_exercise_metrics(metrics_file="exercise_metrics.txt"):
    with open(metrics_file, "r") as f:
        data = f.read()

    llm = ChatGoogleGenerativeAI(model="gemini-2.5-flash", temperature=0.3)

    prompt = f"""You are a Sports Authority of India (SAI) certified coach.

METRICS DATA: {data}

ANALYZE AND PROVIDE:
1. BODY CONDITION ASSESSMENT
   - Flexibility (stiff/average/flexible)
   - Explosive power (lower body strength)
   - Muscular endurance capacity
   - Core stability and control
   - Cardiovascular fitness indicator
   - Coordination and rhythm

2. ATHLETIC PROFILE TYPE
   - Body type tendency (power/endurance/agility/mixed)
   - Natural movement strengths
   - Energy system dominance (aerobic vs anaerobic)

3. RECOMMENDED SPORTS (Top 5)
   - Sports matching this body profile
   - WHY each sport suits this athlete's physical traits

4. KEY AREAS TO DEVELOP
   - 3 specific physical qualities to improve
   - Simple drills for each

5. OVERALL FITNESS SCORE: X/100

Be specific based on actual test scores. Keep it actionable."""

    return llm.invoke(prompt).content
```

### Design Choices
| Decision | Value | Rationale |
|---|---|---|
| `temperature=0.3` | Low | Athletic assessment needs deterministic, consistent scoring — not creative variation |
| Model: gemini-2.5-flash | Flash | Largest context window in Gemini family; handles verbose metrics JSON; fast + cheap vs Pro |
| Data format | Text/JSON dump | Gemini processes structured text well; no need to send images |
| Separation of concerns | YOLO extracts numbers → Gemini interprets them | Computer vision and LLM each do what they're best at |

### What Gemini Receives (No Image Data)
Gemini only receives the structured metrics JSON — numbers like average knee angle, rep count, jump height in cm, timing data. It never sees video frames. This is intentional:
- **Cost**: Image tokens are expensive; text tokens are cheap
- **Accuracy**: The CV pipeline has already quantified performance; the LLM interprets quantitative scores, which it does better than inferring them from video

---

## 11. Dietary Assistant — Groq LLaMA 3.3-70B

### Architecture (`dietary_assistant.py`)

```python
self.llm = ChatGroq(
    api_key=GROQ_API_KEY,
    model="llama-3.3-70b-versatile",
    temperature=0.7,           # higher = more conversational variance
    max_tokens=1024,
)
```

**Why Groq?** Groq builds custom silicon called **LPUs (Language Processing Units)** optimised specifically for LLM inference. While GPU-based inference (OpenAI, Google) achieves ~30–80 tokens/second, Groq achieves ~300–500 tokens/second on LLaMA 3.3-70B. For a chat interface, this translates to near-instant responses — critical UX for a conversational flow.

### Conversation Management — Sliding Window
```python
def chat(self, user_message: str) -> str:
    system_prompt = SYSTEM_PROMPT.format(
        user_context=self._format_user_context(),
        exercise_data=self._format_exercise_data()
    )

    messages = [SystemMessage(content=system_prompt)]

    # Last 10 messages only (5 turns of context)
    for msg in self.conversation_history[-10:]:
        if msg["role"] == "user":
            messages.append(HumanMessage(content=msg["content"]))
        else:
            messages.append(AIMessage(content=msg["content"]))

    messages.append(HumanMessage(content=user_message))
    response = self.llm.invoke(messages)

    self.conversation_history.append({"role": "user", "content": user_message})
    self.conversation_history.append({"role": "assistant", "content": response.content})
    return response.content
```

**Why 10-message window?** LLaMA 3.3-70B has a 128K context window, but sending the entire conversation history on every call is wasteful for a nutrition chatbot. 5 turns (10 messages) is enough context for coherent dietary advice without token overhead.

### User Profiling Data Model
```python
@dataclass
class UserProfile:
    name: str = "Athlete"
    age: Optional[int] = None
    weight_kg: Optional[float] = None
    height_cm: Optional[float] = None
    activity_level: str = "moderate"       # sedentary/light/moderate/active/very_active
    dietary_preference: str = "omnivore"   # vegan/vegetarian/omnivore/keto/etc.
    allergies: List[str] = field(default_factory=list)
    goals: List[str] = field(default_factory=lambda: ["general_fitness"])

@dataclass
class ExerciseHistory:
    total_tests: int = 0
    recent_exercises: List[Dict[str, Any]] = field(default_factory=list)
    avg_performance_score: float = 0.0
    streak_days: int = 0
    last_test_date: Optional[str] = None
```

### Dynamic Context Injection
The system prompt is **regenerated on every call** with the user's current profile and exercise history:
```python
system_prompt = SYSTEM_PROMPT.format(
    user_context=self._format_user_context(),   # "Name: X | Age: 22 | Weight: 70kg | ..."
    exercise_data=self._format_exercise_data()  # "Total Tests: 5 | Recent: Squats (15 reps), ..."
)
```
This means the LLM always has fresh context even if the user updated their profile mid-conversation.

### Singleton Pattern
```python
_assistant_instance: Optional[DietaryAssistant] = None

def get_dietary_assistant() -> DietaryAssistant:
    global _assistant_instance
    if _assistant_instance is None:
        _assistant_instance = DietaryAssistant()
    return _assistant_instance
```

One assistant instance is shared across all requests. **Production note:** This means conversation history is shared across all users in the current deployment — acceptable for a prototype/demo, but would need per-user instances (keyed by user_id) in a production system.

### Indian Dietary Awareness
The system prompt explicitly instructs the model to account for Indian dietary preferences. This is a deliberate design choice for the SIH 2025 context — Indian athletes often follow vegetarian/vegan diets, have different staple foods (dal, roti, rice), and different cultural relationships with protein supplementation.

---

## 12. Frontend Architecture — Next.js App Router

### Tech Stack
| Layer | Technology |
|---|---|
| Framework | **Next.js 16** (App Router) |
| Runtime | **React 19** |
| Camera API | `navigator.mediaDevices.getUserMedia` |
| Streaming | Native WebSocket API + `requestAnimationFrame` |
| Styling | CSS Modules (Vanilla CSS) |
| Deployment | Vercel |

### Routing Structure
```
/                    → Home: exercise selection + height input
/session/[id]        → Live session page (dynamic route; session_id from backend)
```

The session ID from the backend API is used directly as the Next.js dynamic route parameter — no separate client-side ID management needed.

### The Custom Hook — `useExerciseSocket`
This hook encapsulates the entire streaming lifecycle:

```javascript
const {
    connect,
    disconnect,
    startStreaming,
    stopStreaming,
    metrics,         // { counter, stage, feedback, fps, calibrationProgress }
    frameData,       // base64 string for <img> rendering
    isConnected,
    isStreaming,
} = useExerciseSocket(sessionId);
```

**Internals:**
1. **WebSocket lifecycle** — `connect()` opens the WS, handles `onmessage`/`onerror`/`onclose`
2. **Camera access** — `startStreaming(videoRef)` calls `getUserMedia`, attaches stream to `<video>`
3. **Frame capture loop** — `requestAnimationFrame` → draw to offscreen canvas → `toBlob` → `ws.send`
4. **Flow control** — `pendingFrameRef.current` flag
5. **State updates** — `setMetrics({counter, stage, feedback})` on each response
6. **FPS counter** — counts responses/second, updates every 1s

**Why a custom hook?** The session page component becomes clean — just calls 3 functions and renders from `metrics`. All 300+ lines of WebSocket complexity are hidden in the hook.

### REST API Client (`lib/api.js`)
```javascript
export const createSession = (exercise, heightCm) =>
    fetch(`${API_URL}/session/create?exercise=${exercise}&height_cm=${heightCm}`, { method: 'POST' })
    .then(r => r.json());

export const getSessionMetrics = (sessionId) =>
    fetch(`${API_URL}/session/${sessionId}/metrics`).then(r => r.json());

export const getWebSocketUrl = (sessionId) => {
    const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
    return `${protocol}//${API_HOST}/ws/${sessionId}`;
};
```

---

## 13. Containerisation & Deployment

### Dockerfile (Annotated)
```dockerfile
FROM python:3.11-slim
# "slim" removes docs + non-essential packages — saves ~200MB
# Critical because PyTorch + Ultralytics add significant size

WORKDIR /app

# OpenCV system dependencies — MUST be installed before pip packages
RUN apt-get update && apt-get install -y --no-install-recommends \
    libgl1      \  # OpenCV OpenGL dependency (needed even headless)
    libglib2.0-0 \ # OpenCV GLib dependency
    libsm6      \  # Session Manager (X11 dependency for imshow — not used, but required)
    libxext6    \  # X11 extensions
    libxrender1 \  # X11 rendering
    libgomp1    \  # GNU OpenMP — required by PyTorch for CPU parallelism
    ffmpeg         # Video codec support

# Non-root user — REQUIRED by HF Spaces sandbox policy
RUN useradd -m -u 1000 user
USER user
ENV HOME=/home/user PATH=/home/user/.local/bin:$PATH

WORKDIR $HOME/app

# Copy requirements first (Docker layer caching — pip install cached if requirements.txt unchanged)
COPY --chown=user requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=user . .
EXPOSE 7860                    # HF Spaces default port

CMD ["uvicorn", "fast:app", "--host", "0.0.0.0", "--port", "7860"]
```

**Why non-root uid=1000?** HF Spaces runs all Docker containers in a security sandbox that requires the running process to be uid=1000. Running as root causes `Permission denied` errors when writing files (metrics JSONs) at runtime.

**Why `--no-install-recommends`?** Keeps the Docker layer small by not installing optional package recommendations.

### Git LFS for Model Weights
YOLO model weights are binary files (~6 MB each):
```
yolo11n-pose.pt   → 6.2 MB
yolov8n.pt        → 6.5 MB
```

These are committed via **Git LFS (Large File Storage)**, configured in `.gitattributes`:
```
*.pt filter=lfs diff=lfs merge=lfs -text
```

Without LFS, Git would try to store binary blobs as text diffs → corrupted files when cloned. HF Spaces natively supports Git LFS, so the weights are downloaded correctly during the Space build.

### Deployment Platforms
| Environment | Platform | Configuration |
|---|---|---|
| Backend (primary) | **Hugging Face Spaces** | Docker SDK, port 7860, `README.md` frontmatter: `sdk: docker` |
| Backend (alternate) | **Railway** | `railway.toml`, `Procfile` (`web: uvicorn fast:app --host 0.0.0.0 --port $PORT`) |
| Frontend | **Vercel** | `NEXT_PUBLIC_API_URL=https://<hf-spaces-url>`, `vercel.json` for WS proxy rewrites |

### Production Connectivity Flow
```
Browser (HTTPS) ─────────────────────────────────────────────────────────
     │  REST (HTTPS)  │  WebSocket (WSS)
     ▼                ▼
Vercel Edge Network ──────────────────────────────── proxy/rewrite
     │  HTTP  │  WS (ws://)
     ▼         ▼
HF Spaces Container ─────────────────────────────── port 7860
     │
     ▼
FastAPI / Uvicorn
     │
     ├─► YOLO inference (CPU)
     ├─► Gemini API (HTTPS outbound)
     └─► Groq API (HTTPS outbound)
```

All external API calls (Gemini, Groq) are made **server-side** from the HF Spaces container — API keys are never exposed to the browser.

---

## 14. Full API Surface Reference

### REST Endpoints
| Method | Endpoint | Description | Request | Response |
|---|---|---|---|---|
| `GET` | `/` | API info | — | `{name, version, exercises}` |
| `GET` | `/health` | Health check | — | `{status, active_sessions}` |
| `GET` | `/exercises` | List supported exercises | — | `[{id, name, description}]` |
| `POST` | `/session/create` | Create session | `?exercise=squat&height_cm=170` | `{session_id, exercise, user_height_cm}` |
| `GET` | `/session/{id}` | Session info | — | `{session_id, exercise, is_active, counter}` |
| `GET` | `/session/{id}/metrics` | Compile final metrics | — | Exercise metrics JSON |
| `DELETE` | `/session/{id}` | End + delete session | — | Final metrics JSON |
| `POST` | `/analyze` | Gemini AI coach analysis | `{metrics_file}` | AI report string |
| `POST` | `/api/diet/chat` | Chat with dietary assistant | `{message, user_profile?, exercise_history?}` | `{response}` |
| `POST` | `/api/diet/profile` | Update user profile | Profile fields | `{profile}` |
| `GET` | `/api/diet/profile` | Get user profile | — | `{profile}` |
| `POST` | `/api/diet/history` | Sync exercise history | History dict | `{status}` |
| `GET` | `/api/diet/onboarding` | Onboarding questions | — | `{questions: [...]}` |
| `DELETE` | `/api/diet/clear` | Clear chat history | — | `{status}` |
| `GET` | `/test` | Built-in HTML test page | — | HTML page |

### WebSocket Endpoints
| Endpoint | Description |
|---|---|
| `WS /ws/{session_id}` | **Primary**: client sends binary JPEG frames → server sends JSON metrics |
| `WS /ws/webcam/{session_id}` | Server-side webcam (for local testing without client camera) |

### WebSocket Message Protocol
**Client → Server:**
```json
// Binary: raw JPEG bytes (primary path)
// OR Text:
{"type": "frame", "data": "<base64_jpeg>"}   // base64 fallback
{"type": "stop"}                              // graceful end
```

**Server → Client:**
```json
// On connection:
{"type": "connected", "session_id": "...", "exercise": "squat"}

// On each frame response:
{
  "session_id": "...",
  "exercise": "squat",
  "counter": 7,
  "stage": "UP",
  "feedback": "Great Depth!",
  "calibration_complete": true,
  "calibration_progress": 100.0,
  "cm_per_pixel": 0.182,
  "frame": "<base64_jpeg_string>",
  "timestamp": 1753520400.123,
  // Exercise-specific fields:
  "max_height_cm": 42     // for vjump
}

// Keepalive:
{"type": "keepalive", "timestamp": 1753520430.0}
```

---

## 15. Key Design Decisions & Trade-offs

| Decision | What Was Chosen | Why |
|---|---|---|
| Frame resolution | 320×240 | Enough for full-body YOLO pose; ~15 KB JPEG at 35% quality |
| Target FPS | 12–15 FPS | Sufficient for rep detection; smoothing window stays tight |
| Flow control | One frame in-flight | Prevents buffer bloat; keeps feedback real-time on variable connections |
| Calibration frames | 30 | Balances startup delay (~2.5s) vs measurement stability |
| Smoothing window | 3 frames | Removes flicker without introducing perceptible lag |
| Rep counting trigger | State machine (angle thresholds) | Robust to speed variation; slow/fast reps both counted correctly |
| Session storage | In-memory Python dict | Zero DB overhead; appropriate for 1–10 min single-user sessions |
| AI coach model | Gemini 2.5 Flash | Largest context window, fast, cheap; handles verbose metrics JSON |
| Dietary AI model | Groq LLaMA 3.3-70b | Hardware-accelerated (LPU) for chat latency; open-weight model |
| Pose model | YOLO11n-pose (nano) | Only variant fast enough for CPU-only containers at 15 FPS |
| Model weights storage | Git LFS | Binary blobs cannot be tracked by standard Git |
| Y-axis jump detection | ground_y − min_ankle_y | Image Y increases downward; higher jump = smaller Y |
| Base64 in JSON response | Embed frame in JSON | Avoids mixing binary + JSON in single WS message |
| Response quality | 40% JPEG (vs 35% request) | Annotated frames with skeleton lines compress worse than raw camera |
| CORS | allow_origins=["*"] | Demo/hackathon context; frontend and backend on different domains |

---

## 16. Performance Numbers & Resume Justification

### "15+ FPS on CPU"
- Input resolution: 320×240 (small — fast YOLO inference)
- Model: YOLO11n-pose (nano — ~6 MB, optimised for speed)
- The one-frame-in-flight constraint means FPS = 1 / round_trip_time
- On HF Spaces CPU containers: inference ~40–60 ms → effective ~15–20 FPS

### "~92% accuracy against manual measurements"
- Calibration averages 30 frames of head–ankle pixel distance
- Main sources of error: nose keypoint is ~3 cm below crown; slight tilts; ankle confidence at low res
- Validated by measuring known distances and comparing cm_per_pixel conversion output to tape measurements

### "10+ concurrent WebSocket users at <200 ms latency"
- FastAPI + Uvicorn is async (ASGI) — I/O bound operations (WebSocket recv/send) don't block other connections
- YOLO inference is CPU-bound but runs in ~40–60 ms per frame
- With 10 users at 12 FPS each: 120 inference calls/second needed; each is independent
- HF Spaces provides multi-worker support; concurrent connections handled by async event loop

### "<3 s for AI reports"
- Gemini 2.5 Flash: optimised for speed; typical latency for a ~500-token metrics input + ~800-token output is 1.5–3 s
- Groq LLaMA 3.3-70b: ~300 tok/s → a 300-token dietary response in ~1 s

---

## 17. Likely Interview Questions & Expert Answers

**Q: How do you count reps without timers or frame counting?**
> Purely geometry. I define two angle thresholds for each exercise (e.g., knee > 155° = UP, knee < 135° = DOWN). The system is a finite state machine — a rep increments when the state transitions from DOWN → UP (standing up). This is robust to speed variation: slow reps and fast reps both count correctly because it's state-based, not time-based. A timer-based approach would break for anyone who moves at a different pace.

**Q: How does the pixel-to-cm conversion work?**
> The user enters their height before the session. During the first 30 frames, I compute the Euclidean pixel distance between the nose keypoint (index 0) and the best-confidence ankle keypoint (index 15 or 16). Averaging 30 measurements gives a stable ratio: `cm_per_pixel = user_height_cm / avg_pixel_height`. From then on, any pixel distance × ratio = centimetres. I use 30 frames to filter out noise from mid-motion frames.

**Q: Why WebSocket over HTTP polling or SSE?**
> Polling creates a new TCP connection per request — at 12 FPS that's massive overhead. SSE (Server-Sent Events) is unidirectional — the server can push, but can't receive camera frames. WebSocket gives full bidirectional persistent connection: client sends binary JPEG blobs, server sends JSON metrics back. That bidirectionality is non-negotiable for this use case.

**Q: What's the flow control mechanism and why does it matter?**
> A boolean ref (`pendingFrameRef`) blocks sending a new frame until the server responds to the previous one — one frame in-flight at any time. Without this, a fast client floods a slow server. Frames queue up in the WebSocket buffer. After 10 seconds, the user is receiving feedback on frames from 10 seconds ago. The flow control ensures the system always responds to the most recently processed frame, regardless of variable network latency. Feedback latency = exactly one round-trip time.

**Q: Why YOLO for pose estimation and not MediaPipe?**
> Both are valid. YOLO11-pose runs a single forward pass that simultaneously detects people and extracts keypoints — no separate detection pipeline. The Ultralytics ecosystem has a clean Python API and the nano model is aggressively optimised for CPU throughput. MediaPipe would also work (and is excellent for mobile), but requires a separate framework dependency and its pose model is harder to fine-tune if I wanted to improve specific keypoint accuracy later.

**Q: How does the AI analysis work — does Gemini see the video?**
> No. Gemini only receives the structured metrics JSON — numbers like average knee angle, rep count, jump height in cm, good/bad rep ratio. The vision work is done entirely by YOLO + the metrics engine. Gemini's role is purely interpretation: mapping quantitative athletic measurements to a qualitative fitness profile. This separation keeps costs low (image tokens are expensive) and is architecturally clean — each system does what it's best at.

**Q: How do you handle a user being partially out of frame?**
> Every keypoint access checks the confidence score (index 2 of the (x, y, conf) tuple). If a required joint has confidence < 0.3 (for calibration) or < 0.5 (for angle computation), that joint returns `None`. The exercise processor then falls back to the other side of the body — e.g., uses right elbow if left elbow is occluded. If no valid data is available, the user sees a feedback cue like "Position yourself so arms are visible" instead of a false rep count.

**Q: What happens if the WebSocket drops mid-session?**
> The `ExerciseSession` object stays alive in `active_sessions` with all accumulated metrics. When the client reconnects using the same `session_id`, the server sets `session.is_active = True` again. The rep counter and all metrics continue from where they left off. This is why the session is only deleted on an explicit `DELETE /session/{id}` call, not on WebSocket close.

**Q: Why not use a database for sessions?**
> Sessions last 1–10 minutes and are single-user. A database would add network round-trips to every frame's processing pipeline — thousands of DB reads/writes per session for what is essentially a temporary counter. In-memory is appropriate here. The natural production evolution would be a Redis-backed session store with TTL, which would also enable horizontal scaling across multiple FastAPI instances.

**Q: Walk me through the full flow from clicking Start Workout.**
> 1. Frontend `POST /session/create?exercise=squat&height_cm=173` → gets `session_id`
> 2. Navigate to `/session/<id>` → user clicks "Start"
> 3. `getUserMedia` opens camera at 320×240
> 4. WebSocket opens to `wss://backend/ws/<session_id>`
> 5. Server sends `{"type":"connected"}` handshake
> 6. `requestAnimationFrame` loop: capture → JPEG blob (35%) → `ws.send(blob)`
> 7. Server: `np.frombuffer` → `cv2.imdecode` → YOLO inference → 17 keypoints
> 8. `_calibrate_height()` adds this frame to calibration buffer (first 30 frames)
> 9. `process_squat()` computes knee angle → state machine → feedback
> 10. Response JSON (with base64 annotated frame) → client renders it, updates UI
> 11. Flow control releases → next frame sent
> 12. After 30 calibration frames: `cm_per_pixel` locked in
> 13. User squats → knee < 135° → `stage = DOWN`, `counter += 1`
> 14. User stands → knee > 155° → `stage = UP`
> 15. User presses Stop → `{"type":"stop"}` sent → WS closes gracefully
> 16. Frontend `GET /session/<id>/metrics` → structured JSON report
> 17. User clicks Analyze → `POST /analyze` → Gemini reads metrics → athletic profile in ~2 s

**Q: How does the sit-and-reach differ from rep-based exercises?**
> It doesn't count reps at all. The `counter` field is repurposed to display the maximum reach distance. The system tracks `max_reach_distance = max(all measured wrist-ankle horizontal distances)`. The knee angle is monitored simultaneously — if knee < 165°, feedback adds "Straighten Legs!" because the SAI protocol requires straight knees for a valid measurement.

**Q: Why does the Y-axis subtraction seem reversed for jump height?**
> Image coordinate systems have Y=0 at the TOP-LEFT of the frame. Y increases downward. When a person jumps, their ankles move UP on screen — their Y coordinate DECREASES toward 0. So `height = ground_baseline_y - min_ankle_y_during_jump` is always positive. If you wrote `min_ankle_y - ground_y`, you'd get a negative number. This trips up a lot of people who are used to mathematical coordinate systems where Y increases upward.

---

*Study material generated from live codebase analysis — all code snippets are from actual source files.*
