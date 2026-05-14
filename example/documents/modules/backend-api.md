# Backend API Reference

Base URL: `http://localhost:3001`

All responses follow a unified envelope:

```json
// Success
{ "ok": true, "data": { ... } }

// Error
{ "ok": false, "error": { "code": "ERROR_CODE", "message": "...", "details": {} } }
```

---

## Health Check

### `GET /hello`

Verify the server is running.

**Response**
```json
{ "ok": true, "data": { "message": "Hello API" } }
```

---

## Models

A **model** represents one physical object. Its lifecycle is:
`created → captured → built → (pick pose saved)`

### `GET /api/models`

List all model IDs.

**Response**
```json
{
  "ok": true,
  "data": {
    "models": ["my_part", "another_part"]
  }
}
```

---

### `POST /api/models`

Create a new model. The `model_name` is slugified into a `model_id` (lowercased, non-alphanumeric chars replaced with `_`).

**Request body**
```json
{
  "model_name": "My Part",   // required, becomes model_id "my_part"
  "overwrite": false         // optional, default false — set true to reset an existing model
}
```

**Response** — `201 Created`
```json
{
  "ok": true,
  "data": {
    "model_id": "my_part",
    "model_name": "My Part",
    "status": "created",
    "root_path": "output/models/my_part",
    "next_action": "capture"
  }
}
```

**Errors**
| Code | HTTP | Meaning |
|---|---|---|
| `INVALID_MODEL_NAME` | 400 | Name is empty or contains unsafe path characters |
| `MODEL_EXISTS` | 409 | Model already exists and `overwrite` is false |

---

### `GET /api/models/:model_id`

Get full model state including build results and pick poses.

**Response**
```json
{
  "ok": true,
  "data": {
    "model_id": "my_part",
    "model_name": "My Part",
    "status": "built",           // "created" | "captured" | "built"
    "capture_count": 3,          // 0–3 capture images present
    "mask_count": 3,             // 0–3 masks present
    "pick_pose_count": 1,
    "pick_executable": true,
    "pick_execution_blockers": [],
    "latest_pick_pose": {
      "name": "default",
      "frame": "object_T_pick_tcp",
      "saved_at": "2025-01-15T10:00:00+09:00"
    },
    "pick_poses": [ ... ],       // see Pick Pose summary shape below
    "build": {                   // null if not built yet
      "status": "succeeded",
      "build_id": "job_20250115_100000_abc123",
      "built_at": "2025-01-15T10:00:00+09:00",
      "coordinate_frame": "object",
      "point_count": 150,
      "valid_point_count": 120,
      "unchecked_point_count": 20,
      "invalid_point_count": 10,
      "point_cloud_url": "/api/files?path=output/models/my_part/point_cloud_view.png",
      "points": [
        { "xyz": [0.01, 0.02, 0.5], "status": "valid" }
      ]
    },
    "paths": {
      "root": "output/models/my_part",
      "sift_model": "output/models/my_part/sift_3d_model.json"
    }
  }
}
```

**Pick Pose summary shape**
```json
{
  "name": "default",
  "frame": "object_T_pick_tcp",
  "executable": true,
  "saved_at": "2025-01-15T10:00:00+09:00",
  "saved_base_pick_tcp_pose": [x, y, z, rx, ry, rz],
  "object_pick_tcp_pose": [x, y, z, rx, ry, rz]
}
```

**Errors**
| Code | HTTP | Meaning |
|---|---|---|
| `MODEL_NOT_FOUND` | 404 | Model does not exist |

---

### `DELETE /api/models/:model_id`

Permanently delete one model workspace under `output/models/<model_id>`. This removes the model directory and all files inside it, including captures, masks, build outputs, point-cloud previews, logs, and temporary build folders. It does not delete job history, pick run history, or snapshots outside the model directory.

**Response**
```json
{
  "ok": true,
  "data": {
    "model_id": "my_part",
    "deleted": true
  }
}
```

**Errors**
| Code | HTTP | Meaning |
|---|---|---|
| `INVALID_MODEL_ID` | 400 | Model ID is invalid or contains unsafe path characters |
| `MODEL_NOT_FOUND` | 404 | Model does not exist |
| `JOB_CONFLICT` | 409 | The model has a queued or running capture/build/save-pick-pose/pick-workflow job |
| `MODEL_DELETE_FAILED` | 500 | Filesystem deletion failed |

---

### `GET /api/models/:model_id/3d-view`

Get 3-D point cloud data and camera positions for a Three.js viewer.

**Response**
```json
{
  "ok": true,
  "data": {
    "points": [
      { "xyz": [0.01, 0.02, 0.5], "status": "valid" }
    ],
    "cameras": [
      { "id": "observe1", "pose": [x, y, z, rx, ry, rz] }
    ],
    "flange_T_camera": [[...], [...], [...], [...]]  // 4×4 matrix or null
  }
}
```

---

## Captures

Captures are 3 RGB images taken by moving the robot arm to 3 observation poses.

### `GET /api/models/:model_id/captures`

Check whether capture images and the position manifest exist.

**Response**
```json
{
  "ok": true,
  "data": {
    "model_id": "my_part",
    "captures": [
      {
        "image_index": 1,
        "pose_name": "observe1",
        "exists": true,
        "path": "output/models/my_part/captures/image_1.png",
        "url": "/api/files?path=output/models/my_part/captures/image_1.png",
        "updated_at": "2025-01-15T10:00:00+09:00"
      },
      { "image_index": 2, "pose_name": "observe2", "exists": false, "path": "...", "url": null },
      { "image_index": 3, "pose_name": "observe3", "exists": false, "path": "...", "url": null }
    ],
    "position_exists": true,
    "position_path": "output/models/my_part/captures/position_collected.json",
    "position_url": "/api/files?path=output/models/my_part/captures/position_collected.json"
  }
}
```

---

### `POST /api/models/:model_id/capture`

Start a capture job. The robot moves to 3 poses and takes an image at each. Returns a job immediately (async); poll `GET /api/jobs/:job_id` for progress.

**Request body**
```json
{
  "pose_names": ["observe1", "observe2", "observe3"],  // optional, must be exactly 3 strings
  "config_path": "method/sift-3d-pose-estimation-main/position.json",  // optional
  "dry_run": false,   // optional, default false — true generates synthetic images without robot
  "robot": {          // optional overrides
    "ip": "192.168.1.30",
    "speed": 0.08,
    "acceleration": 0.08,
    "settle_time": 0.8
  },
  "camera": {         // optional overrides
    "width": 1280,
    "height": 720,
    "warmup_frames": 5,
    "timeout_ms": 5000
  }
}
```

**Response** — `202 Accepted`
```json
{
  "ok": true,
  "data": {
    "job_id": "job_20250115_100000_abc123",
    "type": "capture_model",
    "status": "queued",
    "model_id": "my_part"
  }
}
```

**Errors**
| Code | HTTP | Meaning |
|---|---|---|
| `JOB_CONFLICT` | 409 | Another capture/build/pick job is already running |
| `INVALID_CAPTURE_REQUEST` | 400 | `pose_names` is not exactly 3 non-empty strings |
| `CAPTURE_PRECONDITION_FAILED` | 422 | Config file missing or robot/camera deps unavailable |

---

## Masks

Each of the 3 capture images needs a binary mask (white = object foreground) before building.

### `GET /api/models/:model_id/masks`

List mask status for all 3 capture images.

**Response**
```json
{
  "ok": true,
  "data": {
    "model_id": "my_part",
    "masks": [
      {
        "image_index": 1,
        "exists": true,
        "path": "output/models/my_part/masks/image_1_mask.png",
        "url": "/api/files?path=output/models/my_part/masks/image_1_mask.png"
      },
      { "image_index": 2, "exists": false, "path": "...", "url": null },
      { "image_index": 3, "exists": false, "path": "...", "url": null }
    ]
  }
}
```

---

### `PUT /api/models/:model_id/masks/:image_index`

Upload or replace the mask for capture image 1, 2, or 3. The uploaded PNG must match the capture image dimensions exactly.

Accepts two upload formats:

**Format A — multipart/form-data**
```
field name: mask
file:       binary PNG
```

**Format B — JSON body**
```json
{
  "content_type": "image/png",
  "data_base64": "<base64-encoded PNG bytes>"
}
```

**Response**
```json
{
  "ok": true,
  "data": {
    "model_id": "my_part",
    "image_index": 1,
    "path": "output/models/my_part/masks/image_1_mask.png",
    "url": "/api/files?path=output/models/my_part/masks/image_1_mask.png"
  }
}
```

**Errors**
| Code | HTTP | Meaning |
|---|---|---|
| `INVALID_IMAGE_INDEX` | 400 | Index is not 1, 2, or 3 |
| `INVALID_MASK_UPLOAD` | 400 | Missing field, bad base64, or unsupported content type |
| `CAPTURE_NOT_FOUND` | 422 | The corresponding capture image does not exist yet |
| `MASK_DIMENSION_MISMATCH` | 422 | Mask size ≠ capture size |

---

## Build

Runs the SIFT 3-D triangulation algorithm to create a point-cloud model from the 3 captures + masks.

### `POST /api/models/:model_id/build`

Start a build job. Returns immediately; poll `GET /api/jobs/:job_id` for progress.

**Request body** (all optional)
```json
{
  "coordinate_frame_mode": "robot_base",   // only "robot_base" is supported
  "paths": {
    "position":    "method/sift-3d-pose-estimation-main/position.json",
    "calibration": "method/sift-3d-pose-estimation-main/calibration_result.json",
    "hand_eye":    "method/sift-3d-pose-estimation-main/hand_eye_result.json"
  }
}
```

**Response** — `202 Accepted`
```json
{
  "ok": true,
  "data": {
    "job_id": "job_20250115_100000_abc123",
    "type": "build_model",
    "status": "queued",
    "model_id": "my_part"
  }
}
```

On job success, `GET /api/jobs/:job_id` `result` contains:
```json
{
  "coordinate_frame": "robot_base",
  "point_count": 150,
  "valid_point_count": 120,
  "output_path": "output/models/my_part/sift_3d_model.json",
  "point_cloud_url": "/api/files?path=output/models/my_part/point_cloud_view.png",
  "build_id": "job_...",
  "built_at": "2025-01-15T10:00:00+09:00"
}
```

**Errors**
| Code | HTTP | Meaning |
|---|---|---|
| `JOB_CONFLICT` | 409 | Another job is already running |
| `UNSUPPORTED_COORDINATE_FRAME_MODE` | 400 | Mode other than `robot_base` requested |
| `BUILD_PRECONDITION_FAILED` | 422 | Captures or masks are missing |

---

## Pick Poses

### `POST /api/models/:model_id/pick-poses`

Save the robot's current TCP pose as a named pick pose (reads live from the robot via RTDE). If the model is in `robot_base` frame and `promote_to_object_frame` is true, the model is promoted to `object` frame using the captured TCP pose as the object origin — this only happens once.

**Request body**
```json
{
  "pick_name": "default",         // optional, default "default", alphanumeric + _-
  "promote_to_object_frame": true,// optional, default true
  "dry_run": false,               // optional, default false
  "pose": [x, y, z, rx, ry, rz], // only allowed when dry_run=true; otherwise read from robot
  "robot": {
    "ip": "192.168.1.30"          // optional override
  }
}
```

**Response** — `202 Accepted` (async job)
```json
{
  "ok": true,
  "data": {
    "job_id": "job_...",
    "type": "save_pick_pose",
    "status": "queued",
    "model_id": "my_part"
  }
}
```

On job success, `GET /api/jobs/:job_id` `result` contains:
```json
{
  "model_id": "my_part",
  "pick_name": "default",
  "replaced": false,
  "frame": "object_T_pick_tcp",
  "coordinate_frame": "object",
  "previous_coordinate_frame": "robot_base",
  "promoted_to_object_frame": true,
  "saved_at": "2025-01-15T10:00:00+09:00",
  "pick_pose_count": 1,
  "saved_base_pick_tcp_pose": [x, y, z, rx, ry, rz],
  "base_T_object": [[...], [...], [...], [...]]
}
```

**Errors**
| Code | HTTP | Meaning |
|---|---|---|
| `JOB_CONFLICT` | 409 | Another job is running |
| `PICK_POSE_PRECONDITION_FAILED` | 422 | Model not built, or robot RTDE unavailable |
| `INVALID_PICK_NAME` | 400 | Pick name contains invalid characters |
| `INVALID_PICK_POSE_REQUEST` | 400 | Bad request body (e.g. `pose` given without `dry_run`) |

---

## Pick Workflow

Runs a full pick cycle: observe pose → capture image → SIFT pose estimation → (optional) execute pick on robot.

### `POST /api/pick/workflow`

Start a pick workflow job. Returns immediately; poll `GET /api/jobs/:job_id` for results.

**Request body**
```json
{
  "model_id": "my_part",         // required
  "observe_name": "observe",     // optional, default "observe"
  "pick_name": "default",        // optional, default "default"
  "approach_height_m": 0.05,     // optional, default 0.05, range (0, 1.0]
  "dry_run": true,               // optional, default true — false executes the physical pick
  "config_path": "...",          // optional
  "calibration_path": "...",     // optional
  "hand_eye_path": "..."         // optional
}
```

**Response** — `202 Accepted`
```json
{
  "ok": true,
  "data": {
    "job_id": "job_...",
    "type": "pick_workflow",
    "status": "queued",
    "model_id": "my_part"
  }
}
```

On job success, `GET /api/jobs/:job_id` `result` contains:
```json
{
  "run_id": "run_20250115_100000_abc123",
  "model_id": "my_part",
  "dry_run": true,
  "detected": true,
  "quality": "GOOD",           // "GOOD" | "BAD"
  "reason": "ok",
  "matches": 85,
  "inliers": 72,
  "inlier_ratio": 0.85,
  "estimated_object_pose": { ... },
  "estimated_pick_pose": {
    "name": "default",
    "frame": "base_T_pick_tcp",
    "pose": [x, y, z, rx, ry, rz],
    "matrix": [[...], [...], [...], [...]]
  },
  "captured_image_url": "/api/files?path=output/runs/run_.../observe.png",
  "result_image_url": "/api/files?path=output/runs/run_.../output_observe.png",
  "pose_results_url": "/api/files?path=output/runs/run_.../pose_results.json",
  "pick_execution": null,        // populated if dry_run=false and pick was executed
  "model_coordinate_frame": "object",
  "model_pick_pose_count": 1
}
```

**Errors**
| Code | HTTP | Meaning |
|---|---|---|
| `JOB_CONFLICT` | 409 | Another job is running |
| `INVALID_PICK_WORKFLOW_REQUEST` | 400 | Bad request body |
| `PICK_WORKFLOW_PRECONDITION_FAILED` | 422 | Model not built, no executable pick pose, or script missing |

---

## Jobs

All long-running operations (capture, build, pick-pose, pick-workflow) are async jobs.

### `GET /api/jobs/:job_id`

Poll a job for its current state. Logs are refreshed on every poll (last 50 lines from the log file).

**Response**
```json
{
  "ok": true,
  "data": {
    "job_id": "job_20250115_100000_abc123",
    "type": "build_model",       // "capture_model" | "build_model" | "save_pick_pose" | "pick_workflow"
    "status": "running",         // "queued" | "running" | "succeeded" | "failed"
    "model_id": "my_part",
    "step": "running",           // free-form current step label
    "progress": 10,              // 0–100
    "created_at": "2025-01-15T10:00:00+09:00",
    "updated_at": "2025-01-15T10:00:05+09:00",
    "started_at": "2025-01-15T10:00:01+09:00",
    "finished_at": null,
    "result": null,              // populated when status=succeeded
    "error": null,               // populated when status=failed: { "code": "...", "message": "..." }
    "log_path": "output/models/my_part/build_log.txt",
    "logs": [
      "build_model job queued",
      "Running triangulation..."
    ]
  }
}
```

**Polling guidance:** Poll every 1–2 seconds while `status` is `queued` or `running`. Stop when `status` is `succeeded` or `failed`.

**Errors**
| Code | HTTP | Meaning |
|---|---|---|
| `JOB_NOT_FOUND` | 404 | Job ID does not exist (jobs are in-memory; lost on server restart) |

---

## Camera

The backend owns a persistent `CameraService` singleton (see `documents/modules/backend-camera-manager.md`). Under normal operation the frontend does not need to call `POST /api/camera/start` — the server warms up the camera automatically. The management endpoints exist for status polling, forced restart after errors, and diagnostic snapshots.

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

Trigger camera warmup. Returns immediately — poll `GET /api/camera/status` until `status` is `"ready"`. Idempotent: already `warming` or `ready` succeeds without restarting.

**Request body** (all optional)
```json
{ "width": 1280, "height": 720, "warmup_frames": 5, "timeout_ms": 5000 }
```

**Response** — `202 Accepted`
```json
{ "ok": true, "data": { "status": "warming" } }
```

**Errors**
| Code | HTTP | Meaning |
|---|---|---|
| `CAMERA_UNAVAILABLE` | 422 | Camera hardware or SDK not present |

---

### `POST /api/camera/stop`

Gracefully stop the camera pipeline.

**Response**
```json
{ "ok": true, "data": { "status": "off" } }
```

---

### `POST /api/camera/snapshot`

Capture one color frame, save it under `output/snapshots/`, and return its URL. Fails immediately if camera is not `ready`.

**Request body** (all optional)
```json
{ "timeout_ms": 5000 }
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
|---|---|---|
| `CAMERA_NOT_READY` | 409 | Camera status is not `ready` |
| `CAMERA_FRAME_FAILED` | 500 | Pipeline returned no frame |

---

## Files

### `GET /api/files?path=<relative-path>`

Serve any file under `output/models/`, `output/jobs/`, `output/runs/`, or `output/snapshots/`. The `url` / `*_url` fields returned by other endpoints are valid values for this `path` parameter.

**Example**
```
GET /api/files?path=output/models/my_part/captures/image_1.png
GET /api/files?path=output/models/my_part/point_cloud_view.png
```

Returns the raw file with appropriate `Content-Type`.

**Errors**
| Code | HTTP | Meaning |
|---|---|---|
| `INVALID_FILE_PATH` | 400 | Path is missing, absolute, or contains `..` |
| `INVALID_FILE_PATH` | 400 | Path resolves outside allowed roots |
| `FILE_NOT_FOUND` | 404 | File does not exist |

---

## Typical Frontend Flow

```
1.  POST /api/models               → create model, get model_id
2.  POST /api/models/:id/capture   → start capture job
    GET  /api/jobs/:job_id         → poll until succeeded
3.  GET  /api/models/:id/captures  → confirm 3 images exist, get URLs
    GET  /api/files?path=...       → display capture images in UI
4.  PUT  /api/models/:id/masks/1   → upload mask for image 1
    PUT  /api/models/:id/masks/2   → upload mask for image 2
    PUT  /api/models/:id/masks/3   → upload mask for image 3
5.  POST /api/models/:id/build     → start build job
    GET  /api/jobs/:job_id         → poll until succeeded
6.  GET  /api/models/:id           → verify status="built", view point_cloud_url
7.  POST /api/models/:id/pick-poses → save pick pose (robot must be at pick position)
    GET  /api/jobs/:job_id          → poll until succeeded
8.  POST /api/pick/workflow         → run pick estimation (dry_run=true first)
    GET  /api/jobs/:job_id          → poll until succeeded
    GET  /api/files?path=...        → display result image
```

---

## Common Error Codes

| Code | Meaning |
|---|---|
| `MODEL_NOT_FOUND` | Model ID does not exist |
| `JOB_NOT_FOUND` | Job ID does not exist |
| `JOB_CONFLICT` | Only one active job allowed at a time |
| `BUILD_PRECONDITION_FAILED` | Missing captures or masks |
| `CAPTURE_PRECONDITION_FAILED` | Missing config or robot/camera unavailable |
| `PICK_POSE_PRECONDITION_FAILED` | Model not built or RTDE unavailable |
| `PICK_WORKFLOW_PRECONDITION_FAILED` | Model not pick-executable or script missing |
| `NOT_FOUND` | Unknown endpoint |
| `INTERNAL_ERROR` | Unexpected server exception |
