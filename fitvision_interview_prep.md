# FitVision AI — Deep Technical Interview Prep

> Everything you need to explain this project confidently in a technical interview — from first principles to production deployment.

---

## 1. What Problem Does FitVision Solve?

Physical fitness testing (push-ups, squats, jump height, flexibility) traditionally requires a trained coach, measuring equipment, and a physical space. The SAI (Sports Authority of India) uses standardised tests to evaluate athletes, but access is limited.

FitVision removes all those constraints. A user opens a browser, points their camera at themselves, and the system:
- Counts their reps with angle-based detection (not pixel counting)
- Measures actual physical distances in centimetres from a plain webcam
- Evaluates movement quality frame by frame
- Generates a coach-level athletic profile via an LLM

**Core technical challenge:** Achieving all of this over a standard WebSocket connection with sub-500ms feedback latency, using only a mobile or laptop camera.

---

## 2. System Architecture — The Big Picture

```
[Browser / Next.js Frontend]
        |
        | POST /session/create      (REST — session bootstrap)
        | WS  /ws/{session_id}      (WebSocket — live frame stream)
        | GET /session/{id}/metrics (REST — final results)
        |
[FastAPI Backend — HF Spaces / Docker]
        |
        |— PoseCalibrator (YOLO11n-pose)
        |— PerformanceMetrics (exercise engine)
        |— AI Analysis (Gemini 2.5 Flash via LangChain)
        |— Dietary Assistant (Groq LLaMA 3.3-70b via LangChain)
```

The backend is **stateful per session** — each active session holds its own YOLO model instance, metrics buffer, calibration state, and exercise thresholds entirely in memory.

---

## 3. The Pose Estimation Layer — How YOLO Works Here

### Model Choice
The system uses **YOLO11n-pose** (nano variant of Ultralytics YOLO11). The "pose" variant outputs, for each detected person:
- A standard bounding box
- **17 keypoints**, each with `(x, y, confidence)` — mapped to nose, eyes, ears, shoulders, elbows, wrists, hips, knees, ankles

The nano model was chosen deliberately — it's fast enough to run on CPU-only HF Spaces containers without a GPU, trading some accuracy for throughput.

### Inference Per Frame
For every received frame:
```python
results = model(frame, verbose=False)
keypoints = results[0].keypoints.data[0].cpu().numpy()
# keypoints shape: (17, 3) — [x, y, confidence] per joint
```

The `.cpu().numpy()` call is important — YOLO returns PyTorch tensors; converting to NumPy allows OpenCV and SciPy operations directly.

### Confidence Filtering
Every keypoint access checks `keypoints[index][2] > threshold` (confidence). If a key joint is below confidence (e.g., 0.3), that frame is skipped or falls back to the other side of the body. This prevents ghost angles from low-quality poses causing false rep counts.

### Angle Calculation — Vector Dot Product
Joint angles are computed from three keypoints using:

```python
v1 = pt1 - pt2   # vector from vertex to point A
v2 = pt3 - pt2   # vector from vertex to point B
cos_angle = dot(v1, v2) / (|v1| * |v2|)
angle = degrees(arccos(clip(cos_angle, -1, 1)))
```

This gives the interior angle at the vertex joint — exactly what a physical therapist or coach measures.

**Interview tip:** The `clip(-1, 1)` is critical — floating point arithmetic can push the cosine slightly outside `[-1, 1]`, making `arccos` return `NaN`. Always clamp.

### Angle Smoothing — Moving Average
Raw keypoint positions jitter frame to frame due to model variance. A **3-frame moving average** is applied per exercise:

```python
self.last_knee_angles.append(knee_angle)
if len(self.last_knee_angles) > 3:
    self.last_knee_angles.pop(0)
return int(np.mean(self.last_knee_angles))
```

This smooths out flickering without introducing noticeable lag (3 frames at 12 FPS = 250ms window).

---

## 4. Height Calibration — Turning Pixels Into Centimetres

This is one of the most technically interesting parts of the system.

### The Problem
Jump height and reach distance are meaningless in pixels — a tall person standing close to the camera has a larger pixel height than a short person far away. We need real-world units.

### The Solution
The user enters their height (e.g., 173 cm) before the session. During the first 30 frames, the system:

1. Detects the `nose` keypoint (index 0) and the best-confidence `ankle` keypoint (index 15 or 16)
2. Computes Euclidean pixel distance: `H_p = sqrt((head_x - ankle_x)² + (head_y - ankle_y)²)`
3. Accumulates 30 such measurements
4. Averages them: `avg_H_p = mean(calibration_frames)`
5. Computes the ratio: `cm_per_pixel = user_height_cm / avg_H_p`

From that point on, any pixel distance multiplied by `cm_per_pixel` gives a real-world centimetre value.

### Why 30 Frames?
Single-frame measurements are noisy (the person may be mid-movement). Averaging over 30 frames (~2.5 seconds at 12 FPS) gives a stable baseline without requiring the user to stand still explicitly.

### Why Check Minimum Pixel Height?
```python
if H_p < 40:  # Relaxed for 320x240 low-res mobile stream
    return
```
If the person is far away or partially out of frame, the detected body height in pixels may be too small to produce a reliable ratio. This guard prevents garbage calibration data from entering the average.

---

## 5. Exercise State Machines — How Reps Are Counted

Every exercise is implemented as a **finite state machine** driven by joint angles. No timers, no frame counting — only geometry.

### Squat Example

States: `UP` → `DOWN` → `UP` (one rep completed on the UP transition)

```
Thresholds:
  knee > 155° → stage = "UP"
  knee < 135° → stage = "DOWN"
  knee < 90°  → "Great Depth!"  (good form)
  else        → "Squat" (cue)
```

A rep is counted **when transitioning from DOWN to UP** — not when hitting the bottom. This mirrors how a coach counts: the rep is complete when you stand back up.

```python
if knee > 155:           # Standing / UP
    if stage == "DOWN":
        counter += 1     # Rep complete
    stage = "UP"
elif knee < 135:         # Squatting / DOWN
    stage = "DOWN"
```

### Push-up Example

Uses **elbow angle** (shoulder-elbow-wrist). Down = arms bent (~90°), Up = arms extended (~140°+).

```python
if elbow > 140:          # Arms extended
    if stage == "DOWN":
        counter += 1
    stage = "UP"
elif elbow < 90:         # Arms bent
    stage = "DOWN"
```

Form check: Hip angle is simultaneously monitored. `hip < 110°` means the back is sagging → `feedback = "Fix Back!"`. This runs every frame independently of the rep counter.

### Vertical Jump — Airtime Detection

The jump state machine is different — it uses **vertical position of the midpoint between both ankles** relative to a ground reference:

```
States: standing → airborne → landing → standing
```

During `standing`, the ankle Y position establishes the ground baseline. When ankle Y rises above `(ground_Y - threshold)`, the state flips to `airborne`. The **peak ankle Y** during airborne phase is recorded. On landing (ankle Y returns to ground level), the height is:

```python
height_px = ground_baseline_y - min_ankle_y_during_jump
height_cm = height_px * cm_per_pixel
```

**Interview tip:** In image coordinates, Y increases downward. So a higher jump = **smaller Y value**. The subtraction order is reversed from what you'd expect — this trips up a lot of people.

### Sit-and-Reach — Continuous Measurement

This exercise doesn't count reps. Instead it tracks the **maximum reach distance achieved** during the session:

```python
reach_distance = wrist_x - ankle_x  # horizontal pixel distance
max_reach_distance = max(max_reach_distance, reach_distance)
```

The counter field is repurposed to display `max_reach_distance` (in pixels during calibration, in cm afterwards). Feedback cues: "Keep Reaching", "MAX REACH!", "Straighten Legs!" (if knee angle < 165°).

---

## 6. The Metrics Engine — What Gets Measured Per Rep

`PerformanceMetrics` is a large class (~2600 lines) that accumulates granular data across the full session.

### What It Tracks Per Rep (Squat Example)
- `squat_depths[]` — knee angle at bottom of each rep
- `concentric_times[]` — time to ascend from bottom
- `eccentric_times[]` — time to descend
- `torso_angles` — rolling deque of back inclination angles
- `shin_angles` — shin angle relative to vertical
- `sticking_points[]` — knee angle at minimum velocity (where reps slow down)
- `good_reps` / `bad_reps` — quality categorisation

### Final Report Generation
At session end, `squat_metrics()` compiles all collected data into a structured dict:
```json
{
  "total_reps": 15,
  "good_reps": 12,
  "bad_reps": 3,
  "avg_depth_angle": 88.3,
  "avg_concentric_time": 1.2,
  "avg_eccentric_time": 1.8,
  "sticking_point_angle": 95,
  "timestamp": "2026-07-25 12:00:00",
  "session_id": "session_20260725_120000_0"
}
```

This JSON is saved to disk (`results/metrics_{session_id}.json`) so it can later be fed to Gemini without rerunning the session.

---

## 7. The WebSocket Pipeline — Frame Streaming Protocol

### Why WebSocket Over HTTP?
HTTP is request-response. For real-time video, you need continuous bidirectional communication. WebSocket provides a persistent TCP connection with minimal overhead per frame — no HTTP headers on every frame.

### Client-Side — What the Frontend Does

The frontend custom hook manages the entire streaming lifecycle:

**Camera Setup:**
```javascript
stream = await navigator.mediaDevices.getUserMedia({
    video: { width: { exact: 320 }, height: { exact: 240 }, frameRate: { max: 15 } }
});
```
320×240 is intentional — small enough to be encoded fast and transferred quickly, large enough for YOLO to detect a full-body pose.

**Frame Encoding & Sending (requestAnimationFrame loop):**
```javascript
ctx.drawImage(videoElement, 0, 0, 320, 240);  // draw to canvas
canvas.toBlob((blob) => {
    ws.send(blob);              // send raw binary JPEG
}, 'image/jpeg', 0.35);        // 35% quality ≈ 15-20 KB
```

**Flow Control — The Key Design Decision:**
```javascript
const pendingFrameRef = useRef(false);

// Before sending:
if (!pendingFrameRef.current) {
    pendingFrameRef.current = true;
    ws.send(blob);
}

// On receiving server response:
pendingFrameRef.current = false;
```

Without this, the client would continuously send frames regardless of server processing speed. On a slow connection or overloaded server, frames would queue up in the WebSocket buffer, creating a growing lag. The flow control ensures **exactly one frame is in-flight at all times** — the client waits for the server's response before sending the next frame. This maintains real-time feedback even when latency is variable.

**FPS Calculation:**
The frontend counts received responses per second, not sent frames per second. This gives the true processed throughput, not the send rate.

### Server-Side — What FastAPI Does

```python
data = await asyncio.wait_for(websocket.receive(), timeout=30.0)

if "bytes" in data:
    # Binary frame (primary path)
    nparr = np.frombuffer(data["bytes"], np.uint8)
    frame = cv2.imdecode(nparr, cv2.IMREAD_COLOR)

elif "text" in data:
    # JSON message (control path)
    message = json.loads(data["text"])
    if message.get("type") == "stop":
        break
```

After processing, the response is sent back:
```python
small_frame = cv2.resize(processed_frame, (320, 240), interpolation=cv2.INTER_NEAREST)
_, buffer = cv2.imencode('.jpg', small_frame, [cv2.IMWRITE_JPEG_QUALITY, 40])
response["frame"] = base64.b64encode(buffer).decode('utf-8')
await websocket.send_json(response)
```

**Why INTER_NEAREST for resize?** It's the fastest interpolation method (no weighted averaging). Since the frame is already 320×240 from the client, the resize is usually a no-op, but the call ensures consistency.

**Why 40% quality on the response vs 35% on the request?** The annotated skeleton lines and text overlays on the server-processed frame compress slightly worse than a raw camera frame. 40% keeps the skeleton visually clean.

**Why base64 in JSON rather than binary?** The response also carries JSON metadata (counter, stage, feedback, calibration). Mixing binary and JSON in a single WebSocket message is messy. Base64-encoding the frame into a JSON string field keeps everything in one clean JSON payload.

### Keepalive
```python
except asyncio.TimeoutError:
    await websocket.send_json({"type": "keepalive", "timestamp": time.time()})
```
If no frame arrives for 30 seconds, the server sends a keepalive ping. This prevents the WebSocket from being closed by intermediate proxies (Vercel, Cloudflare, HF Spaces) that terminate idle connections.

---

## 8. Session Management — In-Memory State

```python
active_sessions: Dict[str, ExerciseSession] = {}
```

Sessions are stored in a Python dictionary in process memory. Each `ExerciseSession` holds:
- The YOLO model instance (`PoseCalibrator`)
- The metrics engine (`PerformanceMetrics`)
- Calibration state and `cm_per_pixel` ratio
- Rep counter, current stage, feedback string
- Smoothing buffers for knee/elbow angles

**Trade-offs of in-memory sessions:**
- ✅ Zero database overhead — no latency for session reads
- ✅ Model is loaded once per session, not per frame
- ❌ Sessions are lost on server restart
- ❌ Doesn't scale horizontally across multiple instances

For the use case (single-user fitness sessions of 1–10 minutes), in-memory is the right call. A Redis-backed approach would add complexity with no meaningful benefit here.

**Session lifecycle:**
1. `POST /session/create` → session created, YOLO loaded
2. `WS /ws/{id}` → frames processed, metrics accumulated
3. WebSocket closes → session marked inactive but NOT deleted
4. `GET /session/{id}/metrics` → metrics compiled and returned
5. `DELETE /session/{id}` → session removed from memory

Step 4 and 5 are intentionally separate — the frontend fetches metrics after disconnecting, so the session must persist briefly post-WebSocket.

---

## 9. The AI Coach — Gemini Integration

### How It Works
After a session, the saved metrics JSON is read from disk and injected into a prompt:

```python
llm = ChatGoogleGenerativeAI(model="gemini-2.5-flash", temperature=0.3)
prompt = f"""You are a SAI certified coach.
METRICS DATA: {data}
ANALYZE AND PROVIDE:
1. BODY CONDITION ASSESSMENT...
2. ATHLETIC PROFILE TYPE...
3. RECOMMENDED SPORTS (Top 5)...
4. KEY AREAS TO DEVELOP...
5. OVERALL FITNESS SCORE: X/100
"""
return llm.invoke(prompt).content
```

**Why temperature=0.3?** Lower temperature = more deterministic, factual output. For an athletic assessment, you want consistent scoring logic, not creative variation.

**Why Gemini 2.5 Flash specifically?** Flash is Google's fastest Gemini model with a very large context window. The metrics JSON can be verbose (several KB for a full session). Flash handles it cheaply and quickly.

### What Gemini Receives
The raw metrics dict includes rep counts, average angles, timing data, depth measurements, form quality flags. Gemini doesn't need image data — just the structured numbers. This is a clean separation: computer vision extracts the measurements, LLM interprets them.

---

## 10. The Dietary Assistant — Groq + LangChain

### Architecture
```python
self.llm = ChatGroq(
    api_key=GROQ_API_KEY,
    model="llama-3.3-70b-versatile",
    temperature=0.7,
    max_tokens=1024,
)
```

**Why Groq?** Groq uses custom silicon (LPUs — Language Processing Units) that run LLaMA inference at ~300–500 tokens/second. For a chat interface, this means near-instant responses — critical for user experience in a conversational flow.

### Conversation Management
```python
# Build messages list with sliding window
messages = [SystemMessage(content=system_prompt)]
for msg in self.conversation_history[-10:]:   # Last 10 messages only
    ...
messages.append(HumanMessage(content=user_message))
response = self.llm.invoke(messages)
```

The **10-message sliding window** is a memory budget decision. LLaMA 3.3-70b has a large context window, but sending the entire conversation history on every call is wasteful. 10 messages (5 turns) keeps enough context for coherent dietary advice without unnecessary token costs.

**Singleton pattern:**
```python
_assistant_instance: Optional[DietaryAssistant] = None

def get_dietary_assistant() -> DietaryAssistant:
    global _assistant_instance
    if _assistant_instance is None:
        _assistant_instance = DietaryAssistant()
    return _assistant_instance
```

One assistant instance is shared across all requests. This means conversation history is shared across all users in the current deployment — acceptable for a demo/prototype but would need per-user instances in production.

### Context Injection
The system prompt is dynamically formatted with the user's profile and exercise history on every call:

```python
system_prompt = SYSTEM_PROMPT.format(
    user_context=self._format_user_context(),    # age, weight, goals, diet
    exercise_data=self._format_exercise_data()   # recent reps, performance score
)
```

This means the LLM always has fresh context even if the user updated their profile mid-conversation.

---

## 11. Frontend Architecture — Next.js App Router

### Routing Structure
```
/              → Exercise selection + height input (Home page)
/session/[id]  → Live exercise session (dynamic route, param = session_id)
```

The session ID returned by the backend API is used directly as the Next.js route parameter — no separate frontend state management needed for navigation.

### The Custom Hook — useExerciseSocket
This is where the complexity lives on the frontend. It encapsulates:
- WebSocket lifecycle (connect, disconnect, reconnect logic)
- Camera access (`getUserMedia`)
- Frame capture loop (`requestAnimationFrame`)
- Flow control flag (`pendingFrameRef`)
- Metrics state updates (counter, stage, feedback, FPS, frame data)

**Why a custom hook?** The session page component becomes clean — it just calls `connect()`, `startStreaming(videoRef)`, `stopStreaming()`, and reads from `metrics`. All the WebSocket complexity is hidden.

### API Client
A thin module wraps all REST calls with typed functions: `createSession()`, `getSessionInfo()`, `getSessionMetrics()`, `deleteSession()`, `getWebSocketUrl()`. The WebSocket URL builder handles `ws:` vs `wss:` protocol based on the page's `https:` status — critical for secure deployments.

---

## 12. Deployment — From Code to Production

### Step 1 — Docker Build (Backend)

```dockerfile
FROM python:3.11-slim

RUN apt-get install -y libgl1 libglib2.0-0 libsm6 libxext6 libxrender1 libgomp1 ffmpeg

RUN useradd -m -u 1000 user   # Non-root required by HF Spaces
USER user

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .
EXPOSE 7860
CMD ["uvicorn", "fast:app", "--host", "0.0.0.0", "--port", "7860"]
```

**Why those system packages?** `libgl1` is OpenCV's OpenGL dependency (even headless mode needs it). `libgomp1` is GNU OpenMP, required by PyTorch for parallel CPU operations. Without these, the container crashes silently on import.

**Why non-root user?** HF Spaces runs all Docker containers as uid=1000 for security sandbox reasons. Using a root user causes permission errors on file writes during runtime.

**Why Python 3.11-slim?** `slim` removes documentation and unnecessary packages, saving ~200MB. PyTorch and Ultralytics add significant image size — keeping the base small matters.

### Step 2 — Hugging Face Spaces Deployment (Backend)

The repository is pushed to `huggingface.co/spaces/<username>/fitvision-api`. HF Spaces detects the `Dockerfile` (SDK: docker specified in README frontmatter), builds the image, and runs it on their managed infrastructure.

The YOLO model weights (`yolo11n-pose.pt`, `yolov8n.pt`, ~13MB each) are committed to the repo via **Git LFS** (Large File Storage). The `.gitattributes` file marks `.pt` files for LFS tracking. Without LFS, Git would store these as regular blobs, corrupting the binary files.

### Step 3 — Vercel Deployment (Frontend)

The Next.js frontend connects the GitHub repository to Vercel. One critical environment variable is set:
```
NEXT_PUBLIC_API_URL=https://<hf-spaces-url>
```

`NEXT_PUBLIC_` prefix is Next.js convention for variables that must be available in the browser bundle (not just server-side). Without this prefix, the API URL would be `undefined` in the client.

The `vercel.json` configures rewrites so WebSocket upgrade requests are correctly proxied to the backend. This handles the `ws:` → `wss:` upgrade that HTTPS deployments require.

### Connectivity Flow in Production

```
Browser (HTTPS/WSS)
  ↓
Vercel Edge Network (HTTPS/WSS proxy)
  ↓
HF Spaces Container (HTTP/WS on port 7860)
  ↓
FastAPI / Uvicorn
  ↓
YOLO inference (CPU) → Gemini API → Groq API
```

All external API calls (Gemini, Groq) are made server-side from the HF Spaces container, so API keys are never exposed to the browser.

---

## 13. Key Design Decisions — Interview Discussion Points

| Decision | What Was Chosen | Why |
|---|---|---|
| Frame resolution | 320×240 | YOLO detects full-body pose accurately; file size ~15KB keeps latency low |
| Frame rate | 12 FPS target | Sufficient for rep counting; smooth enough for visual feedback |
| Flow control | One frame in-flight at a time | Prevents buffer bloat on slow connections |
| Calibration frames | 30 | Balances startup delay vs measurement stability |
| Smoothing window | 3 frames | Removes jitter without introducing perceptible lag |
| Metrics storage | In-memory dict | No DB overhead for short-lived single-user sessions |
| LLM for coach | Gemini 2.5 Flash | Large context window (for verbose metrics), fast, cheap |
| LLM for diet | Groq LLaMA 3.3-70b | Hardware-accelerated inference for chat latency |
| Pose model | YOLO11n-pose (nano) | CPU-compatible, fast enough for 12 FPS on container |
| Model weights | Git LFS | Binary blobs cannot be tracked by standard Git |

---

## 14. Likely Interview Questions & Answers

**Q: How do you count reps without timers?**
A: Purely geometry. I define two angle thresholds (e.g., knee > 155° = UP, knee < 135° = DOWN). A rep increments when the state transitions from DOWN to UP. This is robust to speed variation — slow reps and fast reps both count correctly because it's state-based, not time-based.

**Q: How does the pixel-to-cm conversion work?**
A: The user enters their height. During calibration, we measure the Euclidean pixel distance between their head and ankle keypoints across 30 frames, average it, and divide the real height by this pixel height. That ratio converts any pixel distance to centimetres for the rest of the session.

**Q: Why WebSocket over polling or Server-Sent Events?**
A: Polling is wasteful (repeated HTTP connections). SSE is unidirectional — the server can push but can't receive frames. WebSocket gives full bidirectional persistent connection. The client sends binary JPEG frames; the server sends JSON metrics back — that bidirectionality is essential.

**Q: What's the flow control mechanism and why does it matter?**
A: A boolean flag prevents sending a new frame until the server responds to the previous one. Without it, fast clients flood slow servers. Frames queue up in the WebSocket buffer, feedback becomes increasingly stale, and the latency grows unboundedly. One-in-flight control keeps feedback real-time regardless of variable network conditions.

**Q: Why YOLO for pose estimation and not MediaPipe?**
A: Both are valid. YOLO11-pose runs in a single inference pass (detect + keypoints simultaneously). It's battle-tested in the Ultralytics ecosystem, has a simple Python API, and the nano model is aggressively optimised for CPU. MediaPipe would also work but requires a separate framework dependency and its pose model is less flexible for custom training later.

**Q: How does the AI analysis work — does Gemini see the video?**
A: No. Gemini only receives the structured metrics JSON — numbers like average knee angle, rep count, jump height in cm. The vision work is done entirely by YOLO. Gemini's role is purely interpretation: mapping those quantitative athletic scores to a qualitative fitness profile. This is a deliberate separation of concerns that keeps costs low.

**Q: How do you handle a user being partially out of frame?**
A: Every keypoint access checks the confidence score. If the confidence for a required joint is below 0.3, the frame is skipped or the system falls back to the other side of the body (e.g., right elbow if left elbow is occluded). The user gets a "Position yourself so arms are visible" feedback cue.

**Q: What happens if the WebSocket drops mid-session?**
A: The session object stays alive in memory. The client can reconnect to the same `session_id` — the session's `is_active` flag is reset to `True` on reconnection. The metrics accumulated before the drop are preserved. The rep counter continues from where it left off.

**Q: Why not use a database for sessions?**
A: Sessions last 1–10 minutes and are single-user. Adding a database layer would introduce network round-trips on every frame (thousands per session). In-memory is appropriate here. For a multi-instance production deployment, a Redis-backed session store with TTL would be the natural evolution.

**Q: Walk me through what happens from the moment I click "Start Workout".**
A:
1. Frontend POSTs to `/session/create?exercise=squat&height_cm=173` → gets `session_id`
2. Navigates to `/session/<id>` page
3. User clicks "Start Exercise" → `getUserMedia` opens camera at 320×240
4. WebSocket opens to `wss://<backend>/ws/<session_id>`
5. Server confirms connection with `{"type": "connected"}`
6. `requestAnimationFrame` loop begins capturing frames to canvas, encoding as JPEG blobs
7. First blob sent → server decodes → YOLO runs → calibration frame recorded
8. Server responds with `{"counter": 0, "stage": "UP", "feedback": "Squat", "calibration_progress": 3.3, "frame": "<base64>..."}` 
9. Frontend renders annotated frame, updates UI
10. Flow control releases — next frame sent
11. After 30 frames, calibration locks in — all subsequent spatial measurements converted to cm
12. User squats → knee angle crosses 135° → stage flips to DOWN
13. User stands → knee angle crosses 155° → counter increments to 1, `good_reps++` or `bad_reps++`
14. User clicks "Stop & Get Results" → `{"type": "stop"}` sent → WebSocket closes gracefully
15. Frontend GETs `/session/<id>/metrics` → server compiles full performance report → JSON returned
16. User clicks "Analyze" → `POST /analyze` → Gemini 2.5 Flash reads metrics → returns athletic profile
