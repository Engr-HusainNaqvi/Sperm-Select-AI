# Methodology selection — SpermSelect-AI

This follows a structured design process: frame the problem, list five candidate AI solutions,
five hybrid pipelines, then choose one and justify it against interpretability, compute, data
requirements and expected accuracy.

## 1. Problem summary

**Biology.** In ICSI (intracytoplasmic sperm injection) an embryologist picks *one* spermatozoon
per oocyte, by eye, under a microscope. The fertilised embryos are cultured and the good ones are
**vitrified** (ultra-rapid cryopreservation) for later transfer. Sperm selection is the first
quality gate in that chain: poor motility and kinematics are associated with lower fertilisation
and embryo quality, and selection today is subjective and varies between operators.

**AI problem.** From a phase-contrast microscopy video of a wet semen preparation:

1. **detect** every sperm head in every frame (small ~19 × 17 px objects, crowded, out-of-focus),
2. **track** each one over time (identical-looking objects → identity must come from motion),
3. **measure** WHO/CASA kinematics per sperm (VCL, VSL, VAP, LIN, STR, WOB, ALH, BCF),
4. **grade** each sperm (progressive / non-progressive / immotile) and **rank** the best candidates,
5. **report** sample-level values (progressive %, total motility %, concentration) that match the lab.

**The data obstacle.** No public dataset has a per-sperm motility label. VISEM gives *sample-level*
lab percentages for 85 men; VISEM-Tracking gives boxes and track IDs for 20 videos but no motility
class per track. Any per-sperm grader therefore has to be learned from **sample-level proportions**.

## 2. Five candidate AI solutions

| # | Solution | AI methods | Rationale / biological insight | Main weakness |
|---|---|---|---|---|
| S1 | Classical CASA | DoG blob detection + Kalman/Hungarian tracking + WHO rule thresholds | Transparent, the clinical standard | Hand-set thresholds; fails on debris, crowding |
| S2 | Learned detector + tracker | YOLO (CNN) fine-tuned on VISEM-Tracking + Kalman filter + Hungarian assignment | Robust detection of small, blurred heads | Still needs a grading rule |
| S3 | Video regression | 3D-CNN / CNN+LSTM straight from frames → motility % | End-to-end, no tracking | Black box; 85 samples → over-fits; no per-sperm output |
| S4 | Distribution regression | Gradient boosting (LightGBM) / Ridge on histograms of track kinematics | Strong on small tabular data | Sample-level only, cannot rank individual sperm |
| S5 | Unsupervised motility types | GMM / k-means on per-track kinematics, then map clusters to WHO classes | Finds natural motility sub-populations (e.g. hyperactivated) | Cluster→class mapping is ad hoc |

## 3. Five hybrid methodologies

| # | Hybrid | How it improves on the classical method |
|---|---|---|
| H1 | **YOLO + Kalman tracker + learning-from-label-proportions (LLP) grader** | Per-sperm softmax classifier on kinematics, trained so that the *average* of its per-sperm probabilities reproduces each man's lab percentages. Gives per-sperm grades without per-sperm labels. |
| H2 | YOLO + tracker + GMM mixture whose weights are regressed to lab % | Probabilistic, interpretable mixture; but the mixture components are fixed before supervision. |
| H3 | CNN + Transformer over track sequences (set encoder) → motility % | Attention over all tracks; needs far more than 85 samples. |
| H4 | Self-supervised video autoencoder (336 unlabeled clips) + Ridge | Uses unlabeled data; features are hard to explain to a clinician. |
| H5 | H1 + Bayesian quality score for selection (posterior P(progressive) × straightness × track reliability) | Turns grades into a ranked shortlist with uncertainty, which is what an embryologist needs. |

## 4. Chosen methodology: H1 + H5 ("SpermSelect")

| Criterion | Why H1 + H5 wins |
|---|---|
| Interpretability | Every number is a WHO kinematic quantity; the grader is a small softmax model whose weights can be read. |
| Compute | Detector runs on CPU; the grader trains in seconds. A clinic can run it on a laptop. |
| Data needs | Uses exactly the supervision that exists: boxes for the detector, lab percentages for the grader. |
| Accuracy | Evaluated on the public VISEM benchmark (lab motility, cross-validated by participant) against ZeroR, classical CASA rules and gradient boosting. |
| Discovery | Per-sperm grades make it possible to study sub-populations and their link to DNA fragmentation (DFI). |

The end-to-end video models (S3/H3) were rejected: with 85 participants they over-fit, and they
cannot say *which* sperm to pick, which is the clinical question.

## 5. Implementation outline

1. Detector: YOLO11n fine-tuned on VISEM-Tracking, split by video (no frame leakage). Compared with
   a training-free DoG detector (mAP@0.5, precision, recall).
2. Tracker: constant-velocity Kalman + Hungarian. Scored against the ground-truth track IDs
   (MOTA, IDF1, ID switches).
3. Kinematics: WHO 2021 definitions at 0.495 µm/px (UI-2210C 9.9 µm pixels / (40× objective × 0.5×
   C-mount adapter)). Checked on the images: a human flagellum (~50 µm) spans 80–100 px.
4. Grader: LLP softmax on standardised kinematics, cross-validated by participant on VISEM (85 men,
   lab progressive / non-progressive / immotile %). Baselines: ZeroR, WHO rule thresholds, LightGBM
   on kinematic histograms.
5. Selection: rank tracks by the quality score, show the top candidates with their trajectories.
6. Clinical app: upload a video → overlay, ranked shortlist, WHO-style report and PDF.

**Scope note.** The system ranks sperm on motility and kinematics. It does not observe fertilisation,
embryo development or vitrification survival. Linking the selection score to embryo outcomes needs
paired ICSI data; that is the proposed PhD extension.
