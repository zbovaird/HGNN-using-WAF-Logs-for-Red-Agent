# HGNN using WAF Logs for Red Agent

Colab-ready **HGNN** pipeline for enterprise WAF (Splunk export) request embeddings and **IP-level anomaly scoring** for red-team / detection workflows.

## Quick start (Google Colab)

1. Open [`notebooks/waf_HGNN_colab.ipynb`](notebooks/waf_HGNN_colab.ipynb) in Colab (**File → Upload notebook** or open from this repo).

2. Set runtime to **GPU** (Runtime → Change runtime type → T4 GPU or better).

3. Run all cells. When prompted, upload **`hgnn_waf_events.csv`** from your machine (Splunk WAF export). **Google Drive is not required.**

4. The notebook splits the CSV 70/15/15 into train/val/test, trains on **ALLOW** rows, and writes artifacts under `/content/waf_hgnn_artifacts/`.

Production HGNN training and scoring use **ALLOW** rows only; **BLOCK** rows are evaluated separately in notebook section 8.

## Notebook

| File | Purpose |
|------|---------|
| `notebooks/waf_HGNN_colab.ipynb` | End-to-end WAF HGNN: upload CSV, features, KNN graphs, training, embeddings, IP aggregation, blocklist stub |
| `notebooks/modbus_HGNN_clean.ipynb` | Modbus HGNN prototype: features, KNN graphs, training, embeddings, DBSCAN cluster evaluation |

### Modbus notebook (`modbus_HGNN_clean.ipynb`)

- Enable **GPU** in Colab (Runtime → Change runtime type → GPU) before running.
- Set `REBUILD_KNN = True` once so KNN cache metadata (`.meta.json`) is written; then set `False` for fast reruns.
- DBSCAN defaults are tuned from section 8b grid search: `dbscan_eps=0.4`, `dbscan_min_samples=3`.
- Set `RUN_DBSCAN_SWEEP = True` only for offline DBSCAN tuning (section 8b grid search); leave `False` for normal runs.

## Configuration

Edit the `Config` dataclass in the WAF notebook:

| Setting | Default | Notes |
|---------|---------|-------|
| `DATA_FILE` | `hgnn_waf_events.csv` | Uploaded to `/content/` when prompted |
| `ARTIFACT_DIR` | `/content/waf_hgnn_artifacts/` | Models, embeddings, blocklist outputs |
| `train_frac` / `val_frac` / `test_frac` | `0.70` / `0.15` / `0.15` | Split from single CSV |
| `score_action` | `ALLOW` | Filter for HGNN training/scoring |
| `REBUILD_KNN` | `True` | Set `False` after first run to load cached graphs |
| `RUN_UMAP_DBSCAN` | `False` | Optional exploration in section 8 |

Feature engineering includes `uri_path_length` / `uri_path_depth`, `ua_family` (not raw user-agent), and excludes `client_ip` from model features. IP aggregation and blocklist output are in sections 6–7.

## Artifacts

Written under `ARTIFACT_DIR` by default, for example:

- Cached KNN graphs per split (`*_edge_index.pt`)
- `best_model.pt`, `embeddings.npy`, `training_history.json` (modbus notebook)
- `ip_anomalies.csv`, `blocklist_candidates.json` (WAF notebook)

Download artifacts from Colab (**Files** panel) before the runtime disconnects, or copy them elsewhere if you need persistence.

## Colab sync / next run

- Pull or sync this repo in Colab so notebooks match `main`.
- Re-upload [`notebooks/waf_HGNN_colab.ipynb`](notebooks/waf_HGNN_colab.ipynb) or [`notebooks/modbus_HGNN_clean.ipynb`](notebooks/modbus_HGNN_clean.ipynb) from the repo.
- Delete stale `best_model.pt` in `ARTIFACT_DIR` before retraining.
- During training, verify `val_link` stays positive (watch the validation logs).
- Set `data_percentage=1.0` for full-dataset evaluation runs.
- If KNN/graph code changed, remove cached `*_edge_index.pt` or set `REBUILD_KNN = True` once.

## Troubleshooting

- **Upload prompt every run:** Place `hgnn_waf_events.csv` at `/content/hgnn_waf_events.csv` before section 1, or re-upload when prompted.
- **Slow KNN:** First run builds graphs; set `REBUILD_KNN = False` on later runs in the same session.
- **CPU warning:** Enable GPU runtime in Colab.
- **CUDA OOM:** Reduce `data_percentage` in `Config` for smoke tests.

## License

Use and adapt for your security research; ensure WAF log handling complies with your data policies.
