---
sidebar_position: 6
---

# Egress Control

By default a sandbox can reach the public internet (the NetworkPolicy only blocks cluster-internal addresses). For enterprise data-governance you usually want the opposite: a sandbox should reach **only** the hosts its app explicitly needs. vibeD's egress control enforces a **per-app hostname allow-list** that untrusted code cannot bypass.

## How it works

```
sandbox --HTTPS_PROXY--> [Squid egress proxy] --asks--> [vibed-egress-authz]
                              │  allow → dst host          (src podIP → VibedApp → allowedHosts?)
                              │  deny  → 403 (logged)
NetworkPolicy: a sandbox may egress ONLY to the proxy — there is no direct internet path.
```

- The sandbox NetworkPolicy permits egress **only to the Squid proxy** (port 3128) plus DNS. The broad `0.0.0.0/0` rule is removed, so raw sockets and non-HTTP traffic simply can't leave.
- Sandboxes get `HTTP(S)_PROXY` pointed at Squid, so their HTTP libraries route through it.
- For every connection, Squid asks **`vibed-egress-authz`**, which maps the source pod IP → the owning `VibedApp` → its `egress.allowedHosts`, and allows or denies (default-deny). vibeD's **system hosts** (e.g. the S3/MinIO source store) are always allowed so the agent can still pull source. Denials are logged.

## Declaring an app's allowed hosts

Per app, via the deploy metadata or the MCP tool:

```jsonc
// POST /v1/deploy  metadata
{ "name": "my-app", "egress": { "allowed_hosts": ["api.openai.com", "*.internal.example.com"] } }
```

```jsonc
// deploy_artifact (MCP)
{ "name": "my-app", "files": { ... }, "allowed_hosts": ["api.openai.com"] }
```

This lands on `VibedApp.spec.egress.allowedHosts`. Matching is case- and port-insensitive; `*.example.com` matches any subdomain (`a.example.com`, `a.b.example.com`) but **not** the apex `example.com`. **Omitting the list = no external egress** (default-deny).

## Enabling it

Egress control is off by default. Turn it on alongside vibeD's own sandbox NetworkPolicy:

```yaml
runtime:
  sandboxNetworkPolicy: Unmanaged   # vibeD owns the policy
networkPolicy:
  enabled: true                     # the locked-down sandbox policy
egressControl:
  enabled: true
  systemHosts:                      # always reachable (don't lock yourself out of the source store)
    - s3.us-east-1.amazonaws.com
```

This deploys the Squid proxy + `vibed-egress-authz`, rewrites the sandbox NetworkPolicy to proxy-only egress, and injects the proxy env into sandboxes.

:::tip Source store
With egress control on, the agent pulls source **through the proxy** too — so your `storage.tarball.s3` endpoint host must be in `egressControl.systemHosts`, or source injection will be denied. The chart **auto-includes the in-cluster served-source DNS** when `tarball.backend=served` (the dev/kind backend), and Squid's `Safe_ports` is widened to allow the vibeD Service port (default 8080) — so dev installs work without extra config.
:::

:::caution Non-HTTP egress
The proxy speaks HTTP/HTTPS (CONNECT) only, and the NetworkPolicy blocks everything else — so a sandbox can't open arbitrary raw TCP/UDP connections at all. That's intentional for web apps; workloads needing other protocols aren't supported under egress control.
:::

## Observing denials

`vibed-egress-authz` logs each denied connection (`egress denied src=… host=…`), and Squid's access log records allow/deny per request. These feed the deploy/egress **audit trail** (a later governance milestone).

## Hardening posture (defaults)

The egress chokepoint isn't worth running if the chokepoint itself is the soft spot, so v0.4.0 pins this:

- **Proxy pod**: runs `runAsNonRoot: true` as the Squid uid (`13`), with `readOnlyRootFilesystem: true`, `cap_drop: [ALL]`, and `seccompProfile: RuntimeDefault`. Ephemeral `emptyDir` mounts cover the writable paths Squid actually needs (`/var/spool/squid`, `/var/run`, `/var/log/squid`). No service-account token is mounted.
- **Squid base image** is pinned to `ubuntu/squid:edge` (a stable rolling tag) rather than `:latest`. Swap to a digest pin on the next maintenance cycle for full immutability.
- **DNS-rebinding mitigation**: Squid's `positive_dns_ttl 1 hour` + `negative_dns_ttl 5 minutes` clamp the window where an attacker's resolver could swap the IP behind an allow-listed hostname between the authz approval and the CONNECT.
- **Sandbox DNS lockdown**: when `networkPolicy.enabled: true`, sandbox egress on port 53 is scoped to CoreDNS pods only (`kubernetes.io/metadata.name=kube-system + k8s-app=kube-dns`). This blocks DNS-tunnel exfiltration where a sandbox would otherwise resolve via `8.8.8.8` or an attacker-controlled server. Override `networkPolicy.dnsSelector` if your cluster uses different DNS labels (some managed providers do).

## Debugging the proxy

If a deploy unexpectedly stays in `Starting`, or you suspect the proxy is denying something it shouldn't, flip the proxy into verbose mode:

```yaml
egressControl:
  debug: true
```

This sets `VIBED_SQUID_HELPER_DEBUG=1` on the egress-proxy pod. The helper then prints every raw line Squid sends on stdin plus the parsed `(src, host)` fields to stderr — visible via `kubectl logs deploy/vibed-egress-proxy -n vibed-system`. Pair with the authz pod (`kubectl logs deploy/vibed-egress-authz -n vibed-system`) to see allow/deny decisions per host. Off in production.
