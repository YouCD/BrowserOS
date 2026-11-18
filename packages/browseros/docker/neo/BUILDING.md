# Building a custom browseros-neo Docker image

Working notes from building a local `browseros-neo` image with a patched
`claw-server` and full Chinese/emoji font support — **without rebuilding the
Chromium fork**. Everything here was hit in practice; the gotchas are the point.

## Two build paths — use the right one

BrowserOS has two build paths and only one is relevant for agent-side work:

- `packages/browseros` — the Chromium fork (`bos_build`, `gn gen` + `autoninja`).
  Slow (hours) and produces the browser `.deb`. **Only rebuild this if you touch
  the browser C++.**
- `packages/browseros-agent` — the agent platform (Bun + Rust). `claw-server`
  (the MCP/HTTP server) lives here.

For any agent-side change (MCP behaviour, claw-server config, etc.), **do not
rebuild Chromium**. Ship the published `.deb` and inject a custom `claw-server`
binary via `--server-dir`. The Dockerfile's `neo-server` context replaces the
bundled `browseros-claw-server` in place (the browser launches that path, so the
replacement takes effect with no browser change).

## Prerequisites (local)

- The published `.deb` for the target version, e.g.
  `BrowserOS_neo_v0.51.2.1_amd64.deb` (from the CDN; on this host it is fetched
  through the `127.0.0.1:7890` proxy).
- A custom `claw-server` built from `packages/browseros-agent`, placed at
  `<server-dir>/amd64/browseros-claw-server-rs`.

## glibc gotcha — build claw-server inside `rust:bookworm`

`claw-server` is a Rust binary. If it is built on a host with a newer glibc
(e.g. Arch, glibc 2.44) it will not start inside the Ubuntu 24.04 container
(glibc 2.39): `version 'GLIBC_2.xx' not found`. Build it inside `rust:bookworm`
(glibc 2.36), which produces a binary needing at most `GLIBC_2.35`:

```sh
docker run --rm \
  -v "$PWD/packages/browseros-agent:/src" \
  -v "$HOME/.cargo:/root/.cargo" \
  -v /tmp/clawbuild-out:/out \
  -e CARGO_HOME=/root/.cargo -e CARGO_TARGET_DIR=/build/target \
  rust:bookworm \
  bash -c "cd /src && cargo build --release --offline -p claw-server \
    && cp /build/target/release/browseros-claw-server-rs /out/"
```

`--offline` reuses the host's `~/.cargo` registry cache; the build takes ~4–5 min.
Confirm the result before injecting:

```sh
objdump -T /tmp/clawbuild-out/browseros-claw-server-rs | grep -oE 'GLIBC_[0-9.]+' | sort -V | tail -1
# want GLIBC_2.35 (or lower), never 2.4x
```

## Fonts: Chinese + emoji

The `neo-runtime` stage's `apt-get install` already ships
`fonts-liberation fonts-noto-core fonts-noto-color-emoji fonts-wqy-zenhei
fonts-ipafont-gothic`. `fonts-wqy-zenhei` (WenQuanYi) is older and misses some
glyphs, so for complete Simplified/Traditional Chinese (plus Japanese/Korean)
add `fonts-noto-cjk`:

```dockerfile
libasound2t64 fonts-liberation fonts-noto-core fonts-noto-color-emoji \
fonts-noto-cjk fonts-wqy-zenhei fonts-ipafont-gothic
```

- `fonts-noto-cjk` — Noto Sans/Serif CJK SC/TC/JP/HK/KR. This is what makes
  modern Chinese render fully. Keep `fonts-wqy-zenhei` alongside it.
- `fonts-noto-color-emoji` — color emoji. This is what makes emoji render; do
  not drop it.

Verify after building (override the entrypoint, otherwise it boots the browser):

```sh
docker run --rm --entrypoint bash browseros-neo:v0.51.2.1 -lc \
  'dpkg -l | grep -E "fonts-noto-cjk|fonts-noto-color-emoji|fonts-wqy-zenhei"'
```

## Build + tag

```sh
cd packages/browseros
uv run browseros release docker \
  --deb /tmp/BrowserOS_neo_v0.51.2.1_amd64.deb \
  --server-dir /tmp/neo-server        # holds amd64/browseros-claw-server-rs
# -> ghcr.io/browseros-ai/neo:0.51.2.1 (linux/amd64)

docker tag ghcr.io/browseros-ai/neo:0.51.2.1 browseros-neo:v0.51.2.1
```

The tag is `<repository>:<version>` (`--repository`, default
`ghcr.io/browseros-ai/neo`). There is no direct `--tag` flag, so re-tag with
`docker tag` for a custom name like `browseros-neo:v0.51.2.1`. `--deb` +
`--server-dir` keep it local (never pushed).

## CLAW_LISTEN_HOST — exposing MCP to the network

`claw-server` binds `127.0.0.1` by default. A `CLAW_LISTEN_HOST` env var
(default `127.0.0.1`, non-loopback is opt-in) lets you bind `0.0.0.0`. On a
non-loopback bind, the MCP Host gate is relaxed so external clients can
connect.

Two gotchas, both bitten:

1. **The browser starts the claw-server, and a fresh container works.** A
   container started with `CLAW_LISTEN_HOST=0.0.0.0` binds `0.0.0.0:9210` and
   opens MCP — verified with a clean test container. The entrypoint passes the
   env through, so no entrypoint change is needed.
2. **A typo'd var name silently does nothing.** A container with
   `LAW_LISTEN_HOST=0.0.0.0` (missing the `C`) reads an unset `CLAW_LISTEN_HOST`
   → defaults to loopback → external MCP is `403 Forbidden: Host header is not
   allowed`. This exact typo caused a long debug. Verify the exact name:

   ```sh
   docker inspect <c> --format '{{range .Config.Env}}{{.}}
   {{end}}' | grep LISTEN_HOST
   ```

## Port topology (why external ports differ)

- Browser proxy: `0.0.0.0:9010` (and `9011`) — forwards to the claw-server.
- claw-server: `127.0.0.1:9210` — loopback, behind the proxy.
- DevTools/CDP: `127.0.0.1:9110`.
- noVNC/websockify: `0.0.0.0:6080`.

So a host mapping `-p 9011:9010` exposes MCP at `<host>:9011/mcp`. The external
Host header (carrying the external port) reaches the claw-server; the
loopback-only Host gate rejects it unless relaxed.

## Where the `/mcp` 403 came from

Not the browser C++ (`browseros_server_proxy.cc` is a dumb TCP byte-pump, no
HTTP parsing). It is **rmcp 3.2.0's `StreamableHttpServerConfig::default()`**:
`allowed_hosts = ["localhost","127.0.0.1","::1"]`, rejecting other Hosts with a
dynamically-built `Host header is not allowed` (so the string is not grep-able
in the binary). The fix is in `claw-server`'s `api/mcp/mod.rs`: build the service
with a `StreamableHttpServerConfig` that calls `.disable_allowed_hosts()` when
the listener is non-loopback. `/system/health` is a different handler that
accepts any Host — which is why health looked "open" while `/mcp` was locked.

## Verify end-to-end

```sh
# MCP from outside, with a real Host
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: <host>:9011" \
  -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"p","version":"1"}}}' \
  http://<host>:9011/mcp
# want 200 with "version":"0.0.95" (the patched build), not 403
```
