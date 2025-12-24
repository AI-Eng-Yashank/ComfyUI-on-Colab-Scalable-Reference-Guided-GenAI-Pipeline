#  ComfyUI-on-Colab: Scalable Reference-Guided GenAI Pipeline

This repository contains a **Google Colab notebook** that provisions a production-ready ComfyUI environment optimized for **IP-Adapter** workflows. It enables high-quality, reference-guided image generation by leveraging Google Colab's GPU resources while persisting your models and workflows to your personal Google Drive.

![ComfyUI](https://img.shields.io/badge/ComfyUI-0.5.1-blue)
![Python](https://img.shields.io/badge/Python-3.12-green)
![License](https://img.shields.io/badge/License-MIT-yellow)

## Features

*   **Persistent Storage:** Mounts Google Drive to save models and workflows, preventing data loss after Colab sessions end.
*   **IP-Adapter Ready:** Pre-configured to handle IP-Adapter workflows, including automated download of CLIP Vision models and weights.
*   **Remote Access:** Utilizes **Cloudflared** to create a secure tunnel, allowing you to interact with the ComfyUI interface from your local browser.
*   **Environment Management:** One-click installation of dependencies including `xformers` for VRAM optimization.
*   **Directory Management:** Automated directory creation and verification for checkpoints, clip_vision, and IPAdapter models.

## Tech Stack

*   **Backend:** Python 3.12, PyTorch, ComfyUI
*   **Optimization:** xformers, CUDA 12.x
*   **Tunneling:** Cloudflared
*   **Models:** Stable Diffusion 1.5, IP-Adapter SD1.5

## Prerequisites

1.  A **Google Account** to use Google Colab.
2.  **Google Drive** storage space (approx. 10GB recommended for models).

##  How to Use

### Open in Colab
Click the "Open in Colab" badge below (or copy the notebook content into a new Colab file).

[![Open In Colab]([https://colab.research.google.com/drive/1nI4Qev6IiNERKvkvh7bwqbMMIk3RHL-4?usp=sharing](https://colab.research.google.com/drive/1nI4Qev6IiNERKvkvh7bwqbMMIk3RHL-4#scrollTo=3DioeCN34A79))

### Run Setup Cells
1.  **Environment Setup:** Run the first cell to install ComfyUI and dependencies. (Uncheck update options on subsequent runs to save time).
2.  **Download Models:** Run the cells to download `v1-5-pruned-emaonly.safetensors`, `CLIP Vision`, and `IP-Adapter` weights.
3.  **Launch Server:** Run the final cell to start the ComfyUI server and generate the Cloudflared URL.

### Access ComfyUI
Once the server starts, a URL ending in `.trycloudflare.com` will appear. Click it to open the ComfyUI interface.

## Project Structure
The notebook sets up the following directory structure in your Google Drive:
```text
/content/drive/MyDrive/ComfyUI/
├── models/
│   ├── checkpoints/       # Base SD model (v1-5)
│   ├── clip_vision/      # CLIP Vision model (required for IP-Adapter)
│   ├── ipadapter/        # IP-Adapter weights
│   └── ...
├── custom_nodes/         # ComfyUI-Manager
└── ...
```

## Troubleshooting

*   **ClipVision model not found:** Ensure Cell 4 (CLIP Vision download) has run successfully.
*   **Connection Timed Out:** Restart the Colab runtime (Runtime -> Restart session) and run the cells again.
*   **Low VRAM:** The notebook uses `xformers` to minimize VRAM usage. If you still face OOM (Out of Memory) errors, try reducing the resolution in your workflow (e.g., 512x512).

##  License
This project is licensed under the MIT License.

##  Author
**Yashank Krishna Mhatre**
*AI Engineer | GenAI | Voice AI*

---

### **Code Explanation & Breakdown**

This section details the logic behind each cell in the notebook, explaining the engineering decisions made to ensure stability and performance.

#### **Environment Setup**
This is the foundation of the pipeline. It handles the initial setup and subsequent runs efficiently.
*   **Google Drive Mounting:** Checks if `USE_GOOGLE_DRIVE` is True. If so, it mounts `/content/drive/` to allow the ComfyUI environment to persist data.
*   **Path Management:** Defines the `WORKSPACE` variable pointing to `/content/drive/MyDrive/ComfyUI`.
*   **Dependency Installation:**
    *   Installs `xformers!=0.0.18`, which is critical for memory-efficient attention mechanisms on GPUs (Tesla T4/Colab).
    *   Uses `--extra-index-url` to fetch PyTorch wheels compatible with CUDA 12.x.
*   **ComfyUI Manager:** Clones the ComfyUI-Manager extension into `custom_nodes/`. It also fixes Linux permissions (`chmod 755`) for shell scripts within the manager, which often fail due to Drive permission quirks.
*   **Smart Flags:** `UPDATE_COMFY_UI` and `USE_COMFYUI_MANAGER` are set to `True` by default. The user is instructed to uncheck these on future runs to avoid reinstalling dependencies every time, speeding up the workflow significantly.

#### **Directory Structure Validator**
This is a utility cell for debugging.
*   **File Tree Generation:** Uses Python's `os.walk` to recursively print the file structure of the `WORKSPACE`.
*   **Why?** In Colab, it can be difficult to know exactly where your files are or if specific folders were created. This cell allows the user to visually verify that `models/checkpoints` and `models/clip_vision` folders actually exist before attempting downloads.

#### **Download Base Checkpoint**
*   **Objective:** Downloads the Stable Diffusion 1.5 base model.
*   **Command:** Uses `wget -c` (resume capability) to download `v1-5-pruned-emaonly.safetensors`.
*   **Path:** Saves to `/content/drive/MyDrive/ComfyUI/models/checkpoints/`. This is the standard path ComfyUI scans for base models.

#### **Download CLIP Vision Model**
*   **Critical Fix:** This cell solves the `ClipVision model not found` error.
*   **Mechanism:** IP-Adapter requires a specific "Image Encoder" model to "see" the reference image. This is separate from the main checkpoint.
*   **Execution:** Creates the `models/clip_vision` directory (if missing) and downloads `CLIP-ViT-H-14-laion2B-s32B-b79K.safetensors`.

#### **Verify Checkpoint Integrity**
*   **Sanity Check:** Uses Python `os.path.getsize` to check the downloaded checkpoint size.
*   **Validation:** If the file size is less than 3.5GB, it prints a warning. A typical corrupted download or HTML 404 error file is often only a few KB. This prevents the user from trying to run the workflow with a broken file.

#### **Download IP-Adapter Weights**
*   **Objective:** Downloads the actual IP-Adapter weights (`ip-adapter_sd15.safetensors`).
*   **Path:** Saves to `models/ipadapter`. These files contain the actual neural network layers that inject the style into the UNet.
*   **Logic:** Ensures the specific weights for SD1.5 are downloaded, as SDXL or Flux requires different versions.

#### **Comprehensive Model Verification**
*   **Dual Check:** Verifies *both* the CLIP Vision model (Cell 4) and the IPAdapter Weights (Cell 6).
*   **Feedback:** Prints `FOUND` or `MISSING` status for each. This acts as a final "Green Light" check before launching the server.

#### **Launch Server & Cloudflared Tunnel**
This cell makes the local server accessible globally.
*   **Cloudflared Install:** Downloads and installs the `cloudflared` `.deb` package, a tool for creating secure tunnels.
*   **Threading:**
    *   Python's `threading` module is used to run the tunneling process in parallel with the server.
    *   **`iframe_thread` function:** Waits for ComfyUI to start listening on port `8188`. Once active, it launches `cloudflared tunnel --url http://127.0.0.1:8188`.
*   **URL Extraction:** The script watches the output stream of `cloudflared` (STDERR) for the phrase `trycloudflare.com`. When found, it prints the URL.
*   **Server Launch:** Finally, it runs `!python main.py --dont-print-server` to start the ComfyUI backend.
*   **Result:** The user receives a public URL (e.g., `https://random-name.trycloudflare.com`) which tunnels directly to the Colab GPU, allowing them to use the drag-and-drop UI on their local computer.
