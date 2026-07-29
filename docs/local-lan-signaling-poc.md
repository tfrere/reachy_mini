# Working doc: LAN self-signaling PoC (drop the internet + HF OAuth dependency)

> Status: exploration / feasibility report. Not a spec.
> Scope: can a Reachy Mini be discovered and connected to over WebRTC on the
> local network, without internet and without HF OAuth, and how much work is it?

## TL;DR

Feasible, and closer than it looks. The hardest brick (a WebRTC signaling
server running *on the robot*) already exists and runs at all times. Today we
hide it behind `127.0.0.1` and bridge it to the remote HF "central". Going
LAN-only is mostly transport plumbing plus a small local pairing model, not a
WebRTC rewrite.

The nuance is the client:

- **Python SDK** and **native app (Tauri, mobile/desktop)**: realistic and cheap.
- **Web app served over HTTPS (from a Space)**: blocked by a browser wall
  (mixed-content / secure-context). Out of scope for the PoC.

Estimate: **~3-4 days** for a "works on my LAN" PoC, **~1 to 1.5 week** for a
presentable version (with PIN + clean central fallback + tests).

---

## 1. How it works today

Three tiers:

```
Client (mobile Tauri / desktop / browser / JS SDK)
   |  SSE /events + POST /send   (Authorization: Bearer <HF token>)
   v
Central HF Space  (pollen-robotics-reachy-mini-central.hf.space)  <- stateless matchmaker
   |  SSE /events + POST /send   (Authorization: Bearer <HF token>)
   v
CentralSignalingRelay  (thread in the daemon)
   |  WebSocket ws://127.0.0.1:8443
   v
webrtcsink (GStreamer, run-signalling-server=True)  <- the signaling server ALREADY runs on the robot
   |
   v  WebRTC P2P direct (SDP/ICE via Google STUN, no TURN)
Client
```

Two real "internet + HF" dependencies:

- **Discovery**: the client only finds robots by querying central
  (`GET /api/robot-status`), which filters by HF `username` (`whoami-v2`). The
  HF token *is* the ownership filter.
- **Signaling**: SDP/ICE exchange goes through central (SSE + POST), also gated
  by the HF token.

Central is a plain **stateless matchmaker** (`reachy_mini_central/app.py`): it
relays SDP/ICE, keeps an in-memory `token -> peerId` registry, then steps out.
It never carries media.

Key source files:

| Role | File |
|------|------|
| Daemon signaling client (SSE<->WS, peer/session mgmt, heartbeat) | `src/reachy_mini/media/central_signaling_relay.py` |
| WebRTC/media pipeline, `webrtcsink`, DataChannel, ICE watchdog | `src/reachy_mini/media/media_server.py` |
| Direct WS client to the local signalling server (`:8443`) | `src/reachy_mini/media/webrtc_utils.py`, `webrtc_client_gstreamer.py` |
| mDNS / LAN discovery | `src/reachy_mini/utils/discovery.py` |
| Local FastAPI server + mDNS wiring + bind host | `src/reachy_mini/daemon/app/main.py` |
| JS SDK client (central SSE, STUN, startSession, peerId) | `ts/lib/reachy-mini.ts` |
| Central matchmaker | `reachy_mini_central/app.py` |
| Mobile app central discovery | `reachy_mini_mobile_app/src/features/auth/fetchRobotsFromCentral.ts` |

---

## 2. What already exists for LAN (the good news)

1. **The signaling server runs on the robot.** `media_server.py` creates
   `webrtcsink` with `run-signalling-server=True`, so a real WebSocket signaling
   server listens on `:8443`. Central is just a bridge to it.
2. **A client that talks directly to `:8443` already exists.** `webrtc_utils.py`
   / `webrtc_client_gstreamer.py`: the Python SDK's `WEBRTC` backend connects to
   `signalling_host:8443` and speaks the native GStreamer protocol, no central,
   no token. This is tested (`tests/unit_tests/test_webrtc_loopback.py`).
3. **LAN discovery already exists.** `utils/discovery.py` publishes mDNS
   `_reachy-mini._tcp.local.` with `robot_name`, `address`, `hardware_id`,
   `caps`, etc., and provides `find_robots()` on the client side.
4. **The daemon already listens on the LAN.** On the wireless variant, FastAPI
   binds `0.0.0.0:8000` (`daemon/app/main.py::_resolve_bind_host`).
5. **No STUN/TURN needed on the same LAN.** Two devices on the same Wi-Fi
   negotiate via ICE "host" candidates. A LAN mode can be 100% offline.

Conclusion: the robot is already, technically, self-signaling. We just capped it
to `127.0.0.1` + a central bridge.

---

## 3. The insight that makes it simple

The entire JS SDK signaling client depends only on **two variables**
(`_signalingUrl`, `_token`) and **three HTTP endpoints**:

- `GET  {signalingUrl}/events`          (SSE, `Authorization: Bearer <token>`)
- `POST {signalingUrl}/send`            (client -> server messages)
- `GET  {signalingUrl}/api/robot-status` (robot listing)

So: if the daemon exposes `/events`, `/send`, and `/api/robot-status` locally
speaking the **same protocol**, the JS SDK works by only changing `signalingUrl`
(-> `http://<robot-ip>:8000/local-signaling`) and passing a dummy token (or a
PIN). Zero rewrite of the client WebRTC state machine.

The application protocol is already written and tested server-side in
`reachy_mini_central/app.py` (`handle_set_peer_status`, `handle_start_session`,
`handle_peer_message`, `handle_end_session`, `get_producers_list`). The shim is
that file minus HF auth and minus multi-tenant.

---

## 4. Target architecture (Option A)

```
App (Tauri mobile/desktop) / Python SDK        -- same Wi-Fi -->  Robot

  1. mDNS _reachy-mini._tcp.local.  -> IP + hardware_id (already published)
  2. GET  http://<ip>:8000/local-signaling/api/robot-status  (PIN)
  3. GET  http://<ip>:8000/local-signaling/events   (SSE, welcome)
  4. POST http://<ip>:8000/local-signaling/send     (startSession, SDP/ICE)
       |
       v
  FastAPI daemon :8000
       |  local_signaling router  -- WS bridge -->  ws://127.0.0.1:8443 (webrtcsink)
       v
  WebRTC P2P direct, ICE host candidates, no STUN/TURN, fully offline
```

Important: the shim connects to `webrtcsink` over **loopback**
(`127.0.0.1:8443`), so we never expose port 8443 on the LAN and don't touch the
webrtcsink bind. Only `:8000` (already bound to `0.0.0.0` on the wireless
variant) is the LAN entry point.

Central stays as the remote fallback (hybrid mode: LAN first, central if the
robot isn't reachable locally).

---

## 5. Work breakdown

### Lot 1 - Local signaling shim (daemon)

**Create:** `src/reachy_mini/daemon/app/routers/local_signaling.py`
**Modify:** `src/reachy_mini/daemon/app/main.py` (wire the router).

Reuse from `reachy_mini_central/app.py`: the SSE/POST skeleton and the message
handlers (`get_or_create_peer`, `handle_start_session`, `handle_peer_message`,
`handle_end_session`, `get_producers_list`).

Drop:
- All HF validation (`_resolve_hf_token`, `validate_hf_token`, `whoami-v2`) ->
  replaced by an optional local PIN check.
- `username` filtering and the "you don't own this robot" gate -> on the LAN
  there is a single de-facto owner.
- `install_id` eviction -> single robot, unneeded.

The only genuinely new part: the **bridge to `:8443`**. The shim must not wait
for a remote daemon to connect (as central does) but hook itself onto the local
`webrtcsink` and expose that robot as the single producer. Two ways:

- **5a (recommended)** - reuse the translation logic already written in
  `central_signaling_relay.py` (`_process_local_message` /
  `_process_central_message`, local<->consumer `sessionId` mapping). It's exactly
  a WS(8443) <-> SSE/POST bridge, just served locally instead of consumed
  remotely: flip the direction of the HTTP channel.
- **5b** - a small dedicated bridge that, per SSE consumer, opens a WS session to
  `127.0.0.1:8443` (like `webrtc_utils.connect(...)`), does `startSession`, and
  relays `peer`/`ice`/`sdp` both ways.

Router sketch (5b, deliberately minimal):

```python
# src/reachy_mini/daemon/app/routers/local_signaling.py
router = APIRouter(prefix="/local-signaling")

@router.get("/api/robot-status")
async def robot_status(_pin: None = Depends(check_local_pin)):
    # Ask the local webrtcsink signalling server for its producer list
    # (reuse webrtc_utils.get_producer_list) and reshape it to the central
    # /api/robot-status wire format the SDK already parses.
    producers = get_producer_list("127.0.0.1", 8443)
    return {"robots": [_to_status_entry(pid, meta) for pid, meta in producers.items()]}

@router.get("/events")
async def events(_pin: None = Depends(check_local_pin)):
    # Same SSE frames as central: welcome{peerId} + list{producers},
    # then relay queued messages (EventSourceResponse, like central).
    ...

@router.post("/send")
async def send(request: Request, _pin: None = Depends(check_local_pin)):
    # startSession / peer / endSession -> forward to the WS bridge to :8443
    ...
```

Wire it in `main.py` next to the other routers (gate on `args.wireless_version`,
same condition as the `0.0.0.0` bind).

Effort: **1 - 1.5 day** (protocol copy is quick; real time is the `:8443` bridge
and session mapping, already solved once in `central_signaling_relay.py`).

### Lot 2 - JS SDK client

Micro-changes in `ts/lib/reachy-mini.ts`:

1. **Make the token optional in local mode.** `_sendToServer` currently throws
   without a token; `connect()` and `_fetchOwnedRobots()` always send the Bearer
   header. In local mode, send the PIN (or nothing) instead.
2. **Point `signalingUrl` at the robot.** The constructor already accepts the
   override; the app passes `signalingUrl: 'http://<robot-ip>:8000/local-signaling'`.
3. **(optional, offline cleanliness)** make `iceServers` configurable so LAN mode
   can drop the hardcoded Google STUN. Same-subnet host candidates suffice, so
   even without this it works offline (STUN just fails silently).

Effort: **1 - 2 days** (mostly non-regression tests on the central path, which
must stay identical).

### Lot 3 - LAN discovery in the mobile app

Most platform-dependent lot.

- Python SDK: discovery already exists (`utils/discovery.py::find_robots`).
  Nothing to do.
- Mobile app (Tauri): **no mDNS today**, discovery is 100% central
  (`fetchRobotsFromCentral.ts` even documents "the mobile app has no LAN line of
  sight"). Two options:
  - **Native mDNS** via a Tauri/Rust plugin (Bonjour iOS / NSD Android). The real
    path, but the cost/risk hotspot of the PoC.
  - **PoC shortcut**: the app already knows the robot IP right after BLE
    provisioning (`ble-provisioning/`). Remember that IP and skip mDNS for the
    demo.

Files:
- `reachy_mini_mobile_app/src/shared/env.ts` - add a "LAN-first" flag (mirror the
  existing `HF_REALTIME_CONNECTION_MODE`). Central default stays.
- **New** `reachy_mini_mobile_app/src/features/auth/fetchRobotsFromLan.ts` - clone
  of `fetchRobotsFromCentral.ts` hitting `http://<ip>:8000/local-signaling/api/robot-status`
  with the PIN instead of the Bearer HF. The extraction helpers
  (`extractRobotId`, `extractRobotHardwareId`, ...) are reusable as-is.
- Robot picker: LAN first, central fallback.

Effort: **1 day** with the IP-after-BLE shortcut. **+1-2 days** for real native
mDNS in Tauri.

### Lot 4 - Local pairing / PIN

Without the HF filter, authorization becomes "being on the Wi-Fi". Note: on the
wireless variant the daemon `:8000` API is **already** unauthenticated and bound
to `0.0.0.0` (`_resolve_bind_host`, plus the CORS regime), so DataChannel control
is already LAN-exposed today. The PIN is hardening, not a regression.

Minimal PoC version:
- Robot generates a short PIN (displayed, or transmitted over the existing BLE
  provisioning channel).
- Shim verifies the PIN in a `Depends(check_local_pin)` on `/events`, `/send`,
  `/api/robot-status`.
- Client sends it as a header (`X-Reachy-Pin`) or inside the pseudo-token.

Effort: **0.5 - 1 day**.

### Lot 5 - Integration + real-robot tests

Effort: **1 - 2 days**. End-to-end on Wi-Fi, AP-isolation case, reconnection,
latency, and non-regression on the central path.

---

## 6. Risks / hard points

| Risk | Impact | Mitigation |
|------|--------|------------|
| Web app HTTPS (mixed-content + secure-context) | An `https://` page (Space) can neither call `http://robot:8000` nor `getUserMedia` outside a secure context | Out of PoC scope. LAN-only targets Tauri + desktop + Python; web stays on central |
| Native mDNS in Tauri (mobile) | No auto-discovery without a plugin | Shortcut: reuse the IP known after BLE. Real mDNS = separate lot |
| CORS from the webview | SSE/POST is cross-origin (`tauri://localhost` -> `http://robot-ip:8000`) | Already covered: `CORS_ORIGIN_REGEX` allows `tauri://localhost` and `capacitor://localhost` |
| AP / client isolation | Some Wi-Fi networks isolate clients -> mDNS + P2P fail | Detect/document, not dev work |
| Google STUN offline | Slower negotiation without internet | Host candidates suffice on same subnet; make `iceServers` configurable (Lot 2, optional) |
| Central path regression | Break the existing remote mode | Local mode is additive (new `signalingUrl`); non-regression tests in Lot 5 |

---

## 7. Files summary + estimate

**Create**
- `src/reachy_mini/daemon/app/routers/local_signaling.py` (SSE/POST shim + `:8443` bridge)
- `reachy_mini_mobile_app/src/features/auth/fetchRobotsFromLan.ts` (LAN listing)

**Modify**
- `src/reachy_mini/daemon/app/main.py` (import + include router, wireless gate)
- `ts/lib/reachy-mini.ts` (optional token + PIN, local `signalingUrl`, optional iceServers)
- `reachy_mini_mobile_app/src/shared/env.ts` (LAN-first flag)
- mobile app robot picker (LAN first, central fallback)

**Estimate**

| Lot | Estimate |
|-----|----------|
| 1. Daemon shim | 1 - 1.5 d |
| 2. JS SDK client | 1 - 2 d |
| 3. LAN discovery (IP-after-BLE shortcut) | 1 d |
| 4. Local PIN | 0.5 - 1 d |
| 5. Integration + real-robot tests | 1 - 2 d |

- **"Works on my LAN" PoC** (lots 1+2+3, no PIN): **~3-4 days**
- **Presentable version** (PIN + clean fallback + tests): **~1 to 1.5 week**
- **+1-2 days** for real native mDNS in Tauri instead of the IP-after-BLE shortcut.

The number holds because the heavy part (embedded signaling server + application
protocol + `:8443` bridge) already exists and runs; we collapse "central + relay"
into a local router and flip two constants on the client.

---

## 8. Suggested next steps

1. SSH sanity check on a real robot: confirm
   `webrtc_utils.get_producer_list("127.0.0.1", 8443)` responds and that a
   session initiated from the LAN completes.
2. Write the real `local_signaling.py` skeleton, reusing the session mapping from
   `central_signaling_relay.py`.
3. Add the LAN-first flag + `fetchRobotsFromLan.ts` on the mobile app and wire the
   picker.
