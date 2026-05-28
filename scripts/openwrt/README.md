# Hysteria client on OpenWRT (TUN mode, low-memory targets)

This directory contains everything needed to run a **client-only** Hysteria in
TUN mode on a memory-constrained OpenWRT router — for example an arm64
MediaTek Filogic board with 256 MB of RAM.

- [`client.yaml`](client.yaml) — a TUN client config tuned for low memory.
- [`hysteria.init`](hysteria.init) — a procd init script that sets the Go
  runtime memory limits.

## Why TUN mode uses memory (and how it differs from WireGuard)

WireGuard is a Layer-3 packet tunnel: it encrypts IP packets and forwards them,
keeping **no per-connection state**. Its memory use is essentially constant no
matter how many connections cross it.

Hysteria's TUN mode is a Layer-4 **proxy**. A userspace TCP/IP stack
(`sing-tun`) terminates each TCP connection locally and relays it as a separate
**QUIC stream** to the server. Every flow therefore carries its own state:
a terminated socket, a QUIC stream, two relay goroutines, copy buffers, and
flow-control windows. Because a single web page can open dozens of connections
and a whole LAN multiplies that, **memory scales with the number of concurrent
connections** — unlike WireGuard.

This is inherent to the proxy model and the price of Hysteria's censorship
resistance and lossy-link performance. It can't be made as flat as WireGuard,
but it can be bounded. The settings here do exactly that.

## Building a client-only binary

The server command pulls in a large ACME/certmagic/libdns dependency tree that
a client never needs. Build with the `noserver` tag to drop it and the server
core, producing a smaller binary that is friendlier to router flash:

```sh
# arm64 (Filogic, etc.)
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 \
  go build -trimpath -tags noserver -ldflags "-s -w" \
  -o hysteria ./app

# mipsel (older routers)
CGO_ENABLED=0 GOOS=linux GOARCH=mipsle GOMIPS=softfloat \
  go build -trimpath -tags noserver -ldflags "-s -w" \
  -o hysteria ./app
```

On arm64 this drops the binary from ~15.8 MB to ~12.7 MB. The resulting binary
still has every client feature (TUN, SOCKS5, HTTP, forwarding, etc.); only the
`server` subcommand is removed.

## Memory tuning, in order of impact

1. **`GOMEMLIMIT` + `GOGC`** (set in `hysteria.init`). A soft memory ceiling
   makes the Go GC run aggressively as usage approaches it; a low `GOGC` shrinks
   the heap headroom kept between collections. This is the most effective lever
   when connections churn. Defaults here: `GOMEMLIMIT=64MiB`, `GOGC=20`.
2. **QUIC `maxConnReceiveWindow`** (in `client.yaml`). This is a hard ceiling on
   received-but-unread data buffered across *all* streams at once, so it bounds
   receive memory regardless of connection count. Lowered from 20 MB to 4 MB.
3. **QUIC `maxStreamReceiveWindow`**. Per-stream peak buffering, lowered from
   8 MB to 2 MB.
4. **`tun.timeout`**. Reclaims idle sessions (and their buffers/goroutines)
   faster. Lowered to 60s.

The relay path itself uses a pooled 16 KB buffer per direction (instead of a
freshly allocated 32 KB buffer per connection), so allocation churn stays low
even with thousands of short-lived connections.

### Kernel UDP buffers

quic-go asks the kernel for an 8 MB UDP receive and send buffer. On a router
you generally do **not** want to raise `net.core.rmem_max` / `wmem_max` to
satisfy it — the default cap keeps kernel memory modest, and quic-go logs a
harmless warning and proceeds. Leave them alone unless you need more throughput
and have the RAM to spare.

## Deploy

```sh
# on the router
mkdir -p /etc/hysteria
cp hysteria      /usr/bin/hysteria
cp client.yaml   /etc/hysteria/config.yaml   # edit server/auth/tls first
cp hysteria.init /etc/init.d/hysteria
chmod +x /usr/bin/hysteria /etc/init.d/hysteria

/etc/init.d/hysteria enable
/etc/init.d/hysteria start

logread -e hysteria        # check logs
```

Adjust `GOMEMLIMIT` in `/etc/init.d/hysteria` to your board: ~`64MiB` is safe on
256 MB RAM; raise it for more throughput headroom if you have more memory.
