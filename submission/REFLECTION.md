# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Nguyen Cong Nhat Tan
**Submission date:** 11/05/2026
**Lab repo URL:** https://github.com/godrefuse2610/Day23-Track2-NguyenCongNhatTan

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```json
{
  "docker": {
    "ok": true,
    "version": "29.4.0"
  },
  "compose_v2": {
    "ok": true,
    "version": "5.1.1"
  },
  "ram_gb_available": 7.44,
  "ram_ok": true,
  "required_ports": [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888],
  "bound_ports": [],
  "all_ports_free": true
}
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app` | screenshot `alertmanager-firing.png` |
| _T0+90s_ | `ServiceDown` fired | screenshot `slack-firing.png` |
| _T1_ | restored app | — |
| _T1+60s_ | alert resolved | screenshot `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

Grafana provisioning assigns auto-generated UIDs to datasources at startup, but dashboard JSON files reference hardcoded UIDs like `"uid": "prometheus"`. When these do not match, every panel silently shows "No Data" with no error message. Adding `uid: prometheus` to the datasource provisioning YAML fixed all six panels at once. This taught me that dashboard-as-code only works reliably when datasource UIDs are pinned explicitly — not left to auto-generation.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Trace ID from `make trace`: `191bc378160cd73e5c8dcde02f6102db`

Corresponding structured JSON log line emitted by the service:

```json
{
  "event": "prediction served",
  "model": "llama3-mock",
  "input_tokens": 1,
  "output_tokens": 28,
  "quality": 0.823,
  "duration_seconds": 0.1523,
  "trace_id": "191bc378160cd73e5c8dcde02f6102db",
  "level": "info",
  "timestamp": "2026-05-11T13:24:52.134Z"
}
```

The `trace_id` field is injected by `main.py` from the active OTel span context, making it possible to jump directly from a Loki log line to the corresponding Jaeger trace.

### Tail-sampling math

During `make load` the service processed **1032 requests in 60 s ≈ 17.6 req/s**.

The otel-collector tail-sampling policy has three arms (OR logic — any match keeps the trace):

| Policy | Rule | Kept traces/s |
|---|---|---|
| `keep-errors` | status_code = ERROR | 0 (0 errors in load test) |
| `keep-slow` | latency > 2000 ms | ~1% × 17.6 = **0.18 req/s** |
| `probabilistic-1pct` | random 1% of the rest | 1% × 99% × 17.6 = **0.17 req/s** |

**Total kept ≈ 0.35 req/s out of 17.6 req/s ≈ 2% of all traces.**

A forced-error request (`fail: true`) is always retained by `keep-errors`, while a healthy sub-2s trace has only a 1% chance of surviving. This confirms tail sampling correctly prioritises signal (errors, slow outliers) over noise (routine healthy traffic).

---

## 4. Track 04 — Drift Detection

### PSI scores

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

### Which test fits which feature?

| Feature | Chosen test | Reason |
|---|---|---|
| `prompt_length` | **PSI** | Continuous, roughly normal. PSI is the industry standard for monitoring input feature distributions in production ML — a PSI > 0.2 triggers a retrain signal. Here PSI = 3.46 confirms a severe shift (loc moved from 50 → 85). |
| `embedding_norm` | **KS (Kolmogorov-Smirnov)** | Tight, low-variance continuous feature. KS is distribution-free and sensitive to shape changes, making it ideal for embedding statistics where the scale is small and a subtle shift matters. PSI = 0.019 and KS p-value = 0.13 both correctly report no drift. |
| `response_length` | **KL divergence** | Response length can have a long right tail (rare very long outputs). KL divergence captures asymmetric distributional changes better than PSI when one tail matters more than the other. PSI = 0.016 and KS p = 0.09 confirm no drift. |
| `response_quality` | **MMD (Maximum Mean Discrepancy)** | Quality score is a Beta-distributed value in [0, 1] — it is bounded and multimodal when the distribution shifts (beta(8,2) → beta(2,6) is a full shape reversal, not just a mean shift). MMD operates in kernel space and detects such shape changes without binning assumptions. PSI = 8.85 also catches it, but MMD would be the production choice for bounded scores. |

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

The Day 19 Qdrant vector store would be the hardest to expose. Qdrant serves metrics on a REST endpoint (`/metrics`) but requires the container to be running on a reachable address from inside the Docker Compose network. On a developer laptop, the Qdrant process runs on `host.docker.internal:6333`, which is only accessible if Docker Desktop's host networking is correctly configured — a setting that behaves differently on Windows versus Linux. The Prometheus scrape job is already commented out in `prometheus.yml` with the correct target, but wiring it end-to-end requires verifying network reachability, which adds friction compared to services already inside the `obs` Compose network.

---

## 6. The single change that mattered most

The single most impactful addition to this stack was exposing `inference_quality_score` as a Prometheus Gauge — what the deck calls the **"4th pillar"** of AI observability. Standard RED metrics (rate, errors, duration) tell you the service is *up* and *fast*, but they are blind to whether the model is producing *good* outputs. By treating an eval score as a time-series metric scraped every 15 seconds, I could dashboard it alongside latency and plot their correlation: as load increased and median latency dropped toward 170ms, quality held steady around 0.82, confirming the mock's deterministic behaviour. In a real system, a divergence between low latency and falling quality score would be the first signal of a model regression or a prompt-injection attack inflating token counts — neither of which would appear in any infrastructure metric.

This one gauge transforms the observability stack from *infrastructure monitoring* into *model monitoring*. Without it, a silent quality degradation — the most dangerous failure mode in production LLMs — is completely invisible. With it, a simple Grafana alert on `inference_quality_score < 0.7` closes the loop between the inference pipeline and the on-call engineer.
