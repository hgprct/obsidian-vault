
> [!abstract] Purpose A taxonomy of current driving models (end-to-end planners, VLM-assisted systems, VLAs, WAMs) as a set of **independent design axes**, not exclusive classes. One model sits on every axis at once. Written to support the project on **diffusion-level UQ for driving VLAs / WAMs**: each axis ends with what it implies for uncertainty.
> 
> Driving counterpart of the robotics note [[vla-architecture-taxonomy]]. Related: [[ad-uq-tools-and-models]] (tools, leaderboards, UQ evaluations) · [[ad-uq-literature-map]] (UQ literature).
> 
> Status as of 22 Oct 2026. ⚠️ = not verified from the primary source, or from memory.

---

## 0. Vocabulary pitfalls

> [!warning] "E2E planners vs generative models (VLM / VLA / WAM)" is not a valid split It mixes two axes:
> 
> - **End-to-end** describes the _pipeline_: sensors in, trajectory out, trained jointly. Most VLAs and WAMs are end-to-end.
> - **Generative** describes the _output model_: a learned distribution that is sampled. DiffusionDrive, GoalFlow and BridgeDrive are generative but are neither VLMs nor VLAs. SimLingo is a VLA with a plain regression output.
> - A **VLM alone does not drive**: it answers or reasons in language. It enters a driving system either as an advisor to a separate planner, or by being turned into a VLA.

Other terms used loosely in the literature:

|Term|Usual meaning in driving papers|Caution|
|---|---|---|
|**Closed-loop metrics**|NAVSIM papers use it for PDMS / EPDMS|NAVSIM is pseudo-simulation, not closed loop (see [[ad-uq-tools-and-models]])|
|**World model**|Anything from an auxiliary future-prediction loss to a neural simulator|Always ask _what role_ it plays (Axis 6)|
|**Action**|Usually a trajectory, sometimes low-level controls|See Axis 3|
|**VLA**|A VLM fine-tuned to output driving actions|Rarely takes language _instructions_ (Axis 7)|

---

## 1. Overview of the axes

|#|Axis|Question it answers|Main values|
|---|---|---|---|
|1|**System decomposition**|How is the stack split?|Modular · mid-to-mid · modular E2E · monolithic sensor-to-action|
|2|**Foundation-model role**|Is a pretrained VLM / video model involved, and how?|None · VLM-assisted dual system (explicit / implicit) · VLA · WAM|
|3|**Output representation**|_What_ does the model output?|Meta-actions · goal point · vocabulary / anchors · tokens · waypoints · path + speed · residual · dynamics-parameterised · controls|
|4|**Output generator**|_How_ is the output produced?|Regression · scoring / selection · AR tokens · diffusion · flow matching · discrete diffusion · VAE|
|5|**Execution**|How does the output become vehicle motion?|External tracker (LQR / MPC / PID) · direct control · replanning rate|
|6|**World-model role**|What does future prediction do, if anything?|None · auxiliary loss · latent chained · cascaded · per-candidate evaluation · joint · external simulator / data engine|
|7|**Conditioning and language**|What is the system told, and what is language used for?|Implicit goal · command · route · guidance energies · free text; language as input / interface / output|
|8|**Reasoning**|Does the model "think" before acting?|None · textual CoT · causal traces · adaptive fast/slow · latent or dynamics CoT|
|9|**Inputs**|What does it perceive?|Object-level · camera-only · camera + LiDAR (+ radar) · history length · maps|
|10|**Training regime**|Where does supervision come from?|Imitation of humans · imitation of a privileged expert · RL in (pseudo-)simulation · closed-loop RL · distillation|

---

## 2. The axes in detail

### Axis 1 — System decomposition

|Value|Description|Examples|
|---|---|---|
|**Modular stack**|Separately trained perception, prediction, planning; often a rule-based planner|Industry stacks; PDM-Closed|
|**Mid-to-mid planner**|Learned planner on perceived objects and maps, no sensors|Diffusion Planner, PLUTO, PlanTF (nuPlan)|
|**Modular end-to-end**|Perception / prediction / planning modules inside one network, trained jointly with intermediate supervision|UniAD, VAD, SparseDrive, DiffusionDrive|
|**Monolithic sensor-to-action**|One network, little or no intermediate supervision|TransFuser lineage (partly), most VLAs and WAMs|

> [!tip] UQ implication Intermediate outputs (detections, maps, predictions) give **extra places to measure uncertainty** and to inject it into planning (UncAD, SUPER-AD, UniUncer). Monolithic models offer fewer handles but less error propagation between modules.

### Axis 2 — Role of a foundation model

|Value|Description|Examples|
|---|---|---|
|**None**|Task-specific network trained from scratch on driving data|UniAD, DiffusionDrive, TF++, BridgeDrive|
|**VLM-assisted, dual system — explicit guidance**|A VLM outputs decisions or meta-actions; a separate planner consumes them|DriveVLM-Dual, Senna, DiffVLA (one-hot decisions into a diffusion planner)|
|**VLM-assisted, dual system — implicit transfer**|VLM latents or distillation shape a separate planner|ReCogDrive (VLM latents → diffusion planner; VLM frozen while the planner trains), VLM-AD|
|**VLA (single system)**|The VLM itself is fine-tuned to output driving actions, possibly via an attached action expert or head|Alpamayo 1 / 1.5 / 2, AutoVLA, SimLingo, ORION, DriveMoE, Poutine, VDRive|
|**WAM**|The model also predicts how the world evolves, tied to action generation|Epona, DriveLaW, DriveWAM, WA-JEPA, SimWAM, VaVAM, DriveVA|

The split follows the VLA-for-driving survey (arXiv 2512.16760): vision-action models, end-to-end VLA, and dual-system VLA with explicit vs implicit guidance. Boundaries blur: DriveVLA-W0 is a VLA trained with a future-image loss; Alpamayo 2 and Cosmos 3 aim to unify reasoning, world generation and action.

> [!tip] UQ implication The foundation-model role decides which **extra uncertainty channels** exist beyond the trajectory: language reasoning (VLA), a predicted future (WAM), or the interface between reasoner and planner (dual system).

### Axis 3 — Output representation (_what_ is output)

|Value|What is output|Examples|
|---|---|---|
|**Meta-actions / decisions**|Discrete intent: keep lane, change left, yield, accelerate|DriveVLM, Senna, DiffVLA (as priors), Alpamayo 2 (alongside a trajectory)|
|**Goal point**|Target position at the horizon; trajectory generated toward it|GoalFlow, Mimir|
|**Vocabulary / anchors**|Select or refine one of K prototype trajectories|VADv2, Hydra-MDP (selection); DiffusionDrive, BridgeDrive (anchors refined)|
|**Trajectory tokens**|Discretised trajectory codes in the LLM vocabulary|AutoVLA (action codebook), EMMA (text), Alpamayo training-time tokens|
|**Waypoints**|Future ego poses at fixed times (NAVSIM: 8 poses × 0.5 s)|UniAD, VAD, TF, DiffusionDrive|
|**Path + speed (disentangled)**|Geometric path plus a separate speed profile|SimLingo / CarLLaVA; TF++ ⚠️|
|**Residual**|Deviation from a simple reference (e.g. constant velocity)|ResAD|
|**Dynamics-parameterised**|Accelerations / curvatures integrated through a vehicle model|Alpamayo (unicycle dynamics)|
|**Low-level controls**|Steering, throttle, brake|CIL, TCP-ctrl, VDRive diffusion head, many RL policies|

Evidence for trajectories over controls: on Bench2Drive, TCP's trajectory branch scores 59.90 Driving Score vs 30.47 for its control branch.

> [!tip] UQ implication Uncertainty lives in the output space, so **sample spread is not comparable across representations** (metres over 4 s vs steering at one instant vs curvature profiles). Compare in a common space, ideally the _executed_ trajectory after rolling samples through the same tracker and vehicle model. Dynamics-parameterised outputs make every sample feasible, so spread is more likely to mean real behavioural alternatives. Hierarchical outputs (meta-action + trajectory) separate _decision_ from _execution_ uncertainty.

### Axis 4 — Output generator (_how_ it is produced)

|Value|Examples|Distribution over outputs|
|---|---|---|
|**Regression**|TF++, UniAD, SimLingo|One answer (mean-seeking)|
|**Scoring / selection**|VADv2, Hydra-MDP, GTRS, DriveSuprim|Discrete distribution over candidates|
|**Autoregressive tokens**|AutoVLA, Poutine, EMMA|Token likelihoods|
|**Continuous diffusion (DDPM / DDIM, truncated, bridge)**|DiffusionDrive(V2), BridgeDrive, ReCogDrive, Diffusion Planner, DriveLaW|Sampled; anchored variants start from priors|
|**Flow matching**|Alpamayo, GoalFlow, VaVAM, DriveWAM, SimWAM, VITAL|Sampled; ODE integration, few steps|
|**Discrete / masked diffusion**|Fast-dDrive, DriveFine, WAM-Diff / WAM-Flow|Parallel unmasking over tokens|
|**VAE**|GenAD, ORION|Latent sampling|

> [!tip] UQ implication This is **the axis that defines "diffusion-level" UQ**. It is independent of Axis 2: a diffusion planner (DiffusionDrive) and a diffusion VLA (Alpamayo) expose the same kinds of signal (sample spread, denoising-path stability, score / velocity-field sensitivity). With anchored or truncated diffusion (~2 steps), spread comes mostly from the anchor set and score head. With ODE samplers (flow matching, DDIM), aleatoric variability enters only through the initial noise. Discrete diffusion needs token-level signals instead.

### Axis 5 — Execution (from output to motion)

|Value|Description|Where|
|---|---|---|
|**External tracker**|LQR / MPC / PID converts a trajectory to controls|NAVSIM (LQR), nuPlan (LQR), AlpaSim (MPC)|
|**Agent-side controller**|The model's package includes its own PID|CARLA waypoint models (CARLA expects controls)|
|**Direct control**|The model outputs controls|CIL, TCP-ctrl, VDRive|
|**Replanning rate / executed horizon**|How much of each plan runs before the next call|NAVSIM: whole 4 s plan; closed loop: first fraction|

> [!warning] Harness effects In Bench2Drive, the default PID can **override the model's steering** toward the route target point, in more than half of all frames in some scenarios ("Comfortably Stuck", 2026). Failures in closed loop are then not purely the model's own.

> [!tip] UQ implication The tracker filters uncertainty: late-horizon waypoint spread may never affect motion. Weight uncertainty by what is actually executed, and log controller overrides.

### Axis 6 — Role of a world model

|Value|Description|Driving examples|
|---|---|---|
|**None**|No future prediction|UniAD, DiffusionDrive, AutoVLA|
|**Auxiliary supervision**|Future prediction shapes representations; actions decoded directly|DriveVLA-W0, LAW ⚠️, SimWAM (video branch dropped at inference)|
|**Latent chained**|Video-generator latents feed the planner|DriveLaW|
|**Cascaded ("imagine then act")**|Generate a future, then derive actions|DriveVA ⚠️, World4Drive|
|**Per-candidate evaluation**|Predict a future for each candidate trajectory, score it|DA-WAM, WoTE ("online trajectory evaluation via BEV world model")|
|**Joint**|One network denoises future and action together|DriveWAM, WA-JEPA, MM-Future, Epona|
|**External simulator / data engine**|World model used _around_ the policy|NuRec and OmniDreams (AlpaSim), ReSim, GAIA-2 ⚠️|

Second sub-axis, **prediction substrate**: pixels / video latents (DriveWAM, Epona), learned latents (WA-JEPA, Drive-JEPA), occupancy or BEV (OccWorld, WoTE).

> [!tip] UQ implication Only WAMs that **generate the future at inference** (cascaded, joint, per-candidate) allow a _predicted-vs-observed surprise_ signal. Auxiliary-loss WAMs (SimWAM) do not. Per-candidate evaluation naturally gives an uncertainty over consequences of each action.

### Axis 7 — Conditioning and the role of language

**What the system is told**

|Value|Description|Examples|
|---|---|---|
|**Implicit goal**|"Drive safely, make progress" is baked in by data or reward; style inherited from the expert|All imitation-trained models|
|**Discrete command**|Left / right / straight (± lane change)|UniAD, VAD (nuScenes), NAVSIM, VaVAM, NeuroNCAP|
|**Route geometry**|Target point(s) or route|CARLA models, nuPlan, WOD-E2E routing info|
|**Guidance energies**|Preferences injected at inference, not text|Diffusion Planner (speed, comfort, style)|
|**Free text**|Natural-language commands|LMDrive (LangAuto, CARLA), SimLingo (Action Dreaming), Alpamayo 1.5 (user commands + navigation guidance), AD-H, LAD-Drive|

**What language is used for**

|Role|Description|Examples|
|---|---|---|
|**Input instruction**|User → model|LMDrive, SimLingo, Alpamayo 1.5 (rare)|
|**Internal interface**|Reasoner → planner|DiffVLA, DriveWAM (VLM semantic intent), Senna|
|**Output reasoning / explanation**|Trace before or with the trajectory|Alpamayo Chain-of-Causation, AutoVLA slow mode, ORION VQA|

> [!note] Most driving VLAs do not take instructions The "L" mostly means a **language-pretrained backbone** plus language as _output_ or _interface_. See §4.

> [!tip] UQ implication Coarse instructions leave legitimate multimodality (which lane, when), so spread partly measures **instruction underspecification**. Free text adds instruction ambiguity and instruction-OOD. Sensitivity of the sampled plan to harmless paraphrases or alternative commands is itself a usable signal.

### Axis 8 — Reasoning

|Value|Examples|
|---|---|
|**None**|Most planners|
|**Textual chain-of-thought**|DriveVLM, Senna ⚠️|
|**Structured causal traces**|Alpamayo Chain-of-Causation (CoC)|
|**Adaptive (fast / slow)**|AutoVLA (trajectory-only vs CoT mode, RL reduces needless reasoning)|
|**Latent / dynamics reasoning**|DynVLA (dynamics tokens before actions), DriveVLA-W0 (world prediction)|

> [!tip] UQ implication Reasoning gives two extra signals: **uncertainty of the trace** (entropy, sample disagreement) and **reasoning–action consistency**. Faithfulness is not guaranteed: Alpamayo's RL stage improved reasoning–action consistency by 37%, and DriveBench found driving VLMs often answer from text priors rather than the image.

### Axis 9 — Inputs

|Value|Examples|
|---|---|
|**Object-level** (boxes, vector map)|Diffusion Planner, PLUTO (nuPlan)|
|**Camera-only**|Alpamayo, SimLingo, most VLAs and WAMs|
|**Camera + LiDAR (+ radar)**|TransFuser lineage, Hydra-MDP|
|**Temporal history**|Alpamayo: 4 frames × 4 cameras over 0.4 s + egomotion; ORION: long-term history via QT-Former|
|**Maps**|HD map (nuPlan), online maps (UniAD, VAD), none (many VLAs)|

> [!tip] UQ implication Object-level inputs allow **controlled perception-uncertainty injection** with known ground truth. Camera-only models make perception uncertainty part of the policy's own uncertainty, so it cannot be separated without probes.

### Axis 10 — Training regime and data source

|Value|Examples|
|---|---|
|**Imitation of human logs**|NAVSIM models (navtrain), nuScenes models, Alpamayo pretraining|
|**Imitation of a privileged expert (simulation)**|CARLA models: PDM-Lite, LEAD, Think2Drive (Bench2Drive official data)|
|**RL in pseudo-simulation**|DiffusionDriveV2, ReCogDrive (GRPO in NAVSIM's non-reactive sim)|
|**Closed-loop RL**|AlpaGym (AlpaSim), CaRL (CARLA), Alpamayo 1.5 / 2 RL post-training ⚠️ (environment not detailed)|
|**Distillation**|Hydra-MDP (from rule-based teachers), Bench2DriveZoo models (from Think2Drive), Alpamayo 2 Super as teacher|

> [!tip] UQ implication **RL post-training may collapse sample diversity**, weakening spread-based signals (hypothesis: compare Alpamayo 1, released without RL, vs Alpamayo 1.5, RL post-trained). In CARLA, the expert's driving style is part of what "correct" means, and changes scores a lot (TF++: 84.2 on the leaderboard vs 93.6 Driving Score with LEAD data).

---

## 3. Placement of representative models

|Model|Decomposition|Foundation-model role|Output repr.|Generator|World-model role|Conditioning|Reasoning|Training|
|---|---|---|---|---|---|---|---|---|
|**UniAD**|Modular E2E|None|Waypoints|Regression|None|Command|None|IL (nuScenes)|
|**TF++**|Monolithic (cam + LiDAR)|None|Path + speed ⚠️|Regression / classification ⚠️|None|Target point + command|None|IL (PDM-Lite / LEAD)|
|**Hydra-MDP**|Modular E2E|None|Vocabulary|Scoring|None|Command|None|IL + distillation|
|**DiffusionDrive**|Modular E2E|None|Anchors → waypoints|Truncated diffusion|None|Command|None|IL (NAVSIM)|
|**DiffusionDriveV2**|Modular E2E|None|Anchors → waypoints|Truncated diffusion|None|Command|None|IL + GRPO (NAVSIM)|
|**GoalFlow**|Modular E2E|None|Goal point → waypoints|Flow matching + scoring|None|Command|None|IL (NAVSIM)|
|**Diffusion Planner**|Mid-to-mid|None|Waypoints (+ joint predictions)|Diffusion + guidance|None|Route + guidance energies|None|IL (nuPlan)|
|**BridgeDrive**|Modular E2E|None|Anchors → trajectory|Diffusion bridge|None|Target point|None|IL (CARLA expert)|
|**DiffVLA**|Modular E2E + VLM|Dual, explicit|Anchors → waypoints|Truncated diffusion|None|Command + VLM decisions|VLM decisions|IL ⚠️|
|**ReCogDrive**|Monolithic|Dual, implicit|Waypoints|DDPM diffusion|None|Command ⚠️|VLM pretraining (QA)|SFT + DiffGRPO (NAVSIM)|
|**ORION**|Monolithic|VLA + generative planner|Trajectory|VAE|None|Command / target ⚠️|VQA|IL (Bench2Drive + Chat-B2D)|
|**SimLingo**|Monolithic|VLA|Path + speed|Regression|None|Target point + free text|Commentary|IL (CARLA, PDM-Lite)|
|**AutoVLA**|Monolithic|VLA|Trajectory tokens|AR|None|Command / route ⚠️|Adaptive fast/slow CoT|SFT + GRPO|
|**Alpamayo 1**|Monolithic|VLA + action expert|Dynamics-parameterised|Flow matching|None|⚠️|Chain-of-Causation|SFT (no RL in release)|
|**Alpamayo 1.5**|Monolithic|VLA + action expert|Dynamics-parameterised ⚠️|Flow matching ⚠️|None|Text navigation + user commands|Chain-of-Causation|SFT + RL|
|**VDRive**|Monolithic|VLA + diffusion policy|Controls|Diffusion (actor-critic)|Predicted future tokens ⚠️|⚠️|—|Preference FT + RL|
|**DriveVLA-W0**|Monolithic|VLA + world-model loss|Trajectory|AR / flow / query expert|Auxiliary|Command ⚠️|—|IL (NAVSIM + in-house)|
|**DriveLaW**|Monolithic|WAM|Waypoints|Diffusion planner|Latent chained|Command ⚠️|—|IL (3-stage)|
|**DriveWAM**|Monolithic|WAM|Trajectory|Joint flow matching|Joint|Command + VLM intent|VLM intent (interface)|IL|
|**SimWAM**|Monolithic|WAM (train-time)|Trajectory|Flow matching|Auxiliary (video dropped at inference)|Command ⚠️|—|IL + RL|
|**Epona**|Monolithic|WAM|Trajectory|Diffusion (trajectory DiT)|Joint / modular|⚠️|—|Video + trajectory pretraining|
|**VaVAM**|Monolithic|WAM-like|Trajectory|AR video + flow-matching action expert|Feature-level ⚠️|Command (left / right / straight)|—|Video pretraining + IL|

---

## 4. Why import VLAs / WAMs into driving, if there is no text input?

|Claimed benefit|What it relies on|Evidence so far|
|---|---|---|
|**Long-tail generalisation**|Internet-scale pretrained knowledge|Mixed. WOD-E2E long-tail: VLM-based Poutine 1st (RFS 7.99), but AutoVLA (7.56) below DiffusionLTF (7.72)|
|**Denser supervision**|Language traces and future-video prediction add training signal ("supervision deficit", DriveVLA-W0)|Plausible; WAMs competitive on NAVSIM (SimWAM 91.5–91.9 PDMS)|
|**Interpretability / auditability**|Reasoning traces as output|Faithfulness not guaranteed (DriveBench; Alpamayo +37% consistency from RL)|
|**Scaling**|Reuse of foundation backbones|Claimed (DriveVLA-W0; DriveWAM 4k→100k clips)|
|**Offline teacher / data engine**|Large models label and distil|Alpamayo 2 Super explicitly positioned as a teacher|
|**Text interface**|User commands, QA|Real but secondary (Alpamayo 1.5, SimLingo, LMDrive)|

On standard benchmarks, VLAs do not dominate: NAVSIM v1 top entries are non-VLA (DrivoR 94.6, DriveSuprim 93.5, Hydra-MDP++ 91.0 vs ReCogDrive 90.8, AutoVLA 89.1), and Bench2Drive's top two are non-VLM (AlignDrive 89.07, BridgeDrive 86.87; best VLA SimLingo-Base 85.94). Cost is real: Alpamayo-R1 takes ~99 ms with reasoning vs 29 ms trajectory-only.

> [!important] Consequence for the UQ project The case for VLAs and WAMs rests on **long-tail competence**, which is exactly what UQ should detect. A sharp research question: _does a VLA's uncertainty rise on rare or OOD scenes more informatively than a task-specific diffusion planner with the same output generator?_ This needs long-tail evaluation sets (WOD-E2E, PhysicalAI-AV OOD subset, Fail2Drive) and a matched non-VLA diffusion baseline (e.g. DiffusionDrive or BridgeDrive vs Alpamayo).

---

## 5. Summary — which axis gives which uncertainty signal

|Axis|Signals it makes available|
|---|---|
|1 Decomposition|Uncertainty of intermediate outputs (detection, map, prediction)|
|2 Foundation-model role|Language channel (VLA), future channel (WAM), reasoner–planner interface (dual)|
|3 Output representation|Decision vs execution uncertainty; feasibility of samples|
|4 Generator|Sample spread, denoising-path stability, score / velocity sensitivity, token entropy, anchor-score entropy|
|5 Execution|Executed vs planned uncertainty; controller overrides|
|6 World-model role|Predicted-vs-observed surprise; per-candidate consequence uncertainty|
|7 Conditioning|Instruction sensitivity; instruction OOD|
|8 Reasoning|Trace uncertainty; reasoning–action consistency|
|9 Inputs|Perception uncertainty (separable only with object-level inputs or probes)|
|10 Training|Effect of RL on spread; expert-style dependence|

---

## Sources

**Taxonomy and surveys**: [VLA for AD survey (arXiv 2512.16760)](https://arxiv.org/html/2512.16760v2) · [DriveLaW (world-model paradigms)](https://arxiv.org/html/2512.23421v2) · [DA-WAM](https://arxiv.org/pdf/2608.19085) · [MM-Future](https://arxiv.org/pdf/2609.20377)

**Models**: [Alpamayo-R1](https://arxiv.org/pdf/2511.00088) · [Alpamayo 1.5 card](https://huggingface.co/nvidia/Alpamayo-1.5-10B) · [Alpamayo 1 repo](https://github.com/NVlabs/alpamayo) · [Alpamayo 2 Super](https://blogs.nvidia.com/blog/alpamayo-2-super-open-model-now-available) · [AutoVLA](https://arxiv.org/html/2506.13757v3) · [SimLingo](https://arxiv.org/html/2503.09594v1) · [ORION](https://arxiv.org/pdf/2503.19755) · [ReCogDrive](https://arxiv.org/abs/2506.08052v1) · [DiffVLA](https://www.alphaxiv.org/abs/2505.19381) · [VDRive](https://arxiv.org/html/2510.15446v2) · [DriveMoE](https://arxiv.org/abs/2505.16278v1) · [DriveVLA-W0](https://arxiv.org/html/2510.12796v1) · [DynVLA](https://arxiv.org/html/2603.11041v2) · [Poutine](https://arxiv.org/html/2506.11234v3) · [DiffusionDrive](https://arxiv.org/abs/2411.15139v3) · [DiffusionDriveV2](https://arxiv.org/abs/2512.07745v1) · [GoalFlow](https://arxiv.org/abs/2503.05689) · [Diffusion Planner](https://arxiv.org/pdf/2501.15564) · [BridgeDrive](https://arxiv.org/html/2509.23589v3) · [ResAD](https://www.alphaxiv.org/@zhiyu-zheng) · [Epona](https://arxiv.org/pdf/2506.24113) · [VaViM / VaVAM](https://arxiv.org/pdf/2502.15672) · [DriveLaW](https://arxiv.org/pdf/2512.23421) · [DriveWAM](https://arxiv.org/pdf/2605.28544) · [WA-JEPA](https://arxiv.org/pdf/2608.20974) · [SimWAM](https://arxiv.org/pdf/2608.07468) · [DriveVA](https://arxiv.org/abs/2604.04198) · [WoTE (BEV world model evaluation)](https://arxiv.org/html/2504.01941v1) · [LMDrive](https://openaccess.thecvf.com/content/CVPR2024/papers/Shao_LMDrive_Closed-Loop_End-to-End_Driving_with_Large_Language_Models_CVPR_2024_paper.pdf) · [AD-H](https://arxiv.org/pdf/2406.03474) · [LAD-Drive](https://arxiv.org/pdf/2603.02035)

**Evaluation and evidence**: [Bench2Drive leaderboard](https://carlaleaderboard-bench2drive.hf.space/) · [Driving on Registers (NAVSIM tables)](https://arxiv.org/pdf/2601.05083) · [WOD-E2E](https://arxiv.org/html/2510.26125v2) · [DriveBench](https://drive-bench.github.io/) · [Comfortably Stuck](https://arxiv.org/pdf/2610.21741) · [NAVSIM protocol](https://arxiv.org/pdf/2608.06008) · [PDMS definition](https://huggingface.co/XDimLab/ICCV2025-RealADSim-ClosedLoop-SubmissionDemo/blob/main/docs/metrics.md) · [LEAD / TF++](https://github.com/kesai-labs/lead/releases/tag/v1.4.0) · [Bench2DriveZoo](https://github.com/Thinklab-SJTU/Bench2DriveZoo)