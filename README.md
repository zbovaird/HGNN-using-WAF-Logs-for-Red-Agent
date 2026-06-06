# HGNN using WAF Logs for Red Agent

Colab-ready **HGNN** pipeline for enterprise WAF (Splunk export) request embeddings and **IP-level anomaly scoring** for red-team / detection workflows.

## Quick start (Google Colab)

1. Upload WAF CSV splits to Google Drive:
   - `/content/drive/MyDrive/waf/train.csv`
   - `/content/drive/MyDrive/waf/val.csv`
   - `/content/drive/MyDrive/waf/test.csv`

2. Open [`notebooks/waf_HGNN_colab.ipynb`](notebooks/waf_HGNN_colab.ipynb) in Colab (**File → Upload notebook** or sync from this repo).

3. Set runtime to **GPU** (Runtime → Change runtime type → T4 GPU or better).

4. Run all cells top-to-bottom.

Production HGNN training and scoring use **ALLOW** rows only; **BLOCK** rows are evaluated separately in notebook section 8.

## Notebook

| File | Purpose |
|------|---------|
| `notebooks/waf_HGNN_colab.ipynb` | End-to-end WAF HGNN: features, KNN graphs, training, embeddings, IP aggregation, blocklist stub |
| `notebooks/modbus_HGNN_clean.ipynb` | Modbus HGNN: features, KNN graphs, training, embeddings, DBSCAN cluster evaluation |

### Modbus notebook (`modbus_HGNN_clean.ipynb`)

- Enable **GPU** in Colab (Runtime → Change runtime type → GPU) before running.
- Set `REBUILD_KNN = True` once so KNN cache metadata (`.meta.json`) is written; then set `False` for fast reruns.
- DBSCAN defaults are tuned from section 8b grid search: `dbscan_eps=0.4`, `dbscan_min_samples=3`.
- Set `RUN_DBSCAN_SWEEP = True` only for offline DBSCAN tuning (section 8b grid search); leave `False` for normal runs.

## Configuration

Edit the `Config` dataclass in the notebook:

| Setting | Default | Notes |
|---------|---------|-------|
| `DATA_DIR` | `/content/drive/MyDrive/waf/` | Splunk export CSV location |
| `ARTIFACT_DIR` | `/content/drive/MyDrive/waf_hgnn_artifacts/` | Models, embeddings, outputs |
| `score_action` | `ALLOW` | Filter for HGNN training/scoring |
| `REBUILD_KNN` | `True` | Set `False` after first run to load cached graphs |
| `RUN_UMAP_DBSCAN` | `False` | Optional exploration in section 8 |

Feature engineering includes `uri_path_length` / `uri_path_depth`, `ua_family` (not raw user-agent), and excludes `client_ip` from model features. IP aggregation and blocklist output are in sections 6–7.

## Artifacts (Drive)

Written under `ARTIFACT_DIR` by default, for example:

- Cached KNN graphs per split (`*_edge_index.pt`)
- `best_model.pt`, `embeddings.npy`, `training_history.json` (modbus notebook)
- IP-level scores and blocklist outputs from sections 6–7

## Colab sync / next run

- Pull or sync this repo in Colab so notebooks match `main`.
- Re-upload [`notebooks/modbus_HGNN_clean.ipynb`](notebooks/modbus_HGNN_clean.ipynb) or [`notebooks/waf_HGNN_colab.ipynb`](notebooks/waf_HGNN_colab.ipynb) from the repo.
- Delete stale `best_model.pt` in `ARTIFACT_DIR` before retraining.
- During training, verify `val_link` stays positive (watch the validation logs).
- Set `data_percentage=1.0` for full-dataset evaluation runs.
- If KNN/graph code changed, remove cached `*_edge_index.pt` or set `REBUILD_KNN = True` once.

## Troubleshooting

- **Slow KNN:** First run builds graphs; set `REBUILD_KNN = False` on later runs.
- **CPU warning:** Enable GPU runtime in Colab.
- **CUDA OOM:** Reduce data subsampling in `Config` or use smaller batches if you extend training.

## License

Use and adapt for your security research; ensure WAF log handling complies with your data policies.
