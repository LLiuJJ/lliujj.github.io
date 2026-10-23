# Building, Modifying, and Verifying vLLM on Ubuntu 24.04.1 x86_64

> Target environment: Ubuntu 24.04.1 LTS x86_64 + NVIDIA CUDA GPU.
> Content compiled from `docs/getting_started/installation/gpu.cuda.inc.md`, `docs/contributing/incremental_build.md`, `docs/usage/troubleshooting.md`, `AGENTS.md`, `requirements/build/cuda.txt`, and `requirements/test/cuda.in` in this repository.

## Key Constraints (Read First)

| Item | Requirement | Notes |
|------|-------------|-------|
| Python | 3.10–3.14 (3.12 recommended) | `requires-python: >=3.10,<3.15` |
| PyTorch | `torch==2.13.0` | CUDA/ROCm versions are pinned; do not use system pip |
| CUDA | Default 12.9 (also available: 12.8 / 13.0 variants) | Blackwell GPU requires ≥ 12.8 |
| GCC | ≥ 11.3 | Ubuntu 24.04 ships GCC 13.x by default — **already satisfied**, no need to install gcc-11 as on 22.04 |
| Build Tools | CMake ≥ 3.26.1 + Ninja | See `requirements/build/cuda.txt` |
| Package Manager | `uv` required | System `python3` / bare `pip` are prohibited |

---

## 1. Environment Setup

### 1.1 System Dependencies

```bash
sudo apt update
sudo apt install -y build-essential python3-dev python3-pip git curl wget

# Speed up repeated compilations (highly recommended)
sudo apt install -y ccache
```

### 1.2 CUDA Toolkit

vLLM compiles with CUDA 12.9 by default. If not already installed, install 12.9 from
<https://developer.nvidia.com/cuda-toolkit-archive>, then:

```bash
export CUDA_HOME=/usr/local/cuda
export PATH="${CUDA_HOME}/bin:$PATH"
nvcc --version          # Verify nvcc is in PATH
${CUDA_HOME}/bin/nvcc --version   # Verify CUDA_HOME points correctly
```

### 1.3 Install uv and Create Virtual Environment

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env      # Add uv to PATH

git clone https://github.com/vllm-project/vllm.git
cd vllm

uv venv --python 3.12 --seed
source .venv/bin/activate
```

### 1.4 Build Dependencies

```bash
uv pip install -r requirements/build/cuda.txt --torch-backend=auto
```

`requirements/build/cuda.txt` pins key versions: `cmake>=3.26.1`, `ninja`, `torch==2.13.0`,
`setuptools-rust>=1.9.0`, `protobuf>=5.29.6`, etc.

### 1.5 (Optional) Development Hooks

```bash
uv pip install -r requirements/lint.txt
pre-commit install
```

---

## 2. Three Build Methods (Choose One Based on Your Scenario)

### Method A — Python-Only Changes (No C++/CUDA Compilation, Fastest)

Source: `docs/getting_started/installation/gpu.cuda.inc.md` (Python-only build).

```bash
VLLM_USE_PRECOMPILED=1 uv pip install --editable . --torch-backend=auto
```

How it works:

1. Finds the base commit of the current branch on `main`.
2. Downloads the pre-compiled wheel for that commit.
3. Reuses compiled artifacts (`_C.abi3.so`, `_moe_C.abi3.so`, etc.) and the `vllm-rs` binary.

> Note: If you modified C++/CUDA code under `csrc/` but still use this method, you'll get
> `library not found` or `undefined symbol` errors on import. Switch to Method B instead.

Relevant environment variables:

- `VLLM_PRECOMPILED_WHEEL_LOCATION`: Specify the URL or local path of the pre-compiled wheel.
- `VLLM_PRECOMPILED_WHEEL_COMMIT`: Override the commit; set to `nightly` to use the most recently built main commit.
- `VLLM_PRECOMPILED_WHEEL_VARIANT`: Specify the variant subdirectory, e.g., `cu129`, `cu130`, `cpu`.

If the wheel for the base commit hasn't been built yet (just merged):

```bash
export VLLM_PRECOMPILED_WHEEL_COMMIT=nightly
export VLLM_USE_PRECOMPILED=1
uv pip install --editable .
```

#### Rebuild Rust Frontend (vllm-rs)

```bash
./build_rust.sh          # release build
./build_rust.sh --debug  # faster build for development
```

Artifacts are placed in `vllm/vllm-rs`.

### Method B — Full Source Build (When C++/CUDA Kernels Are Modified)

Source: `docs/getting_started/installation/gpu.cuda.inc.md` (Full build).

```bash
uv pip install -e . --torch-backend=auto
```

Limit parallel jobs when memory is constrained:

```bash
export MAX_JOBS=6
uv pip install -e .
```

Use ccache to speed up repeated builds (must use `--no-build-isolation`, otherwise pip creates random directories each time, invalidating ccache):

```bash
CCACHE_NOHASHDIR="true" uv pip install --no-build-isolation -e .
```

#### Use Existing PyTorch Installation

```bash
git clone https://github.com/vllm-project/vllm.git
cd vllm
python use_existing_torch.py
uv pip install -r requirements/build/cuda.txt
uv pip install --no-build-isolation -e .
```

#### Use Local cutlass

```bash
VLLM_CUTLASS_SRC_DIR=/path/to/cutlass uv pip install -e . --torch-backend=auto
```

### Method C — Docker Image Build

Source: `docs/getting_started/installation/gpu.cuda.inc.md` (Build image from source).

```bash
# Compile only for the current machine's GPU architecture (faster)
DOCKER_BUILDKIT=1 docker build . \
    --target vllm-openai \
    --tag vllm/vllm-openai \
    --file docker/Dockerfile \
    --build-arg torch_cuda_arch_list=""

# Use pre-compiled wheel to speed up when no C++ changes
DOCKER_BUILDKIT=1 docker build . \
    --target vllm-openai \
    --tag vllm/vllm-openai \
    --file docker/Dockerfile \
    --build-arg VLLM_USE_PRECOMPILED="1"
```

> Docker builds are memory-intensive and time-consuming. Recommended for CI environments without GPUs or when reproducible artifacts are needed.

---

## 3. Incremental Build Workflow (Recommended When Frequently Modifying Kernels)

Source: `docs/contributing/incremental_build.md`. First do an editable install using Method A,
then use CMake for incremental compilation to avoid running a full `pip install` each time.

### 3.1 Prerequisites

```bash
# First time: editable install with pre-compiled wheel
VLLM_USE_PRECOMPILED=1 uv pip install -U -e . --torch-backend=auto

# Build tools (cmake/ninja/ccache)
uv pip install -r requirements/build/cuda.txt --torch-backend=auto
```

### 3.2 Generate CMake Configuration

```bash
python tools/generate_cmake_presets.py
# Force overwrite existing configuration:
python tools/generate_cmake_presets.py --force-overwrite
```

Key fields in the generated `CMakeUserPresets.json`:

- `CMAKE_CUDA_COMPILER`: Path to `nvcc`.
- `CMAKE_C/CXX/CUDA_COMPILER_LAUNCHER`: Set to `ccache` for acceleration.
- `VLLM_PYTHON_EXECUTABLE`: Python in the venv.
- `CMAKE_INSTALL_PREFIX`: `${sourceDir}`, installs artifacts back into the source tree for editable use.
- `CMAKE_JOB_POOLS` / `jobs`: Parallelism, adjust based on machine cores/memory.

### 3.3 Build and Install

```bash
cmake --preset release                         # Configure (first time)
cmake --build --preset release --target install  # Compile + install back to source tree
```

After this, every time you modify `csrc/`, just re-run the last command to compile only changed files.

Build artifacts (under `cmake-build-release/`) include `_C.abi3.so`, `_moe_C.abi3.so`,
`_flashmla_C.abi3.so`, `cumem_allocator.abi3.so`, etc. The `--target install` step copies them
into the `vllm` package directory, updating the editable install.

### 3.4 Clean Rebuild

For stubborn errors or after switching branches:

```bash
rm -rf cmake-build-release
cmake --preset release
cmake --build --preset release --target install
```

---

## 4. Verification Workflow After Code Changes

Verify progressively from fastest to slowest. For Python-only changes (Method A), focus on confirming:
**pre-compiled libraries load correctly** + **editable changes take effect**.

### 4.1 Verification Levels

| Level | What to Verify | Time | Applicable To |
|:-----:|----------------|:----:|---------------|
| 1 | Installation succeeded, libraries load | Seconds | Any change |
| 2 | torch.compile / Triton works | Seconds | Any change |
| 3 | Editable changes take effect | Seconds | Python changes |
| 4 | Related unit tests | Minutes | Targeted verification |
| 5 | End-to-end small model inference | Minutes | Integration verification |

### 4.2 Level 1 — Installation and Library Loading

```bash
python -c "import vllm; print('vllm', vllm.__version__); print(vllm.__file__)"
```

- No `undefined symbol` / `library not found` errors → pre-compiled `.so` loaded successfully.
- `__file__` points to your source directory (not site-packages) → editable install is active.

### 4.3 Level 2 — torch.compile Sanity Check

Source: `docs/usage/troubleshooting.md` (torch.compile Error).

```bash
python - << 'PY'
import torch

@torch.compile
def f(x):
    x = x + 1
    x = x * 2
    x = x.sin()
    return x

x = torch.randn(4, 4).cuda()
print(f(x))
print("sanity check is successful!")
PY
```

If you see `sanity check is successful!`, the Triton/Inductor pipeline is working.

### 4.4 Level 3 — Editable Changes Take Effect

Temporarily add a print statement to a Python file to confirm it actually executes:

```bash
# For example, temporarily add a print at the top of vllm/__init__.py
python -c "import vllm"   # You should see your output
```

If you see the output → changes are coming from local source, not a cached wheel.

### 4.5 Level 4 — Related Unit Tests

First install test dependencies (`requirements/test/cuda.in`, includes pytest, pytest-asyncio, etc.):

```bash
uv pip install -r requirements/test/cuda.in
```

Run only tests related to your changes (recommended format from AGENTS.md):

```bash
.venv/bin/python -m pytest tests/<your_module>/test_xxx.py -v
```

Common lightweight entry points:

```bash
# Basic correctness
.venv/bin/python -m pytest tests/basic_correctness/test_basic_correctness.py -v
# Compilation/graph
.venv/bin/python -m pytest tests/compile/fullgraph/test_simple.py -v
# Configuration
.venv/bin/python -m pytest tests/config/test_config_utils.py -v
```

Test writing principles (from AGENTS.md): Reuse existing `conftest.py` fixtures, test one behavior per case,
minimal setup, name by behavior, prefer extending existing test files over creating new ones.

### 4.6 Level 5 — End-to-End Small Model Inference

Use a tiny model for a real generation to verify the entire pipeline:

```bash
python - << 'PY'
from vllm import LLM, SamplingParams

# enforce_eager=True disables CUDA graph for faster cold start
llm = LLM(model="Qwen/Qwen3-0.6B", enforce_eager=True)
out = llm.generate(["Hello, vLLM!"], SamplingParams(max_tokens=8))
print(out[0].outputs[0].text)
PY
```

`enforce_eager` is great for quick smoke tests; without it, `torch.compile` + graph capture are triggered,
which is closer to production but slower on first run due to compilation.

### 4.7 Quick Reference: Minimum Verification in One Command

```bash
source .venv/bin/activate && \
python -c "import vllm; print('vllm', vllm.__version__, 'from', vllm.__file__)" && \
python -m pytest tests/basic_correctness/test_basic_correctness.py -v -x
```

Passing means: pre-compiled library loading OK + editable install OK + basic inference pipeline OK.

---

## 5. Additional Requirements for Model/Accuracy-Related Changes

When changes affect model output, accuracy, or serving behavior (from AGENTS.md):

- Search `tests/evals/` or use `vllm bench` to run model evaluations; include results in the PR.
- Don't wait for reviewers to ask — run them proactively.

```bash
# Install evaluation dependencies as needed
uv pip install -r requirements/test/cuda.in
```

---

## 6. Linting and Type Checking

Source: AGENTS.md (Running linters). Python line width limit is 88 characters.

```bash
# All pre-commit hooks (staged files)
pre-commit run

# All files
pre-commit run --all-files

# Single hook
pre-commit run ruff-check --all-files

# mypy (as in CI)
pre-commit run mypy-3.12 --all-files --hook-stage manual
```

Documentation style uses Google-style docstrings (`Args:`/`Returns:`/`Raises:`),
not reStructuredText/Sphinx fields (`:param:`/`:return:`/`:rtype:`).

---

## 7. Quick Reference: Minimal End-to-End Flow (From Scratch Build + Verify)

```bash
# --- 1. System + uv ---
sudo apt install -y build-essential python3-dev git curl ccache
curl -LsSf https://astral.sh/uv/install.sh | sh && source $HOME/.local/bin/env
export CUDA_HOME=/usr/local/cuda && export PATH="${CUDA_HOME}/bin:$PATH"

# --- 2. Clone + venv ---
git clone https://github.com/vllm-project/vllm.git && cd vllm
uv venv --python 3.12 --seed && source .venv/bin/activate

# --- 3. Build dependencies ---
uv pip install -r requirements/build/cuda.txt --torch-backend=auto

# --- 4a. Python-only changes (fast) ---
VLLM_USE_PRECOMPILED=1 uv pip install --editable . --torch-backend=auto

# --- 4b. Or full source build (includes C++/CUDA) ---
export MAX_JOBS=6
uv pip install -e . --torch-backend=auto

# --- 5. Test dependencies ---
uv pip install -r requirements/test/cuda.in

# --- 6. Verify ---
python -c "import vllm; print('vllm', vllm.__version__, 'from', vllm.__file__)"
.venv/bin/python -m pytest tests/basic_correctness/test_basic_correctness.py -v -x
```

---

## 8. Common Issues Quick Reference

| Symptom | Cause / Fix |
|---------|-------------|
| Import reports `undefined symbol` / `library not found` | Used Method A but modified C++/CUDA; switch to Method B for full build |
| Pre-compiled wheel not found | Base commit just merged, not yet built; set `VLLM_PRECOMPILED_WHEEL_COMMIT=nightly` |
| Compilation OOM | `export MAX_JOBS=1` (WSL defaults to using only 50% of memory) |
| `torch.compile` errors | Check Triton version, run Level 2 sanity check |
| ccache misses | Use `--no-build-isolation` + `CCACHE_NOHASHDIR=true` |
| CUDA driver too old | Install `cuda-compat-12-9`, set `VLLM_ENABLE_CUDA_COMPATIBILITY=1` |
| Want to develop on non-Linux | `export VLLM_TARGET_DEVICE=empty` then `uv pip install -e .` (import only, no execution) |
| Tests report `OSError: Can't load the configuration of '<model>'` | Cannot access HuggingFace from your network; see **Section 9** below |
| `NotExistError [404] record not found` after setting `VLLM_USE_MODELSCOPE` | Test models (e.g., `hmellor/tiny-random-*`) are not available on ModelScope; use `HF_ENDPOINT=https://hf-mirror.com` for tests, and `unset VLLM_USE_MODELSCOPE` |

---

## 9. Accessing HuggingFace from Regions with Restricted Access

Direct access to huggingface.co may be blocked in some regions. When running tests, loading model
configurations/weights may fail with `OSError: Can't load the configuration of '<model>'` (e.g.,
`tests/basic_correctness/test_cpu_offload.py::test_tower_weight_offloading` fails downloading
`Qwen/Qwen3.5-0.8B`). Solutions are provided below by scenario; use Solution 1 for running tests.

### Solution 1: `HF_ENDPOINT` Mirror (Recommended, Simplest for Running Tests)

`huggingface_hub` natively supports the `HF_ENDPOINT` environment variable. Point it to the community mirror
[hf-mirror.com](https://hf-mirror.com), and all `from_pretrained` / `snapshot_download` calls
automatically use the mirror — no code changes or extra packages needed.

```bash
# Temporary (single test run)
export HF_ENDPOINT=https://hf-mirror.com
.venv/bin/python -m pytest tests/basic_correctness/test_cpu_offload.py -v

# Persistent (add to ~/.zshrc or ~/.bashrc)
echo 'export HF_ENDPOINT=https://hf-mirror.com' >> ~/.zshrc
source ~/.zshrc
```

Why this works best for tests: Small test models (e.g., `hmellor/tiny-random-LlamaForCausalLM`)
are available on HF Hub and fully mirrored by hf-mirror; whereas ModelScope may not index such
random test models.

**Choose either mirror or proxy, do not mix them**: If both `http_proxy`/`https_proxy`/`all_proxy`
and `HF_ENDPOINT=https://hf-mirror.com` are set, requests to the domestic mirror will be routed
through the proxy to overseas nodes, becoming slower or even failing. If using the mirror, disable
proxies; if using a proxy, don't set `HF_ENDPOINT` and access HF official directly.

```bash
# Mirror approach (recommended for tests): disable proxies
unset http_proxy https_proxy all_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY
export HF_ENDPOINT=https://hf-mirror.com
# Proxy approach: keep proxies, don't set mirror
unset HF_ENDPOINT
export https_proxy=http://192.168.3.10:7890 http_proxy=http://192.168.3.10:7890 all_proxy=socks5://192.168.3.10:7890
```

### Solution 2: Built-in ModelScope Support (Suitable for Production with Real Models)

vLLM has ModelScope integration in multiple places (`vllm/envs.py`,
`vllm/model_executor/model_loader/weight_utils.py`,
`vllm/transformers_utils/config.py`, `vllm/transformers_utils/repo_utils.py`).
When enabled, weight downloads, config loading, and tokenizers all go through the ModelScope community:

```bash
uv pip install modelscope
export VLLM_USE_MODELSCOPE=True   # Note: True (see parsing in envs.py)
```

> Note: ModelScope mainly hosts domestic/popular models. Models like `Qwen/Qwen3.5-0.8B` are
> likely available, but test models like `hmellor/tiny-random-*` **may not be**. Use Solution 1 for tests.

### Solution 3: Download to Local Cache First, Then Run Offline

After the first download via Solution 1, models are cached in `~/.cache/huggingface`. Subsequent runs work offline:

```bash
export HF_ENDPOINT=https://hf-mirror.com   # First time: download online
.venv/bin/python -m pytest tests/basic_correctness/test_cpu_offload.py -v

# After that: fully offline, no network needed
export HF_HUB_OFFLINE=1
.venv/bin/python -m pytest tests/basic_correctness/test_cpu_offload.py -v
```

### Solution 4: Use Local Paths Directly

When models are already downloaded locally, replace `model="<repo>"` in tests with a local absolute path,
or point to the local cache directory:

```bash
export HF_HOME=/path/to/local/hf/cache
# Or pass a local directory directly to LLM(model=...)
```

### Direct Fix (For This Specific Test)

```bash
export HF_ENDPOINT=https://hf-mirror.com
.venv/bin/python -m pytest \
    tests/basic_correctness/test_cpu_offload.py::test_tower_weight_offloading \
    -v
```

If the mirror occasionally times out, add concurrency and retry controls:

```bash
export HF_ENDPOINT=https://hf-mirror.com
export HF_HUB_DOWNLOAD_TIMEOUT=60
.venv/bin/python -m pytest tests/basic_correctness/test_cpu_offload.py -v \
    --timeout=300
```

**Summary**: Running tests → Solution 1 (`HF_ENDPOINT=https://hf-mirror.com`, zero-cost full mirror);
running production inference with domestic models → can combine Solution 2 (`VLLM_USE_MODELSCOPE=True`).
Both can be set simultaneously without conflicts.

---

## Reference File Index

| File | Content |
|------|---------|
| `docs/getting_started/installation/gpu.cuda.inc.md` | CUDA installation and three build methods |
| `docs/contributing/incremental_build.md` | CMake incremental build workflow |
| `docs/usage/troubleshooting.md` | torch.compile sanity check / CUDA compatibility |
| `requirements/build/cuda.txt` | Build dependency version constraints |
| `requirements/test/cuda.in` | Test dependencies |
| `AGENTS.md` | Development workflow, testing standards, linting, commit conventions |
| `tools/generate_cmake_presets.py` | CMakeUserPresets generation script |
| `build_rust.sh` | Rust frontend build |
| `use_existing_torch.py` | Reuse existing PyTorch |
