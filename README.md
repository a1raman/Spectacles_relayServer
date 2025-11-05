# Spectacles_relayServer

A lightweight polling-based Render Receiver that continuously fetches tasks (image render jobs) from a Relay Server and saves them locally under downloads/.
Designed to integrate with Spectacles (AR glasses) or similar systems for multimodal AI workflows.

## Overview

This script periodically polls a remote Relay Server for new render tasks, downloads image data (or other media) when available, and creates matching JSON sidecar metadata files.
It’s intended to act as the bridge between the RelayServer (task distributor) and the RenderServer (model inference / processing node).

## ✨ Features

Polling-based task retrieval via /task endpoint

Safe, resumable downloads with atomic .part → final rename

Sidecar JSON metadata generated automatically per request

MIME-type → file extension mapping (JPG/PNG/WEBP)

Auto retry/backoff for transient HTTP 5xx or network errors

Proxy, custom CA, and TLS verification toggle

Single-file Python script, easily deployable as a service or Docker container

## 🔁 Workflow

Polls GET {BASE_URL}/task?session={SESSION_ID}&timeout={TIMEOUT_S}

Returns 204 No Content if no task available

When a task arrives:
```
{
  "req_id": "r-1762137472879-332357",
  "prompt": "Describe this image...",
  "mime": "image/jpeg",
  "image_url": "/media/r-1762137472879-332357.jpg"
}
```

Downloads the image → saves as downloads/r-1762137472879-332357.jpg

Writes a sidecar JSON → downloads/r-1762137472879-332357.json

Downstream pipelines can watch the downloads/ folder for processing.

## ⚙️ Requirements

Python 3.8+

requests, urllib3
```
pip install requests urllib3
```

## 🌍 Environment Variables
Variable	Default	Description
RELAY_BASE	""	Base URL of the Relay Server
RELAY_SESSION	""	Session ID (client group identifier)
RELAY_TOKEN	""	Authentication token sent as X-Relay-Token header
RELAY_TIMEOUT	10	Long-poll timeout (seconds)
RELAY_OUTDIR	./downloads	Directory for saving downloaded files
REQUESTS_CA_BUNDLE	(empty)	Path to custom root CA PEM file
INSECURE	(empty)	Set to 1/true/yes to disable TLS verification
HTTP_PROXY	(empty)	HTTP proxy URL
HTTPS_PROXY	(empty)	HTTPS proxy URL

Example .env:
```
RELAY_BASE=https://relay.example.com
RELAY_SESSION=prod-spectacles
RELAY_TOKEN=super-secret
RELAY_OUTDIR=./downloads
RELAY_TIMEOUT=10
```

## 🚀 Usage
Single-shot mode

Run once to fetch a single task (if available):
```
python relay_downloader.py
```

Exit codes:

0 → Task processed successfully

1 → No task (204 response)

2 → Network / SSL / HTTP error

Continuous loop mode

Continuously poll for tasks:
```
python relay_downloader.py --loop
```

Useful options
```
# Disable TLS verification (for testing)
python relay_downloader.py --loop --insecure

## Use a corporate root CA
python relay_downloader.py --loop --cafile /path/to/rootCA.pem

## Add proxy settings
python relay_downloader.py --loop --https-proxy http://proxy:8080 --http-proxy http://proxy:8080
```

## 📁 Output Structure
```
downloads/
  r-1762137472879-332357.jpg      # downloaded image
  r-1762137472879-332357.json     # sidecar metadata
```

Sidecar example:
```
{
  "req_id": "r-1762137472879-332357",
  "prompt": "Describe this image...",
  "mime": "image/jpeg",
  "image_url": "/media/r-1762137472879-332357.jpg",
  "saved_at": 1730799999
}
```

## 🧠 Retry & Backoff

Retries up to 3 times on 502/503/504 responses

Exponential backoff (0.2s factor)

When no tasks available: backoff increases from 100 → 500 ms

After a task or error: short sleep (50 ms) before next poll

## 🔒 Security

Authentication via X-Relay-Token header

Optional TLS verification override (--insecure)

Compatible with internal HTTPS proxy and custom root CAs

## 🧩 Integration Tips

This script is often used together with:

RelayServer → provides /task and /result endpoints

RenderServer (this repo) → downloads and processes media

Post-processors → perform inference, TTS, or logging on downloaded assets
