# WAF AI Agent Detection Strategy

**Audience:** Security engineers, detection architects, and red-team leads who work with WAF logs, Splunk, and Wiz. This document explains *what* we are building, *why*, and *how* to operationalize it without assuming deep machine-learning background.

**Repository:** [HGNN-using-WAF-Logs-for-Red-Agent](https://github.com/zbovaird/HGNN-using-WAF-Logs-for-Red-Agent)

---

## 1. Purpose

We want to detect **AI-driven attacker behavior** in web traffic—the kind of automated, exploratory, tool-assisted activity that increasingly shows up in real attacks—not merely “traffic from our internal Red Agent tool.”

Wiz Red Agent is a **controlled stand-in** for that behavior: it helps us **calibrate and test** detection during exercises, but production success means flagging **agentic patterns** (enumeration, probing, automation fingerprints) even when the client is an unknown agent, script, or browser automation stack.

The long-term outcome: Splunk exports WAF logs on a schedule → analytics scores traffic → **high-confidence client IPs** are sent back to the WAF for blocking, with enough context for analysts to review.

**Near-term focus:** Validate detection on **Splunk-exported WAF CSV files** (manual upload and analysis) **before** building the automated Splunk → cron → WAF pipeline.

---

## 2. Problem in plain terms

### What WAFs already do well

Traditional WAF rules catch **known bad signatures**: SQLi, XSS, rate limits, known malicious paths, etc. Events with `action=BLOCK` are often already handled.

### What we are targeting

Many AI-assisted attacks begin as **allowed traffic** that looks “weird” rather than explicitly malicious:

- Systematic path or API discovery  
- Unusual header combinations (automation clients vs real browsers)  
- High-volume probing from a single IP  
- Behavior that is **similar across requests** (bot-like) but **unlike normal users** on your app  

That traffic may sit in **`action=ALLOW`** rows—the hardest and most realistic detection surface.

### What we are not trying to do (initially)

- Replace WAF managed rules or Wiz’s cloud posture findings  
- Fingerprint Red Agent by a single field (e.g. one user-agent string)  
- Block purely on one indicator like `suspicious_path=1` without broader behavioral context  

---

## 3. Strategic goals and non-goals

| Goals | Non-goals (phase 1) |
|-------|---------------------|
| Detect **agentic ALLOW traffic** that deviates from a learned “normal” baseline | Perfect attribution (“this was ChatGPT”) |
| Roll up suspicion to **client IP** for WAF blocklist actions | Autonomous blocking with zero human review |
| Use Red Agent exercises to **measure recall** and tune thresholds | Train a model that only recognizes Red Agent |
| Keep an auditable trail (scores, sample requests, thresholds) | Real-time inline scoring inside the WAF data plane (can come later) |

---

## 4. Approach at a high level

### 4.1 Behavioral fingerprinting, not signature matching

Instead of a rule like “if user-agent contains X, block,” we:

1. Turn each WAF request into a **numeric behavior profile** (method, header flags, path shape, UA family, etc.).  
2. Connect **similar requests** in a graph (“these requests look like each other”).  
3. Learn a compact **embedding** (a behavior fingerprint vector) for each request.  
4. Flag requests—and **IPs**—that sit far from the bulk of **normal allowed** traffic.

Think of it as: *“Does this traffic cluster with how legitimate users use our app, or does it live in its own island of automation?”*

### 4.2 Why a graph?

In a graph, each request is a **node**. Edges link **similar** requests (nearest neighbors in feature space). Attack automation often forms **dense, separate clusters** or **outlier points**: many requests that resemble each other but not the rest of the site’s traffic.

We use a **Hyperbolic Graph Neural Network (HGNN)** here because security and IT data often have **hierarchical / tree-like** structure (paths, APIs, nested resources). Hyperbolic geometry is a practical way to represent that kind of structure in a compact embedding. You do not need to manage the math day-to-day—the notebook implements it.

**Plain analogy:** Normal users occupy a large “city” of similar behaviors. Agentic scanners often camp in a **small, distant neighborhood**. The model maps the city; scoring finds who is far from downtown.

### 4.3 Universal Hyperbolic Geometry (UHG) vs “traditional” hyperbolic ML

This project uses an **algebraic (UHG)** formulation for distance-like scores in the model. For operators, the important point is:

- It avoids heavier calculus-based steps in the training loop.  
- It is suitable for **batch analytics** on exported logs (Splunk cron), not for microsecond inline WAF decisions.  

### 4.4 Why ALLOW-only for training and scoring

| `action` | Role in our pipeline |
|----------|----------------------|
| **ALLOW** | **Train and score here** — traffic the WAF did not block; this is where novel AI recon often appears. |
| **BLOCK** | **Evaluation only** — confirms what the WAF already catches; useful for dashboards, not for learning “normal.” |

Mixing large amounts of BLOCK traffic into training teaches the model to replicate **existing WAF decisions**, not to find **missed** agentic behavior.

---

## 5. Data strategy

### 5.1 Source and delivery path

**Phase A (current): CSV files from Splunk**

- Run Splunk searches against AWS WAF (or equivalent) telemetry and export results to CSV (e.g. `hgnn_waf_events.csv`).
- Upload the CSV to the Colab notebook for training, scoring, and threshold experiments.
- No scheduled ingestion or WAF API integration in this phase—the goal is to prove the **data, features, and detection quality** on real WAF logs.

**Phase B (later): Automated pipeline**

- Scheduled Splunk export → enterprise runner → blocklist push to WAF API.
- Built only after CSV-based testing shows acceptable recall, precision, and false-positive rates.

Expected scale in early pilots: on the order of **tens of thousands of rows per month** (e.g. ~85k over 30 days is manageable).

### 5.2 Fields we use vs hold back

**Used in the model (behavior):**  
HTTP method, query/referer/cookie presence flags, header count, UA **family** (not full string), URI path **length/depth**, rule type buckets, JA fields where stable, etc.

**Held out of the model (pipeline / blocking / audit):**  
`client_ip`, timestamps, raw URI/query/referer/user-agent strings, `action`, `suspicious_path` (for validation, not as a crutch in training).

**Reasoning:** IPs are for **block actions**, not for teaching “what normal HTTP behavior looks like.” Raw URLs and user-agents have too many unique values and cause overfitting. `suspicious_path` is kept to **cross-check** whether our scores align with known bad paths on ALLOW traffic.

### 5.3 Red Agent mix: training vs evaluation

| Phase | Red Agent in data | Normal ALLOW |
|-------|-------------------|--------------|
| **Baseline training** | **None or &lt;1–2%** | **~98–100%** |
| **Red Team eval windows** | **5–15%** (or hundreds–thousands of ALLOW agent events) | Remainder |
| **Live production** | Naturally **rare** | Vast majority |

Red Agent proves: *“When agentic traffic is present, does it score as anomalous?”* It should **not** dominate the baseline, or the model will treat agent behavior as normal.

### 5.4 CSV-first validation before pipeline build

All initial work uses **Splunk WAF CSV exports** only:

| Step | What you do | What you learn |
|------|-------------|----------------|
| **Export** | Splunk → CSV with agreed WAF schema | Field coverage, volume, ALLOW/BLOCK mix |
| **Analyze** | Run Colab notebook on uploaded CSV | Whether embeddings separate agentic vs normal ALLOW traffic |
| **Tune** | Adjust thresholds, allowlists, features on CSV runs | Block-candidate quality without production automation |
| **Gate** | Document metrics from Red Agent windows | Go/no-go to invest in cron + WAF API wiring |

Do **not** build the production pipeline until CSV testing meets agreed detection and false-positive bars.

---

## 6. Methodology: phases

```text
Phase 0 — CSV validation (current)
  Splunk WAF export → CSV upload → Colab notebook (train, score, IP aggregation)
  Prove features and detection on real logs; tune thresholds with Red Agent windows

Phase 1 — Offline pilot (still CSV-driven)
  Repeatable export playbook; documented baselines; analyst review of block candidates

Phase 2 — Automated pipeline + controlled blocking
  Splunk cron → scoring job → analyst-reviewed blocklist → WAF API (TTL, allowlists)

Phase 3 — Production hardening
  Drift monitoring, retrain cadence, SOC playbooks, integration with Wiz workflows
```

---

## 7. Step-by-step process

### Step 1 — Define “normal” for your application

- Pick a time window of **typical business traffic** (e.g. 7–30 days).  
- Exclude known exercises, pentests, and major incidents from the **training** window if possible.  
- Document which hosts, paths, and client types are **allowlisted** (CDN, monitors, corporate egress).

**Output:** A baseline CSV export policy and allowlist requirements.

### Step 2 — Export WAF logs from Splunk to CSV

- Build and document a Splunk search that returns the agreed WAF fields (see notebook `WAF_SCHEMA_COLUMNS`).
- Export to **`hgnn_waf_events.csv`** (or equivalent single file per evaluation window).
- Validate row counts, `action` distribution, and column completeness before analysis.
- Normalize column names in the notebook; derive URI shape features; **do not** feed raw IP/UA into the model matrix.

**Output:** A reviewed CSV ready for upload to Colab. This step is repeated for each baseline and Red Agent evaluation window until the automated pipeline exists.

### Step 3 — Train the behavioral baseline (offline)

- Filter to **`action=ALLOW`** for training/scoring splits.  
- Split data train/validation/test (notebook default 70/15/15).  
- Build similarity graph (KNN), train HGNN, save `best_model.pt` and scaler/PCA artifacts.

**Output:** A model that encodes “what normal allowed requests look like” in embedding space.

**Operator check:** Training metrics should be stable (validation link loss positive and flat-ish); no negative loss artifacts from bad checkpoints.

### Step 4 — Score requests and aggregate to IPs

- For each ALLOW request, compute an **anomaly score** (distance from the normal centroid in embedding space).  
- Flag requests above a percentile threshold (default top ~5%).  
- **Group by `client_ip`:** require multiple flagged requests and/or high anomaly rate before block candidacy.

**Output:** `ip_anomalies.csv`, `blocklist_candidates.json` (stub today; WAF API integration later).

**Reasoning:** One odd request may be a false positive; **IP-level** aggregation matches how WAF blocklists are applied and how attackers operate.

### Step 5 — Red Agent evaluation (calibration, not training)

- Run Red Agent against the app; export the same Splunk window.  
- Measure on **ALLOW** Red Agent traffic:  
  - **Recall:** What fraction of agent IPs/requests score above threshold?  
  - **Precision:** During the exercise, what fraction of flagged IPs are actually Red Agent vs benign?  
- Tune **`anomaly_percentile`**, minimum request counts, and allowlists—not necessarily retrain.

**Output:** Threshold config backed by exercise metrics, not gut feel.

### Step 6 — Analyst review and WAF action

- SOC reviews top IPs (sample URIs, scores, `suspicious_path` overlap).  
- Push blocks to WAF with **TTL** (temporary blocks) and **allowlist** checks.  
- Log decision rationale for audit.

**Output:** Block events tied to detection scores and analyst approval (human-in-the-loop initially).

### Step 7 — Operate on a schedule

During **CSV validation**, repeat Steps 1–6 manually for each new Splunk export (baseline refresh, Red Agent windows). After the **automated pipeline** is built, use the cadence below.

| Activity | Suggested cadence |
|----------|-------------------|
| **Score new logs** | Every 15–60 minutes (cron; post-pipeline) |
| **Threshold / allowlist tune** | After each Red Team run or monthly |
| **Retrain baseline on normal ALLOW** | Weekly to monthly, or when drift detected |
| **Full feature/model review** | Quarterly or after major app/WAF changes |

**Reasoning:** Agent **patterns** often stay stable; **normal user traffic** and your **app surface** change. Retrain mainly refreshes “normal,” not “Red Agent.”

---

## 8. Reasoning for key design choices

### 8.1 Unsupervised vs supervised

We do not have reliable labels for “AI agent” in production logs. Red Agent gives **periodic ground truth** in exercises, but most traffic is unlabeled. Unsupervised anomaly detection fits: learn normal, flag outliers.

When Red Agent labels exist, use them for **evaluation and threshold tuning**, not as the primary training objective—unless you later add a supervised layer for known campaigns.

### 8.2 Why not block on WAF fields alone?

Single fields (`suspicious_path`, one rule ID, one JA3) are useful but brittle. Agents rotate clients and paths. **Multi-feature embeddings** capture combinations (headers + path shape + method + cadence at IP level) that generalize better.

### 8.3 Why graph-based vs flat clustering

Flat clustering (k-means on raw features) ignores **relationships between requests**. Graph-based methods explicitly encode “who resembles whom,” which matches scanner behavior: many **related** probes, not one isolated bad row.

### 8.4 Why hyperbolic / HGNN (conceptual)

Web behavior is often **tree-structured** (paths, nested APIs). Hyperbolic space represents hierarchical data efficiently. HGNN is our chosen encoder; operators care about **embedding quality and anomaly separation**, not the underlying geometry proofs.

### 8.5 Caching and performance

- **Cache the trained model** and preprocessing objects across cron runs.  
- **Rebuild the KNN graph** on each scoring window (new logs = new neighbors).  
- At ~85k rows/month, batch runs on GPU are practical; UMAP/t-SNE are for **exploration**, not required in production scoring.

---

## 9. Success metrics

### Detection (during Red Agent exercises)

- **IP recall:** % of Red Agent source IPs exceeding block candidate rules on ALLOW traffic.  
- **Request recall:** % of Red Agent ALLOW requests flagged (may be lower; IP metric is primary).  
- **False positives:** Benign IPs flagged during the same window (especially CDN, scanners, employees).

### Operational

- Time from log export → blocklist candidate generation.  
- Analyst time per block decision (should decrease as thresholds stabilize).  
- Block TTL and rollback rate (high rollback = tuning needed).

### Generalization (stretch)

- Detection still works when Red Agent varies speed, paths, or client profile.  
- If detection only works on one Red Agent fingerprint, treat as **overfit** and revisit features/thresholds.

---

## 10. Risks, limitations, and guardrails

| Risk | Mitigation |
|------|------------|
| **False positives** block legitimate users | IP thresholds, allowlists, TTL blocks, human review initially |
| **Training on agent traffic** makes agents look “normal” | Keep Red Agent out of baseline training; use only for eval |
| **Training on BLOCK traffic** copies WAF rules | ALLOW-only scoring path |
| **IP in features** memorizes IPs instead of behavior | Exclude `client_ip` from model; use only at aggregation/block stage |
| **Concept drift** (app changes) | Scheduled baseline retrain; monitor score distributions |
| **Shared NAT / CDN** | Allowlist known egress; require sustained anomaly rate, not one spike |
| **Compliance / privacy** | Handle logs per corporate policy; restrict exports and retention |

---

## 11. Roles and responsibilities (suggested)

| Role | Responsibility |
|------|----------------|
| **Detection engineering** | Splunk exports, cron jobs, model artifacts, thresholds |
| **AppSec / red team** | Red Agent exercises, labeled eval windows, false-positive feedback |
| **SOC** | Review block candidates, approve WAF pushes |
| **WAF ops** | Apply blocklists, TTL, allowlists, measure impact |
| **Wiz stakeholders** | Align on how this complements Wiz findings (attack path vs edge WAF telemetry) |

---

## 12. Current implementation map

| Capability | Status | Location |
|------------|--------|----------|
| Splunk WAF CSV ingest (manual upload) | In use | `notebooks/waf_HGNN_colab.ipynb` |
| WAF schema + feature engineering | Implemented | Same |
| ALLOW/BLOCK split, IP aggregation | Implemented | Same |
| Blocklist JSON stub (manual review) | Implemented | Notebook section 7 |
| Splunk scheduled export + cron runner | Planned | After CSV validation gate |
| WAF API blocklist integration | Planned | After Phase 2 |
| Production drift monitoring | Planned | Phase 3 |

---

## 13. Recommended roadmap (next 90 days)

1. **Splunk CSV export playbook:** Document search, field list, and export steps for baseline and Red Agent windows.
2. **Baseline CSV run:** 30 days normal traffic → upload → train on GPU in Colab; confirm stable training and artifact outputs.
3. **Red Agent CSV eval:** Overlapping export with exercise traffic; measure IP recall on ALLOW rows; tune `anomaly_percentile`.
4. **Allowlist v1:** CDN, corporate egress, known scanners; re-run on same CSVs to measure false-positive reduction.
5. **Analyst review drill:** Manual review of `blocklist_candidates.json` from CSV runs; document accept/reject reasons.
6. **CSV validation gate:** Agree minimum recall/precision before any cron or WAF API automation.
7. **Pipeline design (only after gate):** Splunk schedule → scoring job → WAF blocklist API with TTL.

---

## 14. Glossary

| Term | Meaning |
|------|---------|
| **ALLOW / BLOCK** | WAF decision on the request. We score ALLOW for missed threats. |
| **Embedding** | A numeric fingerprint summarizing request behavior. |
| **KNN graph** | Links each request to its nearest similar requests. |
| **HGNN** | Graph neural network that refines fingerprints using neighbor context. |
| **Anomaly score** | How far a request’s fingerprint is from typical normal traffic. |
| **Red Agent** | Wiz red-team tool; simulates attacker behavior for controlled testing. |
| **Agentic behavior** | Automated, tool-driven exploration/exploitation patterns—not one specific product. |

---

## 15. Summary

We are building a **behavioral anomaly layer** on WAF logs to find **AI-assisted attack patterns**—especially on **allowed** traffic—before or alongside signature-based blocks. Wiz Red Agent is our **ruler**, not our **target class**: it validates that agent-like traffic separates from normal users when the baseline is healthy.

Train on **normal ALLOW** traffic, score on Splunk **CSV exports** first, aggregate to **IPs**, tune with **Red Agent exercises**, and block with **guardrails**. Retrain to follow **normal traffic drift**, not every Red Agent run. Automate Splunk → WAF only after CSV testing meets agreed bars. Success is measured in **generalized agentic detection**, not Red Agent signature matching.

For hands-on execution, start with [`notebooks/waf_HGNN_colab.ipynb`](../notebooks/waf_HGNN_colab.ipynb) and the [repository README](../README.md).
