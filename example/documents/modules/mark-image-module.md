# Mark Image Module

## Purpose

The Mark Image module belongs to the Create page Observe tab. Its job is to let the user mark the usable object area on capture image 1 before building a SIFT 3D model.

The marked area becomes the ROI mask for SIFT detection. White mask pixels allow SIFT keypoints; black mask pixels are ignored. This keeps background texture, fixture edges, and robot-table noise out of the model-building step.

## Current Implementation Status

Implemented with remaining test-depth limitations.

- `CreateScreen` shows image 1 as the editable ROI image after capture.
- The user can paint ROI strokes on image 1 with Brush mode.
- The user can switch to Eraser mode to remove marked area.
- The user can change brush size.
- The user can Undo and Redo completed mark strokes.
- The UI uploads a PNG mask to `PUT /api/models/:model_id/masks/1`.
- Image 2 and image 3 are display-only thumbnails in the current Create Observe UI.
- Continuing without a saved mask is allowed and means full-image SIFT detection.
- Backend build now prefers uploaded masks in `output/models/<model_id>/masks/` and generates full-image fallback masks only where uploads are missing.

Important current limitations:

- The editor still stores mark edits as vector strokes and renders an SVG overlay; an offscreen bitmap canvas is used only during PNG export.
- Automated tests verify UI controls, edit state, upload path, and backend mask path selection, but do not yet decode exported PNG pixels or assert exact mask dimensions.

## Target Behavior

The Observe tab should support a real mark workflow on image 1:

1. Paint mode marks allowed SIFT area on image 1.
2. Brush size can be changed before or during marking.
3. Eraser mode removes previously painted mask area.
4. Undo restores the previous mask edit state.
5. Redo recovers an undone edit state.
6. Saving writes a binary mask PNG for image 1.
7. Build uses the saved image 1 mask for SIFT detection.
8. If no mask is saved, build falls back to a full white image 1 mask.

## Key Files

Frontend:

- `app/src/renderer/src/screens/CreateScreen.tsx`
  - Owns the Create workflow and Observe pane.
  - Currently contains `ObservePane`, `RoiPaintCanvas`, and `strokesToPngBase64`.
- `app/src/renderer/src/globals.css`
  - Contains ROI canvas styles such as `.roi-canvas` and `.roi-svg`.
- `app/src/renderer/src/lib/api.ts`
  - Exposes `uploadMask`.
- `app/src/renderer/src/lib/realApi.ts`
  - Sends `PUT /api/models/:model_id/masks/:image_index`.
- `app/src/renderer/src/lib/mockApi.ts`
  - Demo Mode should accept mask save without backend access.

Backend:

- `backend/app.py`
  - Handles `PUT /api/models/<model_id>/masks/<image_index>`.
  - Builds `BuildModelPaths` for `POST /api/models/<model_id>/build`.
- `backend/services/masks.py`
  - Validates mask upload, checks dimensions, stores `masks/image_1_mask.png`.
- `backend/services/build_model.py`
  - Selects uploaded or generated masks, prepares fallback masks, and runs `triangulate_sift_points.py`.
- `method/sift-3d-pose-estimation-main/triangulate_sift_points.py`
  - Receives `--mask-1`, `--mask-2`, and `--mask-3`.
  - Uses `cv2.SIFT_create().detectAndCompute(gray, mask)`.

Tests:

- `app/src/renderer/src/screens/__tests__/CreateScreen.test.tsx`
- `app/src/renderer/src/lib/__tests__/api.test.ts`
- `backend/tests/test_f02_api.py`
- `backend/tests/test_build_model_service.py`

## Responsibilities

### Frontend Mark Editor

The editor should own interactive mask editing only. It should not run SIFT or infer SIFT keypoints.

Expected controls:

- Brush tool
- Eraser tool
- Brush size control
- Undo
- Redo
- Clear mask
- Save ROI

Expected state:

- Current tool: `brush` or `eraser`
- Brush size in image pixels or normalized image space
- Mask bitmap or vector edit history
- Undo stack
- Redo stack
- Dirty state after edits
- Saved state after successful upload

The current implementation stores ROI as normalized stroke records with tool and brush size metadata. This supports brush, eraser, undo, redo, and PNG export, but a future offscreen bitmap mask canvas would provide stronger pixel-level editing guarantees.

Current frontend model:

- Render image 1 as the base image.
- Render an SVG overlay from normalized mark strokes.
- Store each stroke with `tool`, `size`, and normalized points.
- Use Brush strokes as white marks during export.
- Use Eraser strokes as black marks during export.
- Undo and redo operate on completed strokes.
- Export all strokes to a binary-like PNG mask canvas for `uploadMask(modelId, 1, base64)`.

### Backend Mask Storage

The backend should continue to validate masks strictly:

- Image index must be 1, 2, or 3.
- Upload must be PNG.
- Mask dimensions must match the corresponding capture image.
- Failed validation must not replace an existing mask.

For this module, image 1 is the only editable image in the UI. Backend support for image 2 and image 3 can remain available for API compatibility.

### Build Integration

Build must use the saved mark mask for image 1.

Expected mask selection:

- If `output/models/<model_id>/masks/image_1_mask.png` exists, use it as `--mask-1`.
- If image 1 has no uploaded mask, generate a full white mask for image 1.
- For image 2 and image 3, continue generating full white masks unless future UI adds marking for those images.
- Generated fallback masks should still match each capture image size exactly.

This is implemented by selecting uploaded masks before validation and preparing generated fallback masks only for missing mask inputs.

## Data Flow

1. User creates or opens a model in Create page.
2. User starts Observe capture.
3. Backend captures `image_1.png`, `image_2.png`, and `image_3.png`.
4. Frontend displays image 1 in the mark editor.
5. User paints and erases the ROI mask.
6. User can undo or redo completed strokes.
7. User saves ROI.
8. Frontend uploads PNG base64 to `PUT /api/models/:model_id/masks/1`.
9. Backend stores `output/models/<model_id>/masks/image_1_mask.png`.
10. User continues to Build.
11. Backend chooses uploaded mask 1 plus full-image fallback masks for image 2 and image 3.
12. `triangulate_sift_points.py` runs SIFT only on white pixels in mask 1.

## UI Constraints

- Marking is only available after capture image 1 exists.
- Re-capture must reset local mask edit state and saved-mask state because image dimensions/content may change.
- Image 2 and image 3 remain display-only in this module.
- Save ROI should be disabled when there are no painted pixels or no unsaved changes.
- Build can continue without a saved mask, using full-image fallback.
- Brush size should have a clear numeric range, currently 4 to 160 px.
- Undo and redo should operate on completed strokes, including eraser strokes.
- The editor must support pointer events so mouse, trackpad, and stylus input share the same path.

## SIFT Constraints

The generated mask must be a real image-space mask, not only an SVG preview.

Mask rules:

- Same width and height as `captures/image_1.png`.
- White or nonzero pixels mean "detect SIFT here".
- Black pixels mean "ignore this area".
- PNG format.

SIFT uses the mask through OpenCV `detectAndCompute(gray, mask)`. The mask must be aligned to the original capture image dimensions, not just the displayed browser size.

## Error Handling

Frontend should show a clear failure state if mask upload fails:

- `INVALID_MASK_UPLOAD`: frontend exported an invalid upload body.
- `CAPTURE_NOT_FOUND`: capture was missing or re-captured while saving.
- `MASK_DIMENSION_MISMATCH`: exported mask dimensions do not match image 1.
- Network/backend failure: keep local edits dirty so the user can retry.

Backend should keep the old saved mask if replacement upload fails.

## Acceptance Criteria

- User can paint on image 1 after Observe capture succeeds.
- User can change brush size and see the next brush or eraser stroke use the new size.
- User can switch to eraser and remove painted area.
- User can undo the previous completed edit.
- User can redo a previously undone edit.
- Saved ROI uploads a PNG mask for image index 1.
- Re-capture clears local edits and saved mask state.
- Build uses uploaded `masks/image_1_mask.png` when it exists.
- Build falls back to a full-image mask when image 1 has no saved ROI.
- SIFT detection for image 1 is limited to the marked area when a saved mask exists.

## Testing Notes

Frontend tests should cover:

- Brush tool creates visible mask pixels.
- Brush size changes exported stroke/mask width.
- Eraser removes previously painted pixels.
- Undo and redo update the visible mask and exported mask.
- Save ROI calls `PUT /api/models/:id/masks/1`.
- Re-capture clears edits and disables Save ROI until new paint exists.

Backend tests should cover:

- Uploaded image 1 mask is selected for `--mask-1`.
- Missing image 1 mask generates full-image fallback.
- Image 2 and image 3 still get full-image fallback masks.
- Wrong-size upload is rejected and does not replace an existing mask.
- Triangulation command receives the selected mask paths.

Existing tests that assert uploaded ROI masks are ignored must be updated when this module is implemented.

## Known Limitations

- Current UI edits only image 1.
- Current mark storage is stroke-based rather than bitmap-canvas-based.
- Current automated tests do not yet decode exported PNG mask pixels.
- Current automated tests do not yet assert exact exported PNG dimensions.

## Related Documents

- `documents/modules/GUI設計規劃.md`
- `documents/modules/backend-api.md`
- `documents/implements/F03-CreatePage.md`
- `documents/implements/F04-CreatePage-ObserveUI.md`
