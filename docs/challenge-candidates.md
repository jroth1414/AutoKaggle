# Challenge candidates

*Companion to [system-design.md](system-design.md) §18. Compiled 2026-11-20.*

This page lists the competitions AutoKaggle can be developed and benchmarked on, given the v1 dev hardware: one 16 GB GPU and limited disk.

**How sure each column is**

| Column | Source | Status |
|---|---|---|
| MLE-bench split | Read from the MLE-bench repo split files | Confirmed |
| Metric, size, code-competition status | Prior knowledge | Approximate. Verify each with the Kaggle adapter before use. |
| Late submissions | Not checked; Kaggle pages need a login | Unverified |

**Split names:** Low = MLE-bench Lite (22 competitions, ~158 GB). Med = Medium, High = High, Dev = the MLE-bench dev split. **Big** = more than 50 GB on disk.

## Primary development challenge

| Field | Value |
|---|---|
| **Name** | Solar Filament Segmentation Challenge 2026 (IEEE BigData Cup 2026, NSO) |
| **Slug** | `filament-segmentation-2026` |
| **Task** | Instance segmentation of solar filaments in full-disk GONG H-alpha images, 2048×2048 JPEG |
| **Data** | 707 train images. 1,154 annotator views (1–3 annotators per image). 8,199 polygon instances in COCO-style JSON. 180 test images. About 0.7 GB. *(From participant repos.)* |
| **Metric** | Panoptic Quality: a match needs IoU > 0.5, and each annotator's labels are scored separately. Inter-annotator PQ is only about 0.34. The official scorer seems to pool TP/FP/FN and IoU across annotator views. *(Uncertain. The organizer's self-evaluation notebook on Kaggle is the reference.)* |
| **Submission** | `submission.csv` with columns `filament_id,segmentation_rle`. Each row is one instance, encoded as compressed COCO RLE. Masks must not overlap. File upload, not a code competition. |
| **Status** | Deadline was 2026-11-15, so the competition has ended. Late-submission availability is **unverified**. |
| **Rules** | The public MAGFiLO release is reportedly disallowed because it may contain test labels. COCO and ImageNet pretrained weights are reportedly used by top entries. *(Unverified. Check the rules page.)* |
| **Reference scores** | *(Self-reported by participants.)* Winning private PQ ≈ 0.64. Strong entries reach OOF ≈ 0.46–0.48 and public LB ≈ 0.35–0.42. A plain U-Net with connected components gets about 0.20–0.31. |
| **What worked** | *(From participants.)* Native or high resolution. Flip/8-view TTA. Mask2Former (Swin) or Mask R-CNN combined with U-Net refiners. A learned per-instance quality filter. Selecting non-overlapping masks. CV folds split by date or observation, because same-day images come from different sites. |

**Why it suits development.** It exercises almost every hard part of the harness at small data scale:
- **Leak-free validation design.** Folds must be grouped by date.
- **A nonstandard metric to reproduce.** Multi-annotator PQ.
- **Compute adaptation.** 2048 px images don't fit comfortably on a 16 GB GPU.
- **Instance-segmentation post-processing.**
- **A clear quality ladder** from about 0.2 (naive) to about 0.64 (winning).

**Risks**
- Without late submissions, the only feedback is local OOF plus the organizer's self-evaluation notebook.
- The metric details must be confirmed before the numbers can be trusted.

## Candidate table

| # | Name | Slug | Task | Metric | Size | Code comp | MLE-bench | Late subs | Why / risk |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Aerial Cactus | aerial-cactus-identification | Classification (binary) | AUC | ~25 MB | no | Low | likely | Ideal smoke test: tiny, finishes in minutes |
| 2 | Dog Breed ID | dog-breed-identification | Classification (120) | Log loss | ~0.7 GB | no | Low | likely | Transfer learning, calibration |
| 3 | Plant Pathology 2020 | plant-pathology-2020-fgvc7 | Classification (4) | Mean col. AUC | ~0.8 GB | no | Low | likely | High-res images, augmentation matters |
| 4 | Histopathologic Cancer | histopathologic-cancer-detection | Classification (binary) | AUC | ~7.8 GB | no | Low | likely | 220k tiles: data-loader throughput |
| 5 | APTOS 2019 | aptos2019-blindness-detection | Ordinal classification | QWK | ~10 GB | yes | Low | likely | Ordinal loss, threshold tuning |
| 6 | RANZCR CLiP | ranzcr-clip-catheter-line-classification | Multi-label classification | Mean AUC | ~12 GB | yes | Low | likely | High-res X-rays strain 16 GB VRAM |
| 7 | Denoising Dirty Docs | denoising-dirty-documents | Image-to-image | RMSE | ~60 MB | no | Low | likely | Unusual submission format |
| 8 | Plant Seedlings | plant-seedlings-classification | Classification (12) | Micro-F1 | ~1.7 GB | no | Dev | likely | MLE-bench dev split |
| 9 | TGS Salt | tgs-salt-identification-challenge | Semantic segmentation | mAP over IoU | ~0.5 GB | no | Med | likely | Small classic segmentation, RLE |
| 10 | UW-Madison GI Tract | uw-madison-gi-tract-image-segmentation | Semantic segmentation (2.5D) | Dice + 3D Hausdorff | ~2.5 GB | yes | Med | likely | Composite metric, slice stacking |
| 11 | Whale Categorization | whale-categorization-playground | Retrieval / metric learning | MAP@5 | ~0.8 GB | no | Med | likely | Open-set "new_whale" class |
| 12 | Kuzushiji Recognition | kuzushiji-recognition | Detection + OCR | F1 | ~4.5 GB | no | Med | likely | Dense small objects |
| 13 | Cassava Leaf | cassava-leaf-disease-classification | Classification (5) | Accuracy | ~6 GB | yes | Med | likely | Noisy labels, tight leaderboard |
| 14 | Pawpularity | petfinder-pawpularity-score | Image regression + metadata | RMSE | ~1 GB | yes | Med | likely | Low signal: tests overfitting discipline |
| 15 | Vesuvius Ink | vesuvius-challenge-ink-detection | Segmentation (3D→2D) | F0.5 | ~37 GB | yes | High | likely | Hard, near the disk limit |
| 16 | VinBigData CXR | vinbigdata-chest-xray-abnormalities-detection | Detection | mAP@0.4 | ~200 GB **Big** | no | High | likely | Disk |
| 17 | SIIM COVID-19 | siim-covid19-detection | Detection + classification | mAP@0.5 | ~120 GB **Big** | yes | High | likely | Disk, complex format |
| 18 | NFL Player Contact | nfl-player-contact-detection | Video + tracking | MCC | large | yes | High | likely | Only video task in MLE-bench |
| 19 | Sartorius Cell | sartorius-cell-instance-segmentation | Instance segmentation | mAP @ IoU 0.5–0.95 | ~2.4 GB | yes | no | likely | Best small instance-segmentation test |
| 20 | Global Wheat | global-wheat-detection | Detection | mAP @ IoU 0.5–0.75 | ~0.6 GB | yes | no | likely | Small detection test; watch weight licenses |
| 21 | Severstal Steel | severstal-steel-defect-detection | Semantic segmentation (4) | Mean Dice | ~1.6 GB | yes | no | likely | Class imbalance, empty masks |
| 22 | Airbus Ship | airbus-ship-detection | Instance segmentation | F2 over IoU | ~29 GB | no | no | likely | Mostly empty images |
| 23 | Facial Keypoints | facial-keypoints-detection | Keypoints | RMSE | ~80 MB | no | no | yes | Only keypoints option, very cheap |
| 24 | CSIRO Image2Biomass | csiro-biomass | Image regression | unverified | unverified | unverified | no | unverified | 2026 competition, ended |
| 25 | Recod.ai Image Forgery | recodai-luc-scientific-image-forgery-detection | Forgery segmentation | unverified | unverified | unverified | no | unverified | 2026 competition, reportedly closed in April 2026 |
| 26 | Hyperspectral Object Detection 2026 | hyperspectral-object-detection-challenge-2026 | Detection (hyperspectral) | unverified | unverified | unverified | no | unverified | Community competition; dates unknown |

**Active competitions.** No featured computer-vision competition was confirmed open on 2026-11-20. Check `kaggle competitions list --category featured` when the adapter exists.

## Recommended tiers

| Tier | Competitions | Purpose |
|---|---|---|
| **T0 Smoke** | #1 Aerial Cactus, #7 Denoising, #23 Facial Keypoints | Pipeline correctness in minutes, plus unusual output formats |
| **T1 Dev (local grading)** | #2, #3, #8, #9, #11, #14 | Tuning the harness. MLE-bench grades them locally and they are small. |
| **T1 Dev (primary)** | Solar Filament | The headline development target (see above) |
| **T1 Dev (late submission)** | #19, #21 | Fill the instance and semantic segmentation gaps |
| **T2 Held-out** | #4, #5, #6, #10, #13, plus #15 if disk allows | Never tuned on. Used for medal-rate reporting and the loop-2 promotion gate. |
| **Excluded for now** | #16–18 | Disk and compute; revisit with cloud runners (phase 3) |

## Sources
- MLE-bench: https://github.com/openai/mle-bench (splits in `experiments/splits/*.txt`)
- Solar Filament: https://www.kaggle.com/competitions/filament-segmentation-2026, https://uksolphys.org/general-news/solar-filament-segmentation-challenge-2026/
- Participant repos: https://github.com/Emualliug2705/kaggle_filament_2026, https://github.com/MaxWellApexLab/solar-filament-segmentation-2026, https://github.com/pragyaangaur/NSO-Solar-Filament-Challenge, https://github.com/RenziPaolo/Solar-Filament-Segmentation-Challenge-2026
