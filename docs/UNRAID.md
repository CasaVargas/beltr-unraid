# Beltr on Unraid (and other Docker hosts)

Beltr as a container: a headless karaoke server for a homelab. Same backend as
the desktop apps, no Electron — the TV screen, the host dashboard and the phone
remotes are all just web pages this container serves.

The cpu image is multi-arch: `docker pull ghcr.io/casavargas/beltr` gives you
linux/amd64 on an Intel/AMD box and linux/arm64 on Apple Silicon (Docker
Desktop or OrbStack), a Raspberry Pi 5 or an Ampere VPS. The `-cuda` and
`-openvino` images are amd64 only. On a Mac the container has no access to
CoreML or the GPU, so separation is CPU-only and slower than the native Mac
app; use the `cpu` profile and skip `--device`/`--gpus`.

---

## What you get

| URL | What it is | Who opens it |
|---|---|---|
| `http://<server>:8477/dashboard` | Host/admin: library, settings, imports | You, from a laptop |
| `http://<server>:8477/tv` | The lyrics screen, with a QR code | Whatever screen you're singing at |
| `http://<server>:8477/remote` | Phone remote | Guests, by scanning the QR |

The bare address (`http://<server>:8477`) redirects to the dashboard, and so
does Unraid's **WebUI** button — both when the container was installed from the
Community Applications template and when it was added by hand or by
docker-compose, which have no template to read a WebUI path out of. Older
containers landed on `/tv` in those cases; nothing needs re-creating, the
redirect is server-side.

Everything runs locally. Songs never leave the server. Lyrics are looked up
against [LRCLIB](https://lrclib.net) unless you point `LRCLIB_BASE_URL` at your
own mirror, and force-aligned to the vocal stem for word timing. If no provider
has a song, Beltr drafts the lyrics locally with an on-device speech model
(Parakeet-TDT) and force-aligns that draft instead.

---

## Install on Unraid

### Community Applications

1. **Apps** → search **Beltr**.
2. Pick **Beltr** (CPU), **Beltr-NVIDIA** (an NVIDIA GPU) or **Beltr-Intel**
   (an Intel iGPU or Arc card) — see the two GPU sections below.
3. Set the four paths. Defaults are sensible; the section on shares below
   explains why they are what they are.
4. **Apply**, wait for the pull, then open the WebUI.
5. In **Settings → Music folders**, add `/media`. Beltr does not scan it
   automatically — it will not touch your library until you point it at one.

### Plain Docker

```bash
docker run -d --name beltr \
  -p 8477:8477 \
  -e PUID=99 -e PGID=100 -e TZ=America/New_York \
  -v /mnt/user/appdata/beltr:/config \
  -v /mnt/user/beltr:/library \
  -v /mnt/user/appdata/beltr-cache:/cache \
  -v /mnt/user/Music:/media:ro \
  --restart unless-stopped \
  ghcr.io/casavargas/beltr:latest
```

Add `--gpus all` and use `ghcr.io/casavargas/beltr:latest-cuda` for the NVIDIA
image; add `--device /dev/dri` and use `ghcr.io/casavargas/beltr:latest-openvino`
for the Intel one.

### Compose

[`docker-compose.yml`](../docker-compose.yml), with
[`.env.example`](../.env.example) as a starting point:

```bash
cp .env.example .env    # edit PUID/PGID and the paths
docker compose --profile cpu up -d    # or --profile gpu (NVIDIA), --profile openvino (Intel)
```

Pick one profile. All three publish the same port and share the same volumes.

---

## Licensing

Beltr is a **one-time purchase**, no subscription. The container runs free for
five songs so you can see it work on your own library, then asks for a key.

Three ways to supply one, in the order the server checks them:

1. **`BELTR_LICENSE_KEY`** — the License key field in the Unraid template, or
   the env var in compose. Activates at startup.
2. **`/config/license.key`** — a file containing just the key. Handy if you'd
   rather not put it in a template field, and it survives a template reset.
3. **The dashboard** — paste it into the trial banner. Works from any browser on
   your LAN (unlike the desktop app, where activation is localhost-only). The key
   is written to `/config/license.key` for you, so it survives restarts exactly
   like the other two routes.

Activation happens **once**, against an identity stored at `/config/.machine-id`.
After that the license is cached and revalidated occasionally; it keeps working
offline for weeks if your server has no internet.

**Keep the `/config` volume and you never re-activate.** Updating the image,
recreating the container, changing any other setting — none of that touches the
identity. A key covers two installs, so a spare slot is there for a rebuild or a
second machine.

**Delete `/config` and you spend a slot.** The identity goes with it, so the next
start looks like a new install. If you run out of slots, deactivate an old one
from your license page at beltr.app.

Two things worth knowing up front:

- **Activation needs internet, once.** A fully air-gapped server can't activate.
  After that, outbound access is only used for occasional revalidation, lyrics
  lookups, and artwork — all of which degrade gracefully offline.
- **Nothing about your library is ever sent anywhere.** The license check sends
  your key and the install identity. Song titles, files and usage are not
  transmitted; there is no telemetry.

Check the state any time in the dashboard, or:

```bash
curl -s http://<server>:8477/api/trial/status
```

---

## Volumes, and where to put them

| Mount | Holds | Size | Put it on |
|---|---|---|---|
| `/config` | Database, settings, logs, lyrics | < 500 MB | Cache pool / SSD (`appdata`) |
| `/library` | Separated stems, imported audio, clips | **60–120 MB per song** | The array |
| `/cache` | AI model weights | ~2 GB, once | Cache pool / SSD |
| `/media` | Your music (read-only) | — | Wherever it already is |

Two of these matter more than they look:

- **`/library` must not live in `appdata`.** Every processed song leaves behind
  a pair of stems: AAC by default, about 10 MB a song, or lossless FLAC (about
  40 MB a song) if you pick that under Settings → Preparing songs → How stems
  are stored. Either way a few hundred songs will fill a cache pool sized for
  config files. This is the whole reason `/config` and `/library` are separate
  mounts. `BELTR_STEM_STORE_FORMAT=flac` in the template pre-selects lossless.
- **`/cache` must stay mapped.** It is where the ~2 GB of model weights land.
  Unmapped, they live inside the container and are re-downloaded every single
  time you update the image. On slow storage, model *loading* is noticeably
  slower at the start of each job — this is a reasonable thing to put on an SSD.

`/media` is mounted read-only on purpose. Beltr copies a file into `/library`
when it imports it and never writes back to your music share.

---

## Which GPU, at a glance

GPU acceleration only speeds up **vocal separation** (and, on the NVIDIA image,
local transcription). Everything else, and every image, works on the CPU alone.

| Where Beltr runs | NVIDIA | Intel iGPU / Arc A-series | AMD | Apple Silicon |
|---|---|---|---|---|
| **Container** (this guide) | `latest-cuda` / **Beltr-NVIDIA** | `latest-openvino` / **Beltr-Intel** (6th gen Core through Arrow / Lunar Lake, Arc A; Arc B and Panther Lake not yet) | **Not supported** (see below) | CPU-only: the multi-arch `latest` image runs as linux/arm64 under Docker Desktop or OrbStack, but the VM has no GPU or CoreML |
| **Desktop app** | Optional CUDA GPU pack (Windows, Linux) | Optional DirectML GPU pack (Windows) | Optional DirectML GPU pack (Windows, where the driver supports it) | Built in |

Install one container template, not two: they share the same default paths.

---

## GPU passthrough (NVIDIA)

Worth doing: separation goes from minutes to well under a minute.

1. Install the **Nvidia-Driver** plugin (Apps → search "Nvidia Driver"), then
   reboot.
2. **Settings → Nvidia-Driver** shows your card and its UUID, like
   `GPU-1a2b3c4d-5e6f-...`. Copy it.
3. Install the **Beltr-NVIDIA** template. It already sets
   `--runtime=nvidia` in Extra Parameters.
4. Paste the UUID into `NVIDIA_VISIBLE_DEVICES`. (`all` works on a
   single-GPU server.)
5. Leave `NVIDIA_DRIVER_CAPABILITIES` at `compute,utility`. Beltr needs CUDA
   and `nvidia-smi`; it is not a transcoding container and does not want
   `video` or `graphics`.

**Check that it actually took.** A broken passthrough is not an error — the
image falls back to the CPU and keeps working, so the only symptom is "slow".
Three ways to tell, cheapest first:

```bash
# 1. Does the container see the GPU at all?
docker exec beltr-gpu nvidia-smi

# 2. Does ONNX Runtime have the CUDA provider, and is Beltr asking for it?
docker exec beltr-gpu python -c "import onnxruntime as o, os; print(o.get_available_providers(), '| asking for:', os.environ.get('BELTR_MDX_EP'))"

# 3. Did a separation ACTUALLY run on it? This is the only one that proves
#    dispatch — the list above reports what the build supports, and names CUDA
#    even when the CUDA libraries cannot be loaded at all.
docker exec beltr-gpu sh -c 'cat /config/stems/*/_job.json' | head

# 4. Watch nvidia-smi on the host while a song separates — you should see
#    the python process appear and VRAM climb.
```

In (3), `accelerator` is the provider Beltr asked for and `accelerator_actual`
is what ONNX Runtime accepted. If they disagree, the separation silently ran on
the CPU.

The driver must be new enough for CUDA 12.8 (roughly 525+); the image ships
onnxruntime-gpu built against CUDA 12.8 because earlier CUDA builds have no
kernels for RTX 50-series cards.

---

## GPU passthrough (Intel)

The `-openvino` image runs the separation on Intel integrated graphics — the
GPU inside most Intel desktop and NAS processors since 6th gen Core, N100
boxes included — or an Arc A-series card, through ONNX Runtime's OpenVINO
provider. How much it helps depends on the part: an Iris Xe or Arc-class GPU
gets a 3-4 minute song done in about a minute, while an older UHD 630-class
iGPU is not much faster than a modern CPU. Cheap to try: the image is ~200 MB
bigger than the CPU one and falls back to the CPU if the GPU is unusable.

1. Install the **Intel GPU TOP** plugin (Apps → search "Intel GPU TOP"). It
   loads the `i915` driver; `/dev/dri/renderD128` appears on the host.
2. Install the **Beltr-Intel** template. It already sets `--device=/dev/dri`
   in Extra Parameters. Nothing else goes on the host — the Intel compute
   runtime ships inside the image, and the container joins its service
   account to whatever group owns the device node.
3. Leave **GPU device** at `GPU` unless the box has two Intel GPUs (an Arc
   card beside an iGPU: `GPU.1` picks the second). **GPU precision** stays at
   `FP32`; `FP16` is usually faster and unmeasured for quality — try it and
   listen.

**Check that it actually took**, the same way as NVIDIA — a broken passthrough
only ever looks like "slow":

```bash
# 1. Does the container see a render node, and can its user open it?
docker exec beltr-openvino ls -l /dev/dri
docker logs beltr-openvino 2>&1 | grep -i "joined group"

# 2. Does ONNX Runtime have the OpenVINO provider, and is Beltr asking for it?
docker exec beltr-openvino python -c "import onnxruntime as o, os; print(o.get_available_providers(), '| asking for:', os.environ.get('BELTR_MDX_EP'))"

# 3. Did a separation ACTUALLY run on it? `accelerator_actual` must read
#    OpenVINOExecutionProvider. OpenVINO refuses the session outright when it
#    cannot see a GPU, so a fallback shows up here as CPUExecutionProvider.
docker exec beltr-openvino sh -c 'cat /config/stems/*/_job.json' | head

# 4. Watch the GPU on the host while a song separates:
intel_gpu_top     # from the Intel GPU TOP plugin; the Render/3D bar should move
```

The first separation after install (and after each image update) is slower:
OpenVINO compiles the model's kernels for your GPU and caches them under
`/cache/openvino-cache`. Every separation after that skips the compile.

**Supported parts:** Skylake (6th gen) through Arrow Lake / Lunar Lake, and
Arc A-series. The image carries Intel's 24.35 compute runtime, the last one
that still covers the 6th–10th gen iGPUs most NAS boxes have; Arc B-series and
Panther Lake need a newer runtime and are not supported yet.

**AMD GPUs are not supported.** ONNX Runtime's AMD path (MIGraphX) needs a
full ROCm userspace in the image — several gigabytes — and official support
for the APUs in most AMD NAS boxes is thin. It is on the list, not off it. On
the Windows desktop app, AMD and Intel graphics can use the optional DirectML
GPU pack instead; that path does not exist in the container.

---

## Use another computer's GPU

The container separates on its CPU. If there is a Mac, or a Windows PC with
Beltr's GPU pack, on the same network, that machine can do the separation
instead and the container falls back to itself when it is off.

On the fast machine: Beltr → Settings → Processing → **Share this computer's
separation**. Turn it on and copy the address and token.
On the container: Settings → Processing → **Separation helper**. Paste both,
press **Test**, then turn it on.

Trusted-network model, the same as the host PIN: anyone with the token can send
audio to that machine. Do not expose a sharing install on the internet without a
real auth layer in front of it.

A few things worth knowing before you lean on it:

- **Both installs need the same separation model.** Test says "model mismatch"
  when they differ — usually one of the two is a release behind. A *version*
  difference on its own is only a warning.
- **The order is: this machine's own GPU pack, then the helper, then this
  machine's CPU.** A container has no GPU pack, so it is helper, then CPU.
- **No chaining.** A machine that is itself using a helper does not pass work
  on; a job it is given always runs on the machine that received it.
- **One job at a time, plus one waiting.** Anything beyond that is refused and
  the asking install retries shortly after.
- **Uploads are capped at 200 MB** and have to decode as audio. If the sharing
  install sits behind a reverse proxy, raise the proxy's body-size limit to
  match — see [the reverse-proxy guide](REVERSE-PROXY.md).
- **If the helper is off, unreachable or fails, the song is still prepared** —
  once, on the machine that asked. You lose the speed, not the song.

`BELTR_MDX_HELPER=0` stops an install using a helper at all, whatever its
settings say. There is no environment variable that turns *sharing* on: that is
a deliberate choice made in Settings on the machine doing the work.

---

## First run

The image ships **no AI model weights** beyond the small separation model. On
first use Beltr downloads roughly 2 GB into `/cache`:

| Model | Size | When |
|---|---|---|
| Separation (MDX-Net) | ~65 MB | Bundled — already there |
| Voice activity (Silero VAD) | ~2 MB | Bundled — already there |
| Word alignment (wav2vec2) | ~360 MB | First song processed |
| Transcription (Parakeet-TDT) | ~600 MB | First song **no lyrics provider knows** |

This is once, and it survives image updates as long as `/cache` stays mapped.

If a first song looks stuck, check the log — the download is what it's doing:

```bash
docker logs -f beltr
```

You'll see a line at startup naming what's cached and what isn't:

```
Model cache /cache: word alignment (wav2vec2, ~360 MB), transcription (Parakeet-TDT, ~600 MB) will download on first use
```

Model downloads also show as a progress bar in the dashboard.

---

## How long does a song take?

Measured on a **4-core CPU** against a **3.5-minute track**, from this repo's
own benchmarks (`bench/results_separation.json`, `bench/results_postsep.json`).

> **These figures predate the ONNX migration and have not been re-measured.**
> Every engine in the table below was replaced: Demucs by MDX-Net on ONNX
> Runtime, and local transcription by Parakeet-TDT. MDX is known to be *heavier*
> than htdemucs on CPU (docs/audio-pipeline-migration.md quantifies it for the
> desktop), so treat the separation row as a floor, not an estimate. The GPU
> column is now the accelerated ONNX Runtime rather than CUDA PyTorch.

| Stage | CPU (4 cores) | With an NVIDIA GPU |
|---|---|---|
| Vocal separation (measured on Demucs; now MDX-Net) | **86 s** | seconds |
| Word alignment (wav2vec2) | **89 s** | seconds |
| Pitch analysis | 9 s | faster |
| Lyrics from a provider (LRCLIB) | instant | instant |
| Lyrics via local transcription | **the slow path** — see below | much faster |

So on a 4-core box, a song whose lyrics are already in LRCLIB lands in **about
three minutes**. A modern 8- or 16-core server does better; both dominant
stages scale with cores.

**Transcription is the case that hurts, and it's the uncommon one.** Beltr
tries lyrics providers first and only transcribes when none has the song — so
most of a mainstream library never runs the transcription model at all. When
it does run, Parakeet-TDT on CPU is the longest stage in the pipeline; the
NVIDIA image accelerates it, the Intel image does not (transcription stays on
the CPU there). There is no smaller-model knob any more: the retired
`WHISPER_MODEL_SIZE` is ignored. If you hit this path often (obscure tracks,
live recordings), the practical fix is to paste the lyrics or import an `.lrc`
from the song's detail panel, which skips transcription and goes straight to
alignment.

Songs are **separated** one at a time — the separator runs a sequential queue,
so two separations never run together. Queue a batch and leave it; it works
through them.

Post-processing is a separate matter. On a GPU, Beltr starts the next song's
separation while the previous song's post-processing (vocal analysis, lyrics,
alignment) is still running — the two use different memory (VRAM vs system
RAM), so the peak doesn't add up. On **CPU-only** they'd draw from the same
pool, so Beltr turns that overlap off automatically. You only need to think
about this if you set `BELTR_QUEUE_OVERLAP` by hand.

### If songs fail with no error during a big import

That's the out-of-memory killer, and it means the container's memory limit is
too low. The process is killed outright, so nothing gets logged: no traceback,
no error, and if the whole container goes, no `Beltr server stopped` line
either. Beltr reports the detected limit at startup, warns when it's too low
for the variant you're running, and says on the *next* start whether the
previous run ended without a clean shutdown. Give it **at least 8 GB** for CPU
separation; separation alone peaked around 1.6 GB on Demucs, and MDX-Net is
heavier on the CPU, before lyrics and alignment are counted. Lowering the limit makes this worse, not better.

A support bundle collected after the container has already restarted carries a
`Memory history` section that survives the restart. The live `Memory` section
above it will look healthy, because the kernel's own peak counter dies with the
cgroup — read the history one.

#### A note on the Intel image, and a correction

v1.60.1's notes said the `-openvino` image needs about 10 GB because integrated
graphics borrows from the container's memory. That was wrong, and it is
withdrawn. The kills behind it turned out to be word alignment in Beltr's main
process running a whole sung passage through the model in one go, which every
build does, with or without a GPU. Since v1.60.2 alignment runs in bounded
windows and stays around 1.3 GB regardless of song length. The Intel image
shares the plain image's memory guidance above.

The Intel-specific settings are still worth knowing, for speed rather than
memory:

| Variable | Try | What it does |
|---|---|---|
| `BELTR_MDX_OPENVINO_PRECISION` | `FP16` | Usually faster on Intel GPUs and halves what the GPU holds. Unmeasured for quality here; listen to the result. |
| `BELTR_MDX_OPENVINO_LOAD_CONFIG` | raw JSON | OpenVINO runtime properties, e.g. `{"GPU": {"PERFORMANCE_HINT": "LATENCY", "NUM_STREAMS": "1"}}` (the default). `off` sends none. |
| `BELTR_MDX_OPENVINO_DEVICE` | `CPU` | Takes separation off the GPU entirely while staying on OpenVINO. A bisect tool, not a setting to live on. |
| `BELTR_WARM_SEPARATION_IDLE_S` | `10` | How long the separation worker holds its model between songs. Already the default under a memory limit. |

If you want to rule the GPU out completely, `BELTR_MDX_EP=CPUExecutionProvider`
falls all the way back to plain ONNX Runtime on the CPU.

---

## Settings worth knowing

| Variable | Default | Why you'd change it |
|---|---|---|
| `BELTR_LICENSE_KEY` | unset | Your key from beltr.app — see Licensing above |
| `PUID` / `PGID` | `1000` (Unraid template: `99` / `100`) | Must match whoever owns your shares, or Beltr can't write |
| `TZ` | `UTC` | Log timestamps |
| `KARAOKE_HOST_PIN` | random | **Set this.** Unset, the PIN that unlocks host controls on a phone changes every restart |
| `AUTH_PASSWORD` | unset | Password on the TV/dashboard screens — see the security note below |
| `LRCLIB_BASE_URL` | lrclib.net | Point at your own LRCLIB mirror |
| `BELTR_FIX_PERMS` | `first-run` | `always` if ownership keeps drifting; `never` if you manage it yourself |
| `BELTR_WARM_SEPARATION_IDLE_S` | `60`, or `10` under a memory limit | How long the separation worker holds its model when idle. Lower it if memory is tight, raise it to trade memory for speed on a big import |
| `BELTR_MEMORY_SAMPLE_S` | `15` | How often the memory high-water mark is sampled in a container. Rarely worth touching |
| `ENABLE_*` | mostly on | Individual feature flags — see the main README |

### A note on security, stated plainly

**Beltr assumes everyone who can reach it is trusted.** That is an
architectural assumption, not an oversight: room joins are open by design so a
guest can scan a QR code and start singing without an account.

`AUTH_PASSWORD` puts a password on the host surfaces — the TV and dashboard
pages, and the settings/management endpoints. Phones joining a room are not
affected. It keeps the household out of your settings.

It is **not** internet-grade authentication, because the party surface it
deliberately leaves open is still unauthenticated. Do not port-forward this
container. If you want to reach it from outside your network, put a reverse
proxy with real authentication (or a VPN / Tailscale) in front of it — the same
advice the desktop app gives for its tunnel feature.

If you are setting up that reverse proxy, read
[the reverse-proxy guide](REVERSE-PROXY.md) first. It covers what the proxy
*must* pass through — the `/ws` WebSocket upgrade, byte-range audio, long
separation timeouts — and what to type into each TV app and phone remote.

### Microphones need HTTPS

This one is a browser rule, and it bites self-hosted installs specifically.

Browsers only hand out microphone access in a **secure context**: an `https://`
page, or `localhost`. Reach Beltr at `http://tower.local:8477` — the normal way
to use a server — and the microphone API isn't blocked, it's *absent*. Singing
scores, the TV mic panel and the phone-as-mic feature all depend on it, so all
three go dark. Beltr now says so plainly instead of showing a JavaScript error,
but it cannot grant itself the permission.

Everything else works untouched: queueing, playback, lyrics, stem mixing,
remotes. It is specifically live microphone capture that needs the secure page.

**The built-in way out (default since the container gained it):** Beltr also
serves **https on port 8478** with a self-signed certificate it generates on
first start and keeps in `/config/tls`. Open
`https://tower.local:8478/tv` — or `/remote` on a phone — accept the browser's
one-time warning ("Advanced → Proceed" in Chrome/Edge/Firefox, "Show Details →
visit this website" in iOS Safari), and the microphone works. Beltr's own
"Mic unavailable" message names this address, and the phone remote's mic note
links to it. The QR the TV shows stays plain http on 8477 on purpose, so guests
who only want to queue songs never meet a certificate warning; a guest who
wants to sing follows the link in the mic note. A few notes:

- The certificate is issued once and deliberately **not** reissued when the
  server's IP or hostname changes — every reissue is a fresh warning on every
  device. To reissue anyway, stop the container and delete `/config/tls`. To
  put an extra name or IP in it, set `BELTR_TLS_SANS=karaoke.lan,10.0.0.9`.
- Bring your own certificate with `BELTR_TLS_CERTFILE` + `BELTR_TLS_KEYFILE`
  (paths inside the container); `BELTR_TLS=off` turns the listener off.
- Keep the host and container sides of port 8478 equal: the address Beltr
  shows is built from the container-side number.
- The Apple TV and Android TV apps keep using plain http; nothing changes for
  them.

Three other ways out, for a warning-free setup or a locked-down browser:

1. **Put it behind HTTPS with a real certificate.** A reverse proxy (Nginx
   Proxy Manager, Caddy, Traefik — all in Community Applications), Tailscale
   Serve or Cloudflare Tunnel gives every device a trusted `https://` origin
   with no warning at all. See [the reverse-proxy guide](REVERSE-PROXY.md).
2. **Tell your browser to trust the plain-http origin.** Per-browser,
   per-device, and it survives restarts:
   - Chrome/Edge: `chrome://flags/#unsafely-treat-insecure-origin-as-secure` →
     add `http://tower.local:8477` → Enabled → relaunch.
   - Firefox: `about:config` → set `media.devices.insecure.enabled` and
     `media.getusermedia.insecure.enabled` to `true`.

   Fine for a TV box you control. Do not do it on a shared machine — it lowers
   that browser's guarantees for that origin, not just for Beltr.
3. **Open it on the server itself** at `http://localhost:8477`. Loopback counts
   as secure. Only useful if the server is also the machine driving the TV.

Note that iOS Safari honours no flag equivalent — a phone remote needs option 1
to act as a microphone.

---

## Troubleshooting

**"Permission denied" in the log, or nothing imports.**
`PUID`/`PGID` don't match the owner of your shares. On Unraid that's usually
`99`/`100`. Check with `ls -ln /mnt/user/beltr`, fix the variables, restart. If
ownership is already a mess, set `BELTR_FIX_PERMS=always` for one restart, then
put it back to `first-run`.

**The QR code on the TV points somewhere phones can't reach.**
Beltr advertises whatever address the TV page was opened at. Open `/tv` using
the server's LAN address (`http://tower.local:8477/tv` or
`http://192.168.x.x:8477/tv`) — not `localhost`, which means nothing to a
phone.

**The dashboard says no GPU detected but I passed one through.**
Run the checks in the GPU section above for your vendor. On NVIDIA it's most
often a missing `--runtime=nvidia`, a `NVIDIA_VISIBLE_DEVICES` UUID typo, or the Nvidia-Driver
plugin not loaded after an Unraid update; on Intel, a missing
`--device=/dev/dri` or the Intel GPU TOP plugin not installed (no `i915`, no
render node). Also confirm you installed the GPU template — **Beltr-NVIDIA**
or **Beltr-Intel**: the CPU image ships a CPU-only ONNX Runtime and will never
report a GPU.

When it *is* working, the GPU panel reads **Built in** with an
`ORT CUDAExecutionProvider` (or `ORT OpenVINOExecutionProvider`) line — the
container ships an accelerated ONNX Runtime, so there is no "GPU pack" to
download here and no install button. That is the correct state, not a
half-configured one.

**Native TV apps can't find the server.**
Beltr advertises itself over mDNS, which doesn't cross a Docker bridge network.
Switch the container to host networking if you need discovery. The web UI does
not need it.

**Port 8477 is taken.**
Change the *host* side of the port mapping only. Leave the container port at
8477 — changing `BELTR_PORT` without matching the mapping gives you a container
that starts fine and answers nothing.

**Music-video download fails with "No supported JavaScript runtime could be found".**
yt-dlp needs a JavaScript runtime to solve YouTube's player challenges, and it
arrives with yt-dlp itself: Settings → *Music-video backgrounds* → install the
downloader fetches both halves into `/cache`, where they survive image updates.
Older images (v1.57.9 through v1.60.4) shipped Debian's Node 18 for this, which
yt-dlp refuses — it requires Node 22+ — *and* which made that installer skip the
runtime half as already-present, so the pair could never complete. Pull the
current image, recreate the container, and run the installer again: with no
Node in the way it now installs the runtime it actually needs.

**"Mic unavailable: browsers only allow microphone access over HTTPS or on localhost."**
Not a bug — a browser rule. Open the `https://…:8478` address the message names
on the device with the microphone and accept the warning once; see "Microphones
need HTTPS" above for the details and the warning-free alternatives. Everything
except live mic capture works normally on a plain-http page.

**Everything is slow and the log mentions Parakeet or transcribing.**
You're hitting the transcription path: no lyrics provider had the song. See
the timings section. The fix is to give Beltr the lyrics (paste them or import
an `.lrc` from the song's detail panel) so it only has to align them.

**Out of space mid-import.**
Check `/library`, not `/config`. Stems are the bulk of it, and
`docker system df` won't show them if `/library` is a bind mount.

**"This license has reached its activation limit."**
Both slots on the key are in use. Usually this means `/config` was deleted or
remapped at some point, so an old install still holds a slot. Deactivate it from
your license page at beltr.app and restart the container.

**Licensed, but it dropped back to the trial after a rebuild.**
`/config` didn't survive. The install identity lives at `/config/.machine-id` —
if that path was a container-internal directory rather than a real host mapping,
it was thrown away with the old container. Fix the mapping, then re-activate.

**Activated in the dashboard, but every restart says the trial is used up.**
A bug in images built before this note: a key entered in the dashboard was only
recorded in the license cache, and that cache is ignored when no key is
configured — so the next start fell back to the trial and counted the songs
already in your library against it. Pull the current image and recreate the
container; it recovers the key from the cache on startup and writes it to
`/config/license.key`, with no re-activation and no device slot spent. Check it
took by running `docker exec beltr cat /config/license.key`. If `/config` was
wiped in the meantime, activate once more — repeat activations now reuse the
slot this install already holds instead of spending a new one.

**Trial says "first launch offline — separations blocked" but the server is online.**
Beltr registers with the licensing service once at startup, and it only tries
once per run. If the container started before your network was ready (common on
a NAS that boots everything at once), restart it:
`docker restart beltr`. If it persists, the container genuinely cannot reach the
internet — check DNS from inside it:
`docker exec beltr python -c "import httpx; print(httpx.get('https://beltr.app').status_code)"`.

**Activation says the key is invalid but it works on my desktop.**
Check the container can reach the internet at all
(`docker exec beltr python -c "import httpx; print(httpx.get('https://api.lemonsqueezy.com').status_code)"`).
Activation is the one thing that genuinely requires outbound HTTPS.

---

## Updating

Pull the new image and recreate the container. `/config`, `/library` and
`/cache` are volumes, so your library, settings and downloaded models all
survive — nothing is re-downloaded.

```bash
docker compose --profile cpu pull && docker compose --profile cpu up -d
```

On Unraid, the usual **Check for Updates** → **Apply Update** in the Docker tab.
