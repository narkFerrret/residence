# residence — The Building: 3D Isometric Digital Twin & Spatial Console (`home.local`)

> **Field Notes & Commit Ledger.** This repository tracks the architecture, engineering notes, and chronological commit history of **Home**, the interactive WebGL / Three.js 3D spatial console originally prototyped on an M1 edge node (`edge-node:8844`) and migrated to native hosting on the M3 Pro hub (`http://home.local/` & `:8844`).

---

## Why I Built a 3D Digital Twin of My Flat

2D tables and log rows are great for auditing, but a home is a physical 3D space. When a motion event fires at the front porch or a courier is two minutes away, you want to see *where* in the building things are happening at a glance.

**Home** renders a multi-floor isometric 3D architectural model of the residence in Three.js (`static/home-console.html`), wired directly to the local agent fleet through a unified Node.js adapter (`lib/console_adapter.cjs`). Every room light in the 3D mesh reflects the real-time state and color temperature of the physical Philips Hue bulbs; clicking a camera cone streams live video frames into the 3D viewport; and a spatial HUD overlays live Starlink dish telemetry, local micro-climate weather, active food/parcel deliveries, and a natural-language command omnibox.

---

## Spatial Architecture & Adapter Contract

```
  ┌────────────────────────────────────────────────────────────────────────────┐
  │                3D WebGL / Three.js Spatial Console (home.local)            │
  │  • Multi-Floor Procedural Mesh + Floor-Slice Elevation Toggles             │
  │  • Live Emissive Hue Room Shaders (Bridge v2 State Sync)                   │
  │  • 3.8 fps NDJSON Live Camera Frame Planes (X-Frame-Stream: v1)            │
  │  • 3D Action Effects: Camera Fly-To, Light Pulse, Basket Drawer            │
  └─────────────────────────────────────┬──────────────────────────────────────┘
                                        │
  ┌─────────────────────────────────────▼──────────────────────────────────────┐
  │                 astro-local-adapter (lib/console_adapter.cjs)              │
  ├──────────────────────┬──────────────────────┬──────────────────────────────┤
  │  Astro (:8787)       │  Starlink Dish       │  Remy (:8790) + Jeeves(:8795)│
  │  • /api/cameras/live │  • Direct HTTP/2     │  • Unified /api/deliveries   │
  │  • HTTP 206 Range    │    gRPC to Dish      │    (Food Couriers + Parcels) │
  │    MP4 Clip Proxy    │    (10.0.100.1:9200) │  • Async /api/assistant Jobs │
  └──────────────────────┴──────────────────────┴──────────────────────────────┘
```

---

## Engineering Deep-Dives

### 1. Streaming Live Cameras into WebGL at 3.8 fps Over NDJSON
- WebRTC and HLS introduce multi-second buffering or heavy signaling overhead when you just want four low-latency camera textures inside a Three.js scene on a local LAN.
- `/api/cameras/:id/live` multiplexes base64 JPEG frames at ~3.8 fps over a chunked `application/x-ndjson` stream (`X-Frame-Stream: camera-frames-v1`, `X-Camera-Id: camera_N`), while `/api/cameras/:id/snapshot` stamps every single frame with `X-Snapshot-Fetched-At: <ISO>` so the 3D HUD never displays a frozen frame as live.

### 2. Fixing Safari/WebKit `<video>` MP4 Playback with HTTP `206 Partial Content`
- When clicking a detection event pin in the 3D scene to replay the security clip (`/api/events/:id/clip`), Chrome played the proxied stream fine while Safari/iOS WebKit silently aborted playback.
- **Root Cause:** WebKit starts every `<video>` load with a `Range: bytes=0-1` probe to verify byte-range seek support. If the adapter proxy returns `200 OK` with chunked transfer encoding instead of `206 Partial Content`, WebKit drops the media element immediately.
- **Fix:** `lib/console_adapter.cjs` forwards `Range` headers on both `GET` and `HEAD` requests to Astro `:8787` and preserves `206 Partial Content`, `Accept-Ranges: bytes`, `Content-Range`, and exact `Content-Length`.

### 3. Direct HTTP/2 gRPC Telemetry from the Starlink Dish (`10.0.100.1:9200`)
- Rather than scraping a web page or polling a cloud API, `/api/starlink` speaks raw HTTP/2 gRPC (`SpaceX.API.Device.Device/Handle`, `get_status`) directly to the dish's management IP (`10.0.100.1:9200`) on the WAN interface, surfacing live downlink/uplink throughput, pop ping latency, SNR, and obstruction fraction in the 3D roof HUD.

### 4. M1 (`edge-node`) to M3 (`hub-host`) Consolidation & Zero-Config mDNS
- Consolidating the 3D console onto the M3 host allowed `local.household.serve` to bind `:80`, `:8844`, and `:8800/console` in one process while spawning `dns-sd -P "Home Console" _http._tcp local 80 home.local 10.0.10.10` so any authorised phone or laptop on the LAN can open `http://home.local/` with zero port numbers.
