# AD evaluation tools, driving models and UQ evaluation

> [!abstract] Purpose Working note for the project _uncertainty quantification (UQ) at the diffusion level for driving VLAs and WAMs_. It separates four things that are often mixed up:
> 
> 1. **Simulator software** versus **scenes, datasets and benchmark design** (§2).
> 2. **Models**: what each was trained on and how it produces its trajectory (§3).
> 3. **Model evaluation**: how well a model drives, per benchmark, with leaderboards (§4).
> 4. **UQ evaluation**: how well an uncertainty signal predicts what it claims to, and on which tool each UQ paper was actually evaluated (§5).

---

## 1. Key distinctions

### 1.1 Loop and input type

|Concept|Meaning|Consequence for UQ|
|---|---|---|
|**Open loop**|The model is queried on logged data; its output never affects what happens next.|You can test whether uncertainty _ranks bad plans or unusual scenes_, not whether it _predicts failures the model itself causes_.|
|**Pseudo-closed loop**|One or two decisions per scene, scored by simulating the ego against logged or simple reactive agents (NAVSIM).|No self-correction, no compounding errors; hedging ("slow down, then re-decide") cannot be rewarded.|
|**Closed loop**|The model drives; its actions change the next observation; it replans continuously.|Needed for failure prediction with lead time, uncertainty-triggered fallbacks and adaptive replanning.|
|**Sensor-level vs object-level**|Camera/LiDAR input vs tracked boxes and map polylines.|VLAs and WAMs need sensors. Object-level fits mid-to-mid planners (e.g. Diffusion Planner) and lets you inject perception uncertainty with known ground truth.|
|**Synthetic vs reconstructed rendering**|Game engine (CARLA) vs neural reconstruction of real drives (AlpaSim, HUGSIM, NeuroNCAP, NAVSIM v2 stage 2).|Synthetic rendering creates a domain gap. Reconstruction degrades off the recorded path, so uncertainty may fire on **rendering artifacts** rather than hard situations.|

> [!warning] The core trade-off No tool offers all three of **real sensor data + closed loop + realistic interactive traffic**. Every choice gives up at least one.

### 1.2 Model evaluation vs UQ evaluation

These are different experiments. They need different features from a tool, and their results live in different spaces.

||Model evaluation|UQ evaluation|
|---|---|---|
|**Question**|How well does the policy drive?|Does the uncertainty signal anticipate errors, failures or shifts, and is it calibrated?|
|**Output space**|A driving score per route or scene (PDMS, EPDMS, DS, RFS, AlpaSim score).|A score per decision or timestep, compared to a target (failure label, error, OOD flag).|
|**Metrics**|Benchmark-specific aggregates.|AUROC / AUPRC, risk–coverage curves, ECE / Brier, conformal coverage and set size, detection delay and false-alarm rate (lead time).|
|**What the tool must provide**|Standard scenes, metric, ideally an official leaderboard.|(a) **Labelled targets**: failures or errors, ideally timestamped; (b) **enough failures** to estimate rare-event metrics; (c) **episodes** for lead time; (d) **controllable shift** to separate OOD from benign difficulty; (e) **white-box access** to internals and the ability to draw many samples per step; (f) **ground-truth world state** if perception uncertainty is studied.|
|**Domain transfer**|A model can be evaluated on a benchmark it wasn't trained on, but the score then mixes model quality with domain gap.|A UQ signal evaluated across a domain gap is often the _point_ (OOD detection), but the gap must be characterised, or the signal may only detect "this looks like a different simulator".|

> [!important] Consequence A good driving benchmark can be a poor UQ benchmark. NAVSIM ranks planners well on real data but yields one decision per scene and no model-caused failures. Conversely, a UQ study needs failures, so it often needs a _harder_ or _shifted_ environment than the one the model shines on.

---

## 2. Tools: software vs scenes vs benchmarks

### 2.1 Three layers

Every "tool" bundles up to three separable layers:

1. **Software (simulator / evaluator)**: what advances the world and computes metrics. Physics or vehicle model, renderer, traffic agents, scoring code.
2. **Scenes / data**: where the world content comes from. Hand-built maps, logged drives, neural reconstructions, generated content. Also the _training data_ that comes with a benchmark (often produced by an expert driver).
3. **Benchmark design**: which scenes or routes are evaluated, with which metric, under which rules (splits, submission policy, leaderboard).

|Tool|Software layer|Scene / data layer|Benchmark layer|Are layers separable?|
|---|---|---|---|---|
|**CARLA**|Unreal Engine simulator (0.9.15 used by Bench2Drive; 0.9.16 used by LEAD); physics, scripted traffic, ScenarioRunner.|Hand-built towns (Town01–15), assets, weather. Expert drivers (PDM-Lite, LEAD) generate training data.|Leaderboard 2.0 routes; **Bench2Drive** (220 routes, 44 scenarios, plus an official training set); **Longest6 v2**; **Fail2Drive** (200 routes in Town13 with 17 shift scenario classes).|**Yes, cleanly.** One simulator hosts many benchmarks. However, the _expert used to generate training data_ is de facto part of the benchmark (see §4.3).|
|**nuPlan**|nuPlan devkit: closed-loop simulation (LQR tracker + kinematic model), IDM agents, metrics.|nuPlan logs (Boston, Pittsburgh, Las Vegas, Singapore), object-level.|Splits largely **defined by the community**: Val14 (PDM paper), Test14 / Test14-hard (PlanTF), interPlan.|Mostly. The software is tied to the nuPlan data format, but benchmark splits evolve independently.|
|**NAVSIM**|Devkit = **evaluator**: scores a 4 s trajectory against logged (v1) or IDM (v2) agents. No interactive simulation.|OpenScene (nuPlan redistribution), filtered toward hard scenes. v2 adds **pre-rendered 3D Gaussian Splatting views** around plausible endpoints.|navtrain / navtest (v1, PDMS); navhard two-stage, navsafe (v2, EPDMS); HF leaderboards.|**No — the distinction collapses.** The "simulation" is precomputed with the benchmark: synthetic stage-2 observations exist only for the curated scenes, so the simulator cannot be run on arbitrary new scenes without regenerating reconstructions. Evaluator versions also matter (EPDMS pre- vs post-bug-fix).|
|**AlpaSim**|gRPC microservices: renderer (NuRec or OmniDreams), MPC controller + vehicle model, physics, traffic sim, driver, runtime, eval.|**NuRec reconstructions** of PhysicalAI-AV clips (918 scenes in the 26.01 release, 1,607 clips in 26.04). The 2026 challenge also has a **nuPlan track**.|Test suites (CSV mapping suites to scene artifacts); AlpaSim score; **AlpaSim E2E Closed-Loop Challenge 2026** (capability + safety metrics, PAI-AV and nuPlan tracks).|**Partly.** Software is decoupled from the scene source (nuPlan and PhysicalAI-AV), but each scene is a _learned rendering asset_: scene content and simulator fidelity are the same artifact, and fidelity varies scene by scene.|
|**HUGSIM**|3DGS closed-loop simulator with actor control.|Reconstructions of KITTI-360, Waymo, nuScenes, PandaSet (70+ sequences).|400+ scenarios in difficulty tiers (Easy → Extreme, increasingly adversarial actors); HD-Score.|**Bundled**: simulator and benchmark are released together.|
|**NeuroNCAP**|NeuRAD neural renderer + closed-loop loop around the planner.|14 nuScenes validation sequences, with inserted safety-critical actors.|Stationary / frontal / side scenarios, 100 seeded runs each; NeuroNCAP score (5 if no collision, else decreasing with impact speed).|Bundled; scenario generation is the benchmark.|
|**Waymax**|JAX simulator on GPU/TPU, differentiable, multi-agent; log-replay or IDM agents.|Waymo Open Motion Dataset (570+ h), object-level.|Metrics provided (overlap, offroad, wrong-way, route following, kinematics); no single dominant planning leaderboard ⚠️.|Yes.|
|**PhysicalAI-AV**|None (dataset + devkit).|306,152 × 20 s clips, 7 cameras, LiDAR, radar, egomotion, auto obstacle labels; OOD reasoning labels.|OOD **reasoning** challenge (language, not driving).|Not a simulator. Its reconstructed subset feeds AlpaSim.|
|**WOD-E2E**|None (dataset + metric).|4,021 long-tail segments (2,037 train / 479 val / 1,505 test), 8 cameras.|Rater Feedback Score (RFS) at 3 s and 5 s over 11 scenario types; 2025 challenge leaderboard.|Dataset + metric; open loop.|
|**nuScenes**|None.|1,000 scenes, Boston + Singapore.|Open-loop L2 / collision rate.|Open loop; weak for planning.|

> [!note] Where the distinction breaks down (worth discussing in the paper)
> 
> - **Evaluator-as-benchmark (NAVSIM)**: the simulator is a scoring function over a fixed set of precomputed futures. "Running it on new scenes" is not an option without rebuilding the benchmark.
> - **Learned scene assets (AlpaSim, HUGSIM, NeuroNCAP)**: the scene _is_ a neural model. Rendering quality, hence the observation distribution, varies per scene and with deviation from the logged path. For UQ this means the "environment" itself has an uncertainty profile that must be separated from the policy's.
> - **Training data as benchmark component (CARLA)**: Bench2Drive ships its own training set from a privileged expert. Scores depend strongly on which expert generated the data (§4.3), and SimLingo's authors call Bench2Drive a "training" benchmark because evaluation towns overlap training ⚠️ (secondary source).
> - **Hybrid**: NuRec scenes can also be rendered _inside CARLA_ (NVIDIA NuRec docs), mixing CARLA's tooling with reconstructed real scenes. Maturity unverified ⚠️.

### 2.2 Per-tool details (cost, what it offers, what it lacks)

#### Offline logged data — PhysicalAI-AV

- **Cost**: 133 TB in total, downloadable in chunks via the `physical_ai_av` devkit; one 24 GB GPU runs Alpamayo inference.
- **Offers**: cheapest screening of diffusion-level signals (sample spread, denoising-path stability, epistemic estimates) against OOD-subset membership, conflicts with labelled obstacles, and error against the human future. OOD keyframes allow a first look at lead time.
- **Lacks**: no model-caused failures; no map data; obstacle labels are machine-generated; "OOD" means rare scenario, not necessarily OOD for the model.
- ⚠️ **License**: AV development with NVIDIA technology only. "Confidential Information" includes benchmarking results, derivative works are prohibited, and the licence expires 12 months after download. Check before publishing.

#### NAVSIM v1 / v2

- **Cost**: cheap and reproducible; metric computation only.
- **Offers**: real sensor input; per-scene AUROC of uncertainty against low PDMS/EPDMS or collision; the largest pool of diffusion baselines.
- **Correlation with closed loop**: pseudo-simulation correlates better with closed-loop results (R² = 0.8) than the best open-loop metric (R² = 0.7). A cross-benchmark study of 8 methods found PDMS tracks Bench2Drive DS non-monotonically, with ranking inversions favouring cautious, slow planners.
- **Lacks**: self-correction, compounding errors, replanning.

#### nuPlan

- **Cost**: no rendering; dominated by planner cost and scenario count.
- **Offers**: true closed loop with replanning; the best place for **controlled perception-uncertainty injection** (drop, occlude, jitter or hallucinate objects with the true state known).
- **Lacks**: no sensors, so no VLAs or WAMs; IDM agents don't negotiate; inputs are near ground truth. PDM-Closed, a rule-based planner, scores 92.84 / 92.12 on Val14 (non-reactive / reactive), so check that UQ gains aren't just "drive more cautiously".

#### Waymax

- **Cost**: very high throughput on accelerators, which suits large sampling budgets.
- **Offers**: massive closed-loop rollouts, multi-agent, RL adapters (dm-env, brax).
- **Lacks**: boxes only; Waymo dataset licence (non-commercial ⚠️).

#### CARLA (+ Bench2Drive, Longest6 v2, Fail2Drive)

- **Cost**: light simulator (about 4 h for TF++ on Bench2Drive with 8× RTX 2080 Ti) but unstable: Bench2Drive's README recommends auto-restarting evaluation, and CARLA ignores `CUDA_VISIBLE_DEVICES`. At scale (100+ servers), crashes roughly doubled training time in CaRL. Real-data VLAs must be retrained or fine-tuned on CARLA imagery first.
- **Offers**: controlled shift (weather, towns, scripted events). **Fail2Drive** pairs each shifted route with an in-distribution counterpart at the same location, which suits OOD-vs-benign UQ studies; the authors report an average 22.8% success-rate drop under shift. Episodes with timestamped infractions give lead time.
- **Lacks**: synthetic visuals; scripted traffic; known protocol pitfalls ("Comfortably Stuck", Oct 2026).

#### AlpaSim (+ AlpaGym)

- **Cost**: Docker + NVIDIA Container Toolkit, CUDA ≥ 12.8, driver ≥ 570; gated scene assets (about 1.5 GB per NuRec scene). Alpamayo needs ≥ 24 GB VRAM and NuRec needs > 24 GB (48 GB recommended), so budget one 48–80 GB GPU or two 24 GB GPUs per instance. Scales over SLURM.
- **Offers**: closed loop on realistic imagery with the real diffusion-head VLA run natively; every service request and response is logged and replayable. AlpaGym adds closed-loop RL.
- **Lacks**: bounded deviation from the recorded path; few published baselines; the challenge page lists known scoring issues (false lateral-corridor failures, an off-road bug on the nuPlan track); traffic realism undocumented ⚠️.

#### HUGSIM, NeuroNCAP

- **HUGSIM**: closed loop on real-looking scenes from four datasets, so cross-dataset shift comes for free. Foreground interaction modelling has been criticised as limited (SceneCrafter paper).
- **NeuroNCAP**: purpose-built safety-critical scenarios with 100 seeded runs each. This is the only neural closed-loop tool here already used for a **failure-prediction UQ evaluation** (CATPlan, §5).

### 2.3 Summary — capabilities relevant to UQ evaluation

|Tool|Input|Loop|Real data|Native VLA/WAM|Failure labels with timestamps|Controllable shift|GT world state|Compute|
|---|---|---|---|---|---|---|---|---|
|PhysicalAI-AV|Sensors|Open|✅|✅ Alpamayo|❌ (proxies only)|❌ (OOD subset only)|⚠️ auto labels|1× 24 GB|
|WOD-E2E|Sensors|Open|✅|✅|❌ (rater scores)|❌|❌|Low|
|NAVSIM v1/v2|Sensors|Pseudo|✅|✅|Per scene, not timed|❌|✅ (eval side)|Low|
|nuPlan|Objects|Closed|✅|❌|✅|✅ (via input perturbation)|✅|Low–med|
|Waymax|Boxes|Closed|✅|❌|✅|✅|✅|Very low|
|CARLA (+B2D, Fail2Drive)|Sensors (synthetic)|Closed|❌|Retrain needed|✅|✅✅|✅|Med, unstable|
|AlpaSim|Sensors (neural)|Closed|✅|✅ Alpamayo, VaVAM|✅|⚠️ limited|✅ (sim side)|High|
|HUGSIM|Sensors (3DGS)|Closed|✅|Adapters ⚠️|✅|✅ tiers|✅|Med|
|NeuroNCAP|Cameras (NeRF)|Closed|✅|Adapters ⚠️|✅|✅ scenario types|✅|Med|

---

## 3. Models — what they were trained on and how they output a trajectory

> [!note] Scope of this table This section states only **training data** and **architecture**. Where a model can be or has been evaluated is covered in §4, per benchmark. Any sensor-level model can in principle be run on another benchmark, but then the score mixes model quality with domain gap.

Legend: 🌊 continuous diffusion · 🔀 flow matching · 🧩 discrete/masked diffusion · 🔤 autoregressive tokens · 📏 regression or scoring · 🧬 VAE

|Model|Year / venue|Type|Output generator|Diffusion at output?|Training data (as reported)|Code / weights|
|---|---|---|---|---|---|---|
|**Alpamayo 1 (R1)**|2025|VLA, Cosmos-Reason backbone, 10B|🔀 flow-matching action expert (5 steps)|✅|CoC reasoning traces, Cosmos-Reason Physical AI data, NVIDIA internal AV data; _no RL_ in the released checkpoint|✅ HF (non-commercial)|
|**Alpamayo 1.5**|2026|VLA, Cosmos-Reason2 backbone, 10B|🔀 flow-matching action expert|✅|Same families + public driving data; **RL post-trained**|✅ HF (non-commercial)|
|**Alpamayo 2 Super**|2026|VLA, Cosmos 3 Super Reasoner 32B backbone (card lists 36B ⚠️)|⚠️ decoder not stated|⚠️|⚠️ not detailed; RL post-trained|✅ HF, OpenMDW-1.1|
|**DiffusionDrive**|CVPR 2025|E2E planner|🌊 truncated diffusion from anchored Gaussian, 2 steps|✅|NAVSIM navtrain; separately nuScenes|✅|
|**DiffusionDriveV2**|2025|E2E planner + RL|🌊 truncated diffusion + intra/inter-anchor GRPO|✅|NAVSIM (IL + RL)|✅ Apache 2.0|
|**GoalFlow**|CVPR 2025|E2E planner|🔀 flow matching conditioned on a scored goal point|✅|NAVSIM|✅|
|**Diffusion Planner**|ICLR 2025|Object-level planner|🌊 diffusion + classifier/energy guidance|✅|nuPlan (delivery-vehicle data pending release)|✅|
|**BridgeDrive**|ICLR 2026|E2E planner|🌊 diffusion bridge from anchor to trajectory|✅|CARLA expert data, **PDM-Lite or LEAD**|✅|
|**ReCogDrive**|ICLR 2026|VLM + planner|🌊 DDPM planner on VLM latents + DiffGRPO|✅|VLM: 12 open driving datasets + NAVSIM QA; planner: NAVSIM IL, then RL in NAVSIM's non-reactive sim|✅ code + weights|
|**DiffVLA**|2025|VLM + planner|🌊 DiffusionDrive-based planner with VLM one-hot priors|✅|NAVSIM ⚠️|⚠️|
|**ORION**|ICCV 2025|VLM + generative planner|🧬 VAE (default); diffusion variant tested and worse|❌ (default)|Bench2Drive training set + Chat-B2D VQA; OmniDrive-pretrained weights|✅|
|**AutoVLA**|NeurIPS 2025|VLA|🔤 physical action tokens; GRPO RFT|❌|nuPlan (NAVSIM), Waymo E2E, nuScenes, CARLA, DriveLM annotations|✅|
|**Poutine**|2025|VLA (Qwen2.5-VL)|🔤 trajectory tokens + RL ⚠️|❌|⚠️ vision-language-trajectory pretraining + WOD-E2E|⚠️|
|**SimLingo**|CVPR 2025|VLA|📏 path + speed waypoints|❌|CARLA, PDM-Lite expert|✅ code + data|
|**DriveVLA-W0**|ICLR 2026|VLA + world-model loss|Action expert: query / 🔤 AR / 🔀 flow (AR best at large scale)|Optional|NAVSIM + large in-house|✅|
|**DrivoR**|2026|E2E (ViT + registers)|📏 proposals + scoring ⚠️|❌|NAVSIM navtrain (± SimScale synthetic data)|⚠️|
|**Fast-dDrive**|2026|Block-diffusion VLA|🧩 block-causal masked diffusion|✅ discrete|NAVSIM ⚠️|⚠️|
|**DriveFine**|2026|Masked-diffusion VLA|🧩 masked diffusion LLM + hybrid RL|✅ discrete|NAVSIM|⚠️|
|**WAM-Diff / WAM-Flow**|2025–26|VLA|🧩 masked diffusion / discrete flow|✅ discrete|NAVSIM ⚠️|⚠️|
|**Epona**|ICCV 2025|WAM (AR diffusion world model)|🌊 trajectory DiT + video generation|✅|Driving video ⚠️ (nuPlan, nuScenes)|✅|
|**VaVAM** (Valeo)|2025|Video-action model|AR video model (VQ tokens) + 🔀 flow-matching action expert|✅|⚠️ large driving-video pretraining + imitation|✅|
|**DriveLaW**|CVPR 2026|WAM (latent chained)|🌊 diffusion planner on video-generator latents|✅|nuScenes (video), NAVSIM (planning)|✅|
|**DriveWAM**|2026|Joint WAM|🔀 joint video + action flow matching on a pretrained video DiT|✅|NAVSIM; PhysicalAI-AV (4k–100k clips)|⚠️|
|**WA-JEPA**|2026|Joint WAM|🌊 joint denoising of future scene tokens + trajectory|✅|NAVSIM ⚠️|⚠️|
|**SimWAM**|2026|Joint training, no test-time video|🔀 joint flow matching; video branch dropped at inference; + RL|✅|NAVSIM (65K samples) + pretrained video expert|⚠️|
|**VITAL**|2026|WAM-style|🔀 flow-matching action expert|✅|NAVSIM|⚠️|
|**TransFuser++ / LTF**|2023–25|E2E|📏 waypoint regression|❌|TF++: CARLA (PDM-Lite or LEAD); LTF: NAVSIM|✅|
|**UniAD / VAD / SparseDrive**|2023–24|Modular E2E|📏 query regression|❌|nuScenes (retrained on Bench2Drive data for B2D)|✅|
|**Hydra-MDP(++)**|2024|E2E|📏 scoring over a trajectory vocabulary|❌|NAVSIM|⚠️|
|**GenAD**|2024|E2E|🧬 VAE-style|❌|nuScenes|✅|
|**EMMA** (Waymo)|2024|Multimodal LLM|🔤 text|❌|Waymo internal + public|❌|

---

## 4. Model evaluation — leaderboards per benchmark

> [!warning] How to read these tables
> 
> - Numbers are **as reported** by the cited source. Official leaderboards are linked; where the official page could not be parsed, numbers come from paper tables that compile results.
> - Scores are only comparable **within the same split, metric version and training-data regime**. Known traps are listed under each table.

### 4.1 NAVSIM v1 — navtest, PDMS ↑

Official leaderboard: [AGC2024-P/e2e-driving-navtest](https://huggingface.co/spaces/AGC2024-P/e2e-driving-navtest) (dynamic page, not parsed). Main compilation from the DrivoR paper (Table 1), plus individual papers.

|Method|Output|PDMS|Source|
|---|---|---|---|
|Human driver|—|94.8|DrivoR table|
|DrivoR (+134k SimScale)|📏|94.6|DrivoR table|
|DrivoR (trainval)|📏|93.7|DrivoR table|
|RAP-DINO†|—|93.8|DrivoR table († 10× larger training data)|
|DriveSuprim|📏|93.5|DrivoR table|
|Centaur|—|92.6|DrivoR table|
|SimWAM|🔀|91.5 (abstract) / 91.9 (test-split table) ⚠️|SimWAM paper|
|iPad|—|91.7|DrivoR table|
|DiffusionDriveV2|🌊|91.2|DiffusionDriveV2 paper|
|Hydra-MDP++|📏|91.0|DrivoR table|
|ReCogDrive|🌊|90.8 (89.6 in arXiv v1)|DrivoR table / paper|
|VITAL|🔀|90.8|VITAL paper|
|GoalFlow|🔀|90.3|GoalFlow paper|
|DriveVLA-W0|🔤/🔀|90.2|DrivoR table|
|PDM-Closed (rule-based, privileged)|📏|89.1|DrivoR table|
|AutoVLA|🔤|89.1|DrivoR table|
|DIVER|🌊|88.3|DrivoR table|
|DiffusionDrive|🌊|88.1|DrivoR table / paper|
|VAD-v2|📏|86.2 (80.9 for the 8192-vocab variant in ReCogDrive's table)|DrivoR / ReCogDrive tables|
|LTF|📏|83.8|DrivoR table|
|UniAD|📏|83.4|DrivoR table|
|Ego-status MLP|📏|65.6|DrivoR table|

**Traps**: training on navtrain vs trainval vs extra synthetic data (SimScale) vs much larger private data (RAP†) is not comparable; versions of the same paper report different numbers.

### 4.2 NAVSIM v2 — navhard two-stage, EPDMS ↑

Official leaderboard: [AGC2025/e2e-driving-navhard](https://huggingface.co/spaces/AGC2025/e2e-driving-navhard) (dynamic, not parsed). Paperswithcode lists 8 entries, top "DrivoR (+SimScale) + Ours" at **57.9**. DrivoR paper Table 3 (post-bug-fix evaluator):

|Method|EPDMS|
|---|---|
|DrivoR (+134k SimScale, ViT-S)|54.6|
|SimScale (+185k, V2-99)|53.2|
|DrivoR (+65k SimScale)|52.3|
|DrivoR (ViT-S)|48.3|
|ZTRS (V2-99)|48.1|
|GTRS-DrivoR|45.8|
|GTRS-A|45.4|
|GTRS-D|45.0|
|RAP-DINO|39.6|

**Traps**:

- An official evaluator bug fix changed EPDMS; WA-JEPA reports both EPDMS* (pre-fix) and EPDMS (post-fix). Never mix them.
- "NAVSIM v2" numbers on **navtest with EPDMS** (e.g. DiffusionDriveV2 85.5) are a different split from navhard two-stage (~40–58).
- SUPER-AD reports 69.74 EPDMS on a "NAVHARD & NAVSAFE" setting, again not directly comparable.

### 4.3 Bench2Drive (CARLA) — Driving Score ↑

Official leaderboard: [Bench2Drive HF space](https://carlaleaderboard-bench2drive.hf.space/) (42 entries; success rate not shown on the page). Top entries and diffusion- or VLA-relevant ones:

|Rank|Method|Output|DS|
|---|---|---|---|
|1|AlignDrive|—|89.07|
|2|**BridgeDrive**|🌊 diffusion bridge|86.87|
|3|HiP-AD|—|86.77|
|4|R2SE|—|86.28|
|5|SimLingo-Base (CarLLaVA)|📏 VLA|85.94|
|6|TransFuser++|📏|84.21|
|9|ORION|🧬 VLM + VAE|77.70|
|13|DriveMoE|VLA|74.22|
|16|**ReCogDrive**|🌊 VLM + diffusion|71.36|
|18|**DiffAD**|🌊|67.92|
|20|**VDRive**|VLA + diffusion|66.25|
|35|GenAD|🧬|44.81|
|37|VAD|📏|42.35|
|42|AD-MLP|📏|18.05|

**Traps**:

- **The training expert changes the score a lot.** BridgeDrive reports success rates of 74.99% with PDM-Lite data and 89.25% with LEAD data. The LEAD release of TF++ (CARLA 0.9.16) reports **93.6 ± 1.0 DS** over 3 seeds, versus 84.21 on the leaderboard.
- Leaderboard entries are self-reported; the page states no verification rules.
- Bench2Drive towns overlap training (SimLingo calls it a "training" benchmark ⚠️).
- Harder alternatives: CARLA Leaderboard 2.0 (SimLingo won the 2024 challenge) and Longest6 v2.

### 4.4 nuPlan — closed loop (object-level), score ↑

From the Diffusion Planner repository (NR = non-reactive, R = reactive agents):

|Method|Type|Val14 NR|Val14 R|Test14-hard NR|Test14-hard R|
|---|---|---|---|---|---|
|Expert (log replay)|—|93.53|80.32|85.96|68.80|
|PDM-Closed|Rule-based|92.84|92.12|65.08|75.19|
|PLUTO|Learning + rule refinement|92.88|76.88|80.08|76.88|
|**Diffusion Planner w/ refine**|🌊 + rules|94.26|92.90|78.87|82.00|
|**Diffusion Planner**|🌊 pure learning|89.87|82.80|75.99|69.22|
|PLUTO w/o refine.*|Learning|88.89|78.11|70.03|59.74|
|PlanTF|Learning|84.72|76.95|69.70|61.61|

* Uses pre-searched reference lines as input. Trap: "learning-only" vs "with rule-based refinement" is the decisive split.

### 4.5 WOD-E2E — Rater Feedback Score ↑ (open loop, long-tail)

Official page: [Waymo 2025 E2E challenge](https://waymo.com/open/challenges/2025/e2e-driving/) (login required). Test-split results compiled in the Poutine paper:

|Method|Output|RFS|ADE 5 s|
|---|---|---|---|
|**Poutine** (1st on public leaderboard; special mention)|🔤 VLA|7.99|2.74|
|Poutine-Base|🔤|7.91|2.94|
|ViT-Adapter-GRU|📏|7.85|2.89|
|HMVLM|VLA|7.74|3.07|
|DiffusionLTF|🌊|7.72|2.89|
|UniPlan (DiffusionDrive)|🌊|7.69 (7.779 in the WOD-E2E paper ⚠️)|2.99|
|AutoVLA|🔤|7.56|2.96|
|Swin-Trajectory|📏|7.54|2.81|
|Waymo baseline|—|7.53|3.02|
|Ego-status MLP|📏|7.41|2.94|

**Traps**: RFS has a floor of 4 and saturates (all entries lie between 7.4 and 8.0); trust regions scale with speed.

### 4.6 AlpaSim — AlpaSim score ↑ (closed loop, neural reconstruction)

- **Alpamayo 1.5**: 0.81 ± 0.01 on 910 scenarios (model card). No other public numbers found.
- **AlpaSim E2E Closed-Loop Challenge 2026**: PAI-AV and nuPlan tracks, "capability score + safety metric"; public leaderboard closes **31 Oct 2026**, final results targeted for **15 Nov 2026** (NeurIPS 2026). Organisers warn that local scores are not comparable to official ones and that some metrics may be recomputed.

### 4.7 Other benchmarks without collected leaderboards

HUGSIM (HD-Score), NeuroNCAP (NeuroNCAP score), Fail2Drive (DS, SR and their harmonic mean on paired ID/shift routes), CARLA Leaderboard 2.0, nuScenes open loop. To fill as needed.

---

## 5. UQ evaluation — what each paper actually used

> [!info] Broader map This section focuses on papers closest to diffusion planners and VLA/WAM policies. The full map across perception, prediction, planning, system-level monitoring, conformal safety and VLMs is in [[ad-uq-literature-map]].

> [!important] Rule for this section Each row reports, **from the paper itself**, (1) whose uncertainty is measured, (2) what information the signal uses, (3) what task the uncertainty serves, (4) the tool on which it was evaluated, and (5) whether the _quality of the uncertainty_ was measured directly (AUROC, calibration, coverage) or only indirectly through a downstream driving score.

### 5.1 Driving: uncertainty of the ego policy or planner

| Paper                                                 | Model whose uncertainty                                            | Signal and information used                                                                                  | UQ task                                                                     | Evaluation tool                                                                                      | How UQ is evaluated                                                                                            | Direct UQ metric?                  |
| ----------------------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- | ---------------------------------- |
| **EnDfuser** (NLDL 2026)                              | Own diffusion planner (DDPM, TF++ perception)                      | **Output samples**: variance of desired speed across 128 sampled trajectories                                | Detect safety-critical situations; braking rule when variance > 1.5         | **CARLA, Longest6** (training towns) in arXiv v1 ⚠️; the NLDL abstract reports a **+1.7% DS on LAV** | Qualitative (uncertainty maps, manual review of 100 high-variance scenes: 6 infractions); DS 69.6 → 70.1 in v1 | ❌ no AUROC or calibration          |
| **CATPlan** (2025)                                    | UniAD, VAD (not diffusion)                                         | **Internal states**: planning and motion queries → learned collision classifier (loss-based)                 | Predict collision of the planned trajectory                                 | **NeuroNCAP** (closed loop) + **nuScenes** (open loop)                                               | AUROC, AP, precision at fixed recall (e.g. NeuroNCAP 70.6 AUROC vs 49.4 for a GMM baseline)                    | ✅                                  |
| **Mimir** (2025)                                      | Own hierarchical diffusion planner                                 | **Learned aleatoric head**: Laplace scale on the goal point, injected as guidance into the diffusion planner | Make planning robust to goal-point error                                    | **NAVSIM** v1 navtest (PDMS), v2 navhard (EPDMS)                                                     | Planning score ablations (no uncertainty vs Gaussian vs Laplace); qualitative                                  | ❌                                  |
| **SUPER-AD** (2025)                                   | DiffusionDrive-based candidates                                    | **Perception uncertainty**: MC-sampled BEV semantic logits → entropy-weighted drivable score map             | Score and filter candidate trajectories                                     | **NAVSIM** navtest, navhard + navsafe                                                                | PDMS / EPDMS                                                                                                   | ❌                                  |
| **UncAD** (2025)                                      | VAD, SparseDrive                                                   | **Perception uncertainty**: Laplace online-map uncertainty                                                   | Uncertainty-guided prediction and trajectory selection                      | **nuScenes** (open loop)                                                                             | Collision rate (−26%), drivable-area conflict rate (−42%)                                                      | ❌                                  |
| **Sample-conditioned differentiable planning** (2026) | Own diffusion _predictor_ + differentiable planner                 | **Output samples** of other agents (M = 10) → CVaR tail risk                                                 | Risk-aware planning                                                         | **WOMD (10%)** + **Argoverse 2** open loop; **custom log-replay closed loop**, 100 scenarios         | Collision / off-route / success; minADE/minFDE                                                                 | ❌ (authors note limited diversity) |
| **VAUQ reliability study** (2026)                     | 5 VLMs incl. **Alpamayo-1.5** (language answers, not trajectories) | Predictive entropy + image-information score                                                                 | Failure prediction (correct vs wrong answer), calibration under corruptions | **VQA datasets**: DrivingVQA, NuScenes-QA-mini, STRIDE-QA, OSR, with glare/fog/blur/occlusion        | AUROC, ECE, Brier                                                                                              | ✅ (but QA, not driving)            |

### 5.2 Driving: uncertainty of other agents' predicted motion (used by planners)

|Paper|Predictor|Signal and information used|UQ task|Evaluation tool|How UQ is evaluated|Direct UQ metric?|
|---|---|---|---|---|---|---|
|**Conformal MPC** (Lindemann et al., 2023)|Social LSTM (pedestrians)|**Prediction residuals** → conformal regions|Safe MPC with probabilistic guarantee|**CARLA** intersection, 4 pedestrians (+ ORCA/TrajNet++)|One-step coverage 99.985% (bound 99.83%); 1/100 safety violations|✅ coverage|
|**Robust CP with generative priors** (2026)|⚠️ not named; diffusion model of the nonconformity score|Residuals + **observed nuisance parameter** (e.g. traffic density)|Safe planning under distribution shift|**ORCA** simulator|Empirical safety under shifts vs Conformal MPC|✅ (coverage claimed; details ⚠️)|
|**Adversarially robust CP** (L4DC 2026)|Environment predictor|Residuals under interaction-driven shift|Safe planning when the ego changes others' behaviour|2D car–pedestrian + quadcopter case studies (simulator not named ⚠️)|Safety guarantees ⚠️|⚠️|
|**Scenario-aware CP** (2025)|GameFormer (multi-agent)|Residuals in Frenet frame → CopulaCPTS; reliability discriminator|Calibrated intervals, detect unreliable timestep|**nuPlan dataset** (open loop)|Joint coverage, region area|✅|
|**CUQDS** (2024)|HiVT, LaneGCN, Trajectron++, …|Online conformal update from observed errors|Intervals under temporal shift|**Argoverse 1**|Coverage rate, NLL|✅|
|**Stochasticity in Motion** (2024)|LAformer, PGP, LaPred (ensembles, MC dropout)|**Ensemble samples** → entropy decomposition (aleatoric / epistemic)|Decompose uncertainty, detect OOD|**nuScenes** + synthetic OOD (reversed history, blackout, lane deletion)|Pearson correlation with minADE; distribution shifts of uncertainty|Partial (no AUROC)|
|**DC-MMD OOD monitor** (2026)|GRIP++, FQA|**Only the scalar prediction error**, observed online|Runtime OOD detection as quickest change detection|**NGSIM, nuScenes, ApolloScape** + bounded synthetic perturbations|Detection delay (WADD), mean time to false alarm, AUROC (0.84–0.93), FPR@95|✅|
|**cVMD** (CVPRW 2024)|Conditioned diffusion predictor|**Latent distance**: Mahalanobis in VQ-VAE latent → sets CFG guidance scale|Adaptive guidance|**highD** (highway)|ADE/FDE improvement; qualitative bands|❌|

### 5.3 Cross-domain reference: generative robot policies (manipulation)

|Paper|Policy|Signal|Task|Evaluation tool|Metrics|
|---|---|---|---|---|---|
|**FIPER** (NeurIPS 2025)|Diffusion / flow IL policies|RND on observation embeddings + action-chunk entropy, conformal thresholds|Runtime failure prediction|3 sim tasks (sorting, stacking, push-T) + 2 real (rope, chair pushing) with induced OOD|Balanced accuracy, TPR, normalised detection time|
|**SAFE** (NeurIPS 2025)|OpenVLA, π0, π0-FAST|Last-layer VLA features → MLP/LSTM; functional conformal threshold|Multitask failure detection on unseen tasks|LIBERO-10, SimplerEnv, real Franka and WidowX|ROC-AUC, accuracy vs detection time|
|**DVAC** (2026)|π0.5, π0, others (flow)|Variance of clean-action estimates over the last denoising steps|Adaptive chunk length (not failure prediction)|LIBERO, RoboTwin, CALVIN, real Cobot Magic|Task success, replanning count (signal not calibrated; authors say so)|

### 5.4 What this survey of UQ evaluations shows

1. **Most "uncertainty-aware" driving planners never evaluate their uncertainty.** Mimir, SUPER-AD, UncAD, cVMD and the CVaR planner judge uncertainty only through the downstream driving score. That cannot separate "better uncertainty" from "more conservative driving".
2. **Direct UQ evaluation in driving exists mainly for trajectory _prediction_** (conformal coverage, OOD monitors on nuScenes/NGSIM/Argoverse/nuPlan logs), not for the ego policy of a VLA or WAM.
3. **For ego-policy failure prediction**, the only directly evaluated driving example found is CATPlan (NeuroNCAP + nuScenes), on non-diffusion planners. EnDfuser is the only diffusion-planner UQ paper and reports no quantitative UQ metric.
4. **No UQ paper evaluated on AlpaSim, HUGSIM, Bench2Drive or Fail2Drive** was found, and none evaluates a diffusion VLA or WAM's trajectory uncertainty with AUROC or calibration metrics. The closest is the VAUQ study, which covers Alpamayo-1.5's _language answers_ only.
5. **Information used splits into four families**: output samples (EnDfuser, CVaR planner, FIPER's action entropy); internal states (CATPlan, SAFE); learned uncertainty heads or perception uncertainty (Mimir, SUPER-AD, UncAD); residuals or errors after the fact (conformal methods, DC-MMD). Diffusion-_process_ signals (denoising path, score Jacobians) appear only in manipulation (DVAC) and in generic diffusion-UQ work, not yet in driving.

> [!tip] Implication for this project A diffusion-level UQ paper for driving VLAs/WAMs would fill a clear gap. It should report **direct UQ metrics** (AUROC/AUPRC against failures, risk–coverage, calibration or coverage, detection delay vs false alarms), on at least one **closed-loop** tool where failures are caused by the model (AlpaSim for Alpamayo; Fail2Drive or NeuroNCAP-style setups for controlled shift), with offline PhysicalAI-AV or NAVSIM results as a cheap first stage.

---

## 6. What the output generator implies for diffusion-level UQ

|Generator family|Examples|Native UQ signals|Caveats|
|---|---|---|---|
|**Anchored / truncated diffusion**|DiffusionDrive(V2), BridgeDrive, DiffVLA|Spread across anchors, anchor-score entropy, per-anchor denoising residual|With ~2 steps from anchors, diversity comes mostly from the anchor set and score head, so "diffusion uncertainty" is largely classifier uncertainty.|
|**Full continuous diffusion / flow (ODE sampler)**|Alpamayo, GoalFlow, Diffusion Planner, ReCogDrive, VaVAM, SimWAM|Sample spread over initial noise; x̂₀ path stability (DVAC-style); Laplace / Fisher epistemic; guidance gradients|ODE samplers inject no noise per step, so aleatoric variability enters only via initial noise. **RL post-training may collapse spread** (hypothesis: compare Alpamayo 1 vs 1.5).|
|**Discrete / masked diffusion**|Fast-dDrive, DriveFine, WAM-Diff/Flow|Token confidence at unmasking, remasking frequency, block entropy|Continuous-space tools (Fisher projection, x̂₀ variance) don't transfer directly.|
|**Joint video + action diffusion (WAM)**|DriveWAM, WA-JEPA, Epona, DriveLaW|Predicted-vs-observed future gap; video–action consistency; disagreement between factorizations|Needs closed loop to observe the future; residuals mix model error with agents' legitimate stochasticity, and with rendering error in neural simulators.|
|**Non-diffusion** (AR tokens, regression, VAE, scoring)|AutoVLA, Poutine, SimLingo, ORION, TF++, Hydra-MDP|Token entropy, VAE latent variance, score margins, ensembles|Baselines to beat, to show whether "diffusion-level" adds anything.|

---

## 7. Suggested pipeline for this project

1. **Offline screening** on PhysicalAI-AV with Alpamayo 1 / 1.5. Compute diffusion-level signals and test them against pre-registered targets (OOD subset, obstacle conflicts, error), with nuisance baselines and clip-level bootstrap.
2. **NAVSIM navhard** as a cheap real-data check: per-scene AUROC of uncertainty against low EPDMS, alongside the many diffusion baselines.
3. **AlpaSim** for true failure prediction with lead time, and uncertainty-guided steering, with the real model. Log deviation from the recorded path to control for rendering artifacts.
4. **Controlled shift**: Fail2Drive-style paired routes (CARLA) or perception perturbation in nuPlan, using smaller diffusion planners, to separate OOD from benign difficulty.

---

## 8. Open checks (to do)

- [ ] Alpamayo 2 Super: parameter count (32B text vs 36B card) and trajectory decoder type.
- [ ] Whether PhysicalAI-AV OOD validation clips were excluded from Alpamayo training; official OOD definition (challenge page).
- [ ] EnDfuser: confirm which version (Longest6 vs LAV) is the published NLDL 2026 paper.
- [ ] Parse the official NAVSIM and AlpaSim leaderboards (dynamic pages) and the WOD-E2E test leaderboard (login).
- [ ] HUGSIM / NeuroNCAP support for camera-only VLAs.
- [ ] AlpaSim wall-clock per rollout with Alpamayo on available GPUs.
- [ ] Legal check: publishing results computed on PhysicalAI-AV.
- [ ] Training data for the ⚠️ rows (DiffVLA, Poutine, Fast-dDrive, WAM-Diff/Flow, WA-JEPA, Epona, VaVAM).

---

## 9. Sources

**Tools and benchmarks**

- [PhysicalAI-AV dataset card](https://huggingface.co/datasets/nvidia/PhysicalAI-Autonomous-Vehicles) · [PhysicalAI-AV NuRec](https://huggingface.co/datasets/nvidia/PhysicalAI-Autonomous-Vehicles-NuRec) · [physical_ai_av devkit](https://github.com/NVlabs/physical_ai_av)
- [WOD-E2E paper](https://arxiv.org/html/2510.26125v2) · [WOD-E2E 2025 challenge](https://waymo.com/open/challenges/2025/e2e-driving/) · [Waymo challenges page](https://waymo.com/open/challenges/)
- [NAVSIM v1](https://arxiv.org/html/2406.15349v1) · [Pseudo-Simulation / NAVSIM v2](https://proceedings.mlr.press/v305/cao25a.html) · [NAVSIM repo](https://github.com/autonomousvision/navsim) · [navtest leaderboard](https://huggingface.co/spaces/AGC2024-P/e2e-driving-navtest) · [navhard leaderboard](https://huggingface.co/spaces/AGC2025/e2e-driving-navhard) · [navhard on paperswithcode](https://paperswithcode.co/benchmark/navsim-v2-navhard) · [Driving on Registers (DrivoR, compiled tables)](https://arxiv.org/pdf/2601.05083) · [Driving on Memory](https://arxiv.org/pdf/2608.31029) · [Open-loop vs closed-loop correlation](https://www.emergentmind.com/papers/2605.00066)
- [Waymax paper](https://arxiv.org/pdf/2310.08710) · [Waymax repo](https://github.com/waymo-research/waymax)
- [Bench2Drive repo](https://github.com/Thinklab-SJTU/Bench2Drive) · [Bench2Drive paper](https://export.arxiv.org/pdf/2406.03877) · [Bench2Drive leaderboard](https://carlaleaderboard-bench2drive.hf.space/) · [CARLA garage](https://github.com/autonomousvision/carla_garage) · [LEAD / TF++ release](https://github.com/kesai-labs/lead/releases/tag/v1.4.0) · [CaRL](https://arxiv.org/pdf/2504.17838) · [Comfortably Stuck](https://arxiv.org/pdf/2610.21741) · [Fail2Drive](https://arxiv.org/html/2604.08535v1)
- [Diffusion Planner repo (nuPlan tables)](https://github.com/ZhengYinan-AIR/Diffusion-Planner)
- [AlpaSim docs](https://docsearch.algolia.com/mcp/docs/repo/nvlabs/alpasim) · [AlpaSim onboarding](https://raw.githubusercontent.com/NVlabs/alpasim/main/docs/ONBOARDING.md) · [AlpaSim Challenge 2026](https://nvidia-alpasime2eclosedloopchallenge2026.hf.space/) · [AlpaGym post-training blog](https://developer.nvidia.com/blog/how-to-post-train-autonomous-vehicle-models-in-closed-loop-with-nvidia-alpamayo/) · [NuRec hardware](https://docs.nvidia.com/nurec/basics/hardware.html) · [NuRec for AV](https://docs.nvidia.com/nurec/av/index.html) · [Alpamayo 2 blog](https://huggingface.co/blog/drmapavone/nvidia-alpamayo-2)
- [HUGSIM](https://arxiv.org/pdf/2412.01718) · [NeuroNCAP](https://arxiv.org/pdf/2404.07762) · [SceneCrafter](https://arxiv.org/pdf/2503.18108) · [Bench2Drive-R](https://arxiv.org/abs/2412.09647v1)

**Models**

- [Alpamayo-R1](https://arxiv.org/abs/2511.00088v2) · [Alpamayo 1 card](https://huggingface.co/nvidia/Alpamayo-R1-10B) · [Alpamayo 1.5 card](https://huggingface.co/nvidia/Alpamayo-1.5-10B) · [Alpamayo 2 Super (NVIDIA blog)](https://blogs.nvidia.com/blog/alpamayo-2-super-open-model-now-available)
- [DiffusionDrive](https://arxiv.org/abs/2411.15139v3) · [DiffusionDriveV2](https://arxiv.org/abs/2512.07745v1) · [GoalFlow](https://arxiv.org/abs/2503.05689) · [Diffusion Planner](https://arxiv.org/pdf/2501.15564) · [BridgeDrive](https://arxiv.org/html/2509.23589v3)
- [ReCogDrive](https://arxiv.org/abs/2506.08052v1) · [DiffVLA](https://www.alphaxiv.org/abs/2505.19381) · [ORION](https://arxiv.org/pdf/2503.19755) · [AutoVLA](https://arxiv.org/html/2506.13757v3) · [Poutine](https://arxiv.org/html/2506.11234v3) · [SimLingo](https://openaccess.thecvf.com/content/CVPR2025/html/Renz_SimLingo_Vision-Only_Closed-Loop_Autonomous_Driving_with_Language-Action_Alignment_CVPR_2025_paper.html)
- [DriveVLA-W0](https://arxiv.org/html/2510.12796v1) · [Fast-dDrive](https://arxiv.org/html/2605.23163) · [DriveFine](https://arxiv.org/pdf/2602.14577)
- [Epona](https://openaccess.thecvf.com/content/ICCV2025/html/Zhang_Epona_Autoregressive_Diffusion_World_Model_for_Autonomous_Driving_ICCV_2025_paper.html) · [VaViM / VaVAM](https://arxiv.org/pdf/2502.15672) · [DriveLaW](https://arxiv.org/pdf/2512.23421) · [DriveWAM](https://arxiv.org/pdf/2605.28544) · [WA-JEPA](https://arxiv.org/pdf/2608.20974) · [SimWAM](https://arxiv.org/pdf/2608.07468) · [VITAL](https://arxiv.org/pdf/2610.20604)

**UQ papers**

- [EnDfuser arXiv](https://arxiv.org/html/2506.00560v1) · [EnDfuser NLDL 2026](https://openreview.net/forum?id=7o4nocapUr) · [CATPlan](https://arxiv.org/html/2503.07425v1) · [Mimir](https://arxiv.org/html/2512.07130v1) · [SUPER-AD](https://arxiv.org/html/2511.22865v1) · [UncAD](https://arxiv.org/html/2504.12826v1) · [Sample-conditioned differentiable planning](https://arxiv.org/html/2606.03296) · [VAUQ reliability study](https://arxiv.org/html/2610.01531v1)
- [Conformal MPC](https://arxiv.org/pdf/2210.10254) · [Robust CP with generative priors](https://arxiv.org/html/2602.12616v1) · [Adversarially robust CP](https://proceedings.mlr.press/v331/mirzaeedodangeh26a.html) · [Scenario-aware CP](https://arxiv.org/html/2512.05682v1) · [CUQDS](https://arxiv.org/pdf/2406.12100) · [Stochasticity in Motion](https://arxiv.org/pdf/2410.01628) · [DC-MMD OOD monitor](https://arxiv.org/pdf/2603.14603) · [cVMD](https://arxiv.org/pdf/2405.14384)
- [FIPER](https://arxiv.org/pdf/2510.09459v1) · [SAFE](https://arxiv.org/html/2506.09937v2) · [DVAC](https://arxiv.org/pdf/2606.03847)