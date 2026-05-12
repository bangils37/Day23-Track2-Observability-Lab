# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Nguyễn Bằng Anh - 2A202600136
**Submission date:** 2026-05-12
**Lab repo URL:** [YOUR_GITHUB_URL]

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
PS D:\VinUni_AIThucChien\Day23-Track2-Observability-Lab> make setup  
Pre-pulling 6 images (the FastAPI app builds locally)...
  pulling: prom/prometheus:v2.55.0
  pulling: prom/alertmanager:v0.27.0
  pulling: grafana/grafana:11.3.0
  pulling: grafana/loki:3.3.0
  pulling: jaegertracing/all-in-one:1.62.0
  pulling: otel/opentelemetry-collector-contrib:0.114.0
All images cached.
Docker:        OK  (29.1.3)
Compose v2:    OK  (5.1.3)
RAM available: 7.67 GB (OK)
Ports free:    OK
Report written: D:\VinUni_AIThucChien\Day23-Track2-Observability-Lab\00-setup\setup-report.json
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
| _T0_ | killed `day23-app`         | screenshot `alertmanager-firing.png` |
| _T0+90s_ | `ServiceDown` fired   | screenshot `slack-firing.png` |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | screenshot `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

The dashboards were correctly authored but showed no data until I discovered Grafana had provisioned the Prometheus datasource without a stable `uid`. After adding `uid: prometheus` to `02-prometheus-grafana/grafana/provisioning/datasources/datasources.yml` and restarting Grafana the panels populated. I also learned to wait ~30–45s for Prometheus to perform a couple of scrapes before time-series panels appear filled.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```
PS D:\VinUni_AIThucChien\Day23-Track2-Observability-Lab> curl -X 'POST' \
  'http://localhost:8000/predict' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "prompt": "hello",
  "model": "llama3-mock",
  "fail": false
}'
{"text":"[mock] llama3-mock replied to 'hello...' with 54 tokens","model":"llama3-mock","input_tokens":4,"output_tokens":54,"trace_id":"24905941aef8b67ab130ee2726705958","quality_score":0.82}kuongan@DESKTOP-NF3QU5J:~/vinuni/Day23-Track2-Observability-Lab$ 

```

### Tail-sampling math

If your service produced N traces/sec, what fraction did the policy keep? Show the calculation.

Using the lab's assumptions (P(error) ≈ 1%, P(slow) ≈ 1%, P(healthy) ≈ 98%):

sampled = P(error) * 1.0 + P(slow ∧ ¬error) * 1.0 + P(healthy) * 0.01

Assuming errors and slows are both ~1%:

sampled = 0.01 + 0.01 + 0.98 * 0.01 = 0.01 + 0.01 + 0.0098 = 0.0298 ≈ 2.98%

So the collector will retain roughly 3% of traces under these typical conditions.
---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

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

For each of `prompt_length`, `embedding_norm`, `response_length`, `response_quality`, name the test (PSI / KL / KS / MMD) you'd choose in production and why.

- `prompt_length`: PSI — bucketized population-shift detection is easy to interpret and stable in production sliding windows.
- `embedding_norm`: MMD (or MMD + KS on a scalar summary) — embeddings are high-dimensional; MMD detects multivariate drift, KS is useful on the scalar norm.
- `response_length`: PSI or KS — response length is a 1-D numeric feature; PSI is production-friendly across windows, KS gives a simple significance test.
- `response_quality`: KL or PSI — KL quantifies distributional divergence for full-score distributions; PSI is useful for operational thresholds and alerting on sustained drift.
---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

_(2-3 sentences. If you didn't have prior days running, write about which one would be hardest based on the integration scripts.)_

The Day 19 vector-store integration (Qdrant) would be the hardest to expose because it requires integrating an external datastore with different networking constraints and exposing internal vector-search metrics. Mapping those metrics to request-level traces and the app's `model` label adds semantic and operational complexity compared to in-process metrics.
---

## 6. The single change that mattered most

> **Grader reads this closest.** What one thing about your stack design — a metric you added, a label you dropped, a panel you reorganized, an alert threshold you tuned — made the biggest difference between "works" and "useful"? Write 1-2 paragraphs. Connect it to a concept from the deck.

Adding a compact, well-labeled request counter `inference_requests_total{model, status}` and precomputing failure-rate recording rules (`inference:fail_ratio:rate5m`, `rate30m`, `rate1h`, `rate6h`) had the largest practical impact. With that single metric in place I could express SLOs and burn-rate calculations as cheap recording rules, power the burn-rate dashboard panels, and create concise alerts that map directly to the model dimension. The result was dashboards and alerts that were both responsive and actionable for troubleshooting per-model regressions.

This change connects to the deck advice about designing SLIs that are inexpensive to compute and labeled for the dimensions you operate on. Instead of relying on ad-hoc, expensive PromQL in dashboards, recording rules and minimal, orthogonal labels turned raw telemetry into operational signal — making the stack "useful" rather than merely "working." 
