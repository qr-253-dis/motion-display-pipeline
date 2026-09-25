# Automated Motion-Sensing Photo & Video Frame

An automated media pipeline, mobile ingestion shortcut, and web application that turns an iPad into an intelligent, motion-aware digital media frame.

---

## Overview

This project transforms an old or repurposed iPad into a continuous, motion-sensing video frame. Videos uploaded directly from an iPhone via iOS Share Sheet are automatically ingested, transcoded to web-compatible SDR formats, and hot-reloaded on the iPad display without interrupting playback or requiring manual user interaction.

---

## Architecture & Workflow

```
[ iOS Device ]
      │ (Share Sheet via iOS Shortcut)
      ▼
[ Cloudinary API ] ──(Upload Raw Media)──► [ Staging Storage ]
      │                                          │
      │ (Create Pointer File in /incoming)       │ (Fetch Source Media)
      ▼                                          ▼
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
```

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
- Rotates processed media across a rolling 3-slot storage scheme (`video1.mp4`, `video2.mp4`, `video3.mp4`).

### 3. Automated Storage Maintenance
- Scheduled via a weekly **GitHub Actions Cron Job**.
- Interrogates the Cloudinary Admin API to find and purge staging assets older than 3 days, maintaining storage limits automatically.

### 4. Motion-Aware Web Frontend (iPad)
- **Motion Detection**: Employs an in-browser WebRTC `getUserMedia` video feed sampled onto an HTML5 Canvas to calculate pixel-difference delta thresholds, switching the display between active playback and sleep states based on physical presence.
- **Hot Reloading**: A background polling loop issues HTTP `HEAD` requests to track `Last-Modified` resource headers, seamlessly updating media sources without disrupting current playback or requiring screen refreshes.

---

## Tech Stack

| Domain | Technologies |
| :--- | :--- |
| **Ingestion & Automation** | iOS Shortcuts, Cloudinary REST API, GitHub Contents API |
| **CI/CD & Processing** | GitHub Actions, Bash, FFmpeg (`zscale`, `tonemap=hable`), cURL, `jq` |
| **Frontend Platform** | HTML5 Video, Vanilla JavaScript (ES6+), WebRTC MediaDevices API, HTML5 Canvas API |
| **Hosting & Deployment** | GitHub Pages / Render Static Site Hosting |

---

## Directory Structure

```directory
.
├── .github/
│   └── workflows/
│       ├── process-incoming.yml    # Transcoding & deployment pipeline
│       └── cleanup-staging.yml     # Cloudinary purge cron job
├── incoming/                       # Incoming pointer JSON files (Git-ignored content)
├── media/                          # Transcoded rolling video assets (video1-3.mp4)
├── src/
│   ├── index.html                  # Frontend layout & video container
│   ├── css/
│   │   └── style.css               # Fullscreen minimalist presentation styles
│   └── js/
│       ├── app.js                  # Video rotation & HTTP Last-Modified polling
│       └── motion.js               # WebRTC canvas frame-differencing detector
└── README.md
```

---

## Getting Started & Configuration

### Prerequisites
1. **Cloudinary Account**: Create an account and retrieve your `Cloud Name`, `API Key`, and `Upload Preset`.
2. **GitHub Personal Access Token (PAT)**: Create a token with `repo` scopes to allow the iOS Shortcut to write files to `/incoming`.

### Environment Secrets (GitHub Repository Secrets)
Configure the following secrets in your repository (**Settings > Secrets and variables > Actions**):

- `CLOUDINARY_CLOUD_NAME`
- `CLOUDINARY_API_KEY`
- `CLOUDINARY_API_SECRET`
- `CLOUDINARY_UPLOAD_PRESET`

---

## iPad Display Setup

1. Open **Safari** on your iPad and navigate to your deployed live site URL.
2. Tap the **Share** icon in Safari and select **Add to Home Screen**.
3. Launch the app directly from your Home Screen to enable full-screen presentation mode (hiding browser UI/toolbars).
4. Grant camera access when prompted to enable WebRTC motion sensing.
