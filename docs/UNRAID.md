# Beltr on Unraid (and other Docker hosts)

Beltr as a container: a headless karaoke server for a homelab. Same backend as
the desktop apps, no Electron — the TV screen, the host dashboard and the phone
remotes are all just web pages this container serves.

---

## What you get

| URL | What it is | Who opens it |
|---|---|---|
| `http://<server>:8477/dashboard` | Host/admin: library, settings, imports | You, from a laptop |
| `http://<server>:8477/tv` | The lyrics screen, with a QR code | Whatever screen you're singing at |
| `http://<server>:8477/remote` | Phone remote | Guests, by scanning the QR |

Everything runs locally. Songs never leave the server. Lyrics are looked up
against [LRCLIB](https://lrclib.net) unless you point `LRCLIB_BASE_URL` at your
own mirror; if no provider has a song, Beltr transcribes it locally.

---

## Install on Unraid

### Community Applications

1. **Apps** → search **Beltr**.
2. Pick **Beltr** (CPU) or **Beltr-NVIDIA** (needs an NVIDIA GPU — see below).
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

Add `--gpus all` and use `ghcr.io/casavargas/beltr:latest-cuda` for the GPU
image.

### Compose

[`docker-compose.yml`](../docker-compose.yml), with
[`.env.example`](../.env.example) as a starting point:

```bash
cp .env.example .env    # edit PUID/PGID and the paths
docker compose --profile cpu up -d    # or --profile gpu
```

Pick one profile. Both publish the same port and share the same volumes.

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
  a pair of lossless FLAC stems. A few hundred songs will fill a cache pool
  sized for config files. This is the whole reason `/config` and `/library` are
  separate mounts.
- **`/cache` must stay mapped.** It is where the ~2 GB of model weights land.
  Unmapped, they live inside the container and are re-downloaded every single
  time you update the image. On slow storage, model *loading* is noticeably
  slower at the start of each job — this is a reasonable thing to put on an SSD.

`/media` is mounted read-only on purpose. Beltr copies a file into `/library`
when it imports it and never writes back to your music share.

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

# 2. Does torch?
docker exec beltr-gpu python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"

# 3. Watch nvidia-smi on the host while a song separates — you should see
#    the python process appear and VRAM climb.
```

The driver must be new enough for CUDA 12.8 (roughly 525+); the image ships
PyTorch built against cu128 because earlier CUDA builds have no kernels for
RTX 50-series cards.

**AMD and Intel GPUs are not supported.** Not an oversight: Beltr's separation
model runs a complex-valued STFT that the DirectML/ROCm paths either abort on
or have never been validated against. They would be slower than the CPU path
and less reliable.

---

## First run

The image ships **no AI model weights** beyond the small separation model. On
first use Beltr downloads roughly 2 GB into `/cache`:

| Model | Size | When |
|---|---|---|
| Demucs (htdemucs) | ~80 MB | Bundled — already there |
| Word alignment (wav2vec2) | ~360 MB | First song processed |
| Transcription (Whisper `large-v3`) | ~1.5 GB | First song **no lyrics provider knows** |

This is once, and it survives image updates as long as `/cache` stays mapped.

If a first song looks stuck, check the log — the download is what it's doing:

```bash
docker logs -f beltr
```

You'll see a line at startup naming what's cached and what isn't:

```
Model cache /cache: Whisper large-v3; word alignment (wav2vec2, ~360 MB) will download on first use
```

Whisper's download also shows as a progress bar in the dashboard.

---

## How long does a song take?

Measured on a **4-core CPU** against a **3.5-minute track**, from this repo's
own benchmarks (`bench/results_separation.json`, `bench/results_postsep.json`):

| Stage | CPU (4 cores) | With an NVIDIA GPU |
|---|---|---|
| Vocal separation (Demucs) | **86 s** | seconds |
| Word alignment (wav2vec2) | **89 s** | seconds |
| Pitch analysis | 9 s | faster |
| Lyrics from a provider (LRCLIB) | instant | instant |
| Lyrics via local transcription | **the slow path** — see below | much faster |

So on a 4-core box, a song whose lyrics are already in LRCLIB lands in **about
three minutes**. A modern 8- or 16-core server does better; both dominant
stages scale with cores.

**Transcription is the case that hurts, and it's the uncommon one.** Beltr
tries lyrics providers first and only transcribes when none has the song — so
most of a mainstream library never runs Whisper at all. When it does run,
`large-v3` on CPU is by far the longest stage in the pipeline. If you hit it
often (obscure tracks, non-English, live recordings), set:

```
WHISPER_MODEL_SIZE=large-v3-turbo
```

Substantially faster, slightly less accurate on difficult vocals. On a GPU
there's little reason to move off `large-v3`.

Songs are processed **one at a time** by design — the separator runs a
sequential queue so two jobs can't race each other into an out-of-memory kill.
Queue a batch and leave it; it works through them.

---

## Settings worth knowing

| Variable | Default | Why you'd change it |
|---|---|---|
| `BELTR_LICENSE_KEY` | unset | Your key from beltr.app — see Licensing above |
| `PUID` / `PGID` | `1000` (Unraid template: `99` / `100`) | Must match whoever owns your shares, or Beltr can't write |
| `TZ` | `UTC` | Log timestamps |
| `KARAOKE_HOST_PIN` | random | **Set this.** Unset, the PIN that unlocks host controls on a phone changes every restart |
| `AUTH_PASSWORD` | unset | Password on the TV/dashboard screens — see the security note below |
| `WHISPER_MODEL_SIZE` | `large-v3` | `large-v3-turbo` on CPU-only servers |
| `DEMUCS_MODEL` | `htdemucs` | `htdemucs_ft` is ~4× slower for a small quality gain — reasonable on a GPU |
| `LRCLIB_BASE_URL` | lrclib.net | Point at your own LRCLIB mirror |
| `BELTR_FIX_PERMS` | `first-run` | `always` if ownership keeps drifting; `never` if you manage it yourself |
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

Three ways out, best first:

1. **Put it behind HTTPS.** A reverse proxy (Nginx Proxy Manager, Caddy,
   Traefik — all in Community Applications) with a certificate, then open Beltr
   at that name. Tailscale Serve and Cloudflare Tunnel also produce a real
   HTTPS origin. This is the only one that fixes it for every device at once,
   phones included.
2. **Tell your browser to trust the origin.** Per-browser, per-device, and it
   survives restarts:
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
Run the three checks in the GPU section above. Most often it's a missing
`--runtime=nvidia`, a `NVIDIA_VISIBLE_DEVICES` UUID typo, or the Nvidia-Driver
plugin not loaded after an Unraid update. Also confirm you installed the
**Beltr-NVIDIA** template: the CPU image has no CUDA build of PyTorch and will
never report a GPU.

When it *is* working, the GPU panel reads **Built in** with your card's name and
a torch/CUDA line — the container ships CUDA PyTorch, so there is no "GPU pack"
to download here and no install button. That is the correct state, not a
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
Fixed in images built after this note — the container now ships Node, which
yt-dlp needs to solve YouTube's player challenges. If you see it, you're on an
older image: pull the current one and recreate the container.

**"Mic unavailable: browsers only allow microphone access over HTTPS or on localhost."**
Not a bug and not fixable from inside Beltr — see "Microphones need HTTPS"
above for the three ways out. Everything except live mic capture works normally
on a plain-http page.

**Everything is slow and the log mentions Whisper.**
You're hitting the transcription path. See the timings section — the fix is
usually `WHISPER_MODEL_SIZE=large-v3-turbo` or a GPU.

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
