# vLLM 在 Ubuntu 24.04.1 x86_64 上的编译、改动与验证流程

> 目标环境：Ubuntu 24.04.1 LTS x86_64 + NVIDIA CUDA GPU。
> 内容基于本仓库 `docs/getting_started/installation/gpu.cuda.inc.md`、`docs/contributing/incremental_build.md`、`docs/usage/troubleshooting.md`、`AGENTS.md`、`requirements/build/cuda.txt`、`requirements/test/cuda.in` 整理。

## 关键约束（先读）

| 项 | 要求 | 备注 |
|----|------|------|
| Python | 3.10–3.14（推荐 3.12） | `requires-python: >=3.10,<3.15` |
| PyTorch | `torch==2.13.0` | CUDA/ROCm 版本固定，不要用系统 pip |
| CUDA | 默认 12.9（也提供 12.8 / 13.0 变体） | Blackwell GPU 需 ≥ 12.8 |
| GCC | ≥ 11.3 | Ubuntu 24.04 默认 GCC 13.x **已满足**，无需像 22.04 那样装 gcc-11 |
| 构建工具 | CMake ≥ 3.26.1 + Ninja | 见 `requirements/build/cuda.txt` |
| 包管理 | 必须 uv | 禁止系统 `python3` / 裸 `pip` |

---

## 1. 环境准备

### 1.1 系统依赖

```bash
sudo apt update
sudo apt install -y build-essential python3-dev python3-pip git curl wget

# 加速重复编译（强烈推荐）
sudo apt install -y ccache
```

### 1.2 CUDA Toolkit

vLLM 默认用 CUDA 12.9 编译。若尚未安装，从
<https://developer.nvidia.com/cuda-toolkit-archive> 安装 12.9，然后：

```bash
export CUDA_HOME=/usr/local/cuda
export PATH="${CUDA_HOME}/bin:$PATH"
nvcc --version          # 验证 nvcc 在 PATH
${CUDA_HOME}/bin/nvcc --version   # 验证 CUDA_HOME 指向正确
```

### 1.3 安装 uv 并创建虚拟环境

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env      # 让 uv 进 PATH

git clone https://github.com/vllm-project/vllm.git
cd vllm

uv venv --python 3.12 --seed
source .venv/bin/activate
```

### 1.4 构建依赖

```bash
uv pip install -r requirements/build/cuda.txt --torch-backend=auto
```

`requirements/build/cuda.txt` 固定了关键版本：`cmake>=3.26.1`、`ninja`、`torch==2.13.0`、
`setuptools-rust>=1.9.0`、`protobuf>=5.29.6` 等。

### 1.5（可选）开发钩子

```bash
uv pip install -r requirements/lint.txt
pre-commit install
```

---

## 2. 三种构建方式（按场景选其一）

### 方式 A — 仅改 Python 代码（不编译 C++/CUDA，最快）

来源：`docs/getting_started/installation/gpu.cuda.inc.md`（Python-only build）。

```bash
VLLM_USE_PRECOMPILED=1 uv pip install --editable . --torch-backend=auto
```

工作原理：

1. 查找当前分支在 `main` 上的 base commit。
2. 下载该 commit 的预编译 wheel。
3. 复用其编译产物（`_C.abi3.so`、`_moe_C.abi3.so` 等）和 `vllm-rs` 二进制。

> 注意：若你改了 `csrc/` 下的 C++/CUDA 代码却仍用此方式，import 时会报
> `library not found` 或 `undefined symbol`。此时必须切到方式 B。

相关环境变量：

- `VLLM_PRECOMPILED_WHEEL_LOCATION`：指定预编译 wheel 的 URL 或本地路径。
- `VLLM_PRECOMPILED_WHEEL_COMMIT`：覆盖 commit，可设为 `nightly` 选最近已构建的 main commit。
- `VLLM_PRECOMPILED_WHEEL_VARIANT`：指定变体子目录，如 `cu129`、`cu130`、`cpu`。

若 base commit 的 wheel 尚未构建好（刚合并），可：

```bash
export VLLM_PRECOMPILED_WHEEL_COMMIT=nightly
export VLLM_USE_PRECOMPILED=1
uv pip install --editable .
```

#### 重建 Rust 前端（vllm-rs）

```bash
./build_rust.sh          # release 构建
./build_rust.sh --debug  # 开发用更快构建
```

产物放在 `vllm/vllm-rs`。

### 方式 B — 完整从源码编译（改了 C++/CUDA kernel）

来源：`docs/getting_started/installation/gpu.cuda.inc.md`（Full build）。

```bash
uv pip install -e . --torch-backend=auto
```

内存有限时限制并行任务数：

```bash
export MAX_JOBS=6
uv pip install -e .
```

配合 ccache 加速重复构建（必须用 `--no-build-isolation`，否则 pip 每次建随机目录导致 ccache 失效）：

```bash
CCACHE_NOHASHDIR="true" uv pip install --no-build-isolation -e .
```

#### 使用已有 PyTorch 安装

```bash
git clone https://github.com/vllm-project/vllm.git
cd vllm
python use_existing_torch.py
uv pip install -r requirements/build/cuda.txt
uv pip install --no-build-isolation -e .
```

#### 使用本地 cutlass

```bash
VLLM_CUTLASS_SRC_DIR=/path/to/cutlass uv pip install -e . --torch-backend=auto
```

### 方式 C — Docker 镜像构建

来源：`docs/getting_started/installation/gpu.cuda.inc.md`（Build image from source）。

```bash
# 只为当前机器的 GPU 架构编译（更快）
DOCKER_BUILDKIT=1 docker build . \
    --target vllm-openai \
    --tag vllm/vllm-openai \
    --file docker/Dockerfile \
    --build-arg torch_cuda_arch_list=""

# 不改 C++ 时用预编译 wheel 加速
DOCKER_BUILDKIT=1 docker build . \
    --target vllm-openai \
    --tag vllm/vllm-openai \
    --file docker/Dockerfile \
    --build-arg VLLM_USE_PRECOMPILED="1"
```

> Docker 构建内存大、耗时长，推荐在无 GPU 的 CI 或需要可复现产物时使用。

---

## 3. 增量编译工作流（频繁改 kernel 时推荐）

来源：`docs/contributing/incremental_build.md`。先用方式 A 做一次 editable 安装，
再用 CMake 增量编译，避免每次跑完整 `pip install`。

### 3.1 前置

```bash
# 初次：预编译 wheel 做 editable 安装
VLLM_USE_PRECOMPILED=1 uv pip install -U -e . --torch-backend=auto

# 构建工具（cmake/ninja/ccache）
uv pip install -r requirements/build/cuda.txt --torch-backend=auto
```

### 3.2 生成 CMake 配置

```bash
python tools/generate_cmake_presets.py
# 强制覆盖已有配置：
python tools/generate_cmake_presets.py --force-overwrite
```

生成的 `CMakeUserPresets.json` 关键字段：

- `CMAKE_CUDA_COMPILER`：`nvcc` 路径。
- `CMAKE_C/CXX/CUDA_COMPILER_LAUNCHER`：设为 `ccache` 加速。
- `VLLM_PYTHON_EXECUTABLE`：venv 里的 python。
- `CMAKE_INSTALL_PREFIX`：`${sourceDir}`，把产物装回源码树供 editable 用。
- `CMAKE_JOB_POOLS` / `jobs`：并行度，按机器核数/内存调整。

### 3.3 构建并安装

```bash
cmake --preset release                         # 配置（首次）
cmake --build --preset release --target install  # 编译 + 装回源码树
```

之后每次改 `csrc/` 只需重跑最后一条，仅编译变动文件。

构建产物（`cmake-build-release/` 下）会包含 `_C.abi3.so`、`_moe_C.abi3.so`、
`_flashmla_C.abi3.so`、`cumem_allocator.abi3.so` 等，由 `--target install` 拷入
`vllm` 包目录，更新 editable 安装。

### 3.4 清理重建

遇到顽固错误或切换分支后：

```bash
rm -rf cmake-build-release
cmake --preset release
cmake --build --preset release --target install
```

---

## 4. 改动代码后的验证流程

按从快到慢的梯度逐级验证。仅改 Python（方式 A）时重点确认：
**预编译库可加载** + **editable 改动生效**。

### 4.1 验证梯度

| 级别 | 验证什么 | 耗时 | 适用 |
|:----:|----------|:----:|------|
| 1 | 安装成功、库可加载 | 秒级 | 任何改动 |
| 2 | torch.compile / Triton 正常 | 秒级 | 任何改动 |
| 3 | editable 改动生效 | 秒级 | Python 改动 |
| 4 | 相关单元测试 | 分钟级 | 针对性验证 |
| 5 | 端到端小模型推理 | 分钟级 | 集成验证 |

### 4.2 级别 1 — 安装与库加载

```bash
python -c "import vllm; print('vllm', vllm.__version__); print(vllm.__file__)"
```

- 不报 `undefined symbol` / `library not found` → 预编译 `.so` 加载正常。
- `__file__` 指向你的源码目录（而非 site-packages） → editable 安装生效。

### 4.3 级别 2 — torch.compile sanity check

来源：`docs/usage/troubleshooting.md`（torch.compile Error）。

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

看到 `sanity check is successful!` 即 Triton/Inductor 链路正常。

### 4.4 级别 3 — editable 改动生效

临时在某个 Python 文件加一行打印，确认它被实际执行：

```bash
# 例如临时在 vllm/__init__.py 顶部加 print
python -c "import vllm"   # 应看到你的输出
```

能看到 → 改动走的是本地源码而非缓存 wheel。

### 4.5 级别 4 — 相关单元测试

先装测试依赖（`requirements/test/cuda.in`，含 pytest、pytest-asyncio 等）：

```bash
uv pip install -r requirements/test/cuda.in
```

只跑和改动相关的测试文件（AGENTS.md 推荐格式）：

```bash
.venv/bin/python -m pytest tests/<你的模块>/test_xxx.py -v
```

常用轻量入口：

```bash
# 基础正确性
.venv/bin/python -m pytest tests/basic_correctness/test_basic_correctness.py -v
# 编译/图
.venv/bin/python -m pytest tests/compile/fullgraph/test_simple.py -v
# 配置
.venv/bin/python -m pytest tests/config/test_config_utils.py -v
```

测试编写原则（AGENTS.md）：复用现有 `conftest.py` fixtures、一个用例只测一个行为、
最小 setup、按行为命名、优先在现有测试文件扩展而非新建。

### 4.6 级别 5 — 端到端小模型推理

用极小模型做一次真实生成，确认整条链路通：

```bash
python - << 'PY'
from vllm import LLM, SamplingParams

# enforce_eager=True 关闭 CUDA graph，更快冷启动
llm = LLM(model="Qwen/Qwen3-0.6B", enforce_eager=True)
out = llm.generate(["Hello, vLLM!"], SamplingParams(max_tokens=8))
print(out[0].outputs[0].text)
PY
```

`enforce_eager` 适合快速 smoke；不加则会触发 `torch.compile` + 图捕获，更贴近生产路径，
但首次编译慢。

### 4.7 速查：一条命令最小验证

```bash
source .venv/bin/activate && \
python -c "import vllm; print('vllm', vllm.__version__, 'from', vllm.__file__)" && \
python -m pytest tests/basic_correctness/test_basic_correctness.py -v -x
```

通过即说明：预编译库加载 OK + editable 安装 OK + 基础推理链路 OK。

---

## 5. 模型/精度相关改动的额外要求

当改动影响模型输出、精度或 serving 行为时（AGENTS.md）：

- 搜索 `tests/evals/` 或用 `vllm bench` 跑模型评估，结果写进 PR。
- 不要等评审人要求才跑。

```bash
# 在新环境上加装评估依赖（按需）
uv pip install -r requirements/test/cuda.in
```

---

## 6. Lint 与类型检查

来源：AGENTS.md（Running linters）。Python 行宽上限 88 字符。

```bash
# 所有 pre-commit 钩子（staged 文件）
pre-commit run

# 全量
pre-commit run --all-files

# 单个钩子
pre-commit run ruff-check --all-files

# mypy（按 CI 方式）
pre-commit run mypy-3.12 --all-files --hook-stage manual
```

文档风格用 Google-style docstrings（`Args:`/`Returns:`/`Raises:`），
不要用 reStructuredText/Sphinx 字段（`:param:`/`:return:`/`:rtype:`）。

---

## 7. 速查：最小完整流程（从零编译 + 验证）

```bash
# --- 1. 系统 + uv ---
sudo apt install -y build-essential python3-dev git curl ccache
curl -LsSf https://astral.sh/uv/install.sh | sh && source $HOME/.local/bin/env
export CUDA_HOME=/usr/local/cuda && export PATH="${CUDA_HOME}/bin:$PATH"

# --- 2. clone + venv ---
git clone https://github.com/vllm-project/vllm.git && cd vllm
uv venv --python 3.12 --seed && source .venv/bin/activate

# --- 3. 构建依赖 ---
uv pip install -r requirements/build/cuda.txt --torch-backend=auto

# --- 4a. 仅 Python 改动（快） ---
VLLM_USE_PRECOMPILED=1 uv pip install --editable . --torch-backend=auto

# --- 4b. 或完整源码编译（含 C++/CUDA） ---
export MAX_JOBS=6
uv pip install -e . --torch-backend=auto

# --- 5. 测试依赖 ---
uv pip install -r requirements/test/cuda.in

# --- 6. 验证 ---
python -c "import vllm; print('vllm', vllm.__version__, 'from', vllm.__file__)"
.venv/bin/python -m pytest tests/basic_correctness/test_basic_correctness.py -v -x
```

---

## 8. 常见问题速查

| 现象 | 原因 / 解决 |
|------|------------|
| import 报 `undefined symbol` / `library not found` | 用了方式 A 却改了 C++/CUDA；切到方式 B 完整编译 |
| 预编译 wheel 找不到 | base commit 刚合并未构建；设 `VLLM_PRECOMPILED_WHEEL_COMMIT=nightly` |
| 编译 OOM | `export MAX_JOBS=1`（WSL 默认只用 50% 内存） |
| `torch.compile` 报错 | 检查 Triton 版本，跑级别 2 sanity check |
| ccache 不命中 | 用 `--no-build-isolation` + `CCACHE_NOHASHDIR=true` |
| CUDA driver 太旧 | 装 `cuda-compat-12-9`，设 `VLLM_ENABLE_CUDA_COMPATIBILITY=1` |
| 非 Linux 上想开发 | `export VLLM_TARGET_DEVICE=empty` 后 `uv pip install -e .`（仅可 import，不可运行） |
| 测试报 `OSError: Can't load the configuration of '<模型>'` | 国内无法访问 HuggingFace；见下方**第 9 章** |
| 设了 `VLLM_USE_MODELSCOPE` 后报 `NotExistError [404] record not found` | 该测试模型（如 `hmellor/tiny-random-*`）ModelScope 无收录；跑测试改用 `HF_ENDPOINT=https://hf-mirror.com`，并 `unset VLLM_USE_MODELSCOPE` |

---

## 9. 国内访问 HuggingFace 方案

国内直连 huggingface.co 会被墙，跑测试时加载模型配置/权重会报
`OSError: Can't load the configuration of '<模型>'`（例如
`tests/basic_correctness/test_cpu_offload.py::test_tower_weight_offloading` 下载
`Qwen/Qwen3.5-0.8B` 失败）。下面按场景给方案，跑测试用方案一即可。

### 方案一：`HF_ENDPOINT` 镜像（推荐，跑测试最省事）

`huggingface_hub` 原生支持 `HF_ENDPOINT` 环境变量，指向社区镜像
[hf-mirror.com](https://hf-mirror.com) 后，所有 `from_pretrained` /
`snapshot_download` 自动走镜像，无需改代码、无需装额外包。

```bash
# 临时（单次测试）
export HF_ENDPOINT=https://hf-mirror.com
.venv/bin/python -m pytest tests/basic_correctness/test_cpu_offload.py -v

# 持久化（写进 ~/.zshrc 或 ~/.bashrc）
echo 'export HF_ENDPOINT=https://hf-mirror.com' >> ~/.zshrc
source ~/.zshrc
```

为什么最适合跑测试：测试常用的小模型（如 `hmellor/tiny-random-LlamaForCausalLM`）
在 HF Hub 上有，hf-mirror 做了全量镜像，一定能下到；而 ModelScope 不一定收录这类
测试用随机模型。

**代理与镜像二选一，不要混用**：若同时设了 `http_proxy`/`https_proxy`/`all_proxy`
和 `HF_ENDPOINT=https://hf-mirror.com`，发往国内镜像的请求会被代理绕到境外节点，
反而变慢甚至失败。用镜像就先取消代理；用代理就别设 `HF_ENDPOINT`，直接访问 HF 官方。

```bash
# 镜像路线（推荐跑测试）：取消代理
unset http_proxy https_proxy all_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY
export HF_ENDPOINT=https://hf-mirror.com
# 代理路线：保留代理，不设镜像
unset HF_ENDPOINT
export https_proxy=http://192.168.3.10:7890 http_proxy=http://192.168.3.10:7890 all_proxy=socks5://192.168.3.10:7890
```

### 方案二：项目内置 ModelScope（适合生产用真实国产模型）

vLLM 在多处做了 ModelScope 适配（`vllm/envs.py`、
`vllm/model_executor/model_loader/weight_utils.py`、
`vllm/transformers_utils/config.py`、`vllm/transformers_utils/repo_utils.py`），
开启后权重下载、config 加载、tokenizer 都走魔搭社区：

```bash
uv pip install modelscope
export VLLM_USE_MODELSCOPE=True   # 注意是 True（见 envs.py 的解析）
```

> 注意：ModelScope 上以国产/热门模型为主，像 `Qwen/Qwen3.5-0.8B` 这类 Qwen 模型大概率有
> 镜像，但 `hmellor/tiny-random-*` 这种测试模型**不一定有**。跑测试优先用方案一。

### 方案三：先镜像下载到本地缓存，再离线跑

第一次用方案一下载完后，模型已缓存到 `~/.cache/huggingface`。之后断网也能跑：

```bash
export HF_ENDPOINT=https://hf-mirror.com   # 第一次：联网下载
.venv/bin/python -m pytest tests/basic_correctness/test_cpu_offload.py -v

# 之后：纯离线，不再联网
export HF_HUB_OFFLINE=1
.venv/bin/python -m pytest tests/basic_correctness/test_cpu_offload.py -v
```

### 方案四：直接用本地路径

已手动下好模型时，把测试里的 `model="<repo>"` 换成本地绝对路径，或指向本地缓存目录：

```bash
export HF_HOME=/path/to/local/hf/cache
# 或在 LLM(model=...) 直接传本地目录
```

### 直接修法（针对本测试）

```bash
export HF_ENDPOINT=https://hf-mirror.com
.venv/bin/python -m pytest \
    tests/basic_correctness/test_cpu_offload.py::test_tower_weight_offloading \
    -v
```

镜像偶发超时再加并发与重试控制：

```bash
export HF_ENDPOINT=https://hf-mirror.com
export HF_HUB_DOWNLOAD_TIMEOUT=60
.venv/bin/python -m pytest tests/basic_correctness/test_cpu_offload.py -v \
    --timeout=300
```

**小结**：跑测试 → 方案一（`HF_ENDPOINT=https://hf-mirror.com`，零成本全量镜像）；
跑国产真实模型生产推理 → 可叠加方案二（`VLLM_USE_MODELSCOPE=True`）。
两者可同时设，互不冲突。

---

国内环境跑 Qwen/Qwen3.5-0.8B 测试命令

```
unset http_proxy https_proxy all_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY
unset HF_HUB_OFFLINE
export VLLM_USE_MODELSCOPE=True
export HF_ENDPOINT=https://hf-mirror.com

# 跑测试
.venv/bin/python -m pytest \
    "tests/basic_correctness/cpu_offload/test_cpu_offload.py::test_tower_weight_offloading[True]" \
    -v -s --timeout=600 
```

## 参考文件索引

| 文件 | 内容 |
|------|------|
| `docs/getting_started/installation/gpu.cuda.inc.md` | CUDA 安装与三种构建方式 |
| `docs/contributing/incremental_build.md` | CMake 增量编译工作流 |
| `docs/usage/troubleshooting.md` | torch.compile sanity check / CUDA 兼容性 |
| `requirements/build/cuda.txt` | 构建依赖版本约束 |
| `requirements/test/cuda.in` | 测试依赖 |
| `AGENTS.md` | 开发工作流、测试规范、lint、提交规范 |
| `tools/generate_cmake_presets.py` | CMakeUserPresets 生成脚本 |
| `build_rust.sh` | Rust 前端构建 |
| `use_existing_torch.py` | 复用已有 PyTorch |
