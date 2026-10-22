
> [!summary] Bottom line
> - **CARLA / Bench2Drive**:  X cannot run on DALIA compute nodes (x86-only server binary).
> - **AlpaSim (default NuRec renderer)**: X renderer container is amd64-only.
> - **HUGSIM**: 🟡 likely portable (built from source), but its environment is pinned to x86 + CUDA 11.8 → porting work needed, not yet run.
> - **NAVSIM**: 🟢 very likely fine (pure Python), but it is **pseudo-closed-loop**, not true closed loop.
---

## 1. Machine facts

| Item          | Value                                                                        | How verified           |
| ------------- | ---------------------------------------------------------------------------- | ---------------------- |
| Login nodes   | Intel Xeon Gold 6530, **x86_64**, no GPU                                     | IDRIS doc              |
| Compute nodes | 2× Grace (Neoverse-V2) **aarch64**, 4× GB200 (186 GB)                        | IDRIS doc + `uname -m` |
| GPU driver    | **595.58.03**                                                                | `nvidia-smi`           |
| Page size     | **4096** (no Grace 64K-page issue)                                           | `getconf PAGESIZE`     |
| Containers    | `apptainer`, `singularity`, `enroot` + **pyxis** SLURM plugin; **no Docker** | smoke test             |
| CUDA modules  | `cuda13.1/toolkit`, `nvhpc` 25.9–26.5                                        | `module avail`         |
| Internet      | Compute nodes have access (via `platprox.idris.fr` proxy)                    | `wget` output          |

> [!warning] Login vs compute architecture
> Login nodes are **x86**, compute nodes are **ARM**. Anything installed/compiled on the login node will not run on compute nodes. Build environments on a compute node (IDRIS recommendation).

---

## 2. Test log

### 2.1 CARLA 0.9.15 (used by Bench2Drive)

**Static check (login node)**
```bash
wget https://carla-releases.s3.us-east-005.backblazeb2.com/Linux/CARLA_0.9.15.tar.gz   # 7.8 GB, ~6 min
mkdir carla && tar -xzf CARLA_0.9.15.tar.gz -C carla                                   # slow on Lustre
file carla/CarlaUE4/Binaries/Linux/CarlaUE4-Linux-Shipping
```
Result:
```
ELF 64-bit LSB executable, x86-64, ... interpreter /lib64/ld-linux-x86-64.so.2
```

**Execution check (compute node)**
```bash
srun --gres=gpu:1 --time=00:10:00 --pty bash
uname -m                              # aarch64
./carla/CarlaUE4.sh -RenderOffScreen
```
Result:
```
cannot execute binary file: Exec format error
```

> [!failure] Verdict: CARLA cannot run on DALIA
> Server is distributed as x86-64 only; no x86 emulation on compute nodes.
> Web search found **no ARM build of the CARLA server** (only community aarch64 *client* wheels, e.g. 0.9.13 for Jetson). Building UE4/UE5 + CARLA for aarch64 = months of work → abandoned.

### 2.2 AlpaSim (NVIDIA) – smoke test script `dalia_sim_smoketest.sh`

- Required renderer image (from `src/wizard/configs/base_config.yaml`):
  `nvcr.io/nvidia/nre/nre-ga:26.04`
- Pull on compute node:
```bash
apptainer pull nre.sif docker://nvcr.io/nvidia/nre/nre-ga:26.04
```
Key output:
```
WARNING: Architecture amd64 does not match build arch arm64
FATAL: ... mksquashfs command failed: signal: killed
```

> [!failure] Verdict: AlpaSim (NuRec renderer) cannot run on DALIA
> The architecture warning comes from image metadata → **renderer image is amd64-only**.
> The `FATAL` is a separate issue (mksquashfs killed: job memory limit + Lustre temp dir), irrelevant to the verdict.
> Other AlpaSim services are Python and could probably run natively, but without a renderer there is no simulation.

> [!note] Remaining AlpaSim options (untested)
> - OmniDreams/FlashDreams renderer is built locally (`flashdreams-alpasim:local`) → aarch64 build conceivable; ~48–96 GB VRAM; single-view only.
> - Ask NVIDIA (Alpamayo dev forum) whether an arm64 NRE image is planned.
> - Run AlpaSim on an x86 GPU cluster.

### 2.3 HUGSIM – smoke test (static analysis only)

From `HUGSIM/pixi.toml`:
```toml
platforms = ["linux-64"]
python = "==3.11.10"
torch = { version = "==2.4.1", index = "https://download.pytorch.org/whl/cu118" }
no-build-isolation = ["gsplat", "pytorch3d", "tinycudann", "simple-knn"]
# + apex (old pinned commit) for InverseForm
```

> [!question] Verdict: likely feasible, not yet demonstrated
> Blockers are version pins, not closed binaries:
> 1. `platforms` lacks `linux-aarch64` → must add and re-solve (drop `pixi.lock`).
> 2. torch 2.4.1 + cu118 → no B200 (sm_100) support; need torch ≥ 2.7 with CUDA ≥ 12.8 (verify on pytorch.org for aarch64).
> 3. Source-compiled extensions (gsplat, simple-knn, pytorch3d, tinycudann, apex) must be rebuilt with `TORCH_CUDA_ARCH_LIST="10.0"`; apex and tinycudann are the riskiest.
> 4. nvcc **major** version must match torch's CUDA (cluster module is CUDA 13.1 → either torch cu130 + module, or torch cu128 + conda-forge `cuda-nvcc` 12.8 in pixi).
> Planner clients: start with LTF (NAVSIM client); UniAD/VAD (mmdet3d) likely painful on ARM + Blackwell.
> Estimated effort: 1–3 days to a first rendered step (personal estimate).

**Next check (not done yet)**
```bash
grep -rn -E "import (gsplat|simple_knn|pytorch3d|tinycudann|apex)|from (gsplat|simple_knn|pytorch3d|tinycudann|apex)" \
  closed_loop.py sim/ gaussian_renderer/ scene/ utils/ | sort -u
```
→ decides whether apex / tinycudann can be dropped.

### 2.4 PyTorch on node
`PyTorch not importable here` – no system PyTorch; must be provided by env/container.

---

## 3. Summary table

| Simulator | Type | Status on DALIA | Basis |
|---|---|---|---|
| CARLA / Bench2Drive | Game-engine closed loop | ❌ Not possible | Verified (`file`, `Exec format error`) |
| AlpaSim (NuRec) | Neural-reconstruction closed loop | ❌ Not possible | Verified (amd64-only image) |
| AlpaSim (FlashDreams) | Video-model closed loop | ❓ Untested | Inferred |
| HUGSIM | 3DGS closed loop | 🟡 Likely with porting | Inferred from deps |
| NAVSIM v2 | Pseudo-closed loop | 🟢 Very likely | Pure Python, to verify |

---

## 4. Lessons / gotchas

- `file <binary>` is enough to check architecture without running anything.
- Apptainer pulls on Lustre are slow and may be killed: use node-local `APPTAINER_TMPDIR` (`/tmp/$USER/...`, check `df -h /tmp`) and request more memory (`--mem`, `--cpus-per-task`).
- `lustre.lov` / `xattr` warnings during pulls are harmless noise.
- Clean up after failed pulls: `rm -rf $WORK/caches/apptainer_tmp/build-temp-*`.
- Apptainer silently falls back to amd64 when no arm64 manifest exists → always read the final architecture warning.

---

## 5. Decision & next steps

> [!important] Plan
> - [ ] **NAVSIM on DALIA** as main development loop (calibration / UQ studies at scale).
> - [ ] **Request x86 GPU access** for true closed loop: Jean Zay H100 nodes (Intel Sapphire Rapids, x86) via GENCI *accès dynamique*.
> - [ ] Ask IDRIS: headless **Vulkan** support on Jean Zay GPU partitions (needed by CARLA; not needed by AlpaSim/NuRec)? Shared filesystem between DALIA and Jean Zay?
> - [ ] On Jean Zay: AlpaSim via Apptainer/enroot; Bench2Drive if Vulkan works.
> - [ ] HUGSIM port on DALIA **only as fallback** (time-box ~3 days).
> - [ ] Optional: ask NVIDIA forum about arm64 NRE image.

> [!caution] Why not settle for NAVSIM + HUGSIM
> NAVSIM is non-reactive (no compounding errors, no recovery dynamics). HUGSIM has much lower adoption than Bench2Drive/AlpaSim. Closed-loop failure-prediction claims will need a recognized true closed-loop benchmark.

---

## 6. References

- CARLA releases: https://carla-releases.s3.us-east-005.backblazeb2.com/Linux/CARLA_0.9.15.tar.gz
- Autoware CARLA scenario (server x86_64 only): https://autowarefoundation.github.io/autoware_carla_scenario/latest/installation/
- CARLA ARM issues: https://github.com/carla-simulator/carla/issues/6750 · https://github.com/carla-simulator/carla/discussions/6084
- AlpaSim: https://github.com/NVlabs/alpasim · onboarding: https://github.com/NVlabs/alpasim/blob/main/docs/ONBOARDING.md · tutorial: https://github.com/NVlabs/alpasim/blob/main/docs/TUTORIAL.md
- NuRec container: https://catalog.ngc.nvidia.com/orgs/nvidia/nre/containers/nre-ga
- HUGSIM: https://github.com/hyzhou404/HUGSIM
- NAVSIM / pseudo-simulation: https://arxiv.org/abs/2506.04218
- Jean Zay H100 extension: https://www.hpcwire.com/?p=168505
- Grace page-size gotcha: https://docs.coreweave.com/products/sunk/run_workloads/run-jobs-on-grace-cpu.md

Related: [[VLA & World-Action Models – architectural diagnostic]] · [[Driving VLA-WAM landscape]]
