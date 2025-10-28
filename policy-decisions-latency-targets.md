# Technical Architecture Document

## Policy Decisions & Latency Targets in Session-Aware Agent Gateway (SAG)

**how policy decisions plug into runtime enforcement** in a Session-Aware Agent Gateway (SAG), plus **latency targets** you can hold the platform to.

---

# 1) Where policy runs in the request path

**Pattern:** PEP/PDP/PIP with hot-path execution inside the gateway

* **PEP (Policy Enforcement Point):** the SAG’s request pipeline (ingress & egress hooks for HTTP/WebSocket/SSE/MCP frames).
* **PDP (Policy Decision Point):** an in-process policy engine (e.g., OPA/Cedar compiled to WASM) that evaluates rules synchronously.
* **PIP (Policy Information Point):** fast attribute sources:

  * **Local:** cached JWKS, compiled allow-lists, tool schemas, session graph (USID/WSID/ASID/MSID), risk score.
  * **Nearline (cached):** device/workload posture, entitlement snapshots, DLP patterns, audience maps.
  * **Online (rare/guard-railed):** only when a cache miss is acceptable; decisions must degrade gracefully.

**Why this layout?** All per-request checks stay in-process to avoid network hops. Anything that needs the network (token *exchange*, new JWKS, fresh posture) is amortized or pre-fetched, not performed on every call.

---

# 2) Inline decision flow (fast path)

1. **Connection gate (if mTLS):** certificate validation/binding (handshake amortized).
2. **Token validation (no network):**

   * Parse & verify JWT (sig + exp/nbf/aud).
   * **DPoP**: verify proof sig + (`htm`,`htu`,`iat`,`jti`,`ath`) and check **replay** via jti cache.
3. **Session stitch:** correlate `USID`, `WSID`, `ASID`, protocol session (e.g., MCP stream ID) and current risk score.
4. **PDP evaluate (in-process WASM):** rules over user, agent (SVID), actor chain (`act`), resource/tool schema, environment, risk.
5. **Enforce result:**

   * Allow w/ obligations (redact fields, cap tokens, mask headers).
   * **Step-up** (re-auth/consent) for sensitive tools or irreversible actions.
   * Deny + log with decision rationale.
6. **Side effects:** update replay cache, session graph counters, emit decision metrics.

> **No network calls** on the hot path except the target API call itself.

---

# 3) When token **operations** happen

* **JWT verification:** Always inline with local JWKS (refresh JWKS on a timer / cache miss, out of band).

* **DPoP verification:** Inline every request + `jti` uniqueness check (see caching below).

* **mTLS-bound tokens:** Binding validated during TLS handshake; per-request cost is negligible.

* **OBO / Token Exchange:** **Not** per request. Do it at:

  * session establishment,
  * audience change,
  * token nearing expiry,
  * or policy-driven boundary (e.g., crossing from agent to tool domain).
    Cache the exchanged token per `{actor, audience, scope}` for a short TTL.

* **Token introspection:** Avoid for access tokens in hot path; favor self-contained JWT ATs. If you must introspect, do it **asynchronously** to refresh a local “permit cache,” or only on anomalies.

---

# 4) Caching & data structures that keep it fast

* **JWKS cache:** LRU keyed by issuer; async refresh on expiry or `kid` miss.
* **DPoP replay cache:** in-memory ring buffer + approximate filter (e.g., Bloom) to catch dupes; **write-through to Redis/KeyDB** with TTL ≈ access-token TTL to coordinate cluster-wide.
* **Actor-token cache (OBO):** per `{sub, act, audience, scope}` with short TTL (1–5 min), prefetch 80% into TTL.
* **Policy compilation cache:** pre-compile policies to WASM; **partial evaluation** with static inputs (tool schemas/allow-lists).
* **Session graph cache:** co-located in the gateway (lock-free map) with periodic flush to a durable store for audit.

---

# 5) Latency & performance **tolerances** (targets you can adopt)

These are **end-to-end budgets for the gateway’s security work** (not counting the downstream API latency):

| Operation (inline)                      |  p50 target | p95 target | p99 target | Notes                                        |
| --------------------------------------- | ----------: | ---------: | ---------: | -------------------------------------------- |
| JWT signature verify                    |     ≤0.6 ms |    ≤1.2 ms |    ≤2.5 ms | With cached JWKS; EdDSA/ES256 typical.       |
| DPoP proof verify                       |     ≤0.8 ms |    ≤1.8 ms |    ≤3.0 ms | Includes hash of AT (`ath`).                 |
| DPoP `jti` check                        |     ≤0.1 ms |    ≤0.3 ms |    ≤0.8 ms | Local filter + batched async write to Redis. |
| PDP (policy WASM eval)                  |     ≤0.7 ms |    ≤2.0 ms |    ≤4.0 ms | After partial eval; small input JSON.        |
| **Total security overhead (fast path)** | **≤2.5 ms** |  **≤6 ms** | **≤12 ms** | What most requests should see.               |

**Slow-path (amortized, not per request):**

| Operation (off the hot path)        | Expected                                 |
| ----------------------------------- | ---------------------------------------- |
| OBO token exchange (network to STS) | 15–50 ms typical; cache result.          |
| JWKS refresh                        | Background; 10–100 ms network hit, rare. |
| Introspection (if used)             | 10–40 ms—avoid in hot path.              |

**SLO suggestion for the gateway’s *security layer* overhead**

* **SLO:** 99% of requests add **<20 ms**; 99.9% add **<50 ms** under normal load.
* **Throughput planning:** size for **>10k rps/node** with above latencies on commodity x86/ARM (depends on crypto library).

> If your downstream APIs are latency-sensitive, hold the gateway to the **p95 ≤ 8 ms** security overhead target.

---

# 6) What happens on streams (AG-UI / MCP)

* **First frame:** full validation + PDP evaluation (as above).
* **Subsequent frames:** lightweight checks (rate, content filters, tool step-bound policies) using the **session context**; only re-evaluate full policy on **state change** (new tool, new audience, step-up required).
* **Backpressure:** if DLP/content filters exceed budget, apply throttling; for irreversible actions, pause stream and require human consent (step-up).

---

# 7) Failure & degradation rules (explicit)

* **PDP unreachable / policy store stale:**

  * Default **fail-closed** for high-risk routes (tools with write/delete).
  * **Fail-open** only where read-only + low-risk and a **recent permit** exists in the local cache with the same inputs (time-boxed, e.g., ≤60s).
* **PIP (attribute) timeout:** use cached value with **freshness watermark**; attach higher risk score which can tip PDP to require step-up or deny.
* **STS unavailable for OBO refresh:** keep using nearing-expiry token if still valid; otherwise **block** actions requiring user context until refresh succeeds.

---

# 8) Concrete tuning tips

* Prefer **self-contained JWT ATs** (RFC 9068) over opaque + introspection.
* Use **DPoP** on browser/edge and **mTLS-bound tokens** for s2s to minimize replay risk without per-request network calls.
* Keep **OBO TTL short** (1–5 min) but **refresh proactively** at 80% of TTL to avoid synchronous exchange.
* Compile policies to **WASM** and apply **partial evaluation** on static data (tool schemas, allow-lists) to cut per-request cost.
* Batch `jti` writes; perform duplicate detection with a local filter first, then async replicate to Redis.
* Pin crypto to **ES256** or **EdDSA (Ed25519)** consistently; avoid RSA unless compatibility forces it (slower).

---

# 9) What to measure (and alert on)

* **Latency:** per-component histograms (JWT verify, DPoP verify, `jti` cache op, PDP eval).
* **Hit ratios:** JWKS cache, actor-token (OBO) cache, permit cache.
* **Replay blocks:** DPoP `jti` duplicates rejected.
* **Policy version drift:** requests served on stale policy vs latest (`policy_epoch`).
* **Fail-closed events:** rate & reasons (PIP timeout, PDP error, STS error).
* **End-to-end SLO:** security overhead p50/p95/p99 and error budgets per service.

---

# 10) Example: end-to-end timing (happy path)

1. JWT verify 0.5 ms
2. DPoP verify + `jti` check 1.5 ms
3. PDP evaluate 1.0 ms
4. **Total gateway security overhead ≈ 3.0 ms** → Forward to downstream API.

(When an OBO refresh is due, it happens **asynchronously** before the next call; not on the critical path.)

---

## Bottom line

* **Integration:** Put policy **in-process** at the gateway (WASM PDP), feed it with **local** attributes, and restrict networked operations (STS, introspection) to **amortized** events.
* **Tolerance:** Expect the gateway’s **security overhead** to stay around **≤6 ms p95 / ≤12 ms p99** on the hot path, with strict guardrails for outliers and clear fail-closed rules for high-risk actions.
