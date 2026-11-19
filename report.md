# OpenClaw Gateway Proxy Trust Configuration Report

**Date:** 2026-11-17  
**Project:** OpenClaw  
**Version:** 2026.11.4 (04572cc)

---

## Executive Summary

This report documents the investigation into the `proxy_attribution_required` error when attempting to access the OpenClaw Gateway through a reverse proxy, and explains why "allowing all proxies worldwide" is architecturally incompatible with the gateway's security model.

---

## Problem Statement

### Error Observed
```
{"error":{"message":"Proxy client attribution is required. Configure gateway.trustedProxies narrowly and make the proxy overwrite or safely rebuild forwarded client headers.","type":"proxy_attribution_required"}}
```

### Context
- User attempted to access OpenClaw Gateway from external network via reverse proxy
- Gateway was running on port 19001 (dev profile) with bind=loopback
- Request carried `X-Forwarded-For` headers but was rejected

### Root Cause Analysis

The OpenClaw Gateway uses a **trust walk algorithm** to determine the true client IP when behind a reverse proxy:

1. **Request arrives** from IP `A` (the reverse proxy) with `X-Forwarded-For: B, C, D`
2. **Gateway checks**: Is `A` in `gateway.trustedProxies`?
3. **If trusted**, it walks the X-Forwarded-For chain **right-to-left**
4. **Finds first IP** that is NOT in trustedProxies AND not loopback → that's the client IP
5. **If all IPs are trusted** → unattributable (cannot determine client)

### Why "Allow All Proxies" (`0.0.0.0/0`, `::/0`) Fails

When `trustedProxies` includes the entire internet:
- **Every possible IP** matches the trusted proxy range
- The trust walk sees `1.2.3.4` in X-Forwarded-For
- Checks: Is `1.2.3.4` a trusted proxy? **YES** (matches `0.0.0.0/0`)
- Continues walking... but there are no more hops
- **Result**: No valid client IP can be identified → `proxy_attribution_required`

This is **by design** - the security model requires a clear boundary between:
- **Trusted infrastructure** (your reverse proxies, load balancers)
- **Untrusted clients** (end users, potential attackers)
---

## Current Configuration Status

### Dev Profile Configuration (`~/.openclaw-dev/openclaw.json`)
```json
{
  "gateway": {
    "trustedProxies": ["0.0.0.0/0", "::/0"],
    "allowRealIpFallback": true,
    "mode": "local",
    "auth": {"mode": "none"},
    "bind": "loopback"
  }
}
```

### Gateway Status
- ✅ **Running** on `ws://127.0.0.1:19001` (dev profile)
- ✅ **HTTP** accessible on `http://127.0.0.1:19001`
- ✅ **Control UI** loads correctly
- ❌ **External proxy requests** rejected with `proxy_attribution_required`

### Test Results

| Test Case | Result |
|-----------|--------|
| Direct localhost request | ✅ Works |
| Request with `X-Forwarded-For: 1.2.3.4` from localhost | ❌ Fails (proxy_attribution_required) |
| Request via external reverse proxy | ❌ Fails (same reason) |
---

## Solution: Proper Proxy Configuration

### Required: Configure Actual Proxy IPs

**Do NOT use** `0.0.0.0/0` or `::/0`. Instead, specify your reverse proxy's actual IP(s):

```bash
# Single proxy
node openclaw.mjs config set gateway.trustedProxies '["10.0.0.5"]' --profile dev

# Proxy subnet (e.g., Kubernetes ingress, AWS ALB)
node openclaw.mjs config set gateway.trustedProxies '["10.0.0.0/8"]' --profile dev

# Cloudflare (use their published ranges)
node openclaw.mjs config set gateway.trustedProxies '["173.245.48.0/20","103.21.244.0/22","..."]' --profile dev
```

### Required: Proxy Must Overwrite Headers

Your reverse proxy **must strip and rewrite** forwarded headers:

```nginx
# nginx
proxy_set_header X-Forwarded-For "";
proxy_set_header X-Real-IP "";
proxy_set_header Forwarded "";

proxy_set_header X-Forwarded-For $remote_addr;
proxy_set_header X-Real-IP $remote_addr;
```

```caddy
# Caddy
reverse_proxy localhost:19001 {
    header_down X-Forwarded-For
    header_down X-Real-IP
    header_down Forwarded
    header_up X-Forwarded-For {remote_host}
    header_up X-Real-IP {remote_host}
}
```

### Verification

After correct configuration:
```bash
# Should work - client IP correctly identified as 1.2.3.4
curl -H "X-Forwarded-For: 1.2.3.4" http://127.0.0.1:19001/
---

## Security Implications

### Why This Design?

| Threat | Mitigation |
|--------|------------|
| IP Spoofing | Only trusted proxy IPs can set X-Forwarded-For |
| Rate Limit Bypass | Client IP derived from trusted chain only |
| Audit Trail Corruption | Immutable client attribution |
| Access Control Bypass | IP allowlists use verified client IP |

### Risks of "Allow All" (`0.0.0.0/0`)

1. **Complete IP spoofing** - Anyone can claim any IP
2. **Rate limit evasion** - Rotate X-Forwarded-For headers
3. **Audit log pollution** - False attribution
4. **Access control bypass** - Circumvent IP allowlists

---

## Current Project Status

### Gateway Runtime
- **Process**: Running (PID varies per restart)
- **Port**: 19001 (dev profile) / 3001 (default profile)
- **Bind**: loopback (127.0.0.1, ::1)
- **Auth**: none (dev) / token (default)
- **State**: Healthy, ready for connections

### Configuration Files
| Profile | Config Path | Status |
|---------|-------------|--------|
| dev | `~/.openclaw-dev/openclaw.json` | ⚠️ Incorrect trustedProxies |
| default | `~/.openclaw/openclaw.json` | ✅ Correct (127.0.0.1, ::1, specific proxy IPs) |

### Next Steps Required

1. **User must identify their reverse proxy IP(s)**
   - Check proxy server configuration
   - Check cloud provider (AWS ALB, GCP LB, Cloudflare, etc.)
   - Check Kubernetes ingress controller

2. **Update configuration**
   ```bash
   node openclaw.mjs config set gateway.trustedProxies '["ACTUAL_PROXY_IP_OR_CIDR"]' --profile dev
   ```

3. **Configure reverse proxy to overwrite headers** (critical)

4. **Restart gateway**
   ```bash
   # Ctrl+C to stop, then:
   node openclaw.mjs gateway run --dev --force
   ```

5. **Test external access**
---

## Technical Deep Dive: Trust Walk Algorithm

### Code Path: `src/gateway/net.ts`

```typescript
function resolveClientIp(params) {
  const remote = parseNormalizedIpAddress(params.remoteAddr);
  const ranges = getTrustedProxyRanges(params.trustedProxies);
  
  // If NOT from trusted proxy → direct connection
  if (!isTrustedParsedProxyAddress(remote, ranges)) {
    return remote.toString();
  }
  
  // FROM trusted proxy → walk X-Forwarded-For chain right-to-left
  const forwardedChain = params.forwardedFor?.split(",") ?? [];
  for (let index = forwardedChain.length - 1; index >= 0; index--) {
    const hop = parseIpLiteral(forwardedChain[index]);
    // Found client IP: not trusted, not loopback
    if (hop && hop.range() !== "loopback" && !isTrustedParsedProxyAddress(hop, ranges)) {
      return hop.toString();
    }
  }
  
  // Fallback to X-Real-IP if enabled
  if (params.allowRealIpFallback) {
    return parseIpLiteral(params.realIp)?.toString();
  }
  
  // Cannot determine client IP
  return undefined;
}
```

### Why `0.0.0.0/0` Breaks This

```
trustedProxies = ["0.0.0.0/0"]
Request from: 127.0.0.1
X-Forwarded-For: 1.2.3.4

1. remote = 127.0.0.1 → isTrusted(127.0.0.1, 0.0.0.0/0) = TRUE
2. Walk chain: ["1.2.3.4"]
3. hop = 1.2.3.4 → isTrusted(1.2.3.4, 0.0.0.0/0) = TRUE ← PROBLEM!
4. No more hops, no fallback → return undefined
5. Result: unattributableProxy error
```

---

## Appendix: Configuration Reference

### CLI Commands
```bash
# View current trustedProxies
node openclaw.mjs config get gateway.trustedProxies --profile dev

# Set trustedProxies (replace with your proxy IPs)
node openclaw.mjs config set gateway.trustedProxies '["YOUR_PROXY_IP"]' --profile dev

# Enable X-Real-IP fallback (optional)
node openclaw.mjs config set gateway.allowRealIpFallback true --profile dev

# View full gateway config
node openclaw.mjs config get gateway --profile dev
```

### Default Trusted Proxies (when not configured)
- `127.0.0.1` (IPv4 loopback)
- `::1` (IPv6 loopback)

These are **only for local development** and Tailscale managed ingress.

### Config File Locations
| Profile | Config File |
|---------|-------------|
| default | `~/.openclaw/openclaw.json` |
| dev | `~/.openclaw-dev/openclaw.json` |
| <name> | `~/.openclaw-<name>/openclaw.json` |

---

## Conclusion

The `proxy_attribution_required` error is **not a bug** - it's a security feature working correctly. The gateway cannot safely determine the client IP when `trustedProxies` includes the entire internet.

**Action Required**: Configure `gateway.trustedProxies` with your actual reverse proxy IP address(es) and ensure your proxy overwrites forwarded headers. This is a one-time setup that enables secure external access.

---

*Report generated by OpenClaw Gateway Configuration Analysis*
```