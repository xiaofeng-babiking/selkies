# HOWTO: SuperSplat in a GPU-accelerated remote browser (Selkies on an A100 host)

This deployment streams a GPU-accelerated Linux desktop from an A100 server to
your laptop browser via WebRTC, with the PlayCanvas **SuperSplat** editor served
locally inside it. Everything runs in one container.

Deployed and verified on `ai-hzidc-dev02` (8× A100 80GB, Ubuntu 22.04 host,
driver 590.x), 2026-07-24.

## 1. Architecture

```
 laptop browser (192.168.x.x)
   │  HTTP + WebSocket, TCP 8080 (basic auth)
   ▼
 nginx :8080 ──proxy──► selkies-gstreamer :8081 (web app, signaling)
   │                        │
   │  WebRTC media          │ captures X display, encodes x264 (CPU!)
   ▼                        ▼
 coturn :3478 (TCP) ◄── webrtcbin ◄── Xvfb :20 (virtual desktop, 2D = CPU)
   relay-ip=ethernet IP          ▲
                                 │ OpenGL via VirtualGL → NVIDIA EGL → A100
                          Firefox ── SuperSplat http://localhost:8082
                                     (python http.server, loopback only,
                                      serves host-mounted supersplat/dist)
```

Division of labor — know what is and isn't on the GPU:

| Piece                        | Runs on |
|------------------------------|---------|
| SuperSplat WebGL2 rendering  | **A100** (VirtualGL EGL) |
| Desktop 2D / compositing     | CPU (Xvfb) |
| Video encode of the stream   | CPU (`x264enc`) — **A100 has no NVENC silicon**, never set `nvh264enc` |
| Video decode                 | your laptop |

## 2. Prerequisites

- Docker + `nvidia-container-runtime`; the standalone **`docker-compose`** binary
  (this host has no `docker compose` v2 plugin).
- NVIDIA driver on the host (any recent; user-space libs are injected into the
  container at start, so host driver upgrades are picked up on recreate).
- A built SuperSplat checkout on the host:

  ```bash
  cd ~/codebases/supersplat
  npm install && npm run build        # produces dist/
  ```

- A data directory for your scenes: `~/selkies-data` (mounted as `~/Documents`
  in the desktop — this is the only persistent directory).
- Free host ports: `8080` (web), `8081`/`8082` (loopback), `3478` (TURN),
  UDP `49152–65535` (TURN relays). The container uses **host networking**.

## 3. Files (all in this directory)

- `docker-compose.deploy.yml` — the single `selkies` service. Highlights:
  `network_mode: host`, `runtime: nvidia`, `shm_size: 2gb`,
  mounts `supersplat` (read-only), `selkies-data`, and the supervisor include below.
- `.env` — all site-specific settings (image, auth password, GPU index, TURN).
  Mode 600; the plaintext password lives here and nowhere else.
- `supervisor-supersplat.conf` — extra supervisord program mounted into the
  image's empty `/etc/supervisor/conf.d/`; serves `supersplat/dist` on
  `127.0.0.1:8082` without rebuilding the image, restart-proof.

## 4. Configure `.env` — the four decisions that matter

1. **`SELKIES_TURN_HOST` / `TURN_EXTERNAL_IP`** — the host IP that *client
   laptops can actually route to*. On a multi-homed host pick the ethernet
   interface (here `10.19.2.98` on bond0), not the InfiniBand one
   (`10.10.100.3` on ibs110). Re-check after any lab re-addressing — a stale
   IP here is invisible until streams fail.
2. **`TURN_EXTRA_ARGS=--relay-ip=<same ethernet IP>`** — REQUIRED on
   multi-homed hosts. Without it coturn binds relay sockets on the
   primary-route interface while advertising the external IP; media
   black-holes (`peer usage: rp=0, rb=0` in the coturn log) and the browser
   sticks at "Waiting for stream" forever.
3. **`SELKIES_TURN_PROTOCOL=tcp`** — office/laptop subnets here can reach TCP
   3478 but high UDP ports are filtered, so media must relay over TCP. Only
   set `udp` if you have verified a UDP path end to end.
4. **`SELKIES_GPU_INDEX`** — pick the GPU with the most *free VRAM*, not the
   least utilization (on this shared box all GPUs burst to 100% anyway; what
   kills WebGL is running out of memory → context lost). Currently `1`.

Also in `.env`: `SELKIES_ENCODER=x264enc` (pinned; see NVENC note above) and
`SELKIES_BASIC_AUTH_USER`/`_PASSWORD`.

## 5. Deploy

```bash
cd ~/codebases/selkies/deploy
docker-compose -f docker-compose.deploy.yml --env-file .env up -d --force-recreate
```

**Always `up -d --force-recreate`. Never revive a stopped container with
`docker start`**: the old rw layer keeps stale `/tmp` X11 sockets, which make
selkies' "wait for X" check false-pass against a dead display — it crashes 5×
in seconds and supervisord gives up (`FATAL`). A fresh container has an empty
`/tmp`, so every readiness gate actually waits. (Nothing of value lives in the
container layer; only `~/selkies-data` persists, on the host.)

Startup takes ~20–30 s. Then launch SuperSplat's browser once per recreate:

```bash
docker exec -d selkies-supersplat bash -c \
  'vglrun -d egl firefox --new-instance http://localhost:8082/ >/tmp/firefox.log 2>&1'
```

(`vglrun -d egl` routes Firefox's GL through VirtualGL to the A100.)

## 6. Verify (in order — each step isolates a layer)

All `curl` from this host needs `--noproxy '*'`: the corporate `http_proxy`
intercepts even `127.0.0.1` and fakes 503s.

```bash
# 1. All 8 supervised programs RUNNING (incl. selkies-gstreamer and supersplat):
docker exec selkies-supersplat supervisorctl status

# 2. Auth + web:  expect 401 then 200
curl --noproxy '*' -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/
curl --noproxy '*' -s -o /dev/null -w '%{http_code}\n' -u jing:<pw> http://127.0.0.1:8080/

# 3. RTC config advertises the right relay:  expect "turn:10.19.2.98:3478?transport=tcp"
curl --noproxy '*' -s -u jing:<pw> http://127.0.0.1:8080/turn | grep -o '"turn:[^"]*"'

# 4. SuperSplat served:  expect <title>SuperSplat
curl --noproxy '*' -s http://127.0.0.1:8082/ | grep -o '<title>[^<]*'

# 5. Relay binds on the advertised interface:  expect 10.19.2.98
docker exec selkies-supersplat grep "Local relay addr" /tmp/selkies-gstreamer-entrypoint.log | tail -2

# 6. GL really reaches the A100:  expect "OpenGL renderer ... NVIDIA A100"
docker exec selkies-supersplat bash -c 'DISPLAY=:20 vglrun -d egl glxinfo -B | grep renderer'

# 7. (Optional) End-to-end TURN relay echo — 100% received proves the media path:
docker exec selkies-supersplat bash -c '
  PW=$(tr "\0" "\n" < /proc/$(pgrep -x turnserver | head -1)/cmdline | grep -m1 "^--user=" | sed "s/--user=selkies://")
  turnutils_uclient -t -u selkies -w "$PW" -y -n 5 10.19.2.98 2>&1 | tail -2'
```

## 7. Use

1. Open `http://10.19.2.98:8080` from your laptop (add the IP to your browser's
   proxy-bypass list if you use the corporate proxy). Log in with the
   credentials from `.env`.
2. The streamed Xfce desktop appears with Firefox already on SuperSplat
   (`localhost:8082`). Load/save scenes via `~/Documents` (= host
   `~/selkies-data`).
3. Sanity check inside the stream: SuperSplat's WebGL renderer will report an
   NVIDIA string (Firefox masks the exact model as "GeForce 8800 GTX, or
   similar" for anti-fingerprinting — NVIDIA = A100; `llvmpipe`/"Software"
   would mean CPU fallback).

## 8. Operations

| Task | Command |
|------|---------|
| Stop | `docker stop selkies-supersplat` (stays down across reboots) |
| Start / apply `.env` changes | the `up -d --force-recreate` from §5, then relaunch Firefox |
| Change password | edit `.env`, recreate |
| Change GPU | edit `SELKIES_GPU_INDEX`, recreate |
| Logs | `docker exec selkies-supersplat sh -c 'ls /tmp/*.log'` — supervisord children log to `/tmp/*.log` in the container; `/tmp/selkies-gstreamer-entrypoint.log` holds both selkies and coturn output |

## 9. Troubleshooting

| Symptom | Cause → fix |
|---------|-------------|
| Browser stuck at spinner, "Waiting for stream" | ICE/media failure. Check coturn lines in `/tmp/selkies-gstreamer-entrypoint.log`: allocations with `peer usage: rp=0, rb=0` + `allocation timeout` = relay black hole → verify `--relay-ip` (§4.2) and `transport=tcp` (§4.3). Client side: Firefox `about:webrtc` shows which candidate pair stalls. |
| `selkies-gstreamer FATAL` in supervisorctl | Stale container state after `docker start` → recreate (§5). |
| `curl` to the service returns 503 | You forgot `--noproxy '*'` — that 503 is the corporate proxy, not selkies. |
| Stream connects but no video / instant disconnects | Someone set `SELKIES_ENCODER=nvh264enc` — A100 has no NVENC; restore `x264enc`. |
| SuperSplat viewport black / "context lost" on big scenes | GPU out of VRAM — move to the GPU with most free memory (§4.4). |
| Web UI loads but spinner forever *and* no `/turn` request in nginx log | The app's `fetch("/turn")` promise died silently (it has no `.catch`). Usually a client-side quirk — e.g. Firefox with `user:pass@` embedded in the URL. Log in via the auth prompt instead. |
| Port 8080/3478 already in use at start | Host networking shares ports with every other service on this box — `ss -ltnp | grep -E ':8080|:3478'` and evict or re-port. |
| Everything worked last month, dead today | Check `ip -4 addr` — if the lab re-addressed the host, update the three IPs in `.env` (§4.1–4.2) and recreate. |

## 10. Known limitations

- Stream encoding is CPU x264: heavy host CPU load (training dataloaders)
  degrades stream FPS even when the GPU is idle.
- GPU compute is shared with training jobs: SuperSplat frame rate dips when
  they saturate the SMs. VRAM headroom, not utilization, decides which GPU to
  pin (§4.4).
- Anything opened inside the desktop dies on recreate; only `~/Documents`
  survives. Firefox must be relaunched after each recreate (§5) — if that
  becomes annoying, promote it to another supervisord include like
  `supervisor-supersplat.conf`.
