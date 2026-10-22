# UQ in autonomous driving — literature map

> [!abstract] Purpose and scope A structured map of uncertainty quantification (UQ), OOD detection and failure prediction work in autonomous driving, organised by **pipeline stage**. For each paper it records: what is uncertain, the method family, **what information the signal uses**, the **UQ task**, the **evaluation tool or dataset**, and whether the **quality of the uncertainty is measured directly** (AUROC, calibration, coverage, detection delay) or only through a downstream driving score.
> 

---
## 0. How this map was built (and its limits)

- **Searches** (web, Oct 2026) covered: surveys; UQ/OOD benchmarks; perception (3D detection, segmentation anomalies, BEV/occupancy, online maps); trajectory prediction (ensembles, evidential, conformal, OOD monitoring); planning and end-to-end driving (imitation, RL, diffusion planners, trajectory selection); system-level misbehaviour prediction (software-testing community); conformal safe planning; VLM/VLA reliability; world models and video-prediction anomaly detection.
- **Verification depth**: each row was checked against at least the abstract; rows with method/tool/metric details were checked against the paper body where accessible. Remaining gaps are marked ⚠️.
- **Not a systematic review** (no PRISMA protocol, no citation snowballing to exhaustion). Perception UQ in particular has a much larger literature than shown; I kept representative or driving-specific entries.

---

## 1. One-page overview

| Stage                               | Main method families                                                                                                                                                                                                                           | Where UQ quality is measured directly                                 | Typical evaluation tools                                                                   |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **Perception**                      | Calibration (post-hoc, density-aware), ensembles, MC dropout, evidential, feature-space OOD, learned introspection, conformal boxes                                                                                                            | Often (ECE, D-ECE, AUROC, AUPR, FPR95)                                | KITTI, nuScenes(-C), MultiCorrupt, BDD100K, SMIYC, Fishyscapes, StreetHazards, CODA, CARLA |
| **Mapping**                         | Laplace heads on map vertices                                                                                                                                                                                                                  | Rarely; judged through prediction or planning gains                   | nuScenes                                                                                   |
| **Prediction**                      | Ensembles, entropy decomposition, evidential, heteroscedastic GPs, conformal (split, online, copula), error-stream change detection                                                                                                            | Often (coverage, NLL, correlation with error, AUROC, detection delay) | nuScenes, Argoverse 1/2, WOMD, highD, NGSIM, ApolloScape, Shifts, nuPlan logs              |
| **Planning / E2E**                  | Ensembles of imitative models, distributional RL ensembles, BNN controllers, sample spread of diffusion planners, learned uncertainty heads, perception uncertainty fed to planner, learned collision predictors, evidential fusion with rules | **Rarely**: mostly via driving scores                                 | CARLA (CARNOVEL, Longest6), NeuroNCAP, nuScenes, NAVSIM, nuPlan, custom sims               |
| **System-level runtime monitoring** | Autoencoder reconstruction, attention maps, MC dropout / ensembles on the driving model, telemetry forecasting, black-box failure classifiers                                                                                                  | Yes (precision/recall/F3, AUROC, anticipation window)                 | Udacity simulator, CARLA, real test-drive logs                                             |
| **Conformal safe planning**         | CP regions on predicted agents → MPC / reachability                                                                                                                                                                                            | Yes (coverage, violation rate)                                        | CARLA, ORCA, nuScenes, WOMD + Nocturne, toy sims                                           |
| **VLM / VLA**                       | Entropy + grounding scores, robustness benchmarks, hallucination persistence                                                                                                                                                                   | Yes for QA (AUROC, ECE, Brier); **not yet for trajectories**          | DriveBench, DrivingVQA, NuScenes-QA, DIQ-H                                                 |
| **World models / video prediction** | Prediction-error anomaly scores, epistemic UQ for occupancy                                                                                                                                                                                    | Sometimes                                                             | A3D, nuScenes-based occupancy                                                              |

---

## 2. Surveys

|Survey|Venue / year|Scope|
|---|---|---|
|_Uncertainty Quantification for Safe and Reliable Autonomous Vehicles: A Review of Methods and Applications_|IEEE T-ITS, 2025|UQ methods across the AV stack ⚠️ (full text not accessed)|
|_Uncertainty Quantification for Safety of the Intended Functionality (SOTIF) of Autonomous Driving: A Comprehensive Survey_|2025 ⚠️|UQ framed by SOTIF (ISO 21448)|
|_A Survey on an Emerging Safety Challenge for Autonomous Vehicles: SOTIF_|Engineering, 2024|SOTIF, including uncertainty sources|
|_Trajectory Prediction for Autonomous Driving: Progress, Limitations, and Future Directions_|arXiv 2025|§5 on uncertainty sources, quantification and mitigation in prediction|
|_Anomaly Detection in Autonomous Driving: A Survey_ (Bogdoll et al.)|CVPRW 2022|Anomaly/corner-case detection across sensors|
|_Out-of-distribution detection in 3D applications: a review_|arXiv 2025|OOD for 3D perception|
|_Diffusion Models for End-to-End Autonomous Driving: A Survey_ (Choi et al.)|IEEE Access 2026|Mentions sample-based and ensemble UQ in diffusion pipelines|

> [!note] Gap in the survey literature I found no survey dedicated to UQ for **end-to-end, VLA or WAM driving policies**. Existing surveys centre on perception and prediction, or on SOTIF.

---

## 3. Benchmarks and datasets built for UQ / OOD / failures in driving

| Benchmark                                             | Stage                | What it provides for UQ                                                                                                                     | Tool type                                               |
| ----------------------------------------------------- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| **Shifts** (Malinin et al., NeurIPS 2021 D&B)         | Prediction           | Vehicle motion prediction from an industrial fleet with real, in-the-wild shifts (location, weather ⚠️); retention-curve metrics (R-AUC ⚠️) | Real logs, open loop                                    |
| **CARNOVEL** (Filos et al., ICML 2020)                | Planning             | Novel driving scenes for testing distribution-shift detection and recovery                                                                  | CARLA ⚠️ (CARLA confirmed for the RIP workshop version) |
| **SegmentMeIfYouCan** (Chan et al., NeurIPS 2021 D&B) | Perception           | Road anomaly and obstacle segmentation; pixel and component-level metrics                                                                   | Real images                                             |
| **Fishyscapes** (Blum et al., IJCV 2021)              | Perception           | Anomaly segmentation on urban scenes                                                                                                        | Real + synthetic overlays                               |
| **StreetHazards** (Hendrycks et al., ICML 2022)       | Perception           | Anomaly segmentation with inserted unknown objects                                                                                          | CARLA-rendered                                          |
| **CODA** (Li et al., ECCV 2022)                       | Perception           | Real-world road corner cases for object detection                                                                                           | Real images                                             |
| **nuScenes-C / MultiCorrupt**                         | Perception           | Corrupted camera/LiDAR versions of nuScenes for calibration under shift                                                                     | Real + synthetic corruption                             |
| **BEV-segmentation UQ benchmark** (2024)              | Perception           | Calibration, misclassification and OOD detection for BEV segmentation                                                                       | CARLA, nuScenes, Lyft, nuScenes-C                       |
| **nuScenes OOD for LiDAR detection** (2024)           | Perception           | Protocol for OOD objects in 3D detection without editing point clouds                                                                       | nuScenes                                                |
| **A3D** (Yao et al., 2019)                            | System / video       | Dashcam accident videos for unsupervised anomaly detection                                                                                  | Real video                                              |
| **NeuroNCAP** (ECCV 2024)                             | Planning             | Closed-loop safety-critical scenarios, 100 seeded runs each                                                                                 | NeRF-rendered nuScenes                                  |
| **Fail2Drive** (2026)                                 | Planning             | 200 routes with paired in-distribution / shifted versions                                                                                   | CARLA 0.9.15                                            |
| **WOD-E2E** (Waymo, 2025)                             | Planning             | Long-tail scenarios (< 0.03% frequency), rater-scored trajectories                                                                          | Real logs, open loop                                    |
| **PhysicalAI-AV OOD reasoning subset** (NVIDIA, 2026) | Planning / reasoning | Human-verified OOD scenarios with causal reasoning labels                                                                                   | Real logs                                               |

---

## 4. Papers by pipeline stage

Legend for the last column: ✅ uncertainty quality measured directly · ◐ partly (e.g. correlation only, qualitative) · ❌ only through a downstream task score.

### 4.1 Perception (detection, segmentation, BEV/occupancy, mapping)

|Paper|Venue / year|What is uncertain|Method family|Information used|UQ task|Evaluation tool|Direct?|
|---|---|---|---|---|---|---|---|
|_Can We Trust You? Calibration of a Probabilistic Object Detector_ (Feng et al.)|ITSC 2019 ⚠️|LiDAR 3D box regression|Probabilistic detector + 3 recalibration methods|Detector outputs|Calibration|Several datasets ⚠️ (names not in abstract)|✅ ⚠️|
|**Query2Uncertainty**|arXiv 2026|3D detection class + box|Density-aware post-hoc calibration: normalizing flows on DETR object queries modulate TS / IR|Latent object queries|Calibration under shift|nuScenes + **MultiCorrupt**; Boston→Singapore|✅ (D-ECE, LaECE, MCA)|
|_Revisiting OOD Detection in LiDAR 3D Object Detection_|arXiv 2024|Unknown objects|MLP on frozen detector features; synthetic OOD by perturbing known classes|Detector features|OOD object detection|nuScenes OOD benchmark|✅ ⚠️ (metrics not in abstract)|
|_Two-step Conformal Prediction for Bounding Boxes_|ECCV 2024|2D box extent incl. misclassification|Conformal prediction (ensemble / quantile variants)|Detector outputs + class sets|Coverage-guaranteed boxes|Real-world 2D datasets ⚠️|✅ (coverage)|
|**LFA** (Layer Feature Attention)|arXiv 2026|Frame-level detection failures|Learned introspection over backbone layers|Multi-layer features of frozen detector|Predict missed detections|KITTI, BDD100K|✅ (AUROC, F1, FNR)|
|_Predictive UQ for BEV Segmentation: Benchmark + Loss_|arXiv 2024|BEV semantic classes|Entropy, energy, ensembles, MC dropout, **evidential** (UFCE loss)|Model outputs|Calibration, misclassification, OOD|CARLA, nuScenes, Lyft, nuScenes-C|✅ (ECE, AUROC, AUPR, FPR95)|
|**OCCUQ**|ICRA 2025|3D occupancy|Efficient epistemic UQ (vs ensembles, MC dropout)|Model features ⚠️|OOD detection, calibration|Camera corruptions incl. single-camera ⚠️ dataset|✅ ⚠️|
|**EvOcc**|CVPR 2025|Occupancy ground truth (unobserved / conflicting)|Evidence theory|Raw LiDAR|Better labels and training loss|⚠️ dataset|❌ (ray-based mIoU)|
|_Producing and Leveraging Online Map Uncertainty_ (Gu et al.)|CVPR 2024|Online HD-map elements|Uncertainty heads on online mappers|Mapper outputs|Feed uncertainty to trajectory prediction|nuScenes|❌ (prediction gains, up to 15%)|

### 4.2 Trajectory prediction

|Paper|Venue / year|Method family|Information used|UQ task|Evaluation tool|Direct?|
|---|---|---|---|---|---|---|
|**Shifts** — motion prediction track|NeurIPS 2021 D&B|Baselines with ensemble-based uncertainty ⚠️|Model outputs|Robustness + uncertainty under real shift|Shifts data (industrial fleet)|✅ (retention metrics ⚠️)|
|**Stochasticity in Motion**|2024|Ensembles + MC dropout, entropy decomposition (aleatoric / epistemic)|Ensemble samples|Decompose uncertainty, detect OOD|nuScenes + synthetic OOD (reversed / blacked-out history, lane deletion)|◐ (correlation with minADE; no AUROC)|
|**Evidential multi-modal prediction** (Marvi et al.)|arXiv 2025|Evidential (Normal-Inverse-Gamma positions, Dirichlet modes), single pass|Model outputs|Position and mode uncertainty|Argoverse 1 and 2|✅ ⚠️ (metrics not in abstract)|
|**SHIFT** (rule-regularised heteroscedastic classification)|RSS 2025|Heteroscedastic spectral-normalised GPs; priors from LLM-encoded driving rules|Model outputs + rule priors|Separate epistemic / aleatoric; calibration|nuScenes incl. low-data and cross-location|✅ ⚠️|
|**RuleFuser**|ISRR 2024 ⚠️|Evidential Bayes fusion of learned planner with rule-based prior (normalizing-flow evidence)|Learned-model density + rules|Fall back to rules when OOD|**nuPlan cross-city** (Boston ↔ Singapore)|❌ (safety metrics, −38% OOD)|
|**Adaptive Prediction Ensemble**|arXiv 2024|Router between deep predictor and constant-velocity expert; also tests ensemble variance|Model outputs|OOD generalisation|WOMD ↔ Argoverse zero-shot|◐ (ensemble variance found too noisy)|
|**CUQDS**|2024|Online conformal under temporal shift|Observed errors|Calibrated intervals|Argoverse 1|✅ (coverage, NLL)|
|**Scenario-aware CP**|arXiv 2025|CopulaCPTS in Frenet frame + reliability discriminator|Residuals|Joint coverage, detect unreliable timestep|nuPlan logs|✅ (joint coverage, area)|
|**Multi-agent reachability calibration** (Muthali et al.)|arXiv 2023 ⚠️ venue|Quantile regression + rolling conformal + HJ reachability|Forecaster uncertainty + residuals|Calibrated reachable sets for planning|nuScenes; WOMD + **Nocturne** (MPC)|✅ (coverage, set size)|
|**DC-MMD OOD monitor**|arXiv 2026|Quickest change detection over the prediction-error stream|Only the scalar error|Runtime OOD detection|NGSIM, nuScenes, ApolloScape + bounded perturbations|✅ (WADD, MTFA, AUROC, FPR95)|
|**cVMD**|CVPRW 2024|Mahalanobis in VQ-VAE latent → CFG scale of diffusion predictor|Latent distance|Adaptive guidance|highD|❌ (ADE/FDE)|

### 4.3 Planning and end-to-end driving

|Paper|Venue / year|Policy|Method family|Information used|UQ task|Evaluation tool|Direct?|
|---|---|---|---|---|---|---|---|
|**Robust Imitative Planning (RIP)**|NeurIPS-W 2019 / ICML 2020|Deep imitative models|**Ensemble** of density estimators; plan robustly over members|Ensemble disagreement (epistemic)|Detect and recover from distribution shift; AdaRIP queries expert|CARLA, **CARNOVEL**; nuScenes prediction|❌ ⚠️ (driving outcomes)|
|_UQ with Statistical Guarantees in E2E Driving Control_ (Michelmore et al.)|ICRA 2020|BNN controllers|Bayesian inference variants|Posterior predictive|Calibrated collision-risk measures with guarantees|Driving simulator (CARLA ⚠️)|✅|
|**Ensemble Quantile Networks** (Hoel et al.)|2021 ⚠️ venue|Distributional RL|Quantiles (aleatoric) + bootstrapped ensemble (epistemic)|Return distributions|Flag decisions with unacceptable uncertainty|Occluded intersections (simulator ⚠️)|◐|
|**RACER**|2024|Off-road RL|Ensemble of distributional critics (epistemic)|Critic disagreement|Risk-sensitive fast driving|Sim + 1/10-scale F1TENTH car|❌ (crashes, speed)|
|**CATPlan**|2025|UniAD, VAD|Learned collision classifier (loss prediction)|**Planner internal queries**|Predict collision of planned trajectory|**NeuroNCAP** + nuScenes|✅ (AUROC, AP)|
|**EnDfuser**|NLDL 2026|Own **diffusion** planner|Sample spread (128 trajectories)|Output samples|Flag safety-critical situations; brake rule|CARLA Longest6 (arXiv v1) / LAV (NLDL abstract) ⚠️|◐ (qualitative)|
|**UncAD**|2025|VAD, SparseDrive|Laplace map uncertainty → planning and selection|Perception uncertainty|Safer planning|nuScenes|❌|
|**SUPER-AD**|2025|DiffusionDrive candidates|MC-sampled BEV semantic logits → safety map|Perception uncertainty|Score / filter trajectories|NAVSIM navtest, navhard + navsafe|❌|
|**Mimir**|2025|Hierarchical **diffusion** planner|Laplace head on goal point → guidance|Learned aleatoric head|Robust planning|NAVSIM v1/v2|❌|
|**UniUncer**|2026|SparseDrive, DiffusionDrive|Laplace heads on map + agent vertices, fusion + gating|Perception uncertainty|Robust planning|nuScenes, NAVSIM v2 navhard|❌ (calibration claimed, not measured ⚠️)|
|**Sample-conditioned differentiable planning**|2026|Diffusion predictor + differentiable planner|Sample-based CVaR|Output samples of others|Risk-aware planning|WOMD / Argoverse 2; custom closed loop|❌|

Found but not read (to triage): _Uncertainty-Aware Safety-Critical Decision and Control for Autonomous Vehicles at Unsignalized Intersections_ (arXiv 2505.19939).

### 4.4 System-level runtime monitoring and misbehaviour prediction

This line of work comes largely from the **software-engineering / testing** community (ICSE, ICST, ASE). It is rarely cited in ML driving papers, yet it is the closest existing practice to "predict failures of an end-to-end driving model ahead of time".

| Paper                                                                    | Venue / year       | Model monitored               | Signal                                              | Information used                       | Evaluation tool                                                        | Metrics                                                              | Direct?                              |
| ------------------------------------------------------------------------ | ------------------ | ----------------------------- | --------------------------------------------------- | -------------------------------------- | ---------------------------------------------------------------------- | -------------------------------------------------------------------- | ------------------------------------ |
| **SelfOracle** (Stocco et al.)                                           | ICSE 2020          | DAVE-2-style E2E lane keeping | Autoencoder reconstruction error                    | Input images (black-box w.r.t. policy) | Udacity simulator                                                      | Precision / recall, anticipation                                     | ✅                                    |
| **ThirdEye** (Stocco et al.)                                             | ASE 2022           | E2E lane keeping              | Attention-map statistics                            | Policy attention maps (white-box)      | Udacity simulator                                                      | As above                                                             | ✅                                    |
| **Fail-safe execution through uncertainty monitoring** (Weiss & Tonella) | ICST 2021          | DL systems                    | Supervisors (MC dropout, ensembles, …)              | Model uncertainty                      | ⚠️                                                                     | ⚠️                                                                   | ✅ ⚠️                                 |
| **Predicting safety misbehaviours using UQ** (Grewal, Tonella, Stocco)   | ICST 2024 ⚠️ venue | DAVE-2                        | **MC dropout, deep ensembles** on the driving model | Output variance                        | Udacity simulator; OOD weather/lighting + mutated models; 265 failures | Precision, recall, **F3**, AUROC, 1–3 s windows; ensembles F3 90–94% | ✅                                    |
| _Robustness of UQ-based misbehaviour predictors: a case study_           | SE4ADS 2025        | ⚠️                            | ⚠️                                                  | ⚠️                                     | ⚠️                                                                     | ⚠️                                                                   | ⚠️ (abstract unavailable)            |
| **Foresee** (misbehaviour forecasting for focused testing)               | ICSE 2026          | InterFuser, TransFuser        | Forecast trajectory intersection from telemetry     | Ego + NPC telemetry                    | CARLA 0.9.10.1, Town10                                                 | Failures exposed, efficiency                                         | ◐ (used for testing, not runtime UQ) |
| **Introspective failure prediction (late fusion)** (Kuhn et al.)         | IEEE T-ITS 2022    | Full AV stack (black box)     | Classifiers on camera + vehicle state, fused        | Sensor sequences                       | 14 h of BMW test-drive disengagements                                  | >85% accuracy 7 s ahead at 20% FPR                                   | ✅                                    |

### 4.5 Conformal and statistical safety for planning

|Paper|Venue / year|What is calibrated|Planner|Evaluation tool|Metrics|Direct?|
|---|---|---|---|---|---|---|
|**Conformal MPC** (Lindemann et al.)|2023|Pedestrian trajectory predictions (social LSTM)|MPC|**CARLA** intersection; ORCA / TrajNet++|Coverage, 1/100 violations|✅|
|**Robust CP with generative priors**|L4DC 2026 ⚠️|Nonconformity under observed shift parameter (diffusion model)|MPC|ORCA|Safety under shift|✅ ⚠️|
|**Iterative policy updates + adversarially robust CP**|L4DC 2026|Predictions under endogenous, interaction-induced shift|MPC|2–3-agent sims; car–pedestrian, quadcopter|Success +9.6%, collisions|✅ ⚠️|
|**Perceive with Confidence**|CoRL 2024 (prelim.) / journal ext.|Perception misdetection rate (boxes, occupancy)|Any safe planner|**Indoor quadruped**, not driving|Misdetection, safety|✅|

### 4.6 VLM / VLA reliability in driving

|Paper|Venue / year|What is uncertain|Signal|Evaluation tool|Metrics|Direct?|
|---|---|---|---|---|---|---|
|**DriveBench** ("Are VLMs ready for AD?")|ICCV 2025|Grounding and reliability of driving VLMs|Behaviour under 17 input settings (clean, corrupted, text-only)|19,200 frames, 20,498 QA pairs; 12 VLMs|Accuracy, proposed grounding metrics|◐ (reliability, not UQ scores)|
|**DIQ-H**|arXiv 2025|Hallucination persistence after transient corruption|Uncertainty-filtered pseudo-labels (UIR)|500+ degraded video sequences, 16 VLMs|Hallucination / recovery rate, temporal consistency|◐|
|**VAUQ reliability study**|arXiv 2026|Correctness of VLM answers (incl. **Alpamayo-1.5**)|Entropy + image-information score|DrivingVQA, NuScenes-QA-mini, STRIDE-QA, OSR + corruptions|AUROC, ECE, Brier|✅ (QA only)|

> [!warning] Gap No paper found that measures the calibration or failure-predictive power of a **driving VLA's trajectory output** (Alpamayo, AutoVLA, ReCogDrive, …). Existing work measures VLM _answers_.

### 4.7 World models and video-prediction-based anomaly detection

| Paper                                                     | Venue / year   | Signal                                                        | Evaluation tool                    | Metrics | Direct?                |
| --------------------------------------------------------- | -------------- | ------------------------------------------------------------- | ---------------------------------- | ------- | ---------------------- |
| **Unsupervised traffic accident detection** (Yao et al.)  | 2019 (IROS ⚠️) | Accuracy and consistency of predicted future object locations | A3D (new) + another public dataset | ⚠️      | ✅ ⚠️                   |
| **OCCUQ**                                                 | ICRA 2025      | Epistemic UQ for (current) occupancy, not forecasting         | See §4.1                           |         |                        |
| Driving world-model papers (DriveWAM, Epona, DriveLaW, …) | 2025–26        | —                                                             | —                                  | —       | No UQ evaluation found |

> [!note] Link to the WAM idea The 2019 accident-detection work is an early instance of "surprise = predicted vs observed future", the signal proposed for driving WAMs in [[ad-uq-tools-and-models]]. Modern video WAMs have not been evaluated this way.

---

## 5. Cross-cutting analysis

### 5.1 Where direct UQ evaluation happens

1. **Perception and prediction** have mature UQ evaluation: calibration (ECE variants), OOD detection (AUROC, AUPR, FPR95), conformal coverage. Dedicated benchmarks exist (SMIYC, Fishyscapes, StreetHazards, Shifts, nuScenes-C, MultiCorrupt).
2. **Planning and end-to-end** mostly evaluate uncertainty _indirectly_: UncAD, SUPER-AD, Mimir, UniUncer, RuleFuser and RACER report driving or safety scores only. Exceptions with direct metrics are CATPlan (AUROC/AP on NeuroNCAP) and Michelmore et al. (calibrated risk).
3. **System-level misbehaviour prediction** (software-testing community) does evaluate directly, with anticipation windows and failure labels, but almost entirely on **lane-keeping models in the Udacity simulator**. A notable exception is Kuhn et al., on **real BMW disengagement data**.

### 5.2 Information used by the signal

|Family|Examples|Cost|Notes|
|---|---|---|---|
|Output samples / sample spread|EnDfuser, CVaR planner, FIPER-style action entropy|N forward passes of the head|Captures multimodality; aleatoric vs epistemic confounded|
|Ensembles / Bayesian|RIP, EQN, RACER, Grewal et al., Michelmore et al.|M models|Strongest empirical results in misbehaviour prediction (Grewal: ensembles ≫ MC dropout)|
|Learned uncertainty heads|Mimir, UncAD, UniUncer, Gu et al.|Single pass|Aleatoric only; rarely validated|
|Internal features / introspection|CATPlan, LFA, ThirdEye, Query2Uncertainty|Single pass + probe|Needs failure labels (except density-based)|
|Input-space novelty|SelfOracle (autoencoder), LiDAR OOD MLP|Extra model|Black-box w.r.t. policy|
|Residuals / errors after the fact|Conformal methods, DC-MMD|Cheap|Needs the future to be observed; guarantees under exchangeability|
|Rule / prior fusion|RuleFuser, SHIFT|Single pass|Uses domain knowledge as fallback prior|

### 5.3 Gaps relevant to this project

1. **No direct UQ evaluation for diffusion VLAs or WAMs.** Diffusion-planner UQ in driving is limited to EnDfuser (sample spread, qualitative) and uses of diffusion samples for risk (CVaR planner, cVMD). Diffusion-_process_ signals (denoising path, score Jacobians, Fisher–Laplace) are absent from driving.
2. **Two communities, little overlap.** ML driving papers (NAVSIM, Bench2Drive) rarely report failure-prediction metrics. The SE testing community does, but on small lane-keeping models. Combining their _evaluation protocol_ (failure labels, anticipation windows, F-scores, false-alarm rates) with modern _policies and tools_ (Alpamayo + AlpaSim, NAVSIM) is open.
3. **Closed-loop UQ evaluation with neural rendering** exists only via NeuroNCAP (CATPlan). AlpaSim, HUGSIM and Fail2Drive have no UQ papers yet.
4. **Perception uncertainty fed into planners is never validated as uncertainty**: it is judged by planning gains only.
5. **Interaction-induced shift** is recognised in the conformal literature (adversarially robust CP), but not in end-to-end driving UQ.

---

## 6. Search log (queries run, Oct 2026)

Surveys: "survey uncertainty quantification autonomous driving"; "uncertainty end-to-end autonomous driving survey". Benchmarks: Shifts, CARNOVEL, SMIYC/Fishyscapes, CODA. Perception: 3D detection calibration; online map uncertainty; BEV/occupancy uncertainty; LiDAR OOD; conformal bounding boxes; introspective perception. Prediction: UQ benchmarks for trajectory prediction; evidential; conformal multi-agent; OOD trajectory prediction. Planning: epistemic uncertainty imitation learning; uncertainty-aware RL; uncertainty in NAVSIM trajectory selection; diffusion planner uncertainty; learned planner fallback. System level: SelfOracle; ThirdEye; uncertainty monitoring (Weiss & Tonella); misbehaviour prediction 2024–25. VLM/VLA: hallucination benchmarks; VLA confidence calibration. World models: world-model uncertainty; video-prediction anomaly detection. Conformal: perception-to-planning conformal; LLM planners asking for help.

**Next steps to make this exhaustive**: forward-citation search from Filos et al. 2020, Stocco et al. 2020, Michelmore et al. 2020 and SMIYC; query IEEE IV / ITSC / T-IV / T-ITS proceedings, which are under-represented in web search; add the anomaly-segmentation leaderboard methods.

---

## 7. Sources

**Surveys**: [UQ for safe and reliable AVs (T-ITS 2025)](https://dl.acm.org/doi/abs/10.1109/TITS.2025.3532803) · [UQ for SOTIF survey](https://www.researchgate.net/publication/394993206_Uncertainty_Quantification_for_Safety_of_the_Intended_Functionality_of_Autonomous_Driving_A_Comprehensive_Survey) · [SOTIF survey (Engineering)](https://www.sciencedirect.com/science/article/pii/S2095809924000274) · [Trajectory prediction survey](https://arxiv.org/html/2503.03262v3) · [Anomaly detection in AD survey](https://arxiv.org/pdf/2204.07974) · [OOD in 3D review](https://arxiv.org/pdf/2507.00570) · [Diffusion for E2E AD survey](https://www.researchgate.net/publication/407282404_Diffusion_Models_for_End-to-End_Autonomous_Driving_A_Survey_of_Perception_Prediction_Planning_and_Control)

**Benchmarks**: [Shifts](https://arxiv.org/abs/2107.07455v2) · [Filos et al. / CARNOVEL](https://arxiv.org/abs/2006.14911v1) · [RIP workshop version](https://oatml.cs.ox.ac.uk/publications/201912_Tigas2019Robust.html) · [SegmentMeIfYouCan](https://neurips.cc/virtual/2021/29819) · [CODA](https://arxiv.org/pdf/2203.07724) · [NeuroNCAP](https://arxiv.org/pdf/2404.07762) · [Fail2Drive](https://arxiv.org/html/2604.08535v1) · [WOD-E2E](https://arxiv.org/html/2510.26125v2) · [PhysicalAI-AV](https://huggingface.co/datasets/nvidia/PhysicalAI-Autonomous-Vehicles)

**Perception**: [Feng et al. calibration](https://arxiv.org/abs/1909.12358v1) · [Query2Uncertainty](https://arxiv.org/pdf/2605.05328) · [LiDAR OOD](https://arxiv.org/abs/2404.15879v1) · [Two-step CP boxes](https://dare.uva.nl/id/729c3f35-6547-4dbe-8ac3-fe2a2b8bc40c) · [LFA](https://arxiv.org/pdf/2606.00372) · [BEV UQ benchmark](https://arxiv.org/html/2405.20986v2) · [OCCUQ](https://arxiv.org/abs/2503.10605) · [EvOcc](https://openaccess.thecvf.com/content/CVPR2025/html/Kalble_EvOcc_Accurate_Semantic_Occupancy_for_Automated_Driving_Using_Evidence_Theory_CVPR_2025_paper.html) · [Online map uncertainty](https://arxiv.org/abs/2403.16439v1)

**Prediction**: [Stochasticity in Motion](https://arxiv.org/pdf/2410.01628) · [Evidential multi-modal prediction](https://arxiv.org/abs/2503.05274v1) · [SHIFT (RSS 2025)](https://arxiv.org/abs/2504.13111v3) · [RuleFuser](https://arxiv.org/html/2405.11139v2) · [Adaptive Prediction Ensemble](https://arxiv.org/pdf/2407.09475v2) · [CUQDS](https://arxiv.org/pdf/2406.12100) · [Scenario-aware CP](https://arxiv.org/html/2512.05682v1) · [Multi-agent reachability calibration](https://arxiv.org/pdf/2304.00432) · [DC-MMD](https://arxiv.org/pdf/2603.14603) · [cVMD](https://arxiv.org/pdf/2405.14384)

**Planning / E2E**: [Michelmore et al.](https://arxiv.org/abs/1909.09884) · [EQN](https://export.arxiv.org/abs/2105.10266) · [RACER](https://arxiv.org/pdf/2405.04714) · [CATPlan](https://arxiv.org/html/2503.07425v1) · [EnDfuser](https://arxiv.org/html/2506.00560v1) · [EnDfuser NLDL](https://openreview.net/forum?id=7o4nocapUr) · [UncAD](https://arxiv.org/html/2504.12826v1) · [SUPER-AD](https://arxiv.org/html/2511.22865v1) · [Mimir](https://arxiv.org/html/2512.07130v1) · [UniUncer](https://arxiv.org/html/2603.07686) · [Sample-conditioned planning](https://arxiv.org/html/2606.03296) · [Unsignalized intersections](https://arxiv.org/pdf/2505.19939)

**System-level monitoring**: [SelfOracle](https://arxiv.org/pdf/1910.04443) · [ThirdEye](https://conf.researchr.org/details/ase-2022/ase-2022-research-papers/111/ThirdEye-Attention-Maps-for-Safe-Autonomous-Driving-Systems) · [Weiss & Tonella ICST 2021](https://icst2021.icmc.usp.br/details/icst-2021-papers/7/Fail-Safe-Execution-of-Deep-Learning-based-Systems-through-Uncertainty-Monitoring) · [Grewal et al.](https://arxiv.org/html/2404.18573v2) · [UQ predictors robustness (SE4ADS)](https://www.researchgate.net/publication/393210993_Evaluating_the_Robustness_of_Uncertainty_Quantification-Based_Misbehavior_Predictors_for_Autonomous_Driving_Systems_A_Case_Study) · [Foresee](https://arxiv.org/html/2512.18823) · [Kuhn et al.](https://portal.fis.tum.de/en/publications/introspective-failure-prediction-for-autonomous-driving-using-lat/)

**Conformal safety**: [Conformal MPC](https://arxiv.org/pdf/2210.10254) · [Robust CP](https://arxiv.org/html/2602.12616v1) · [Adversarially robust CP](https://arxiv.org/pdf/2511.10586) · [Perceive with Confidence](https://arxiv.org/html/2403.08185v3)

**VLM / VLA / world models**: [DriveBench](https://drive-bench.github.io/) · [DIQ-H](https://arxiv.org/html/2512.03992v1) · [VAUQ reliability study](https://arxiv.org/html/2610.01531v1) · [Accident detection (Yao et al.)](https://arxiv.org/abs/1903.00618v4)