# ADAS Neural Perception Platform

**Author:** Void / Mitadru Karmakar

---

## 1. Model Training & Pipeline Workflow (Kaggle Notebook)
* **Execution Environment:** Cloud-based Kaggle Notebooks utilizing high-performance GPU hardware accelerators for efficient custom dataset training.
* **Perception Scope:** Configured Ultralytics YOLO to detect unstructured road objects including cars, two-wheelers, auto-rickshaws, pedestrians, stray animals, potholes, and speed bumps.
* **Artifact Generation:** Final trained weights compiled and exported as `best.pt` for local workspace deployment.

---

## 2. Model Conversion Pipeline (`.pt` -> `.onnx` -> `.engine`)
* **PyTorch Training Source (`best.pt`):**
  * Format: Native PyTorch weights file.
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
* **Backend (`app.py`):** FastAPI and Uvicorn server implementing adaptive model binding, spatial depth/kinematic triangulation via `SensitiveADASPerceptionEngine`, ego-vehicle dashboard masking (`ny2 >= 0.93`), duplicate filtering, and real-time WebSocket video/telemetry streaming at 10-19 FPS.
  * **Adaptive Model Binding:** Automatically prioritizes compiled TensorRT GPU engine (`best.engine`), falling back to CUDA PyTorch or CPU runtimes.
  * **SensitiveADASPerceptionEngine:** Computes real-world spatial depth ($z$) and lateral position ($x$) using camera focal length calibration and bounding box height triangulation, tracks objects across frames with velocity vectors, calculates Time-To-Collision (TTC), and evaluates threat levels into Critical, Caution, or Normal tiers based on ego-corridor proximity.
  * **Ego-Vehicle & Duplicate Filtering:** `is_ego_vehicle_part` masks out the vehicle's dashboard hood at the bottom edge (`ny2 >= 0.93`) to prevent false self-wiping of traffic, and `deduplicate_detections` suppresses overlapping bounding boxes and nested misclassifications.
  * **WebSocket Stream Pipeline (`/ws/live/{job_id}`):** Decodes video frames frame-by-frame, runs strided inference, compresses via Turbo JPEG, and broadcasts video + JSON telemetry at 10-19 FPS.
* **Frontend (`templates/index.html`):** Tailwind CSS dashboard managing file uploads (`/upload_live`) to trigger websocket streaming, WebSocket client communication, base64 JPEG frame rendering, and dynamic threat-tier HUD visualization.
  * **Tailwind CSS Dashboard:** Modern, high-contrast HUD layout optimized for vehicular telemetry.
  * **WebSocket Client & Video Renderer:** Connects to backend websocket, receives base64-encoded JPEG frames, and renders them onto the UI canvas.
  * **Dynamic HUD & Radar Map:** Draws bounding box brackets color-coded by threat severity (Red = Critical, Amber = Caution, Cyan = Normal) alongside a live 2D radar blip map.

---

## 4. Local Automation & Startup (`run.bat`)
Automated batch script to run FastAPI backend and Cloudflare Quick Tunnel concurrently:

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
```

Check out our detailed [Model Weights & Large Files Guide](MODELS.md) for download links.
