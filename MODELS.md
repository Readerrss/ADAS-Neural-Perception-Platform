# Model Weights & Large Files

Due to GitHub's strict file size limits, large model artifacts, raw training weights, and compiled runtime engines (such as `best.engine`) are hosted externally.

You can access and download all required large files from the official Google Drive repository:

👉 **[Download Large Files & Models (Google Drive)](https://drive.google.com/drive/folders/1z-5atlzDQsm5Sv25egvtrrqg3YRIArFQ)**

---

### Setup Instructions

1. **Download Artifacts:** Open the Google Drive link above and download the necessary model weights or compiled engines (`best.pt`, `best.onnx`, or `best.engine`).
2. **Place in Project:** Move the downloaded files directly into your local workspace inside the **`adas-web-system/`** folder.
3. **Verify Structure:** Ensure your folder structure looks like this before launching:
   ```text
   adas-web-system/
   ├── app.py
   ├── best.engine  <-- (Place downloaded engine here)
   ├── run.bat
   ├── classes.json
   ├── requirments.txt
   ├── conver.py
   ├── uploads
   ├── processed
   └── templates/
       └── index.html
