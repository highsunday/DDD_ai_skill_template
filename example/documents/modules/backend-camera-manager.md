# Backend Camera Manager

## Overview

The Camera Manager is a persistent **singleton service** that keeps the RealSense camera pipeline alive for the entire server lifetime. This eliminates the ~10-second warmup penalty that previously occurred on every capture job and every pick workflow run.

Before this change, every job followed the pattern:

```
start_camera() → warmup_camera() [~10 s] → read frames → pipeline.stop()
```

After this change the server owns the camera:

```
server start → CameraService.start() → warmup once
job runs     → CameraService.snapshot() [instant]
server stop  → CameraService.stop()
```

---

## CameraService

**File:** `backend/services/camera_service.py`

A thread-safe singleton that wraps the RealSense pipeline.

### States

| State | Meaning |
|-------|---------|
| `off` | Pipeline not started |
| `warming` | `start_camera()` + `warmup_camera()` in progress (background thread) |
| `ready` | Camera is warm; `snapshot()` may be called |
| `error` | Last start or snapshot raised an exception |

### Key methods

| Method | Behaviour |
|--------|-----------|
| `start(width, height, warmup_frames, timeout_ms)` | Launches warmup in a background thread. Idempotent — does nothing if already `warming` or `ready`. |
| `stop()` | Stops the pipeline and transitions to `off`. Safe to call from any state. |
| `snapshot(timeout_ms) → (color_image, depth_image)` | Blocks until `ready`, then reads one frame. Raises `CameraNotReadyError` if state is not `ready` after a short wait. |
| `status() → dict` | Returns current state dict (see `/api/camera/status` response shape). |

---

## New API Endpoints

Base URL: `http://localhost:3001`

---

### `GET /api/camera/status`

Return the current camera state.

**Response**
```json
{
  "ok": true,
  "data": {
    "status": "ready",
    "width": 1280,
    "height": 720,
    "started_at": "2025-01-15T10:00:00+09:00",
    "error": null
  }
}
```

`status` values: `"off"` | `"warming"` | `"ready"` | `"error"`

---

### `POST /api/camera/start`

Trigger camera warmup. Returns immediately — poll `GET /api/camera/status` until `status` is `"ready"`. Idempotent: if the camera is already `warming` or `ready`, the call succeeds without restarting.

**Request body** (all optional)
```json
{
  "width": 1280,
  "height": 720,
  "warmup_frames": 5,
  "timeout_ms": 5000
}
```

**Response** — `202 Accepted`
```json
{
  "ok": true,
  "data": {
    "status": "warming"
  }
}
```

**Errors**
| Code | HTTP | Meaning |
|------|------|---------|
| `CAMERA_UNAVAILABLE` | 422 | Camera hardware or SDK not present |

---

### `POST /api/camera/stop`

Gracefully stop the camera pipeline.

**Response**
```json
{
  "ok": true,
  "data": {
    "status": "off"
  }
}
```

---

### `POST /api/camera/snapshot`

Capture one color frame. Saves the image under `output/snapshots/` and returns its URL.
Fails immediately if the camera is not `ready`.

**Request body** (all optional)
```json
{
  "timeout_ms": 5000
}
```

**Response**
```json
{
  "ok": true,
  "data": {
    "path": "output/snapshots/snap_20250115_100000_abc123.png",
    "url": "/api/files?path=output/snapshots/snap_20250115_100000_abc123.png",
    "captured_at": "2025-01-15T10:00:00+09:00"
  }
}
```

**Errors**
| Code | HTTP | Meaning |
|------|------|---------|
| `CAMERA_NOT_READY` | 409 | Camera status is not `ready` |
| `CAMERA_FRAME_FAILED` | 500 | Pipeline returned no frame |

---

## How Existing Jobs Change

### `POST /api/models/:model_id/capture`

`run_real_capture()` in `backend/services/capture_model.py`:

| Before | After |
|--------|-------|
| `camera = start_camera(w, h)` | _(removed)_ |
| `warmup_camera(camera, ...)` | _(removed)_ |
| `_capture_color_frame(camera, ...)` | `color_image, _ = camera_service.snapshot(timeout_ms)` |
| `camera["pipeline"].stop()` in `finally` | _(removed — service owns lifetime)_ |

The camera object is no longer passed into helper functions; `CameraService` is injected at the service layer.

---

### `POST /api/pick/workflow`

`run_pick_workflow()` in `backend/services/pick_workflow.py`:

The external script (`estimate_pose_from_sift_3d.py`) is called with `--capture`, which starts its own camera pipeline internally. After this refactor the backend pre-captures the observe image instead:

| Before | After |
|--------|-------|
| Script launched with `--capture` | Backend calls `camera_service.snapshot()` first, saves to `run_dir/observe.png` |
| Script owns camera init + warmup | Script launched with `--image run_dir/observe.png` (no `--capture`) |

This keeps the external script's camera code path intact for standalone use while removing the duplicate warmup from the workflow path.

---

## Server Lifecycle Integration

**File:** `backend/app.py`

```python
# on startup (after app is created)
camera_service.start()          # non-blocking, warms up in background

# on shutdown (atexit / signal handler)
camera_service.stop()
```

The frontend does **not** need to call `POST /api/camera/start` under normal operation. The management endpoints exist for:
- Forced restart after a camera error (`POST /api/camera/stop` → `POST /api/camera/start`)
- Status polling during startup (`GET /api/camera/status`)
- Diagnostic snapshots

---

## Frontend Polling Pattern

```
App opens
  ↓
GET /api/camera/status          → poll every 1 s until status === "ready"
  ↓
Capture / Pick workflow jobs    → run normally; no camera warmup delay
  ↓
App closes (optional)
  POST /api/camera/stop
```

If status is `"error"` the frontend should surface a warning and offer a "Restart Camera" button that calls `POST /api/camera/stop` then `POST /api/camera/start`.
