# Putting Beltr behind a reverse proxy

Beltr's desktop **Remote access → Reverse proxy / my own address** mode, and any
self-hosted install, expect you to run the proxy yourself (nginx, Caddy,
Traefik, Nginx Proxy Manager, Tailscale Serve, a plain port forward). Beltr does
nothing extra in that mode: it only learns which address to print on the TV and
into QR codes so a phone joining from outside gets a link that works.

The requirements below are not optional polish. Every one of them is a hard
constraint in the code, and most fail *silently* if you get them wrong: the room
looks alive, phones connect, and nothing ever moves. Read the whole page before
you write a proxy config.

If all you want is a working microphone on your LAN, **you no longer need a
proxy**: the container serves self-signed https on port 8478 out of the box
(open `https://<server>:8478/tv`, accept the warning once) — see
[the Unraid microphone note](UNRAID.md#microphones-need-https). A proxy is the
way to get a *trusted* certificate, i.e. no warning on any device; see
[HTTPS is what makes phone mics work](#https-is-what-makes-phone-mics-work).

## The rules, shortest first

1. Serve Beltr at the **root of its own hostname**, never a subpath.
2. Forward the **WebSocket upgrade on `/ws`**.
3. Let **byte-range requests** through **unbuffered**.
4. Turn off **response buffering** and set **generous read timeouts**.
5. Use **HTTPS** if you want phone microphones to work.
6. Do **not** put the proxy's own auth (basic auth, Authelia, oauth2-proxy) in
   front of Beltr. Use Beltr's `AUTH_PASSWORD` instead.

## 1. Root of a hostname, not a subpath

Beltr's frontend uses **absolute** paths. `frontend/js/api.js` fetches
`/api/...`, and `frontend/js/ws-client.js` builds its socket URL from
`location.host + '/ws'`. Neither knows about a path prefix you invented.

That means `https://home.example.com/beltr/` will **not** work — the browser
loads the page at `/beltr/` but then asks for `/api/songs`, `/ws`, `/static/...`
at the domain root, which your proxy is not serving. This is the first thing
most people try, and it breaks in confusing ways.

Give Beltr its own name: `https://karaoke.example.com`, or a subdomain like
`https://sing.home.example.com`. Point the whole hostname at the container's
`http://<beltr-host>:8000` and forward everything under `/`.

## 2. Forward the WebSocket upgrade on /ws

Every client — the web remote, the TV page, the native Apple TV / Google TV
apps, the phone remotes — talks to Beltr over a WebSocket at `/ws`. If the proxy
does not forward the `Upgrade` and `Connection` headers for that path, the
socket never opens.

This is the **worst** failure mode Beltr has, because it looks like success:
the HTML and REST both work, so phones load the join screen and appear to join,
the room shows connected devices, and then nothing happens. No song advances, no
score arrives, no lyric moves. Always test that a phone can actually queue a
song and see the TV react, not just that the page loaded.

### nginx

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 443 ssl;
    server_name karaoke.example.com;

    # ... your ssl_certificate / ssl_certificate_key ...

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket upgrade (rule 2)
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        # No buffering, long timeouts (rules 3 and 4)
        proxy_buffering off;
        proxy_request_buffering off;
        proxy_read_timeout  3600s;
        proxy_send_timeout  3600s;
    }
}
```

The `location /` block covers `/ws` too — you do not need a separate one. The
`map` block goes at the `http {}` level, once.

### Caddy

Caddy forwards WebSocket upgrades, streams byte ranges, and does not buffer, all
by default. A one-liner is enough:

```caddy
karaoke.example.com {
    reverse_proxy 127.0.0.1:8000 {
        # Long-running separation/lyrics jobs — raise the read timeout.
        transport http {
            read_timeout 3600s
        }
    }
}
```

Caddy also gets you an automatic Let's Encrypt certificate, which satisfies
rule 5 with no extra work.

### Traefik

Traefik forwards WebSockets automatically; the piece you must add is a long
timeout so separation and lyrics jobs are not cut off. On the static config:

```yaml
entryPoints:
  websecure:
    address: ":443"
    transport:
      respondingTimeouts:
        readTimeout: "3600s"
        idleTimeout: "3600s"
```

Router / service (labels or file provider), pointing the whole host at Beltr:

```yaml
http:
  routers:
    beltr:
      rule: "Host(`karaoke.example.com`)"
      entryPoints: [websecure]
      tls: {}
      service: beltr
  services:
    beltr:
      loadBalancer:
        servers:
          - url: "http://127.0.0.1:8000"
```

## 3. Byte-range requests must pass through unbuffered

Stem and CDG audio is served from `/api/audio/...` and is **seeked** — the
player issues HTTP `Range` requests to jump around a track. A proxy that
buffers the whole response before sending it, or that strips `Range` / `Accept-Ranges`,
breaks scrubbing and can stall playback on large stems.

The nginx snippet handles this with `proxy_buffering off`. Caddy and Traefik
pass ranges through untouched by default. If you use a different proxy, make
sure range requests reach Beltr and the `206 Partial Content` responses reach
the client.

## 4. No response buffering, generous read timeouts

Separation and lyrics are **long-running**. Preparing a song can take minutes,
and the endpoints stream progress while they work. A proxy with the usual 60-second
gateway timeout will cut the connection mid-job, and a proxy that buffers
responses will hide the progress the UI is waiting on.

Set the read/send timeout to an hour (`3600s` above is a safe ceiling) and turn
buffering off. The snippets above already do both.

### If this install shares its separation

An install with **Share this computer's separation** switched on receives whole
songs by upload, up to 200 MB, on `/api/helper/jobs`. Most proxies cap request
bodies far below that and answer `413` long before Beltr sees the file, which
looks on the other machine like the helper simply rejecting every job. Raise the
limit: nginx `client_max_body_size 200m;`, Caddy `request_body { max_size
200MB }`. Traefik has no body limit unless you added the `buffering` middleware,
in which case set `maxRequestBodyBytes: 209715200`.

## 5. HTTPS is what makes phone mics work

This is the single biggest reason to put a proxy in front of Beltr at all.

Browsers only expose the microphone API in a **secure context** — an `https://`
page, or `localhost`. Reached over plain `http://` on the LAN, the microphone
API is not permission-denied, it is simply **absent**, so singing scores, the TV
mic panel and the phone-as-mic feature all go dark. No setting inside Beltr can
grant the browser that permission; only a secure origin can.

Beltr's own self-signed listener on port 8478 already gives any laptop or
phone a secure context after one click through the browser's warning. A
reverse proxy with a real certificate (or Caddy's automatic one, Tailscale
Serve, Cloudflare Tunnel) removes that warning for every device at once, and
lets the QR itself point at an `https://` address. That is what this page buys
you on top of the built-in listener.

The plain-`http://` fallbacks — per-browser insecure-origin flags, opening the
server on its own `localhost` — are covered in
[the Unraid microphone note](UNRAID.md#microphones-need-https).

## 6. Do not put the proxy's own auth in front of Beltr

It is tempting to wrap Beltr in basic auth, Authelia, or oauth2-proxy so the
whole thing is "protected." **Do not.** The phone remotes and the WebSocket have
no way to carry those credentials — there is no login prompt in the native apps,
and a browser cannot attach basic-auth to a WebSocket handshake it did not
initiate with a URL you control. Proxy-level auth locks out exactly the clients
the party needs: the phones and the TV apps all break.

Use Beltr's own gate instead. Set `AUTH_PASSWORD` (see
`backend/services/web_auth.py`) and be precise about what it covers:

- **Gated:** `/` and `/dashboard` (the host pages) and the management endpoints
  under `/api/admin`, `/api/config`, `/api/gpu`, `/api/wizard`,
  `/api/preferences`, `/api/support`, `/api/music-library`, `/api/export`,
  `/api/library/sources`, `/api/library/detect`, and `/api/trial/activate`.
- **Deliberately open:** the party surface phones use — joining a room, browsing,
  queuing, singing, scoring — and `/ws` itself, which is the room-join transport
  and is already gated by the 4-character room code. `/tv` is open too: the
  native TV apps render the same stage off open endpoints, so gating the page
  would only lock out the one browser TV.

That split is intentional. `AUTH_PASSWORD` keeps the household out of your
settings; it is **not** internet-grade authentication, because the party surface
it leaves open is unauthenticated by design. Which brings us to the standing
warning.

## What to type into each client

Once the proxy is up, point each device at the **full `https://` hostname** —
scheme included, no port when the proxy is on the standard 443.

| Client | Where | What to enter |
| --- | --- | --- |
| **Apple TV** | Connect screen, **Enter address manually** | `https://karaoke.example.com` |
| **Google TV / Android TV** | Connect screen, **Enter address manually** | `https://karaoke.example.com` |
| **Phone remote (iOS / Android app)** | **Enter a code** → **Server URL** | `https://karaoke.example.com` |
| **Phone (web remote)** | Just open the link | scan the QR code, or open `https://karaoke.example.com` |

The address parsers on every client agree on the shape: when you type an
`https://` host with no port, they use it as-is and talk to the proxy on 443. A
bare address with no scheme (`192.168.1.50`) still means the LAN default —
`http://` on port `8000` — so behind a proxy you **must** type the full
`https://` form. A non-standard proxy port is appended as usual
(`https://karaoke.example.com:8443`).

For the TV apps this depends on the app build accepting an `https://` address;
older builds only took a LAN `host:port`. If a TV app rejects the proxy address,
it is on an older version — update it, or reach that TV over the LAN and let
phones use the proxy.

## The standing warning, repeated

A reverse proxy gives you TLS and a name. **It does not, by itself, make Beltr
safe to expose to the open internet.**

Beltr is trusted-LAN by design (see `CLAUDE.md`, "Auth model: trusted LAN
only"). Its write endpoints — lyrics correction, song upload, search rewrite,
the ASR trigger — are not PIN-gated, and the host PIN is itself fetchable over an
open endpoint. `backend/config.py` logs a warning at startup whenever
`BELTR_PUBLIC_URL` is set, for exactly this reason.

So: put Beltr behind a proxy for TLS (phone mics) and a friendly name, on a
network you trust — your LAN, or a private overlay like Tailscale where only your
own devices can reach it. Do **not** port-forward it to the public internet and
rely on the proxy alone. If you genuinely need it reachable from anywhere, put a
real authentication layer in front — but remember that layer breaks the phones
and TV apps (rule 6), so in practice a VPN or Tailscale, which authenticates the
*network* rather than each HTTP request, is the sane way to reach a Beltr party
from outside.
