# ADAS Neural Perception Platform | Autonomous Telemetry Cockpit

**Author:** Void / Mitadru Karmakar

---

## 🚀 Overview
The **ADAS Neural Perception Platform** is an enterprise-grade, high-throughput autonomous driving assistance system. It combines real-time computer vision object detection (YOLOv8/TensorRT), spatial kinematic triangulation with anti-flicker smoothing, a dynamic ego-corridor risk arbitrator, a top-down phosphor radar HUD, and zero-buffer browser-based client camera ingestion supporting local PC webcams and mobile phone dashcams.

---

## 1. Model Training & Pipeline Workflow (Kaggle Notebook)
* **Execution Environment:** Cloud-based Kaggle Notebooks utilizing high-performance GPU hardware accelerators for efficient custom dataset training.
* **Perception Scope:** Configured Ultralytics YOLO to detect unstructured road objects including cars, two-wheelers, auto-rickshaws, pedestrians, stray animals, potholes, and speed bumps.
* **Artifact Generation:** Final trained weights compiled and exported as `best.pt` for local workspace deployment. *(See our detailed [Model Weights & Large Files Guide](MODELS.md) for external download links and workspace setup instructions).*

---

## 2. Model Conversion Pipeline (`.pt` -> `.onnx` -> `.engine`)
* **PyTorch Training Source (`best.pt`):**
  * Format: Native PyTorch weights file (Downloadable via [MODELS.md](MODELS.md)).
  * Usage: Primary baseline model trained on custom road dataset.
* **PyTorch to ONNX Export (`export_onnx.py`):**
  * Converts `.pt` to portable Open Neural Network Exchange format.
  * Python Script Code:
    ```python
    import os
    from ultralytics import YOLO
    BASE_DIR = os.path.dirname(os.path.abspath(__file__))
    model = YOLO(os.path.join(BASE_DIR, "best.pt"), task="detect")
    model.export(format="onnx", imgsz=640, simplify=True, opset=12)
    ```
  * CLI Alternative: `yolo export model=best.pt format=onnx imgsz=640 simplify=True`
* **ONNX to TensorRT Engine Compilation (`convert.py`):**
  * Compiles ONNX into optimized NVIDIA TensorRT engine using FP16 half-precision and hardware Tensor Core acceleration[cite: 1].
  * Python Script Code:
    ```python
    import os
    from ultralytics import YOLO
    BASE_DIR = os.path.dirname(os.path.abspath(__file__))
    model = YOLO(os.path.join(BASE_DIR, "best.onnx"), task="detect")
    model.export(format="engine", device=0, half=True, imgsz=640, workspace=4)
    ```

---

## 3. Backend & Frontend Architecture (VS Code Workspace)

### 🖥️ Backend (`app.py`)
* **FastAPI & Uvicorn Server:** Implements adaptive model binding, spatial depth/kinematic triangulation via `SensitiveADASPerceptionEngine`, ego-vehicle dashboard masking (`ny2 >= 0.93`), duplicate filtering, and real-time WebSocket video/telemetry streaming.
* **Adaptive Model Binding:** Automatically prioritizes compiled TensorRT GPU engine (`best.engine`), falling back to CUDA PyTorch or CPU runtimes.
* **SensitiveADASPerceptionEngine:** 
  * Computes real-world spatial depth ($z$) and lateral position ($x$) using camera focal length calibration and bounding box height triangulation.
  * Implements **Exponential Moving Average (EMA) box smoothing** and **5-frame track coasting** to eliminate bounding box jitter, tracking drops, and visual flickering.
  * Uses dual-threshold hysteresis to lock hazard classification tiers (Critical, Caution, Normal) and calculates Time-To-Collision (TTC).
* **Universal Client Camera Ingestion (`/ws/camera/{session_id}`):** Non-blocking asynchronous frame decoders running at uninhibited throughput, allowing remote or local client webcams and mobile phone cameras to stream directly to the GPU engine without server-side camera locks.

### 🌐 Frontend (`templates/index.html`)
* **Tailwind CSS Cockpit Dashboard:** High-contrast tactical HUD layout optimized for vehicular telemetry.
* **Hardened Hardware Isolation:** Automatically detects device capability (`maxTouchPoints` / `pointer: coarse`) and completely isolates hardware buttons:
  * *Desktop/Laptop:* PC webcam controls are rendered; mobile options are permanently purged from the DOM.
  * *Mobile Handheld:* Mobile dashcam controls are rendered; PC webcam options are permanently purged from the DOM.
* **High-Speed Client Streaming Loop:** Uses an asynchronous `requestAnimationFrame` and backpressure-guarded `toBlob` pipeline to stream mobile and PC camera feeds at maximum frame rates.
* **Dynamic HUD & Radar Map:** Renders tactical framing brackets, boresight crosshairs, threat-tier color coding (Red = Critical/Braking, Amber = Caution, Cyan = Normal/Far), and a live $360^\circ$ rotating phosphor radar map.
* **Hidden Easter Egg Warp Portal:** Clicking the `VOID WEBSOCKET STREAM` header badge **10 times** triggers a synthetic space-warp sound effect and reveals the `🚀 WARP_CORE_PORTAL` button. The portal auto-reverts back to normal after 10 seconds. Clicking the portal button opens the spatial black hole simulation in a new tab (see [`SECRET.md`](SECRET.md)).

---

## 4. Local Automation & Startup (`run.bat`)
Automated batch script to run the FastAPI backend and Cloudflare Quick Tunnel concurrently:

```cmd
@echo off
TITLE ADAS Neural Perception Platform

:: 1. Start FastAPI backend server
start "ADAS Backend" cmd /k "python app.py"

:: 2. Wait 3 seconds for Uvicorn to initialize on port 8000
timeout /t 3 /nobreak > nul

:: 3. Start Cloudflare Quick Tunnel from parent folder
start "Cloudflare Tunnel" cmd /k "..\cloudflared.exe tunnel --url http://localhost:8000"

echo ADAS Platform and Cloudflare Tunnel are online!
pause
