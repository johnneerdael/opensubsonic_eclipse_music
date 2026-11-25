# Eclipse Subsonic — container distribution

**Bit-perfect Eclipse Music on Android, through USB Audio Player PRO.**

## Why this exists

Eclipse Music serves lossless FLAC from private add-ons, but on a **OnePlus 15
with an iBasso DC4 Ultra DAC** the Eclipse app did not give us bit-perfect USB
output. Other Android players, including **Flick**, did not manage it either. The
one app in our testing that took control of the USB DAC and played the file's
native sample rate and bit depth was **USB Audio Player PRO (UAPP)** by eXtream
Software Development — the €8 app.

UAPP supports OpenSubsonic, so the shortest reliable path was: keep Eclipse as
the source, put its lossless streams behind a private OpenSubsonic server, and
let UAPP's USB driver do the output. This image is that server.

It is not a downloader or a replacement for Eclipse. It signs in to your account,
resolves Home, library and queue through the same add-ons, and serves the
provider's FLAC bitstream to a normal Subsonic client.

## Source and IP

The source repository is private. This repository contains only the compose file
and documentation, and the published image ships an obfuscated bundle with no
source tree. The goal is to protect **eclipsemusic.app's intellectual property**
as much as we reasonably can: keep a readable copy of the add-on contract out of
circulation, the same way Eclipse ships compiled clients rather than source.

Obfuscation raises the cost of casual extraction; it is not a technical or legal
guarantee. Not affiliated with Eclipse Music; Eclipse's terms apply.

## Validation scope

Validated only with our **personal Eclipse supporter account and the paid
supporter add-ons attached to it**. Not tested with free accounts, other add-ons,
other regions, or libraries those providers do not carry.

Matching depends on ISRC/search availability and a clear, non-DRM lossless
stream. If your taste falls outside those catalogs, or a track exists only as a
DRM stream, it may not resolve or play. No coverage or compatibility guarantee.

## The chain

```
Eclipse account + private add-ons
        │  authenticated, lossless FLAC
        ▼
eclipse-subsonic  ── HLS → FLAC stream copy, cache ──►  OpenSubsonic /api
                                                              │
                                                              ▼
                                              USB Audio Player PRO
                                                              │  custom USB driver
                                                              ▼
                                              iBasso DC4 Ultra DAC
```

## What "bit-perfect" means here

- The gateway does not resample or re-encode on the direct path. HLS is
  repackaged once with `ffmpeg -c:a copy`; progressive FLAC is cached
  byte-for-byte.
- UAPP must use its USB DAC output. EQ, normalization, Android mixing, or a
  non-exclusive path can still alter the stream.
- Verify with the DAC's sample-rate/bit-depth display. The gateway controls what
  it serves; the phone/DAC chain controls what comes out.
- Pulsar lossless can be Widevine-only. The gateway skips DRM and resolves the
  same ISRC on a clear provider; it does not decrypt.

This is one private Eclipse account on one private host. It is not a public
service, a redistribution tool, or a way to make a non-bit-perfect app
bit-perfect. Not affiliated with Eclipse, eXtream, or iBasso.

- Image: `ghcr.io/johnneerdael/opensubsonic_eclipse_music:latest`
- Platforms: `linux/amd64`, `linux/arm64`
- Admin UI: `/admin`
- Subsonic API: `/api` (clients append `/rest`)

## Run

```sh
cp .env.example .env
# set ADMIN_PASSWORD
docker compose up -d
```

Then open `http://<host>:4533/admin`:

1. Sign in with the admin password.
2. **Sign in with TV code** and approve at `eclipsemusic.app/tv`.
3. Wait for the library refresh (first run can take a few minutes).
4. Copy the Subsonic URL, username and API key into your client.

Client URL with the default compose file: `http://<host>:4533/api`.

## UAPP setup

1. Add a **Subsonic/OpenSubsonic** server in UAPP: URL `https://<host>/api`,
   username `eclipse`, and the API key from `/admin` (or the password).
2. Select the **USB DAC** as output and enable UAPP's direct/bit-perfect USB mode.
3. Disable EQ, normalization, crossfade, and software volume/DSP you do not want
   in the chain.
4. Play a 16/44.1 and a 24/96 track. The DAC display should follow the file's
   sample rate; if it stays at 48 kHz, something is still resampling.

## Behind Caddy

`deploy/Caddyfile.example` exposes `/api/*` and `/admin*` over TLS. Set
`PUBLIC_URL=https://music.example.com` and `BASE_PATH=/api` so the admin UI shows
the right client address.

## What the image contains

- Node 22, ffmpeg, tini
- `/app/dist/server.mjs` — the obfuscated, minified bundle
- no source tree, no build tools, no tests

The bundle uses string-array/RC4 and control-flow obfuscation, so plain `grep`
does not reveal the endpoint strings. It is still a speed bump, not a secret
store: the strings are decoded at runtime and can be recovered by dynamic
analysis. Packet captures remain possible. The Eclipse account
token lives in `/data` and must be protected.

## Encryption at rest

Encryption is mandatory. The JWT, Subsonic credentials and add-on capability URLs
are AES-256-GCM encrypted in `library.db` using two shares:

1. `<data>/install.key` — generated on first boot and stored with the data.
2. a static code share embedded in the obfuscated image.

The random data key is wrapped with `scrypt(install.key, code share + salt)`, so
the database alone cannot be read without the image, and the image alone cannot
read it without the data. There is no user password and no `data.key`.

Keep `./data` and the image release together: losing `install.key`, or running an
image built with a different code share, makes the JWT and capability URLs
unrecoverable. This does not protect a running host (the key is in memory), and a
technical analyst can extract the static code share.

## Credentials

`ADMIN_PASSWORD` is required. `SUBSONIC_PASSWORD`/`SUBSONIC_API_KEY` are generated
on first boot and printed in the container log if omitted; the admin UI shows them.
Data persists in `./data` (`library.db`, audio and artwork caches).
