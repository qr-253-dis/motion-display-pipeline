# Automated Motion-Sensing Photo & Video Frame

An automated media pipeline, mobile ingestion shortcut, and web application that turns an iPad into an intelligent, motion-aware digital media frame.

---

## Overview

This project transforms an iPad into a continuous, motion-sensing video frame. Videos uploaded directly from an iPhone via the iOS Share Sheet are automatically ingested, transcoded to web-compatible SDR formats, and hot-reloaded on the iPad display without interrupting playback or requiring manual user interaction.

---

## Architecture & Design Decisions

### Why Cloudinary? (Media Ingestion & Staging)
- **Bypassing Git & API File Limits**: GitHub enforces a 100 MB file size limit and imposes strict HTTP payload limits on raw API commits. Mobile 4K HDR video captures frequently exceed these constraints.
- **Offloading Raw Payloads**: iPhone video clips upload directly to Cloudinary’s high-throughput API. The repository only receives a tiny JSON pointer file, keeping Git commit history lightweight and avoiding binary bloat in source control.

### Why GitHub Pages & Automated Pipelines? (Production Hosting)
- **Always-On Edge Delivery**: GitHub Pages provides zero-cost, continuous static site hosting with built-in SSL/HTTPS support—satisfying iPadOS WebRTC camera security requirements for motion detection.
- **Automated CI/CD Integration**: GitHub Actions workflows process incoming media payloads, execute FFmpeg color-space conversions, and deploy web assets directly to the live GitHub Pages environment.

---

## Architecture Workflow

[ iOS Device ]
│ (Share Sheet via iOS Shortcut)
▼
[ Cloudinary API ] ──(Upload Raw Media)──► [ Staging Storage ]
│                                           │
│ (Create Pointer File in /incoming)        │ (Fetch Source Media)
▼                                           ▼
[ GitHub Repository ] ─────────────────► [ GitHub Actions Pipeline ]
│
├─► FFmpeg HDR->SDR Tone Mapping (Hable)
├─► Rolling Storage Rotation (slot1–3)
└─► Deploy Static Web App to Hosting
│
▼
[ iPad Web App Frontend ]
├─► Motion Detection (WebRTC)
└─► Hot-Reloading (HTTP Last-Modified)


### 1. Mobile Ingestion (iOS Shortcut)
- Triggered natively from the iOS Share Sheet when selecting any video.
- Uploads raw video payload directly to **Cloudinary** via REST API.
- Generates a Base64-encoded pointer JSON object containing metadata and the Cloudinary source URL.
- Commits a timestamped pointer file into the repository's `/incoming` directory using the **GitHub Contents API**.

### 2. Automated Transcoding Pipeline (GitHub Actions)
- Triggered automatically on push events targeting `/incoming/*.json`.
- Downloads the source video asset from Cloudinary staging storage.
- Performs hardware-neutral color profile transcoding via **FFmpeg**:
  - Converts **BT.2020 (HDR10/HLG)** color spaces down to **BT.709 (SDR)** using the **Hable tone-mapping algorithm** to prevent color washout on web browsers.
  - Normalizes pixel formats to `yuv420p` at H.264 / AAC MP4 for maximum iOS Safari compatibility.
- Rotates processed media across a rolling 3-slot storage scheme (`yourvideo.mp4`, `yourvideo2.mp4`, `yourvideo3.mp4`).

### 3. Automated Storage Maintenance
- Scheduled via a weekly **GitHub Actions Cron Job**.
- Interrogates the Cloudinary Admin API to find and purge staging assets older than 3 days, maintaining storage limits automatically.

### 4. Motion-Aware Web Frontend (iPad)
- **Motion Detection**: Employs an in-browser WebRTC `getUserMedia` video feed sampled onto an HTML5 Canvas to calculate pixel-difference delta thresholds, switching the display between active playback and sleep states based on physical presence.
- **Hot Reloading**: A background polling loop issues HTTP `HEAD` requests to track `Last-Modified` resource headers, seamlessly updating media sources without disrupting current playback or requiring screen refreshes.

---

## iOS Shortcut Setup & Workflow Steps

The iOS Shortcut acts as the entry point for media ingestion. It accepts video files from the native iOS Share Sheet and executes the following sequence:

[ Share Sheet Input (Video) ]
│
▼
[ Action 1: Upload to Cloudinary ] ──► POST https://api.cloudinary.com/v1_1/{cloud_name}/video/upload
│                         (Includes upload_preset)
▼
[ Action 2: Extract Media URL ]    ──► Parse secure_url from Cloudinary JSON response
│
▼
[ Action 3: Construct Payload ]    ──► Format JSON: { "url": "{secure_url}", "timestamp": "{Current Date}" }
│
▼
[ Action 4: Encode Payload ]       ──► Base64 encode JSON payload for GitHub API compliance
│
▼
[ Action 5: Commit to GitHub ]     ──► PUT https://api.github.com/repos/qr-253-dis/motion-display-pipeline/contents/incoming/{ISO_Timestamp}.json
(Headers: Authorization: Bearer {GITHUB_PAT})


### Detailed Shortcut Configuration
1. **Receive Input**: Enable **"Show in Share Sheet"** and restrict input types to **Media / Videos**.
2. **Cloudinary Upload (`POST`)**:
   - **URL**: `https://api.cloudinary.com/v1_1/<YOUR_CLOUD_NAME>/video/upload`
   - **Method**: `POST`
   - **Request Body (Form)**:
     - `file`: `Shortcut Input`
     - `upload_preset`: `<YOUR_UPLOAD_PRESET>`
3. **Parse Cloudinary Response**:
   - Use **Get Dictionary Value** to extract `secure_url` from the returned JSON.
4. **Construct Pointer JSON**:
   - Text block:
     ```json
     {
       "source_url": "Dictionary Value (secure_url)",
       "created_at": "Current Date (ISO 8601 format)"
     }
     ```
5. **Base64 Encode**:
   - Pass the text payload through **Base64 Encode** (required by GitHub API for creating binary/text content).
6. **Commit File to GitHub (`PUT`)**:
   - **URL**: `https://api.github.com/repos/qr-253-dis/motion-display-pipeline/contents/incoming/upload_<Current Date format:yyyyMMdd_HHmmss>.json`
   - **Method**: `PUT`
   - **Headers**:
     - `Authorization`: `Bearer <YOUR_GITHUB_PAT>`
     - `Accept`: `application/vnd.github.v3+json`
   - **Request Body (JSON)**:
     - `message`: `Ingest video via iOS Shortcut`
     - `content`: `Base64 Encoded Text`

---

## Tech Stack

| Domain | Technologies |
| :--- | :--- |
| **Ingestion & Automation** | iOS Shortcuts, Cloudinary REST API, GitHub Contents API |
| **CI/CD & Processing** | GitHub Actions, Bash, FFmpeg (`zscale`, `tonemap=hable`), cURL, `jq` |
| **Frontend Platform** | HTML5 Video, Vanilla JavaScript (ES6+), WebRTC MediaDevices API, HTML5 Canvas API |
| **Hosting & Deployment** | GitHub Pages |

---

## Directory Structure

```directory
.
├── .github/
│   └── workflows/
│       ├── process-incoming.yml    # Transcoding & deployment pipeline
│       └── cleanup-staging.yml     # Cloudinary purge cron job
├── incoming/                       # Incoming pointer JSON files (.gitkeep tracked)
├── index.html                      # Frontend application entry point
├── README.md                       # Architectural documentation
├── robots.txt                      # Crawler instructions
├── slot.txt                        # Rolling rotation tracking state
├── yourvideo.mp4                   # Active media slot 1
├── yourvideo2.mp4                  # Active media slot 2
└── yourvideo3.mp4                  # Active media slot 3
