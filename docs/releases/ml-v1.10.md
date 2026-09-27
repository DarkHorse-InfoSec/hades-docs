# HADES ML Model v1.10.0 Release Notes

**Release date:** 2026-05-03
**Component:** Machine-learning detection bundle (`models/baseline_classifier_v1.*` production alias)
**Product version:** HADES source product remains v1.4.2; this is a model-only update
**Schema:** v6 (31 features), signed RSA-PSS-SHA256, fail-closed at runtime

## TL;DR

HADES has promoted ML bundle v1.10.0 to the production alias. The new model lowers the aggregate per-category false-positive rate from 0.83% to 0.78%, eliminates a long-standing OEM-driver false-positive cluster (Epson `E_WPUI10.DLL` and `EBPNET6.DLL` from `e_wf1kde`), and clears every per-category false-positive gate under the project ceiling of 1.5%. True-positive recall rises to 99.34% (v1.9.0: 98.88%), measured on the held-out test set of 3,034 malicious and 774 clean files recorded in the v1.10.0 model metadata (`models/baseline_classifier_v1_10_0.metadata.json`, source repo, 2026-05-03). Customers do not need to take any action; the bundle will load on the next phone-home or process restart, and signed-bundle verification fail-closes if the artifact is tampered.

## What changed

v1.10.0 ships two new layers of false-positive defense in front of and inside the model. The product version stays at v1.4.2; only the ML artifact is new.

**Tier 1: deterministic SHA-256 allowlist.** A new file at `core/known_clean_oem_hashes.json` ships pinned SHA-256 entries for OEM components confirmed clean through dual-source provenance review. The ML detection engine consults this allowlist before feature extraction; on a hit, the file is returned as clean with confidence 100 and an audit-log line is emitted. The two files that drove the v1.7-v1.9 windows_pe false-positive cluster are pinned in this allowlist as a deterministic guarantee. A 90-day review cadence is declared in the policy block of the file.

**Tier 2: schema v6 structural OEM-driver features.** The metadata feature extractor expands from 27 columns to 31, adding four structural signals targeted at the residual OEM-driver false-positive class:

- f27 `is_epson_driver_naming`: regex match on Epson basename patterns (E_*, EBP*, EBPNET*, EN*).
- f28 `is_generic_oem_driver_naming`: generalized vendor-prefix regex covering broader OEM naming conventions.
- f29 `is_in_oem_driver_path`: path-token detection for `DriverStore`, `inf_amd64`, and `FileRepository`.
- f30 `parent_dir_vendor_token_present`: 30 known vendor name tokens in the parent directory basename.

f29 ranked 4th and f27 ranked 7th in XGBoost feature importance after retraining. Both features pulled measurable weight as designed. f28 and f30 ship as scaffolding (zero importance in the current corpus); they activate as new corpus diversity is ingested in v1.11.

Schema v5 bundles continue to load through the v6 extractor via `feature_names` truncation; backward compatibility is verified.

## Why this approach: layered defense

Earlier model iterations attacked the OEM-driver false-positive cluster with single-tier solutions: corpus expansion alone, or a single new heuristic feature, or a per-category threshold sweep. None of those held. Static unpack of modern Epson `_Lite_NA.exe` driver bundles misses the actual driver DLLs because the launcher runtime-downloads them, so further corpus expansion cannot reach them. A heuristic-only feature without a deterministic backstop is one model regression away from re-introducing the same false positive.

v1.10.0 applies the layered defense pattern that production-grade EDR vendors apply to false-positive management: a deterministic guarantee on confirmed-clean files plus structural features that generalize to similar components, with sandboxed corpus collection and a customer feedback loop following in the next cycle. The Tier 1 allowlist guarantees that the two confirmed-clean files in the v1.7-v1.9 cluster cannot be flagged by HADES regardless of model behavior. Tier 2 features generalize the structural signal so files of similar shape (OEM driver-store paths, vendor-prefixed naming) lean clean in the model's own scoring without needing to be individually pinned. Tier 3 (sandboxed runtime corpus collection) and Tier 4 (portal false-positive feedback loop) are the next two layers and are scoped for v1.11+.

## Metrics

### Aggregate

| Metric                          | v1.9.0 (prior)     | v1.10.0 (current) | Delta       |
|---------------------------------|--------------------|-------------------|-------------|
| Precision                       | 0.9980             | 0.9980            | 0.0000      |
| Recall                          | 0.9888             | 0.9934            | +0.0046     |
| ROC-AUC                         | 0.9991             | 0.9995            | +0.0004     |
| F1                              | 0.9934             | 0.9957            | +0.0023     |
| Aggregate FP rate (per-cat)     | 0.83% (6/720)      | 0.78% (6/774)     | -0.05pp     |
| Test set size (clean)           | 720                | 774               | +54         |

Source for both columns: the signed model metadata in the source repo, `models/baseline_classifier_v1_9_0.metadata.json` (2026-04-30) and `models/baseline_classifier_v1_10_0.metadata.json` (2026-05-03), per-category-threshold metrics. The v1.9.0 column was corrected on 2026-09-27; it previously carried figures that did not match that metadata.

### Per-category false-positive rates (project gate: under 1.5%)

| Category    | v1.9.0 FP rate    | v1.10.0 FP rate    | Threshold | Gate   |
|-------------|-------------------|--------------------|-----------|--------|
| windows_pe  | 1.53% (2/131)     | 0.66% (1/152)      | 0.50      | PASS   |
| pdf         | 0.74% (1/136)     | 0.00% (0/110)      | 0.50      | PASS   |
| powershell  | 0.59% (1/169)     | 1.23% (2/163)      | 0.75      | PASS   |
| linux_elf   | 0.00% (0/133)     | 1.34% (2/149)      | 0.50      | PASS   |
| python      | 0.00% (0/40)      | 0.00% (0/57)       | 0.50      | PASS   |
| image       | 25.0% (2/8)       | 0.00% (0/47)       | 0.50      | PASS   |
| font        | 0.00% (0/4)       | 0.00% (0/2)        | 0.50      | PASS   |
| other       | 0.00% (0/99)      | 1.06% (1/94)       | 0.50      | PASS   |

Every per-category rate sits under the 1.5% project gate. The two categories that produced slim-margin or sample-noise violations in v1.9.0 (windows_pe at 1.53%, image at 25% on n=8) are both cured: windows_pe drops to 0.66%, and image goes to 0% on a 6x larger clean image corpus (47 samples vs 8). Three categories rose while staying under the gate: powershell 0.59% to 1.23%, linux_elf 0.00% to 1.34%, and other 0.00% to 1.06%.

### Cluster validation

Both files in the v1.7-v1.9 windows_pe false-positive cluster are now classified clean by the **model itself**, separately from the Tier 1 allowlist:

| File                     | Tier 1 hit | Model anomaly score | Model verdict | Confidence |
|--------------------------|------------|---------------------|---------------|------------|
| E_WPUI10.DLL (e_wf1kde)  | yes        | 0.0026              | clean         | 99         |
| EBPNET6.DLL (e_wf1kde)   | yes        | 0.0042              | clean         | 99         |

Both anomaly scores sit far below the 0.5 decision threshold. The Tier 1 allowlist is a redundant safety net; the model's own structural-feature dispatch is sufficient for these specific files. The redundancy is intentional: a future retrain that drifts on these files cannot reintroduce the false positive while the allowlist is in place.

## How customers benefit

No action required. The signed v1.10.0 bundle ships in HADES v1.4.2 source and binaries published to `dl.darkhorseinfosec.com`, and existing installations pick it up automatically on next process restart. Signed-bundle runtime verification fail-closes on signature mismatch; if the artifact is tampered or the public key fingerprint changes, the engine declines to load the model rather than silently downgrading.

Practical effects:
- Slightly lower aggregate false-positive rate (0.83% to 0.78%).
- The Epson `E_WPUI10.DLL` / `EBPNET6.DLL` cluster, if present in your environment, will no longer flag.
- Recall 99.34% on the held-out test set (3,034 malicious files; model metadata, 2026-05-03). Per-family recall is 100% for 8 of 12 held-out families; the other four are 83.3% (n=6), 84.6% (n=13), 97.9% (n=663) and 99.9% (n=2,178).
- No CLI, API, or configuration changes.

Customers running their own retraining pipeline against the documented schema should regenerate features against the v6 extractor; v5 features continue to load through the backward-compatibility path.

## What is coming in v1.11

- **Tier 3: sandboxed runtime corpus collection.** Sandbox-execute Epson `_Lite_NA.exe` installers in a snapshotted Windows VM, scripted through InstallNavi, with post-install filesystem capture. Brings the actual `E_*.DLL` files runtime-extracted by the launcher into the clean training corpus.
- **Tier 4: portal false-positive feedback loop.** Customer-reported false positives flow into the Tier 1 allowlist queue and the next-retrain corpus delta, with an admin review workflow on the portal side.
- **Corpus diversification.** Activate f28 and f30 by ingesting non-Epson OEM components (HP, Dell, Lenovo) and known-vendor parent-directory snapshots. Tier 3 covers the Epson side; v1.11 expands to the rest.

The Tier 1 allowlist is the architectural guarantee that the v1.7-v1.9 cluster cannot recur regardless of model drift. Tier 2 + Tier 3 + Tier 4 are how that guarantee scales to similar files we have not yet seen.

---

*Layered defense. Deterministic guarantees on confirmed-clean. Structural features that generalize. Customer feedback loop coming.*
