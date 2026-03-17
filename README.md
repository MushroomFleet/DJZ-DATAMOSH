# DJZ-DATAMOSH

A desktop video glitch effects tool with 8 datamosh and pixel-sorting modes. Load any MP4 or WebM video, apply effects, preview in real-time, and export as a ZIP containing both a WebM video and numbered PNG frame sequence.

Built with Tauri (Rust + WebView2) for lightweight, portable Windows distribution. No Python, no FFmpeg, no server required.

![Platform](https://img.shields.io/badge/platform-Windows%20x64-blue)
![Version](https://img.shields.io/badge/version-1.2.0-green)
![License](https://img.shields.io/badge/license-MIT-yellow)

---

## Download

Grab the latest release from the [Releases](https://github.com/MushroomFleet/DJZ-DATAMOSH/releases) page:

- **`DJZ-DATAMOSH_x64-setup.exe`** — NSIS installer (recommended)
- **`DJZ-DATAMOSH_x64_en-US.msi`** — MSI installer for managed deployments

No dependencies to install. Just run the installer and launch.

---

## The 8 Effect Modes

| # | Mode | Description |
|---|------|-------------|
| 1 | **Glide** | Block-based motion estimation + displacement. Motion smears and bleeds across frames. |
| 2 | **Multi-Mosh** | Three sub-modes: `glide` (propagate motion from first 2 frames), `movement` (frame-by-frame tracking), `copy` (passthrough). |
| 3 | **I-Frame Kill** | Simulates classic datamosh via `iframe_removal` (motion bleeds across scenes) or `delta_repeat` (stuttering motion loops). |
| 4 | **Vector Transfer** | Extracts motion vectors from a source video and applies them to a target video — motion-based style transfer. |
| 5 | **Size Sort** | Re-orders frames by their compressed file size within a selectable range. |
| 6 | **Pixel Sort (Sobel)** | Sobel edge detection defines segments; pixels within each segment are sorted by luminance. |
| 7 | **Pixel Sort (Advanced)** | Multi-mode pixel sorting (luminance / hue / saturation / laplacian) with rotation and multi-pass options. |
| 8 | **Pixel Sort (Masked)** | Same as Advanced but with mask support — paint a mask manually, generate from luminance threshold, or use MODNet AI for automatic portrait segmentation. |

All effects are JavaScript ports of the [DJZ Datamosh ComfyUI custom nodes](https://github.com/MushroomFleet/DJZ-Nodes) (V1–V8).

---

## How to Use

### 1. Load a Video
Click the **+** button in the Video Gallery strip or drag-and-drop an MP4/WebM file onto the app. Frames are extracted automatically with a progress overlay showing the frame count in real-time.

### 2. Select an Effect
Choose one of the 8 modes from the **Effect** dropdown. Each mode reveals its own parameter controls (sliders, toggles, dropdowns) matching the original ComfyUI node parameters.

### 3. Adjust Parameters
Tweak block size, shift range, thresholds, rotation, and other mode-specific settings. Parameters use the same defaults and ranges as the original Python nodes.

### 4. Process
Click **Process**. A progress bar tracks each frame. Processing yields to the UI so the app stays responsive. Click **Cancel** at any time to abort.

### 5. Preview
The processed result plays back in the preview canvas at the original video's frame rate. Use play/pause, frame stepping, scrub bar, and the Original/Processed toggle to compare.

### 6. Export
Click **Export ZIP** to download a ZIP containing:
- `datamosh_<effect>.webm` — the assembled output video
- `frames/frame_0001.png` through `frame_NNNN.png` — the full frame sequence
- `manifest.json` — effect name, parameters, resolution, FPS, timestamp

Exports are automatically saved to the **Output History** panel for later re-download.

### Special Features

- **MODNet AI Masking** (Mode 8) — Click "Load MODNet Model" to download a ~26 MB ONNX model (cached in IndexedDB for future sessions). Generates automatic portrait segmentation masks for targeted pixel sorting.
- **Vector Transfer** (Mode 4) — Load a second "source" video to extract motion vectors from. Vectors can be saved and reused across sessions.
- **Debug Console** — Press `Ctrl+F9` to toggle a detailed on-screen log for troubleshooting.
- **Keyboard Shortcuts** — `Space` (play/pause), `Left/Right` (step frames), `Home/End` (first/last frame).

---

## Build Your Own

This project uses the **TINS** (There Is No Source) methodology — comprehensive specification documents that any capable LLM can implement from scratch.

### TINS Plans Included

- **`DJZ-DataMosh-TINS-plan.md`** — Full specification for the 8-effect datamosh application: UI layout, data models, all algorithms with complete JavaScript code, IndexedDB storage, ZIP export, and accessibility requirements.

---

## Origin

The 8 effects are JavaScript ports of the **DJZ Datamosh** series of ComfyUI custom nodes (V1–V8), originally written in Python with PyTorch/NumPy/SciPy. The web ports use Canvas 2D API and ImageData for all pixel processing, with no external dependencies beyond JSZip and ONNX Runtime Web (optional, for MODNet).

---

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{djz_datamosh,
  title = {DJZ-DATAMOSH: Browser-Based Video Glitch Effects Tool},
  author = {Drift Johnson},
  year = {2025},
  url = {https://github.com/MushroomFleet/DJZ-DATAMOSH-dev},
  version = {1.2.0}
}
```

### Donate:

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
