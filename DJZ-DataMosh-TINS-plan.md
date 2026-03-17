# DJZ-DATAMOSH

<!-- TINS Specification v1.0 -->
<!-- ZS:COMPLEXITY:HIGH -->
<!-- ZS:PRIORITY:HIGH -->
<!-- ZS:PLATFORM:WEB -->
<!-- ZS:LANGUAGE:JAVASCRIPT -->

---

## Description

DJZ-DATAMOSH is a fully client-side HTML5 single-page application that applies eight distinct datamosh and pixel-sorting glitch effects to video files. The user loads an MP4 or WebM video, selects one of eight effect modes, adjusts mode-specific parameters, processes the video frame-by-frame, previews the result, and exports a ZIP archive containing the processed frame sequence as numbered PNG images.

All video and frame storage uses **IndexedDB** via named object stores. No server-side backend exists. No API calls are made. The application is delivered as a single portable `index.html` file with all CSS and JS inlined. The eight effects are JavaScript ports of the Python ComfyUI custom nodes found in `glitch-nodes/` — the original source files (`DjzDatamosh.py` through `DjzDatamoshV8.py`) are available at implementation time for reference.

The target audience is glitch artists, VJs, and video experimenters who want browser-based datamoshing without installing Python, ComfyUI, or FFmpeg.

### The Eight Effect Modes

| # | Mode Name | Source Node | Summary |
|---|-----------|-------------|---------|
| 1 | **Glide** | `DjzDatamosh.py` (V1) | Block-based motion estimation + displacement; motion smears across frames |
| 2 | **Multi-Mosh** | `DJZDatamoshV2.py` | Three sub-modes: `glide` (propagate motion from first 2 frames), `movement` (frame-to-frame tracking), `copy` (passthrough) |
| 3 | **I-Frame Kill** | `DjzDatamoshV3.py` | Simulates classic datamosh by removing keyframe resets (`iframe_removal`) or repeating delta frames (`delta_repeat`) |
| 4 | **Vector Transfer** | `DjzDatamoshV4.py` | Extracts motion vectors from a source video and applies them to a target video — style transfer via motion |
| 5 | **Size Sort** | `DjzDatamoshV5.py` | Re-orders frames within a range by their compressed file size |
| 6 | **Pixel Sort (Sobel)** | `DjzDatamoshV6.py` | Sobel edge detection defines segments; pixels within each segment are sorted by luminance |
| 7 | **Pixel Sort (Advanced)** | `DjzDatamoshV7.py` | Multi-mode pixel sorting (luminance / hue / saturation / laplacian) with rotation and multi-pass |
| 8 | **Pixel Sort (Masked)** | `DjzDatamoshV8.py` | Same as V7 but with a user-painted mask controlling where sorting is applied |

---

## Functionality

### Core Features

1. **Video input** — file picker or drag-and-drop accepting MP4 and WebM. The loaded video is stored as a Blob in the `videoGallery` IndexedDB store. A `<video>` element is used to decode frames.

2. **Frame extraction** — on load, the video is stepped through frame-by-frame using `requestVideoFrameCallback` (with `seekToNextFrame` fallback via `video.currentTime` stepping at 1/fps intervals). Extracted frames are stored as an array of `ImageData` objects in memory for the active session. A frame count and resolution summary is displayed.

3. **Effect selection** — a tabbed panel or dropdown selects one of the eight modes. Selecting a mode reveals its parameter controls (described per-mode below). Only one effect is applied at a time.

4. **Parameter controls** — each mode exposes sliders, dropdowns, toggles, and number inputs matching the parameters of the original Python node. Controls use the same defaults, ranges, and step sizes as the source.

5. **Processing** — clicking **Process** runs the selected effect across the extracted frame array. A progress bar shows `frame N / total`. Processing is done inside a `requestAnimationFrame` / `setTimeout(0)` yield loop (or Web Worker) to keep the UI responsive. The result is a new array of processed `ImageData` frames.

6. **Preview** — processed frames play back in a `<canvas>` element at the original video's frame rate. Play/pause and frame-scrub controls are provided. A toggle switches between original and processed preview.

7. **Export as ZIP** — clicking **Export** bundles all processed frames as sequentially-numbered PNGs (`frame_0001.png`, `frame_0002.png`, ...) into a ZIP file using the `JSZip` library (inlined). The ZIP is offered as a browser download. Export also stores a `manifest.json` inside the ZIP containing: effect name, all parameter values, source video filename, frame count, resolution, and timestamp.

8. **Video gallery** — a thumbnail strip of all previously loaded videos (pulled from IndexedDB). Clicking a thumbnail reloads that video for processing.

9. **Output history** — each processed result is saved to the `outputGallery` IndexedDB store as a metadata record (effect name, parameters, timestamp, source video ID). The actual frame PNGs are NOT persisted to IndexedDB (too large); only the ZIP Blob is stored if the user explicitly clicks **Save to Library** after export.

10. **Settings panel** — output format (PNG quality), max frame extraction limit (default 300 frames), target processing resolution (original / 720p / 480p downscale for speed), and a **Clear DB** button.

---

### UI Layout

```
+================================================================+
|  DJZ-DATAMOSH                              [Settings]          |
+================================================================+
|                                                                |
|  +----------------------------------------------------------+ |
|  |                                                          | |
|  |                  VIDEO PREVIEW CANVAS                    | |
|  |                  (original or processed)                 | |
|  |                                                          | |
|  +----------------------------------------------------------+ |
|  |  [|<] [<] [ ▶ Play ] [>] [>|]   Frame: 042/120   [⟳]   | |
|  |  [=============================○-----------] scrub bar   | |
|  |  ( ) Original   (●) Processed                           | |
|  +----------------------------------------------------------+ |
|                                                                |
|  +------ VIDEO GALLERY (thumbnails) -----------------------+  |
|  | [thumb1] [thumb2] [thumb3]  ...        [+ Load Video]   |  |
|  +----------------------------------------------------------+ |
|                                                                |
|  +------ EFFECT PANEL -------------------------------------+  |
|  |  Mode: [▼ Glide / Multi-Mosh / I-Frame Kill / ... ]    |  |
|  |                                                          | |
|  |  ┌─ Parameters (varies per mode) ────────────────────┐  | |
|  |  │  Block Size:   [====○====] 16                     │  | |
|  |  │  Max Shift:    [====○====] 8                      │  | |
|  |  │  Shift Range:  [====○====] 2                      │  | |
|  |  └──────────────────────────────────────────────────┘  | |
|  |                                                          | |
|  |  [▶ Process]                     [💾 Export ZIP]         | |
|  |  [=========================================] 0%          | |
|  +----------------------------------------------------------+ |
+================================================================+
```

---

### Mode-Specific Parameter Panels

#### Mode 1: Glide (V1)

| Parameter | Type | Default | Min | Max | Step | Description |
|-----------|------|---------|-----|-----|------|-------------|
| `block_size` | INT slider | 16 | 4 | 64 | 4 | Block size for motion analysis |
| `max_shift` | INT slider | 8 | 1 | 32 | 1 | Maximum pixel displacement |
| `shift_range` | INT slider | 2 | 1 | 4 | 1 | Search step size (higher = faster, less accurate) |

#### Mode 2: Multi-Mosh (V2)

| Parameter | Type | Default | Min | Max | Step | Description |
|-----------|------|---------|-----|-----|------|-------------|
| `sub_mode` | dropdown | `glide` | — | — | — | `glide` / `movement` / `copy` |
| `block_size` | INT slider | 16 | 4 | 64 | 4 | Block size for motion analysis |
| `max_shift` | INT slider | 8 | 1 | 32 | 1 | Maximum pixel displacement |
| `shift_range` | INT slider | 2 | 1 | 4 | 1 | Search step size |
| `sequence_length` | INT slider | 30 | 1 | 300 | 1 | Frames to generate (glide sub-mode only) |

#### Mode 3: I-Frame Kill (V3)

| Parameter | Type | Default | Min | Max | Step | Description |
|-----------|------|---------|-----|-----|------|-------------|
| `sub_mode` | dropdown | `iframe_removal` | — | — | — | `iframe_removal` / `delta_repeat` |
| `start_frame` | INT input | 0 | 0 | 999 | 1 | First frame to affect |
| `end_frame` | INT input | -1 | -1 | 999 | 1 | Last frame to affect (-1 = end) |
| `delta_frames` | INT slider | 5 | 1 | 30 | 1 | P-frames to repeat (delta_repeat only) |

**Web port note:** The original V3 encodes frames to AVI, manipulates raw I-frame/P-frame bytes, then decodes back. Since the browser cannot do raw AVI byte manipulation, the web port **simulates** these effects at the pixel level:

- **`iframe_removal` simulation:** Each frame in the affected range is replaced by the result of accumulating pixel differences onto the previous output frame. Specifically: `output[i] = output[i-1] + (input[i] - input[i-1])`. This mimics what happens when a video decoder never receives a keyframe reset — motion deltas keep accumulating on a stale reference, creating the classic "pixel bloom" datamosh look.

- **`delta_repeat` simulation:** Collect the first `delta_frames` frame-differences within the range, then cyclically replay those differences onto subsequent frames: `output[i] = output[i-1] + deltas[i % delta_frames]`. This mimics repeating P-frame motion data.

#### Mode 4: Vector Transfer (V4)

| Parameter | Type | Default | Min | Max | Step | Description |
|-----------|------|---------|-----|-----|------|-------------|
| `mode` | dropdown | `extract_and_transfer` | — | — | — | `extract_and_transfer` / `extract_only` / `transfer_only` |
| `method` | dropdown | `add` | — | — | — | `add` (additive) / `replace` |
| `source_video` | file picker | — | — | — | — | Second video to extract motion from |

**Web port note:** The original V4 uses `ffgac` and `ffedit` (FFGlitch tools) to extract/apply MPEG motion vectors. The web port uses the same block-matching algorithm from V1/V2 to compute motion vectors in JavaScript, then applies them to the target frames. This produces a visually equivalent effect without requiring FFGlitch:

1. **Extract:** Run `find_shifts_fast()` on consecutive source frames → produces an array of per-block motion vector grids, one per frame.
2. **Transfer (add):** For each target frame pair, compute target motion vectors, then add the source vectors element-wise.
3. **Transfer (replace):** Apply source motion vectors directly to target frames via `apply_shifts()`.
4. **Extract only:** Computes vectors and stores them in IndexedDB as a JSON blob for later use by `transfer_only`.

The `source_video` is loaded via a second file picker. When mode is `transfer_only`, a dropdown lists previously saved vector sets from IndexedDB.

#### Mode 5: Size Sort (V5)

| Parameter | Type | Default | Min | Max | Step | Description |
|-----------|------|---------|-----|-----|------|-------------|
| `reverse_sort` | toggle | true | — | — | — | true = largest-first, false = smallest-first |
| `start_frame` | INT input | 0 | 0 | 999 | 1 | First frame to sort |
| `end_frame` | INT input | -1 | -1 | 999 | 1 | Last frame to sort (-1 = end) |

**Implementation:** For each frame in the sort range, render to an offscreen canvas, call `canvas.toBlob('image/png')`, record the resulting Blob size. Sort frames by size (ascending or descending per `reverse_sort`). Frames outside the range keep their original order.

#### Mode 6: Pixel Sort — Sobel (V6)

| Parameter | Type | Default | Min | Max | Step | Description |
|-----------|------|---------|-----|-----|------|-------------|
| `threshold` | FLOAT slider | 128.0 | 0.0 | 255.0 | 1.0 | Sobel edge detection sensitivity |

#### Mode 7: Pixel Sort — Advanced (V7)

| Parameter | Type | Default | Min | Max | Step | Description |
|-----------|------|---------|-----|-----|------|-------------|
| `sort_mode` | dropdown | `luminance` | — | — | — | `luminance` / `hue` / `saturation` / `laplacian` |
| `threshold` | FLOAT slider | 0.5 | 0.0 | 1.0 | 0.05 | Segment creation threshold |
| `rotation` | INT dropdown | -90 | -180 | 180 | 90 | Sorting direction angle |
| `multi_pass` | toggle | false | — | — | — | Apply all four modes sequentially |
| `seed` | INT input | 42 | 0 | 4294967295 | 1 | Random seed for reproducibility |

#### Mode 8: Pixel Sort — Masked (V8)

Same parameters as Mode 7, plus:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `mask_mode` | dropdown | `paint` | `paint` (user draws) / `auto_luma` (auto-generate from luminance threshold) |
| `mask_brush_size` | INT slider | 20 | Brush radius for paint mode |
| `mask_threshold` | FLOAT slider | 0.5 | Luminance threshold for auto mode |
| `mask_invert` | toggle | false | Invert the mask |

In `paint` mode, the user paints white areas onto a transparent overlay canvas (visible as semi-transparent red). White = apply sorting, black = keep original. In `auto_luma` mode, a luminance threshold generates the mask automatically from the first frame and applies it to all frames.

---

### User Flows

#### Flow 1 — Happy Path (full processing cycle)

1. User opens `index.html` in a modern browser.
2. User clicks **Load Video** or drags an MP4/WebM file onto the drop zone.
3. Video is stored in `videoGallery` IndexedDB. A thumbnail appears in the Video Gallery strip.
4. Frames are extracted. Progress shows `"Extracting frames... 42/120"`. On completion, the first frame displays in the preview canvas and frame count is shown.
5. User selects **Glide** from the effect mode dropdown.
6. Parameter sliders appear. User adjusts `block_size` to 32, leaves others at defaults.
7. User clicks **Process**.
   - Progress bar advances: `"Processing frame 12/120"`.
   - UI remains responsive (processing yields to event loop).
8. Processing completes. Preview automatically switches to the processed result. Play button starts playback.
9. User scrubs through frames, toggles between original and processed views.
10. User clicks **Export ZIP**.
    - ZIP is built in-browser (JSZip). Progress: `"Packing frame 42/120"`.
    - Browser download dialog offers `datamosh_glide_20260316_143022.zip`.
11. ZIP contains `frame_0001.png` through `frame_0120.png` plus `manifest.json`.

#### Flow 2 — Vector Transfer (dual-video)

1. User loads the **target** video (primary Load Video flow).
2. User selects **Vector Transfer** mode.
3. User clicks **Load Source Video** (a second file picker specific to this mode).
4. Source video frames are extracted to a separate in-memory array.
5. User selects `extract_and_transfer` mode, `add` method, clicks **Process**.
6. Motion vectors are extracted from source, applied additively to target frames.
7. Result previews. User exports as ZIP.

#### Flow 3 — Masked Pixel Sort (paint mask)

1. User loads a video and selects **Pixel Sort (Masked)** mode.
2. `mask_mode` defaults to `paint`. A **Draw Mask** button appears.
3. User clicks **Draw Mask**. The first frame shows in the canvas with a painting overlay.
4. User paints red regions (white in mask data) over areas to sort. An eraser tool and clear button are available.
5. User clicks **Done Drawing**. The mask `ImageData` is captured.
6. User adjusts `sort_mode`, `threshold`, and `rotation`, then clicks **Process**.
7. Each frame is pixel-sorted only within the masked region.

#### Flow 4 — Re-process with Different Effect

1. After processing, the user changes the mode dropdown to a different effect.
2. Parameters update to the new mode's controls.
3. User clicks **Process** again. The **original** extracted frames are used (never the previously processed output). This ensures effects don't stack unintentionally.
4. New result replaces the processed preview.

#### Flow 5 — Settings Adjustment

1. User opens Settings modal.
2. Changes max frame limit to 600, processing resolution to 720p.
3. Clicks Save. Settings persist in `localStorage`.
4. Next video load will extract up to 600 frames, downscaled to 720p for processing.

---

### Edge Cases

- **No video loaded** — all effect controls and Process/Export buttons are disabled. Tooltip: `"Load a video first"`.
- **Video with 0 or 1 extractable frame** — show error: `"Video too short — need at least 2 frames"`. Modes requiring 2+ frames disable the Process button.
- **Video exceeds max frame limit** — extraction stops at the limit. Warning banner: `"Video truncated to 300 frames (adjust in Settings)"`.
- **Unsupported codec** — if `<video>` cannot decode, show error: `"Browser cannot decode this video format. Try MP4 (H.264) or WebM (VP8/VP9)"`.
- **Processing cancelled** — a **Cancel** button appears during processing. Clicking it aborts the loop and restores the previous state.
- **Memory pressure** — at 1080p with 300 frames, the frame array is ~1.5 GB of ImageData. The Settings "processing resolution" option (720p/480p) reduces this. If `canvas.getContext('2d')` returns null due to memory, show: `"Out of memory — try reducing resolution or frame count in Settings"`.
- **IndexedDB unavailable** — app shows warning: `"Storage unavailable — videos won't persist across sessions"`. App continues in-memory only.
- **Export with no processed frames** — Export button is disabled until processing completes.
- **Vector Transfer: source/target frame count mismatch** — use `min(source.length, target.length)` frames. Warn: `"Frame count mismatch — processing N frames"`.
- **V4 transfer_only with no saved vectors** — dropdown is empty, Process disabled. Message: `"No saved vector sets — extract vectors first"`.
- **V8 mask not drawn** — if `paint` mode is selected but no mask has been drawn, Process uses a full-white mask (sort everything, equivalent to V7).
- **Size Sort: all frames identical size** — output order equals input order. No error needed.

---

## Technical Implementation

### Architecture

Pure client-side single-page application. No backend. Everything inlined in a single `index.html`. JSZip is the only library, inlined as a `<script>` block.

```
┌──────────────────────────────────────────────────────────────┐
│  UI Layer (HTML5 + CSS3 + Vanilla JS)                        │
│  Video preview canvas, playback controls, parameter panels,  │
│  gallery strip, mask painter, modals                         │
├──────────────────────────────────────────────────────────────┤
│  Effects Engine (Vanilla JS — Canvas API + ImageData)        │
│  8 effect processors, block matching, pixel sorting,         │
│  motion vector extraction, frame diff accumulation           │
├──────────────────────────────────────────────────────────────┤
│  Video I/O Layer (HTMLVideoElement + Canvas)                  │
│  Frame extraction, playback, resolution scaling              │
├──────────────────────────────────────────────────────────────┤
│  Export Layer (JSZip + Canvas.toBlob)                         │
│  ZIP packing, PNG encoding, manifest generation              │
├──────────────────────────────────────────────────────────────┤
│  Storage Layer (IndexedDB + localStorage)                    │
│  videoGallery / outputHistory / vectorSets (IDB)             │
│  Settings (localStorage)                                      │
└──────────────────────────────────────────────────────────────┘
```

**Logical module breakdown (all inlined into a single `index.html`):**

```
db.js                 ← IndexedDB wrapper (open, put, get, getAll, delete, clearAll)
settings.js           ← localStorage load/save for AppSettings
video-io.js           ← frame extraction from <video>, resolution scaling
frame-player.js       ← canvas playback engine with scrub/play/pause
gallery.js            ← video thumbnail strip, output history strip
effects/glide.js      ← Mode 1: V1 block motion glide
effects/multi-mosh.js ← Mode 2: V2 glide/movement/copy
effects/iframe-kill.js← Mode 3: V3 iframe_removal + delta_repeat simulation
effects/vector-xfer.js← Mode 4: V4 motion vector extract + transfer
effects/size-sort.js  ← Mode 5: V5 frame size sorting
effects/pixel-sort-sobel.js    ← Mode 6: V6 Sobel pixel sort
effects/pixel-sort-advanced.js ← Mode 7: V7 multi-mode pixel sort
effects/pixel-sort-masked.js   ← Mode 8: V8 masked pixel sort
mask-painter.js       ← canvas overlay for drawing masks (V8)
export.js             ← ZIP creation via JSZip, manifest generation
app.js                ← top-level orchestration, state management, UI wiring
```

---

### Dependencies

**One inlined library:**
- **JSZip** (v3.10+) — ZIP file creation in-browser. Inline the minified source directly in a `<script>` tag. Source: the standard JSZip distribution (`jszip.min.js`, ~100 KB).

**Native browser APIs used:**
- Canvas 2D API (`getImageData`, `putImageData`, `drawImage`, `toBlob`)
- `HTMLVideoElement` + `requestVideoFrameCallback` (frame extraction)
- IndexedDB API (persistent storage)
- `localStorage` (settings)
- `File` / `FileReader` / `URL.createObjectURL` (video loading)
- `crypto.randomUUID` (ID generation)
- `Blob` / `URL.createObjectURL` (ZIP download)

No other CDN libraries. No build step.

---

### Data Models

#### AppSettings (persisted in `localStorage` under key `"djz_datamosh_settings"`)

```javascript
{
  outputFormat:       string,  // "image/png" (only PNG for frame sequences)
  maxFrames:          number,  // Maximum frames to extract, default 300
  processingRes:      string,  // "original" | "720p" | "480p", default "original"
  frameRate:          number   // Playback FPS override, default 0 (= use source video FPS)
}

const DEFAULT_SETTINGS = {
  outputFormat: 'image/png',
  maxFrames: 300,
  processingRes: 'original',
  frameRate: 0
};
```

#### VideoRecord (stored in `videoGallery` object store, keyPath `'id'`)

```javascript
{
  id:            string,  // generateUUID()
  filename:      string,  // Original filename e.g. "clip.mp4"
  mimeType:      string,  // "video/mp4" | "video/webm"
  blob:          Blob,    // Original video file blob
  width:         number,  // Video intrinsic width
  height:        number,  // Video intrinsic height
  duration:      number,  // Duration in seconds
  fps:           number,  // Detected or estimated frame rate
  frameCount:    number,  // Actual extracted frame count
  thumbnailBlob: Blob,    // JPEG thumbnail of first frame
  addedAt:       number   // Date.now()
}
```

#### VectorSetRecord (stored in `vectorSets` object store, keyPath `'id'`)

```javascript
{
  id:            string,  // generateUUID()
  sourceVideoId: string,  // FK → VideoRecord.id
  vectors:       Array,   // Array of per-frame motion vector grids (JSON-serialisable)
  blockSize:     number,  // Block size used during extraction
  maxShift:      number,  // Max shift used
  frameCount:    number,  // Number of frames vectors were extracted from
  createdAt:     number   // Date.now()
}
```

#### OutputRecord (stored in `outputHistory` object store, keyPath `'id'`)

```javascript
{
  id:            string,  // generateUUID()
  videoId:       string,  // FK → VideoRecord.id
  effectName:    string,  // "glide" | "multi_mosh" | "iframe_kill" | etc.
  parameters:    object,  // Snapshot of all parameter values used
  frameCount:    number,  // Number of output frames
  width:         number,  // Output frame width
  height:        number,  // Output frame height
  zipBlob:       Blob,    // ZIP file blob (only if user clicked Save to Library)
  createdAt:     number   // Date.now()
}
```

---

### Key Algorithms

All effect algorithms below are JavaScript ports of the Python originals in `glitch-nodes/`. The original `.py` files should be open side-by-side during implementation. Variable names, parameter names, and algorithmic structure should match the originals as closely as idiomatic JavaScript allows.

---

#### 1. Frame Extraction (`video-io.js`)

```javascript
// Extract frames from a video element into an array of ImageData
// Uses requestVideoFrameCallback where available, falls back to currentTime stepping
async function extractFrames(videoElement, maxFrames, targetRes) {
  const frames = [];
  const fps = estimateFPS(videoElement); // from videoElement metadata or default 30
  const frameDuration = 1 / fps;

  // Determine output dimensions based on targetRes setting
  const { width, height } = computeTargetDimensions(
    videoElement.videoWidth, videoElement.videoHeight, targetRes
  );

  const canvas = document.createElement('canvas');
  canvas.width = width;
  canvas.height = height;
  const ctx = canvas.getContext('2d');

  videoElement.currentTime = 0;
  await waitForSeek(videoElement);

  while (videoElement.currentTime < videoElement.duration && frames.length < maxFrames) {
    ctx.drawImage(videoElement, 0, 0, width, height);
    frames.push(ctx.getImageData(0, 0, width, height));

    videoElement.currentTime += frameDuration;
    await waitForSeek(videoElement);

    // Yield to UI thread every 10 frames
    if (frames.length % 10 === 0) {
      await yieldToUI();
      onProgress?.(frames.length);
    }
  }

  return { frames, width, height, fps };
}

function computeTargetDimensions(srcW, srcH, targetRes) {
  if (targetRes === 'original') return { width: srcW, height: srcH };
  const targetH = targetRes === '720p' ? 720 : 480;
  if (srcH <= targetH) return { width: srcW, height: srcH };
  const scale = targetH / srcH;
  // Ensure even dimensions (required for many video operations)
  return {
    width:  Math.round(srcW * scale / 2) * 2,
    height: Math.round(srcH * scale / 2) * 2
  };
}

function waitForSeek(video) {
  return new Promise(resolve => {
    if (video.seeking) {
      video.addEventListener('seeked', resolve, { once: true });
    } else {
      resolve();
    }
  });
}

function yieldToUI() {
  return new Promise(resolve => setTimeout(resolve, 0));
}
```

---

#### 2. Block Matching — Shared by Modes 1, 2, 4 (`effects/shared-block-match.js`)

This is the core motion estimation algorithm used by V1, V2, and V4. Port directly from `DjzDatamosh.py` `find_shifts_fast()` and `apply_shifts()`, converting from PyTorch tensors to Canvas `ImageData`.

```javascript
// prevFrame, currFrame: ImageData objects (same dimensions)
// Returns: 2D array of { dx, dy } motion vectors, one per block
function findShiftsFast(prevFrame, currFrame, blockSize, maxShift, shiftRange) {
  const { width, height } = prevFrame;
  const hBlocks = Math.floor(height / blockSize);
  const wBlocks = Math.floor(width / blockSize);
  const shifts = Array.from({ length: hBlocks }, () =>
    Array.from({ length: wBlocks }, () => ({ dx: 0, dy: 0 }))
  );

  const prevData = prevFrame.data;  // Uint8ClampedArray RGBA
  const currData = currFrame.data;

  const step = shiftRange;
  const shiftValues = [];
  for (let s = -maxShift; s <= maxShift; s += step) shiftValues.push(s);

  for (let h = 0; h < hBlocks; h++) {
    for (let w = 0; w < wBlocks; w++) {
      const y = h * blockSize;
      const x = w * blockSize;
      const bh = Math.min(blockSize, height - y);
      const bw = Math.min(blockSize, width - x);

      let minDiff = Infinity;
      let bestDx = 0, bestDy = 0;

      for (const dy of shiftValues) {
        for (const dx of shiftValues) {
          const py = ((y + dy) % height + height) % height;
          const px = ((x + dx) % width + width) % width;

          // Check bounds: block must fit entirely
          if (py + bh > height || px + bw > width) continue;

          let diff = 0;
          for (let by = 0; by < bh; by++) {
            for (let bx = 0; bx < bw; bx++) {
              const ci = ((y + by) * width + (x + bx)) * 4;
              const pi = ((py + by) * width + (px + bx)) * 4;
              diff += Math.abs(currData[ci] - prevData[pi])
                    + Math.abs(currData[ci+1] - prevData[pi+1])
                    + Math.abs(currData[ci+2] - prevData[pi+2]);
            }
          }

          if (diff < minDiff) {
            minDiff = diff;
            bestDx = dx;
            bestDy = dy;
          }
        }
      }

      shifts[h][w] = { dx: bestDx, dy: bestDy };
    }
  }

  return shifts;
}

// Apply motion vectors to a frame, producing a new ImageData
function applyShifts(frame, shifts, blockSize) {
  const { width, height } = frame;
  const hBlocks = Math.floor(height / blockSize);
  const wBlocks = Math.floor(width / blockSize);

  const output = new ImageData(new Uint8ClampedArray(frame.data), width, height);

  for (let h = 0; h < hBlocks; h++) {
    for (let w = 0; w < wBlocks; w++) {
      const yStart = h * blockSize;
      const xStart = w * blockSize;
      const bh = Math.min(blockSize, height - yStart);
      const bw = Math.min(blockSize, width - xStart);

      const { dx, dy } = shifts[h][w];
      const srcY = ((yStart + dy) % height + height) % height;
      const srcX = ((xStart + dx) % width + width) % width;

      // Check bounds
      if (srcY + bh > height || srcX + bw > width) continue;

      for (let by = 0; by < bh; by++) {
        for (let bx = 0; bx < bw; bx++) {
          const di = ((yStart + by) * width + (xStart + bx)) * 4;
          const si = ((srcY + by) * width + (srcX + bx)) * 4;
          output.data[di]   = frame.data[si];
          output.data[di+1] = frame.data[si+1];
          output.data[di+2] = frame.data[si+2];
          output.data[di+3] = frame.data[si+3];
        }
      }
    }
  }

  return output;
}
```

---

#### 3. Mode 1 — Glide (`effects/glide.js`)

Direct port of `DJZDatamosh.datamosh_glide()`. Refer to `DjzDatamosh.py` lines 135-168.

```javascript
// frames: ImageData[], blockSize, maxShift, shiftRange: ints
// onProgress: (frameIndex) => void
// Returns: ImageData[]
async function processGlide(frames, { blockSize, maxShift, shiftRange }, onProgress) {
  if (frames.length < 2) return frames;

  const output = [frames[0]];
  let currentFrame = frames[0];

  for (let i = 1; i < frames.length; i++) {
    const shifts = findShiftsFast(currentFrame, frames[i], blockSize, maxShift, shiftRange);
    const moshed = applyShifts(currentFrame, shifts, blockSize);
    output.push(moshed);
    currentFrame = moshed;
    onProgress?.(i);
    await yieldToUI();
  }

  return output;
}
```

---

#### 4. Mode 2 — Multi-Mosh (`effects/multi-mosh.js`)

Direct port of `DJZDatamoshV2.py`. Three sub-mode functions. Refer to lines 142-206.

```javascript
async function processMultiMosh(frames, { subMode, blockSize, maxShift, shiftRange, sequenceLength }, onProgress) {
  if (frames.length < 2) return frames;

  if (subMode === 'copy') return [...frames];

  if (subMode === 'glide') {
    // Compute shifts from first two frames, then propagate
    const shifts = findShiftsFast(frames[0], frames[1], blockSize, maxShift, shiftRange);
    const output = [frames[0]];
    let current = frames[0];
    for (let i = 0; i < sequenceLength - 1; i++) {
      current = applyShifts(current, shifts, blockSize);
      output.push(current);
      onProgress?.(i + 1);
      await yieldToUI();
    }
    return output;
  }

  // subMode === 'movement'
  const output = [frames[0]];
  let current = frames[0];
  for (let i = 1; i < frames.length; i++) {
    const shifts = findShiftsFast(current, frames[i], blockSize, maxShift, shiftRange);
    const moshed = applyShifts(current, shifts, blockSize);
    output.push(moshed);
    current = moshed;
    onProgress?.(i);
    await yieldToUI();
  }
  return output;
}
```

---

#### 5. Mode 3 — I-Frame Kill (`effects/iframe-kill.js`)

Simulates V3 without AVI manipulation. See `DjzDatamoshV3.py` lines 67-135 for the byte-manipulation logic being simulated.

```javascript
async function processIFrameKill(frames, { subMode, startFrame, endFrame, deltaFrames }, onProgress) {
  if (frames.length < 2) return frames;

  const end = endFrame < 0 ? frames.length : Math.min(endFrame, frames.length);
  const start = Math.min(startFrame, end);
  const output = [];
  const { width, height } = frames[0];

  if (subMode === 'iframe_removal') {
    // Simulate: accumulate frame deltas without keyframe resets
    for (let i = 0; i < frames.length; i++) {
      if (i === 0 || i < start || i >= end) {
        output.push(cloneImageData(frames[i]));
      } else {
        // output[i] = output[i-1] + (frames[i] - frames[i-1])
        const prev = output[i - 1];
        const currInput = frames[i];
        const prevInput = frames[i - 1];
        const result = new ImageData(width, height);
        for (let p = 0; p < result.data.length; p += 4) {
          for (let c = 0; c < 3; c++) {
            const delta = currInput.data[p + c] - prevInput.data[p + c];
            result.data[p + c] = clamp(prev.data[p + c] + delta, 0, 255);
          }
          result.data[p + 3] = 255;
        }
        output.push(result);
      }
      onProgress?.(i);
      if (i % 5 === 0) await yieldToUI();
    }
  } else {
    // delta_repeat: collect deltaFrames frame-diffs, then loop them
    const deltas = []; // each delta is a Float32Array of per-pixel RGB diffs

    for (let i = 0; i < frames.length; i++) {
      if (i === 0 || i < start || i >= end) {
        output.push(cloneImageData(frames[i]));
      } else if (deltas.length < deltaFrames) {
        // Collect delta
        const delta = new Float32Array(width * height * 3);
        for (let p = 0, d = 0; p < frames[i].data.length; p += 4, d += 3) {
          delta[d]     = frames[i].data[p]     - frames[i - 1].data[p];
          delta[d + 1] = frames[i].data[p + 1] - frames[i - 1].data[p + 1];
          delta[d + 2] = frames[i].data[p + 2] - frames[i - 1].data[p + 2];
        }
        deltas.push(delta);
        output.push(cloneImageData(frames[i]));
      } else {
        // Replay deltas cyclically
        const deltaIdx = (i - start - deltaFrames) % deltaFrames;
        const delta = deltas[deltaIdx];
        const prev = output[i - 1];
        const result = new ImageData(width, height);
        for (let p = 0, d = 0; p < result.data.length; p += 4, d += 3) {
          result.data[p]     = clamp(prev.data[p]     + delta[d],     0, 255);
          result.data[p + 1] = clamp(prev.data[p + 1] + delta[d + 1], 0, 255);
          result.data[p + 2] = clamp(prev.data[p + 2] + delta[d + 2], 0, 255);
          result.data[p + 3] = 255;
        }
        output.push(result);
      }
      onProgress?.(i);
      if (i % 5 === 0) await yieldToUI();
    }
  }

  return output;
}

function clamp(v, min, max) { return v < min ? min : v > max ? max : Math.round(v); }
function cloneImageData(src) {
  return new ImageData(new Uint8ClampedArray(src.data), src.width, src.height);
}
```

---

#### 6. Mode 4 — Vector Transfer (`effects/vector-xfer.js`)

Web port of `DjzDatamoshV4.py`. Instead of `ffgac`/`ffedit`, uses the shared `findShiftsFast` for extraction. Refer to `DjzDatamoshV4.py` lines 60-99 (`get_vectors`) and 101-162 (`apply_vectors`) for the structural logic.

```javascript
// Extract motion vectors from a frame sequence
function extractVectors(frames, blockSize, maxShift, shiftRange) {
  const vectors = [null]; // first frame has no motion
  for (let i = 1; i < frames.length; i++) {
    vectors.push(findShiftsFast(frames[i - 1], frames[i], blockSize, maxShift, shiftRange));
  }
  return vectors;
}

async function processVectorTransfer(targetFrames, { mode, method, sourceFrames, savedVectors, blockSize, maxShift, shiftRange }, onProgress) {
  let vectors = null;

  if (mode === 'extract_and_transfer' || mode === 'extract_only') {
    vectors = extractVectors(sourceFrames, blockSize || 16, maxShift || 8, shiftRange || 2);
    if (mode === 'extract_only') {
      // Return vectors for saving — caller stores in IndexedDB
      return { vectors, frames: targetFrames };
    }
  }

  if (mode === 'transfer_only') {
    vectors = savedVectors; // loaded from IndexedDB
  }

  if (!vectors) return targetFrames;

  // Apply vectors to target
  const numFrames = Math.min(targetFrames.length, vectors.length);
  const output = [targetFrames[0]];

  for (let i = 1; i < numFrames; i++) {
    if (!vectors[i]) {
      output.push(cloneImageData(targetFrames[i]));
      continue;
    }

    if (method === 'replace') {
      // Apply source vectors directly to target frame
      output.push(applyShifts(targetFrames[i], vectors[i], blockSize || 16));
    } else {
      // 'add': compute target vectors, add source vectors, apply combined
      const targetVecs = findShiftsFast(
        targetFrames[i - 1], targetFrames[i], blockSize || 16, maxShift || 8, shiftRange || 2
      );
      const combined = addVectors(targetVecs, vectors[i]);
      output.push(applyShifts(targetFrames[i], combined, blockSize || 16));
    }
    onProgress?.(i);
    await yieldToUI();
  }

  return { vectors, frames: output };
}

function addVectors(a, b) {
  return a.map((row, i) =>
    row.map((v, j) => ({
      dx: v.dx + (b[i]?.[j]?.dx || 0),
      dy: v.dy + (b[i]?.[j]?.dy || 0)
    }))
  );
}
```

---

#### 7. Mode 5 — Size Sort (`effects/size-sort.js`)

Port of `DjzDatamoshV5.py`. Uses `canvas.toBlob` to estimate compressed frame size. Refer to `DjzDatamoshV5.py` lines 41-100.

```javascript
async function processSizeSort(frames, { reverseSort, startFrame, endFrame }, onProgress) {
  if (frames.length < 2) return frames;

  const end = endFrame < 0 ? frames.length : Math.min(endFrame, frames.length);
  const start = Math.min(startFrame, end);

  // Measure compressed size of each frame in the sort range
  const canvas = document.createElement('canvas');
  canvas.width = frames[0].width;
  canvas.height = frames[0].height;
  const ctx = canvas.getContext('2d');

  const sizes = []; // { size, index }

  for (let i = start; i < end; i++) {
    ctx.putImageData(frames[i], 0, 0);
    const blob = await new Promise(r => canvas.toBlob(r, 'image/png'));
    sizes.push({ size: blob.size, index: i });
    onProgress?.(i);
  }

  // Sort by size
  sizes.sort((a, b) => reverseSort ? b.size - a.size : a.size - b.size);

  // Build output: pre-range + sorted + post-range
  const output = [];
  for (let i = 0; i < start; i++) output.push(frames[i]);
  for (const { index } of sizes) output.push(frames[index]);
  for (let i = end; i < frames.length; i++) output.push(frames[i]);

  return output;
}
```

---

#### 8. Mode 6 — Pixel Sort Sobel (`effects/pixel-sort-sobel.js`)

Port of `DjzDatamoshV6.py`. All scipy.ndimage operations replaced with manual convolution. Refer to `DjzDatamoshV6.py` lines 29-105.

```javascript
function pixelSortSobel(imageData, threshold) {
  const { width, height, data } = imageData;
  const output = new ImageData(new Uint8ClampedArray(data), width, height);

  // 1. Compute luma
  const luma = new Float32Array(width * height);
  for (let i = 0, p = 0; i < luma.length; i++, p += 4) {
    luma[i] = 0.2126 * data[p] + 0.7152 * data[p+1] + 0.0722 * data[p+2];
  }

  // 2. Sobel edge detection
  const sobelMask = sobelEdgeDetect(luma, width, height, threshold);

  // 3. For each row, find segments defined by edges, sort pixels by luma within segments
  for (let y = 0; y < height; y++) {
    let segStart = 0;
    for (let x = 0; x <= width; x++) {
      const isEdge = x === width || sobelMask[y * width + x];
      if (isEdge && x > segStart) {
        // Sort this segment by luma
        sortSegment(output.data, luma, y, segStart, x, width);
        segStart = x;
      } else if (isEdge) {
        segStart = x;
      }
    }
  }

  return output;
}

// Sobel operator implementation
function sobelEdgeDetect(luma, width, height, threshold) {
  const dx = new Float32Array(width * height);
  const dy = new Float32Array(width * height);

  // Sobel kernels applied manually
  for (let y = 1; y < height - 1; y++) {
    for (let x = 1; x < width - 1; x++) {
      const idx = y * width + x;
      // Horizontal Sobel
      dx[idx] = -luma[(y-1)*width+(x-1)] - 2*luma[y*width+(x-1)] - luma[(y+1)*width+(x-1)]
                +luma[(y-1)*width+(x+1)] + 2*luma[y*width+(x+1)] + luma[(y+1)*width+(x+1)];
      // Vertical Sobel
      dy[idx] = -luma[(y-1)*width+(x-1)] - 2*luma[(y-1)*width+x] - luma[(y-1)*width+(x+1)]
                +luma[(y+1)*width+(x-1)] + 2*luma[(y+1)*width+x] + luma[(y+1)*width+(x+1)];
    }
  }

  // Magnitude + threshold
  const mask = new Uint8Array(width * height);
  let maxMag = 0;
  const magnitude = new Float32Array(width * height);
  for (let i = 0; i < magnitude.length; i++) {
    magnitude[i] = Math.hypot(dx[i], dy[i]);
    if (magnitude[i] > maxMag) maxMag = magnitude[i];
  }
  for (let i = 0; i < mask.length; i++) {
    mask[i] = (magnitude[i] / maxMag * 255) > threshold ? 1 : 0;
  }
  return mask;
}

function sortSegment(data, luma, y, start, end, width) {
  const len = end - start;
  const indices = Array.from({ length: len }, (_, i) => start + i);
  indices.sort((a, b) => luma[y * width + a] - luma[y * width + b]);

  // Copy sorted pixels into output
  const temp = new Uint8Array(len * 4);
  for (let i = 0; i < len; i++) {
    const srcIdx = (y * width + indices[i]) * 4;
    temp[i * 4]     = data[srcIdx];
    temp[i * 4 + 1] = data[srcIdx + 1];
    temp[i * 4 + 2] = data[srcIdx + 2];
    temp[i * 4 + 3] = data[srcIdx + 3];
  }
  for (let i = 0; i < len; i++) {
    const dstIdx = (y * width + (start + i)) * 4;
    data[dstIdx]     = temp[i * 4];
    data[dstIdx + 1] = temp[i * 4 + 1];
    data[dstIdx + 2] = temp[i * 4 + 2];
    data[dstIdx + 3] = temp[i * 4 + 3];
  }
}

// Batch wrapper for processing all frames
async function processPixelSortSobel(frames, { threshold }, onProgress) {
  const output = [];
  for (let i = 0; i < frames.length; i++) {
    output.push(pixelSortSobel(frames[i], threshold));
    onProgress?.(i);
    if (i % 3 === 0) await yieldToUI();
  }
  return output;
}
```

---

#### 9. Mode 7 — Pixel Sort Advanced (`effects/pixel-sort-advanced.js`)

Port of `DjzDatamoshV7.py`. All numpy/scipy replaced with manual JS. Refer to the full file for structure — particularly `calculate_hue`, `calculate_saturation`, `calculate_laplacian`, `calculate_luminance` (lines 44-97), `apply_pixel_sorting` (lines 134-171), and the multi-pass logic (lines 217-235).

```javascript
// Value calculation functions — port each from DjzDatamoshV7.py
const VALUE_FUNCTIONS = {
  luminance: (data, w, h) => {
    const vals = new Float32Array(w * h);
    for (let i = 0, p = 0; i < vals.length; i++, p += 4) {
      vals[i] = 0.2126 * data[p] + 0.7152 * data[p+1] + 0.0722 * data[p+2];
    }
    return vals;
  },

  hue: (data, w, h) => {
    const vals = new Float32Array(w * h);
    for (let i = 0, p = 0; i < vals.length; i++, p += 4) {
      const r = data[p] / 255, g = data[p+1] / 255, b = data[p+2] / 255;
      const hue = Math.atan2(Math.sqrt(3) * (g - b), 2 * r - g - b);
      vals[i] = (hue + Math.PI) / (2 * Math.PI); // normalise to 0-1
    }
    return vals;
  },

  saturation: (data, w, h) => {
    const vals = new Float32Array(w * h);
    for (let i = 0, p = 0; i < vals.length; i++, p += 4) {
      const r = data[p] / 255, g = data[p+1] / 255, b = data[p+2] / 255;
      const max = Math.max(r, g, b);
      const min = Math.min(r, g, b);
      vals[i] = max > 1e-7 ? (max - min) / max : 0;
    }
    return vals;
  },

  laplacian: (data, w, h) => {
    // First compute luminance, then convolve with Laplacian kernel
    const lum = new Float32Array(w * h);
    for (let i = 0, p = 0; i < lum.length; i++, p += 4) {
      lum[i] = (data[p] + data[p+1] + data[p+2]) / (3 * 255);
    }
    const kernel = [0, -1, 0, -1, 4, -1, 0, -1, 0]; // 3x3
    const vals = new Float32Array(w * h);
    for (let y = 1; y < h - 1; y++) {
      for (let x = 1; x < w - 1; x++) {
        let sum = 0;
        for (let ky = -1; ky <= 1; ky++) {
          for (let kx = -1; kx <= 1; kx++) {
            sum += lum[(y + ky) * w + (x + kx)] * kernel[(ky + 1) * 3 + (kx + 1)];
          }
        }
        vals[y * w + x] = Math.abs(sum);
      }
    }
    // Normalise
    let min = Infinity, max = -Infinity;
    for (let i = 0; i < vals.length; i++) {
      if (vals[i] < min) min = vals[i];
      if (vals[i] > max) max = vals[i];
    }
    const range = max - min + 1e-7;
    for (let i = 0; i < vals.length; i++) vals[i] = (vals[i] - min) / range;
    return vals;
  }
};

// applyPixelSorting — port of DjzDatamoshV7.py apply_pixel_sorting()
// Rotates image, computes values, finds edges from threshold, sorts segments, rotates back
function applyPixelSorting(imageData, valueFn, threshold, rotation) {
  // Rotate the image data
  const kRotations = (((rotation / 90) % 4) + 4) % 4;
  let { data, width, height } = rotateImageData(imageData, kRotations);

  // Compute values
  const values = valueFn(data, width, height);

  // Normalise values to 0-1
  let vMin = Infinity, vMax = -Infinity;
  for (let i = 0; i < values.length; i++) {
    if (values[i] < vMin) vMin = values[i];
    if (values[i] > vMax) vMax = values[i];
  }
  if (vMax > vMin) {
    for (let i = 0; i < values.length; i++) values[i] = (values[i] - vMin) / (vMax - vMin);
  }

  // Create edge mask from threshold
  // Edge = where (value > threshold) changes from one pixel to the next in same row
  const edges = new Uint8Array(width * height);
  for (let y = 0; y < height; y++) {
    for (let x = 1; x < width; x++) {
      const curr = values[y * width + x] > threshold ? 1 : 0;
      const prev = values[y * width + x - 1] > threshold ? 1 : 0;
      if (curr !== prev) edges[y * width + x] = 1;
    }
  }

  // Sort segments per row (same logic as V6 but using precomputed values)
  for (let y = 0; y < height; y++) {
    let segStart = 0;
    for (let x = 0; x <= width; x++) {
      if (x === width || edges[y * width + x]) {
        if (x > segStart) {
          sortSegmentByValues(data, values, y, segStart, x, width);
        }
        segStart = x;
      }
    }
  }

  // Rotate back
  const result = rotateImageData(new ImageData(data, width, height), (4 - kRotations) % 4);
  return new ImageData(result.data, result.width, result.height);
}

// Sort pixels in a row segment by their precomputed values
function sortSegmentByValues(data, values, y, start, end, width) {
  const len = end - start;
  const indices = Array.from({ length: len }, (_, i) => start + i);
  indices.sort((a, b) => values[y * width + a] - values[y * width + b]);

  const temp = new Uint8ClampedArray(len * 4);
  for (let i = 0; i < len; i++) {
    const src = (y * width + indices[i]) * 4;
    temp.set(data.slice(src, src + 4), i * 4);
  }
  for (let i = 0; i < len; i++) {
    const dst = (y * width + (start + i)) * 4;
    data[dst]     = temp[i * 4];
    data[dst + 1] = temp[i * 4 + 1];
    data[dst + 2] = temp[i * 4 + 2];
    data[dst + 3] = temp[i * 4 + 3];
  }
}

// Rotate ImageData by k * 90 degrees
function rotateImageData(imageData, k) {
  k = ((k % 4) + 4) % 4;
  if (k === 0) return { data: new Uint8ClampedArray(imageData.data), width: imageData.width, height: imageData.height };

  const { width: w, height: h, data: src } = imageData;
  let newW, newH;
  if (k === 1 || k === 3) { newW = h; newH = w; }
  else { newW = w; newH = h; }

  const dst = new Uint8ClampedArray(newW * newH * 4);
  for (let y = 0; y < h; y++) {
    for (let x = 0; x < w; x++) {
      let nx, ny;
      if (k === 1)      { nx = h - 1 - y; ny = x; }
      else if (k === 2) { nx = w - 1 - x; ny = h - 1 - y; }
      else              { nx = y; ny = w - 1 - x; }
      const si = (y * w + x) * 4;
      const di = (ny * newW + nx) * 4;
      dst[di] = src[si]; dst[di+1] = src[si+1]; dst[di+2] = src[si+2]; dst[di+3] = src[si+3];
    }
  }
  return { data: dst, width: newW, height: newH };
}

// Main entry for V7
async function processPixelSortAdvanced(frames, { sortMode, threshold, rotation, multiPass, seed }, onProgress) {
  // Seed a simple PRNG (for consistency with original — not heavily used in sorting)
  const output = [];
  const modes = multiPass ? ['luminance', 'hue', 'saturation', 'laplacian'] : [sortMode];

  for (let i = 0; i < frames.length; i++) {
    let current = frames[i];
    for (const mode of modes) {
      current = applyPixelSorting(current, VALUE_FUNCTIONS[mode], threshold, rotation);
    }
    output.push(current);
    onProgress?.(i);
    if (i % 2 === 0) await yieldToUI();
  }
  return output;
}
```

---

#### 10. Mode 8 — Pixel Sort Masked (`effects/pixel-sort-masked.js`)

Port of `DjzDatamoshV8.py`. Identical to V7 but `process_row` checks the mask before sorting each segment. Refer to `DjzDatamoshV8.py` lines 105-138 for the mask-aware `process_row` and lines 140-186 for `apply_pixel_sorting` with mask parameter.

```javascript
// Same as V7's applyPixelSorting but accepts a mask (Uint8Array, 0 or 255 per pixel)
function applyPixelSortingMasked(imageData, mask, valueFn, threshold, rotation) {
  const kRotations = (((rotation / 90) % 4) + 4) % 4;
  let rotated = rotateImageData(imageData, kRotations);
  let rotatedMask = mask ? rotateMask(mask, imageData.width, imageData.height, kRotations) : null;

  const { data, width, height } = rotated;
  const values = valueFn(data, width, height);

  // Normalise
  let vMin = Infinity, vMax = -Infinity;
  for (let i = 0; i < values.length; i++) {
    if (values[i] < vMin) vMin = values[i];
    if (values[i] > vMax) vMax = values[i];
  }
  if (vMax > vMin) {
    for (let i = 0; i < values.length; i++) values[i] = (values[i] - vMin) / (vMax - vMin);
  }

  // Edges
  const edges = new Uint8Array(width * height);
  for (let y = 0; y < height; y++) {
    for (let x = 1; x < width; x++) {
      const curr = values[y * width + x] > threshold ? 1 : 0;
      const prev = values[y * width + x - 1] > threshold ? 1 : 0;
      if (curr !== prev) edges[y * width + x] = 1;
    }
  }

  // Sort segments, skipping those entirely outside the mask
  for (let y = 0; y < height; y++) {
    let segStart = 0;
    for (let x = 0; x <= width; x++) {
      if (x === width || edges[y * width + x]) {
        if (x > segStart) {
          // Check if any pixel in this segment is inside the mask
          if (!rotatedMask || segmentInMask(rotatedMask, y, segStart, x, width)) {
            sortSegmentByValues(data, values, y, segStart, x, width);
          }
        }
        segStart = x;
      }
    }
  }

  const result = rotateImageData(new ImageData(data, width, height), (4 - kRotations) % 4);
  return new ImageData(result.data, result.width, result.height);
}

function segmentInMask(mask, y, start, end, width) {
  for (let x = start; x < end; x++) {
    if (mask[y * width + x]) return true;
  }
  return false;
}

function rotateMask(mask, w, h, k) {
  // Reuse rotateImageData logic but for single-channel data
  k = ((k % 4) + 4) % 4;
  if (k === 0) return new Uint8Array(mask);
  let newW, newH;
  if (k === 1 || k === 3) { newW = h; newH = w; } else { newW = w; newH = h; }
  const dst = new Uint8Array(newW * newH);
  for (let y = 0; y < h; y++) {
    for (let x = 0; x < w; x++) {
      let nx, ny;
      if (k === 1)      { nx = h - 1 - y; ny = x; }
      else if (k === 2) { nx = w - 1 - x; ny = h - 1 - y; }
      else              { nx = y; ny = w - 1 - x; }
      dst[ny * newW + nx] = mask[y * w + x];
    }
  }
  return dst;
}

async function processPixelSortMasked(frames, { mask, sortMode, threshold, rotation, multiPass, seed }, onProgress) {
  const output = [];
  const modes = multiPass ? ['luminance', 'hue', 'saturation', 'laplacian'] : [sortMode];

  for (let i = 0; i < frames.length; i++) {
    let current = frames[i];
    for (const mode of modes) {
      current = applyPixelSortingMasked(current, mask, VALUE_FUNCTIONS[mode], threshold, rotation);
    }
    output.push(current);
    onProgress?.(i);
    if (i % 2 === 0) await yieldToUI();
  }
  return output;
}
```

---

#### 11. Mask Painter (`mask-painter.js`)

Provides a canvas overlay for painting the V8 mask.

```javascript
class MaskPainter {
  constructor(containerEl, width, height) {
    this.width = width;
    this.height = height;

    // Create overlay canvas positioned over the preview canvas
    this.canvas = document.createElement('canvas');
    this.canvas.width = width;
    this.canvas.height = height;
    this.canvas.style.cssText = 'position:absolute;top:0;left:0;cursor:crosshair;';
    containerEl.style.position = 'relative';
    containerEl.appendChild(this.canvas);

    this.ctx = this.canvas.getContext('2d');
    this.painting = false;
    this.erasing = false;
    this.brushSize = 20;

    // Mouse/touch events
    this.canvas.addEventListener('mousedown', e => this.startPaint(e));
    this.canvas.addEventListener('mousemove', e => this.paint(e));
    this.canvas.addEventListener('mouseup', () => this.painting = false);
    // Mirror for touch events
  }

  startPaint(e) {
    this.painting = true;
    this.paint(e);
  }

  paint(e) {
    if (!this.painting) return;
    const rect = this.canvas.getBoundingClientRect();
    const scaleX = this.width / rect.width;
    const scaleY = this.height / rect.height;
    const x = (e.clientX - rect.left) * scaleX;
    const y = (e.clientY - rect.top) * scaleY;

    this.ctx.globalCompositeOperation = this.erasing ? 'destination-out' : 'source-over';
    this.ctx.fillStyle = 'rgba(255, 0, 0, 0.4)'; // semi-transparent red for visual
    this.ctx.beginPath();
    this.ctx.arc(x, y, this.brushSize, 0, Math.PI * 2);
    this.ctx.fill();
  }

  getMask() {
    // Convert painted overlay to binary mask (Uint8Array: 0 or 1)
    const imgData = this.ctx.getImageData(0, 0, this.width, this.height);
    const mask = new Uint8Array(this.width * this.height);
    for (let i = 0, p = 0; i < mask.length; i++, p += 4) {
      mask[i] = imgData.data[p + 3] > 0 ? 1 : 0; // any non-zero alpha = masked
    }
    return mask;
  }

  clear() {
    this.ctx.clearRect(0, 0, this.width, this.height);
  }

  destroy() {
    this.canvas.remove();
  }
}
```

---

#### 12. ZIP Export (`export.js`)

```javascript
async function exportAsZip(frames, effectName, parameters, sourceFilename, fps, onProgress) {
  const zip = new JSZip();

  // Add manifest
  zip.file('manifest.json', JSON.stringify({
    app: 'DJZ-DATAMOSH',
    version: '1.0.0',
    effect: effectName,
    parameters,
    source: sourceFilename,
    frameCount: frames.length,
    width: frames[0]?.width,
    height: frames[0]?.height,
    fps,
    exportedAt: new Date().toISOString()
  }, null, 2));

  // Add frames as PNGs
  const canvas = document.createElement('canvas');
  if (frames.length > 0) {
    canvas.width = frames[0].width;
    canvas.height = frames[0].height;
  }
  const ctx = canvas.getContext('2d');

  for (let i = 0; i < frames.length; i++) {
    ctx.putImageData(frames[i], 0, 0);
    const blob = await new Promise(r => canvas.toBlob(r, 'image/png'));
    const arrayBuf = await blob.arrayBuffer();
    const padded = String(i + 1).padStart(4, '0');
    zip.file(`frame_${padded}.png`, arrayBuf);
    onProgress?.(i);
  }

  // Generate ZIP blob
  const zipBlob = await zip.generateAsync({ type: 'blob' });

  // Trigger download
  const timestamp = new Date().toISOString().replace(/[:.]/g, '').slice(0, 15);
  const filename = `datamosh_${effectName}_${timestamp}.zip`;
  const url = URL.createObjectURL(zipBlob);
  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  a.click();
  URL.revokeObjectURL(url);

  return zipBlob; // caller can optionally save to IndexedDB
}
```

---

#### 13. IndexedDB Wrapper (`db.js`)

Same pattern as `DJZ-CropReplacer` — refer to the CropReplacer plan's `db.js` section. Adapt store names:

```javascript
const DB_NAME    = 'DJZDatamosh';
const DB_VERSION = 1;
let _db = null;

const STORES = ['videoGallery', 'outputHistory', 'vectorSets'];

function openDB() {
  return new Promise((resolve, reject) => {
    if (_db) return resolve(_db);
    const req = indexedDB.open(DB_NAME, DB_VERSION);
    req.onupgradeneeded = (e) => {
      const db = e.target.result;
      for (const store of STORES) {
        if (!db.objectStoreNames.contains(store)) {
          db.createObjectStore(store, { keyPath: 'id' });
        }
      }
    };
    req.onsuccess = (e) => { _db = e.target.result; resolve(_db); };
    req.onerror = (e) => reject(e.target.error);
  });
}

async function dbPut(store, record) {
  const db = await openDB();
  return new Promise((resolve, reject) => {
    const tx = db.transaction(store, 'readwrite');
    tx.objectStore(store).put(record);
    tx.oncomplete = () => resolve();
    tx.onerror = (e) => reject(e.target.error);
  });
}

async function dbGet(store, id) {
  const db = await openDB();
  return new Promise((resolve, reject) => {
    const tx = db.transaction(store, 'readonly');
    const req = tx.objectStore(store).get(id);
    req.onsuccess = () => resolve(req.result);
    req.onerror = (e) => reject(e.target.error);
  });
}

async function dbGetAll(store) {
  const db = await openDB();
  return new Promise((resolve, reject) => {
    const tx = db.transaction(store, 'readonly');
    const req = tx.objectStore(store).getAll();
    req.onsuccess = () => resolve(req.result);
    req.onerror = (e) => reject(e.target.error);
  });
}

async function dbDelete(store, id) {
  const db = await openDB();
  return new Promise((resolve, reject) => {
    const tx = db.transaction(store, 'readwrite');
    tx.objectStore(store).delete(id);
    tx.oncomplete = () => resolve();
    tx.onerror = (e) => reject(e.target.error);
  });
}

async function dbClearStore(store) {
  const db = await openDB();
  return new Promise((resolve, reject) => {
    const tx = db.transaction(store, 'readwrite');
    tx.objectStore(store).clear();
    tx.oncomplete = () => resolve();
    tx.onerror = (e) => reject(e.target.error);
  });
}
```

---

#### 14. Settings (`settings.js`)

```javascript
const SETTINGS_KEY = 'djz_datamosh_settings';

function loadSettings() {
  try {
    const raw = localStorage.getItem(SETTINGS_KEY);
    if (raw) {
      const parsed = JSON.parse(raw);
      return {
        outputFormat:  parsed.outputFormat  || 'image/png',
        maxFrames:     parsed.maxFrames     ?? 300,
        processingRes: parsed.processingRes || 'original',
        frameRate:     parsed.frameRate     ?? 0
      };
    }
  } catch { /* ignore */ }
  return { ...DEFAULT_SETTINGS };
}

function saveSettings(settings) {
  localStorage.setItem(SETTINGS_KEY, JSON.stringify(settings));
}
```

---

#### 15. Frame Player (`frame-player.js`)

```javascript
class FramePlayer {
  constructor(canvas) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d');
    this.frames = [];
    this.currentIndex = 0;
    this.playing = false;
    this.fps = 30;
    this._rafId = null;
    this._lastTime = 0;
  }

  setFrames(frames, fps) {
    this.frames = frames;
    this.fps = fps || 30;
    this.currentIndex = 0;
    if (frames.length > 0) {
      this.canvas.width = frames[0].width;
      this.canvas.height = frames[0].height;
      this.renderFrame(0);
    }
  }

  renderFrame(index) {
    if (index >= 0 && index < this.frames.length) {
      this.currentIndex = index;
      this.ctx.putImageData(this.frames[index], 0, 0);
    }
  }

  play() {
    if (this.frames.length === 0) return;
    this.playing = true;
    this._lastTime = performance.now();
    this._tick();
  }

  pause() {
    this.playing = false;
    if (this._rafId) cancelAnimationFrame(this._rafId);
  }

  _tick() {
    if (!this.playing) return;
    this._rafId = requestAnimationFrame((now) => {
      const elapsed = now - this._lastTime;
      const interval = 1000 / this.fps;
      if (elapsed >= interval) {
        this._lastTime = now - (elapsed % interval);
        this.currentIndex = (this.currentIndex + 1) % this.frames.length;
        this.ctx.putImageData(this.frames[this.currentIndex], 0, 0);
        this.onFrameChange?.(this.currentIndex);
      }
      this._tick();
    });
  }

  seekTo(index) {
    this.renderFrame(Math.max(0, Math.min(index, this.frames.length - 1)));
  }

  stepForward() { this.seekTo(this.currentIndex + 1); }
  stepBackward() { this.seekTo(this.currentIndex - 1); }
  seekStart() { this.seekTo(0); }
  seekEnd() { this.seekTo(this.frames.length - 1); }
}
```

---

### Style Guide

- **Colour palette:** Dark theme — `#0a0a0a` background, `#1a1a2e` panel backgrounds, `#16213e` accent panels, `#e94560` primary accent (buttons, active states), `#f5f5f5` text, `#888` secondary text.
- **Font:** System monospace stack: `'SF Mono', 'Fira Code', 'Cascadia Code', monospace`.
- **Layout:** Single-column, max-width 1200px centred. Canvas preview fills available width with `max-width: 100%; height: auto`.
- **Controls:** Sliders use native `<input type="range">` styled with CSS to match theme. Dropdowns use native `<select>` styled. Toggles use `<input type="checkbox">` styled as pill switches.
- **Progress bar:** A `<div>` with CSS width transition, `#e94560` fill on `#1a1a2e` track.
- **Gallery strip:** Horizontal scrollable row of 80px square thumbnails with 4px border-radius. Active item has `#e94560` border.
- **Modal:** Overlay with `rgba(0,0,0,0.7)` backdrop, `#1a1a2e` modal body, `border-radius: 12px`.
- **Buttons:** Rounded (`border-radius: 8px`), `#e94560` primary, `#2a2a3e` secondary. Disabled buttons at 40% opacity.
- **Responsive:** Below 768px width, effect panel stacks below the preview. Gallery strip scrolls horizontally with touch.

---

### Performance Considerations

1. **Processing resolution** — the Settings "720p" / "480p" option downscales frames before processing, dramatically reducing computation time (4x fewer pixels at 720p vs 1080p).

2. **Yielding to UI** — all effect processors call `await yieldToUI()` every few frames to prevent the browser from becoming unresponsive. This is a `setTimeout(0)` promise.

3. **Memory management** — at 1080p, 300 frames of RGBA `ImageData` requires ~1.5 GB. The `maxFrames` setting and resolution downscaling are the primary memory controls. After processing completes and the user exports, the original frames can be released if memory is tight (re-extractable from the stored video blob).

4. **Block matching optimisation** — V1/V2/V4's `findShiftsFast` is O(blocks * shift_values^2) per frame. The `shift_range` parameter controls the step size of the search — higher values skip more candidates. For very large frames, consider processing blocks in parallel via Web Workers (optional enhancement).

5. **Pixel sorting** — V6/V7/V8 process each frame independently. Each row's segments are sorted in O(n log n). Total: O(width * height * log(width)) per frame.

6. **JSZip** — ZIP generation is async and chunked. For 300 PNG frames, expect 5-15 seconds. Progress is reported per-frame.

---

### Accessibility

- All controls have associated `<label>` elements.
- Slider values display numerically next to the slider.
- Focus order follows visual layout: video controls → gallery → effect panel → action buttons.
- Keyboard: `Space` = play/pause, `←`/`→` = step frame, `Home`/`End` = first/last frame.
- High-contrast text on dark background meets WCAG AA (4.5:1 minimum).
- All interactive elements have `:focus-visible` outlines.
- Progress and status messages use `aria-live="polite"` regions.
