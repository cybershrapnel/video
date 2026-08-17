# NanoCheeZe MEQUAVIS — LTX Video Swarm

**Public queue:** [https://video.nanocheeze.com](https://video.nanocheeze.com)

Distributed home-GPU pipeline for **LTX AI video**. Anyone can queue jobs on the website. Machines running this package pull work from the hive, render with ComfyUI, upload results, and mark jobs complete.

This is not a cloud farm. It is a **shared queue + home GPUs**.

---

## What this is

The system has two sides:

| Side | Role |
|------|------|
| **Website / hive** | Accepts uploads, stores job folders, serves zips to workers, receives finished videos |
| **Worker node** | Your PC: ComfyUI + scripts that claim jobs, generate video, upscale, upload |

Jobs can be:

1. **MP3 → video** (music / long audio, chunked into clips, then concat + original audio mux)
2. **Text-to-video** (no MP3; single clip from prompt + 1 or 3 images)

Workers advertise what they can handle (max seconds, audio yes/no). The server only hands out matching jobs.

---

## Architecture

```
User browser
    │
    ▼
video.nanocheeze.com  (PHP)
    ├── jobs/          waiting work
    ├── processing/    claimed by a worker
    ├── completed/     done + final video
    ├── paused/        held / maintenance
    └── trash/         admin discard
    │
    │  GET process.php?max_seconds=&accept_audio=&client_version=
    │  POST complete / release
    │  POST upload.php (final mp4, chunked)
    ▼
Worker PC
    ├── ComfyUI (LTX workflow)
    ├── worker.py   (renders local queue/*.zip)
    ├── checker.py           (fetches zips, upscales, uploads, completes)
    └── start_service.py     (starts both)
```

### Claim flow

1. Checker sees empty local `queue/`
2. Calls `process.php` with filters + protocol version
3. Server zips oldest **matching** job, moves it to `processing/`, returns the zip
4. Worker processes the zip through ComfyUI
5. Checker CPU-upscales final to 1080p, uploads, then `action=complete` so server moves `processing/ → completed/`

**Multi-worker safe:** no global “drain everything.” Each job is claimed once; complete is per `job_name`.

---

## Hardware requirements

| | Minimum | Preferred |
|--|---------|-----------|
| GPU | NVIDIA, **12GB+ VRAM** | **16GB+** |
| System RAM | 32GB | **64–128GB+** (256GB ideal for long jobs) |
| OS | Windows 10/11 | Windows 11 |
| Disk | Fast SSD; models are large (~20GB+ downloads) | — |

**Notes**

- 12GB cards should use **short, no-audio** filters (`MAX_SECONDS=5`, `ACCEPT_AUDIO=0`).
- 16GB cards can take full music jobs (`MAX_SECONDS=45`, `ACCEPT_AUDIO=1`).
- Long music videos are chunked (default 45s segments). Generation is slow: on the order of **~28 minutes wall time per 45 seconds of video** on a single 4060 Ti-class card (varies by settings/load).

---

## Repository layout (swarm package)

Place these **in your ComfyUI install folder** (same place as `main.py` / workflows):

```text
ComfyUI/
├── worker.py      # worker: watches queue/, runs LTX jobs
├── checker.py              # hive client: fetch / upscale / upload / complete
├── start_service.py        # launches worker + checker together
├── ltx_23_audio.json       # API-format workflow (with audio)
├── ltx_23_no_audio.json    # API-format workflow (no audio)
├── queue/                  # incoming .zip jobs (created automatically)
│   ├── completed/
│   ├── failed/
│   └── _work/
├── audio_chunks/           # temp mp3 segments
├── output_concat/          # finals + upscaled/
│   └── upscaled/
└── swarm-data/
    └── patched_api/        # debug dumps of patched API graphs
```

Workflow JSON files stay next to the scripts. Runtime junk for patched graphs goes under `swarm-data/patched_api/`.

Comfy **input** files are staged into Comfy’s real input directory (Shared or install `input/`), not into `swarm-data`.

---

## One-time ComfyUI / workflow setup

You need a working ComfyUI with the LTX stack this swarm uses.

### High-level install (tested procedure)

1. Load the provided workflow and open **Details** for missing nodes/files.
2. Install missing packages → **restart** ComfyUI.
3. Install missing again → **restart** again.
4. ~4 items may still be missing. Manually install those **plus one extra** (5 total) from the models zip: drop all **5 folders** into your ComfyUI `models` folder.

   Example path:

   ```text
   C:\Users\<you>\AppData\Local\Comfy-Desktop\ComfyUI-Installs\ComfyUI\ComfyUI\models
   ```

5. For the last missing **node**, use Details → **Search** (do not rely on the auto name). Search for:

   ```text
   custom-switch
   ```

   Install it (underscore names often fail; search install is required).

6. Restart ComfyUI. Node errors should clear.
7. You should only be missing **3 safetensor** files. **Download all** (~20 GB).
8. Put a test image in the workflow’s **3** image slots and **Run**. If a short clip succeeds, you are ready for the hive.

**Also required**

- Matching **LoRA** / distilled assets the workflow expects (if the log says a LoRA was skipped, quality will be wrong).
- Correct **VideoVAE** vs **AudioVAE** wiring (do not plug video VAE into the audio path).
- `ffmpeg` and `ffprobe` on PATH (used for chunking, concat, mux, 1080p upscale).

### Headless ComfyUI

Run Comfy from the install folder (adjust python path for your install):

```bat
python main.py --listen 127.0.0.1 --port 8188 --disable-auto-launch --disable-pinned-memory
```

Desktop embeds may need the embedded python:

```bat
.\python_embeded\python.exe main.py --listen 127.0.0.1 --port 8188 --disable-auto-launch --disable-pinned-memory
```

Worker talks to `http://127.0.0.1:8188` by default (`COMFY_URL`).

### Input directory mismatch

Comfy must see the same folder the worker stages into. On startup Comfy logs:

```text
Setting input directory to: ...
```

If the worker stages elsewhere, set:

```bat
set COMFY_INPUT_DIR=C:\path\to\that\input
```

The worker also tries common Desktop Shared / install paths automatically.

---

## Running a swarm node

### 1. Copy scripts into ComfyUI root

Put `worker.py`, `checker.py`, `start_service.py`, and the two workflow JSON files in the ComfyUI directory.

### 2. Set machine flags (checker)

Edit the top of `checker.py` (or use env vars):

```python
MAX_SECONDS = 45        # only claim jobs with seconds.txt <= this (missing = 45)
ACCEPT_AUDIO = True     # False = no-audio jobs only (reject any zip with .mp3)
CLIENT_VERSION = "2"    # must match server PROTOCOL_VERSION
```

**Examples**

| Machine | MAX_SECONDS | ACCEPT_AUDIO |
|---------|-------------|--------------|
| Strong 16GB+, music OK | `45` | `True` |
| Weak 12GB / short only | `5` | `False` |

Env overrides:

```bat
set MAX_SECONDS=5
set ACCEPT_AUDIO=0
set CLIENT_VERSION=2
set COMFY_INPUT_DIR=C:\...\ComfyUI\input
```

### 3. Start ComfyUI, then the service

```bat
cd /d C:\path\to\ComfyUI
python start_service.py
```

`start_service.py` starts:

- `worker.py` — processes `queue/*.zip`
- `checker.py` — fetches from hive when queue is empty; after local finals exist, upscales + uploads + completes

Ctrl+C stops both (auto-restart on crash is enabled by default).

---

## Job zip format

Each job zip (from the website) contains:

| File / folder | Purpose |
|---------------|---------|
| `promp.txt` or `prompt.txt` | Positive prompt |
| `anti.txt` | Negative prompt |
| `seconds.txt` | Optional. Clip length 5–45. Missing ⇒ **45** |
| `audio.mp3` | Present for music path |
| `noaudio.txt` | Marker for text-to-video path (no mp3) |
| Images | See below |

### Images

- **1 image at zip root** → used as image1 = image2 = image3
- **3 images at root** → image1 / image2 / image3 (sorted alphabetically; pad with last if needed)
- **Folder of images** (e.g. `images/`) → **rotate**: each chunk uses the next image for all three slots (loops)

### Worker behavior

- **With MP3:** split audio into `seconds`-sized chunks (last chunk shortened to `ceil(leftover)+1` seconds of video), run audio workflow per chunk, concat, remux **original** mp3 onto final (VAE audio is discarded).
- **No audio:** single run of no-audio workflow at `seconds` length.

---

## Server endpoints (hive)

Base: your deploy of the PHP app (e.g. `https://xtdevelopment.net/video-gen/` or `https://video.nanocheeze.com/`).

### `process.php`

**GET** — claim one job:

```text
?max_seconds=45&accept_audio=1&client_version=2
```

| Result | Meaning |
|--------|---------|
| `200` + zip | Job claimed; headers include `X-Job-Name`, `X-Protocol-Version` |
| `204` / `NO_JOB` | Nothing matching filters |
| `409` / `CLAIM_RACE` | Another worker won the race |
| `426` / `SERVER_UPDATED_CHECKER_MUST_UPDATE` | Protocol mismatch — update scripts |

**POST**

- `action=complete&job_name=...&client_version=2` — `processing/ → completed/`
- `action=release&job_name=...&client_version=2` — return job to `jobs/` (optional)

### Protocol version

Server defines `PROTOCOL_VERSION` (currently `"2"`). Checker sends `CLIENT_VERSION`. Exact match required. When the API changes, bump both; old checkers stop receiving work instead of corrupting state.

### `upload.php`

Accepts final 1080p mp4 for a `job_name` (single body or **chunked** resume for large files).

### Website extras

- Maintenance mode: `maintenance.txt` present → new jobs go to `paused/`
- Admin: password-gated **unpause**, **pause**, **trash** (`unpause_password.php`, `pause_password.php`, `trash_password.php` — not web-accessible)
- Queue stats / ETA header, comments on completed/paused jobs, visitor footer, etc.

---

## Checker responsibilities (detail)

1. If local `queue/` has zips → do not claim; wait for worker.
2. Else GET filtered job from `process.php`.
3. Save zip into `queue/`.
4. Worker produces `output_concat/<job>_final_*.mp4`.
5. Checker CPU-upscales to `output_concat/upscaled/*_1080p.mp4`.
6. Upload to hive (`upload.php`, chunked if large).
7. `action=complete` so the site shows the job under completed.
8. Markers under `upscaled/` avoid double-upload; progress files allow resume after network failure.

---

## Worker responsibilities (detail)

1. Poll Comfy `/system_stats`.
2. Take oldest `queue/*.zip`.
3. Extract → read prompt/anti/seconds/images/audio.
4. Stage images/audio into Comfy `INPUT_DIR`.
5. Patch API workflow nodes (LoadImage, LoadAudio, length/frames, text, orchestrator audio toggle).
6. `POST /prompt`, wait history.
7. Collect output videos, concat if needed, mux original mp3.
8. Write final under `output_concat/`.
9. Move zip to `queue/completed/` (or `failed/` on error).

Patched graphs for debugging:

```text
swarm-data/patched_api/ltx23_patched_api_<tag>.json
```

---

## Environment variables (summary)

| Variable | Used by | Meaning |
|----------|---------|---------|
| `COMFY_URL` | worker | Default `http://127.0.0.1:8188` |
| `COMFY_INPUT_DIR` | worker | Force Comfy input path |
| `WORKFLOW_FILE` | worker | Audio workflow JSON |
| `WORKFLOW_NO_AUDIO` | worker | No-audio workflow JSON |
| `MAX_SECONDS` | checker | Claim filter (5–45) |
| `ACCEPT_AUDIO` | checker | `1` / `0` |
| `CLIENT_VERSION` | checker | Must match server `PROTOCOL_VERSION` |
| `PROCESS_URL` | checker | Claim/complete endpoint |
| `UPLOAD_URL` | checker | Final video upload |
| `COMPLETE_URL` | checker | Defaults to `PROCESS_URL` |
| `SWARM_DATA` | worker | Base for `patched_api/` (default `swarm-data`) |
| `QUEUE_DIR` | checker | Local zip queue (default `queue`) |
| `FINAL_OUT` | checker | Finals folder (default `output_concat`) |
| `EMPTY_WAIT` | checker | Seconds between empty claims (default 30) |
| `BUSY_WAIT` | checker | Seconds when local queue has work (default 15) |
| `UPGRADE_WAIT` | checker | Backoff on protocol mismatch (default 120) |
| `MUSIC_IDLE_CYCLES` | optional checker | Empty polls before starting music |
| `RESTART_DELAY` | start_service | Child restart delay |
| `AUTO_RESTART` | start_service | `0` to disable |

---

## Website user flows

### MP3 → video

1. Upload 1 or 3 images (or a group for rotate-per-chunk).
2. Upload required MP3.
3. Prompt + anti-prompt (default anti often `cartoon, animation`).
4. Optional clip length (friends page / index2: 5–45 seconds per chunk).
5. Job lands in `jobs/` (or `paused/` under maintenance).

### Text-to-video (no audio)

1. 1 image (all slots) or 3 images (first / middle / last).
2. Prompt + anti; no MP3.
3. Single clip at `seconds.txt` length (default 45 if missing).

### Queue / processing / completed / paused

- **Queue** — waiting to be claimed  
- **Processing** — a worker has the zip  
- **Completed** — final video available; comments supported  
- **Paused** — held (maintenance or admin); unpause with password  

Abuse, spam long empty music jobs, or flooding the queue can get submissions removed or banned. The service is free on the honor system; donations help expand GPU capacity.

---

## Admin / operator notes

### Maintenance

Create `maintenance.txt` in the site root. Contents can be hours of expected downtime (used by the maintenance banner). New uploads go to `paused/` until you remove the file and move folders back to `jobs/` manually.

### Passwords

Keep these **off the web** (`.htaccess` deny or outside docroot):

- `unpause_password.php` — `$UNPAUSE_PASSWORD`
- `pause_password.php` — `$PAUSE_PASSWORD`
- `trash_password.php` — `$TRASH_PASSWORD`

### Protocol bumps

1. Raise `PROTOCOL_VERSION` in `process.php`.
2. Ship checkers with matching `CLIENT_VERSION`.
3. Old nodes print upgrade message and stop claiming.

### Stuck jobs

If a worker dies after claim, the job can sit in `processing/`. Admin can move it back to `jobs/` or use `action=release` if implemented on your build.

---

## Troubleshooting

| Symptom | Likely fix |
|---------|------------|
| `No such file` on LoadImage | `COMFY_INPUT_DIR` must match Comfy’s input path |
| Bad / blank video quality | Missing LoRA; check Comfy log for “skipping” |
| `VideoVAE` / `latent_frequency_bins` | Wrong VAE on audio node |
| Checker `NameError: time` | Ensure `import time` in checker |
| `SERVER_UPDATED_CHECKER_MUST_UPDATE` | Bump `CLIENT_VERSION` to server protocol |
| Upload HTTP 500 / timeout | Chunked upload + retries; slow hosts need patience |
| OOM / huge system RAM use | Lower length, use short-job filter, restart Comfy periodically |
| Worker not picking zip | Comfy down; or zip still being written; check `queue/` |

Comfy can leak host RAM over long runs; schedule occasional full restarts of the Comfy process on dedicated nodes.

---

## Credits

- LTX / workflow authors as noted in the workflow files and site copy  
- ComfyUI and custom node authors  
- NanoCheeZe MEQUAVIS — swarm orchestration, site, and worker tooling  

Support: CashApp `$nanocheeze` · [Patreon](https://www.patreon.com/hybridtales) · [X @mequavis](https://x.com/mequavis)

---

## License / use

Provided for community GPU sharing around the public queue. Do not abuse the queue. Operators may remove junk jobs and restrict access. Model and node licenses remain with their respective authors—follow those terms when redistributing weights or workflows.
```
