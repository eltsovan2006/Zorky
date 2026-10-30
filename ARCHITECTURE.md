# Zorky architecture

## Components

| Component | Role |
|---|---|
| ML worker | Python: decoding, detectors, tracking, rules, backend client |
| Backend | FastAPI: routes, services, domain models and rules, storage (SQLite, snapshots, previews, sessions, users) |
| Web | React + Vite PWA for the operator, foreman and manager |
| LinQ integration | Connection to the LinQ accelerator, `.tpu` build, frame service |
| Contracts | The single contract: `openapi.yaml`, `events.schema.json`, `violation_types.json` |

## Data flow

```
video (demo clips / RTSP)
   │
   ▼
ML worker — Python, next to LinQ
  decoding → YOLO on LinQ → tracking → rules (no helmet ≥ N frames, feet inside a zone polygon)
   │  outgoing requests only:
   │  POST /api/events, snapshots, previews, metrics
   ▼
Backend — FastAPI
  events, cameras, zones → SQLite (or in-memory in development mode)
  SSE /api/stream — the live feed
  foreman alerts — the `master` event in SSE, escalation
   │
   ▼
Frontend — React
  monitoring · journal · zone editor · analytics · LinQ metrics
```

## Key decisions
- **The worker only sends.** Incoming connections to the LinQ bench are closed, so the worker pushes events, snapshots, previews and metrics to the backend itself.
- **The backend doesn't decode frames.** It stores the worker's JPEGs and serves them as is: an MJPEG preview for `<img>`, snapshots by link.
- **An event is the start of a violation on a track.** Tracking and debouncing: a violation must persist for N frames in a row, a repeat on the same track comes no earlier than after a cooldown. Without this one person without a helmet would produce hundreds of events per minute.
- **Zones are drawn in the frontend and stored by the backend.** The worker re-reads them every few seconds (`GET /api/cameras/{id}/zones`).
- **A danger zone needs no training**: an off-the-shelf people detector + a polygon.
- **The live feed is SSE, not WebSocket**: data flows one way, and the browser reconnects by itself.
- **LinQ metrics are on the dashboard**: FPS, inference time, “frame → event” latency.
- **ML and the backend share one language — Python**: types come from one contract.
- **Alerts stay inside the perimeter**: the foreman's web app, no Telegram and no Google or Apple push services.
- **The production PPE weights are YOLO11s/640** from the `ppe-v2` Release; YOLO11n remains a fallback.

## Security
- **Sign-in.** All `/api/*` except `/healthz` require a session: login and password from the users file (a users file outside the code base), brute-force protection, an Origin check. Without the file and a `WORKER_TOKEN` — development mode, everything is open.
- **Roles.** `master`, `operator`, `auditor`, `admin`: the foreman and the operator confirm events, the auditor only reads, the admin manages users and zones.
- **HTTPS.** The web is served over HTTPS (port 8443, self-signed certificate); the PWA and alert sounds only work in a secure context.
- **Worker token.** The ML worker uses `Authorization: Bearer <WORKER_TOKEN>` and can only `POST /api/events`, `/frame`, `/snapshots`, `/metrics`.
- **Everything stays inside the perimeter.** Video, snapshots and events don't leave it. Secrets live only in the environment configuration.

## Decision log
- 2026-10-21 — backend in Python + FastAPI: one language with ML, nothing extra for the demo.
- 2026-10-23 — alerts only inside the perimeter, no external push services.
- 2026-10-23 — the production PPE weights are YOLO11s/640, not YOLO11n: cigarette mAP50-95 0.397 → 0.464, the other classes are no worse.
