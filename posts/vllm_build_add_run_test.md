# vLLM 在 Ubuntu 24.04 x86_64 上的编译与验证流程

> 目标环境：Ubuntu 24.04.1 LTS x86_64 + NVIDIA CUDA GPU。
> 内容基于本仓库 `docs/getting_started/installation/gpu.cuda.inc.md`、`docs/contributing/incremental_build.md`、`docs/usage/troubleshooting.md`、`AGENTS.md` 整理。

## 关键约束

| 项 | 要求 | 备注 |
|----|------|------|
| Python | 3.10–3.14（推荐 3.12） | `requires-python: >=3.10,<3.15` |
| PyTorch | `torch==2.13.0` | CUDA 版本固定，不要用系统 pip |
| CUDA | 默认 12.9（也提供 12.8 / 13.0 变体） | Blackwell GPU 需 ≥ 12.8 |
| GCC | ≥ 11.3 | Ubuntu 24.04 默认 GCC 13.x 已满足 |
| 构建工具 | CMake ≥ 3.26.1 + Ninja + ccache | 见 `requirements/build/cuda.txt` |
| 包管理 | **必须 uv** | 禁止系统 `python3` / 裸 `pip` |

---

## 1. 环境准备与构建

```bash
# --- 系统依赖 ---
sudo apt install -y build-essential python3-dev git curl ccache
export CUDA_HOME=/usr/local/cuda
export PATH="${CUDA_HOME}/bin:$PATH"

# --- uv + venv ---
curl -LsSf https://astral.sh/uv/install.sh | sh && source $HOME/.local/bin/env
git clone https://github.com/vllm-project/vllm.git && cd vllm
uv venv --python 3.12 --seed && source .venv/bin/activate

# --- 构建依赖 ---
uv pip install -r requirements/build/cuda.txt --torch-backend=auto
```

### 三种构建方式

| 方式 | 命令 | 适用 |
|:----:|------|------|
| **A — 仅 Python**（最快） | `VLLM_USE_PRECOMPILED=1 uv pip install --editable . --torch-backend=auto` | 不改 C++/CUDA，复用预编译 `.so` |
| **B — 完整源码** | `uv pip install -e . --torch-backend=auto` | 改了 `csrc/` 下的 C++/CUDA kernel |
| **C — Docker** | `DOCKER_BUILDKIT=1 docker build . --target vllm-openai --tag vllm/vllm-openai --file docker/Dockerfile` | 无 GPU 的 CI 或需可复现产物 |

> 方式 A 工作原理：查找当前分支在 main 上的 base commit → 下载该 commit 的预编译 wheel → 复用其编译产物（`_C.abi3.so` 等）和 `vllm-rs` 二进制。若改了 `csrc/` 却仍用方式 A，import 时会报 `undefined symbol` / `library not found`。

方式 B 内存有限时：`export MAX_JOBS=6`；配合 ccache 加速重复构建（必须用 `--no-build-isolation`）：

```bash
CCACHE_NOHASHDIR="true" uv pip install --no-build-isolation -e .
```

方式 A 找不到 base commit 的预编译 wheel 时，设 `VLLM_PRECOMPILED_WHEEL_COMMIT=nightly` 选最近已构建的 main commit。

---

## 2. 增量编译（频繁改 kernel 时推荐）

先用方式 A 做一次 editable 安装，再用 CMake 增量编译，避免每次跑完整 `pip install`：

```bash
# 初次：预编译 wheel 做 editable 安装
VLLM_USE_PRECOMPILED=1 uv pip install -U -e . --torch-backend=auto

# 生成 CMake 配置（首次或切换分支后）
python tools/generate_cmake_presets.py

# 增量编译 + 装回源码树（之后每次改完 csrc/ 只跑这条）
cmake --preset release
cmake --build --preset release --target install

# 顽固错误或切换分支后清理重建
rm -rf cmake-build-release && cmake --preset release && cmake --build --preset release --target install
```

---

## 3. 验证流程（从快到慢）

### 级别 1 — 库加载 + editable 生效（秒级）

```bash
python -c "import vllm; print('vllm', vllm.__version__); print(vllm.__file__)"
```

- 不报 `undefined symbol` / `library not found` → 预编译 `.so` 加载正常。
- `__file__` 指向源码目录（而非 site-packages） → editable 安装生效。
- 临时在 Python 文件加一行 print，再 import 一次，能看到输出 → 走的是本地源码而非缓存 wheel。

### 级别 2 — torch.compile sanity check（秒级）

```bash
python - << 'PY'
import torch

@torch.compile
def f(x):
    x = x + 1; x = x * 2; x = x.sin()
    return x

x = torch.randn(4, 4).cuda()
print(f(x))
print("sanity check is successful!")
PY
```

看到 `sanity check is successful!` 即 Triton/Inductor 链路正常。

### 级别 3 — 相关单元测试（分钟级）

```bash
uv pip install -r requirements/test/cuda.in   # 首次装测试依赖
.venv/bin/python -m pytest tests/<你的模块>/test_xxx.py -v
```

### 级别 4 — 端到端小模型推理（分钟级）

```bash
python - << 'PY'
from vllm import LLM, SamplingParams
llm = LLM(model="Qwen/Qwen3-0.6B", enforce_eager=True)
out = llm.generate(["Hello, vLLM!"], SamplingParams(max_tokens=8))
print(out[0].outputs[0].text)
PY
```

`enforce_eager=True` 关闭 CUDA graph，更快冷启动，适合快速 smoke。

> 改动影响模型输出/精度/serving 时（AGENTS.md 要求）：搜索 `tests/evals/` 或用 `vllm bench` 跑模型评估，结果写进 PR，不要等评审人要求才跑。

---

## 4. Lint 与提交

Python 行宽上限 88 字符。文档风格用 Google-style docstrings（`Args:`/`Returns:`/`Raises:`），不要用 reST 字段。

```bash
pre-commit run                      # staged 文件
pre-commit run --all-files          # 全量
pre-commit run mypy-3.12 --all-files --hook-stage manual   # mypy（按 CI 方式）
```

---

## 5. 常见问题速查

| 现象 | 原因 / 解决 |
|------|------------|
| import 报 `undefined symbol` / `library not found` | 用了方式 A 却改了 C++/CUDA；切到方式 B 完整编译 |
| 预编译 wheel 找不到 | base commit 刚合并未构建；设 `VLLM_PRECOMPILED_WHEEL_COMMIT=nightly` |
| 编译 OOM | `export MAX_JOBS=1`（WSL 默认只用 50% 内存） |
| `torch.compile` 报错 | 检查 Triton 版本，跑级别 2 sanity check |
| ccache 不命中 | 用 `--no-build-isolation` + `CCACHE_NOHASHDIR=true` |
| CUDA driver 太旧 | 装 `cuda-compat-12-9`，设 `VLLM_ENABLE_CUDA_COMPATIBILITY=1` |
| 非 Linux 上想开发 | `export VLLM_TARGET_DEVICE=empty` 后 `uv pip install -e .`（仅可 import，不可运行） |
| Rust `resolver setting 3 is not valid` | apt 装的旧 rustc（<1.85）不支持 `resolver="3"`（2024 edition）；装 rustup 自动读 `rust-toolchain.toml`（指定 1.95） |
| 测试报 `OSError: Can't load the configuration of '<模型>'` | 国内无法访问 HuggingFace；见下方**第 6 章** |
| 测试无输出/看起来卡住 | pytest 捕获了 server 子进程日志 + torch.compile 编译慢；加 `-s` 看实时日志，见**第 7 章** |
| `LocalEntryNotFoundError: Cannot find an appropriate cached snapshot` | 设了 `HF_HUB_OFFLINE=1` 但模型未缓存；先联网下载，下完再开离线 |
| `Error retrieving safetensors: timed out` | `try_get_safetensors_metadata`（`config.py:1097`）未分流 ModelScope，强制走 HF API 超时 ~22s；超时后 `return None` 不致命测试可继续；或 `HF_HUB_OFFLINE=1` 直接跳过 |
| `ImportError: Using SOCKS proxy, but the 'socksio' package is not installed` | 环境残留 `all_proxy=socks5://...`，httpx2 缺 `socksio`；`unset all_proxy ALL_PROXY` 改用 HTTP 代理，或 `uv pip install socksio` |

---

## 6. 国内访问 HuggingFace 方案

国内直连 huggingface.co 会被墙，跑测试时加载模型配置/权重会报
`OSError: Can't load the configuration of '<模型>'`。

### 方案一：`HF_ENDPOINT` 镜像（推荐，跑测试最省事）

`huggingface_hub` 原生支持 `HF_ENDPOINT`，指向社区镜像 [hf-mirror.com](https://hf-mirror.com) 后自动走镜像，无需改代码、无需装额外包。测试常用的小模型（如 `hmellor/tiny-random-*`）在 HF Hub 上有全量镜像，一定能下到。

```bash
export HF_ENDPOINT=https://hf-mirror.com
.venv/bin/python -m pytest tests/basic_correctness/test_cpu_offload.py -v
```

下载完后模型缓存到 `~/.cache/huggingface`，之后可断网跑：`export HF_HUB_OFFLINE=1`。

### 方案二：ModelScope（适合生产用真实国产模型）

vLLM 在多处做了 ModelScope 适配（`vllm/envs.py`、`weight_utils.py`、`config.py`、`repo_utils.py`），开启后权重下载、config 加载、tokenizer 都走魔搭社区。Qwen 系模型在 ModelScope 必有镜像。

```bash
uv pip install modelscope
export VLLM_USE_MODELSCOPE=True   # 注意是 True（见 envs.py 的解析）
```

> 注意：ModelScope 上以国产/热门模型为主，`hmellor/tiny-random-*` 这种测试模型**不一定有**。跑测试优先用方案一。

### 代理路线：HTTP 代理直连 HF 官方（已验证绕境外成功）

**代理与镜像二选一，不要混用**：若同时设了代理和 `HF_ENDPOINT`，发往国内镜像的请求会被代理绕到境外，反而变慢甚至失败。

```bash
# 镜像路线（推荐跑测试）：取消代理
unset http_proxy https_proxy all_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY
export HF_ENDPOINT=https://hf-mirror.com

# 代理路线：只用 HTTP 代理，不设镜像，直连 HF 官方
unset HF_ENDPOINT
export http_proxy=http://192.168.3.10:7890 https_proxy=http://192.168.3.10:7890
# 注意：不要用 all_proxy=socks5://...（见下方踩坑）
```

> **踩坑：socks5 代理导致 ImportError（已验证）**
>
> 代理路线最初用了 `all_proxy=socks5://192.168.3.10:7890`，结果测试在 1.63s 内快速失败，报
> `ImportError: Using SOCKS proxy, but the 'socksio' package is not installed`。原因：新版
> `huggingface_hub` 底层用 `httpx2`，其 `_init_proxy_transport` 检测到 socks5 scheme 时尝试
> `import socksio`，该包未安装即抛 `ImportError`，被 transformers 转为
> `OSError: Can't load the configuration`。
>
> **解法**：代理路线只用 `http_proxy`/`https_proxy`（HTTP 代理），`unset all_proxy ALL_PROXY`
> 清掉 socks5。改用纯 HTTP 代理后，代理直连 HF 官方（`huggingface.co`）成功绕境外，
> 模型下载与测试均通过。若必须用 socks5 代理，则 `uv pip install socksio` 补依赖。

### 实战：`test_tower_weight_offloading`（已验证通过）

该测试用 `vllm_runner`（直接 `LLM` 类）+ `Qwen/Qwen3.5-0.8B`（1.6G 真实权重），需先下载模型再跑：

```bash
# 1. 用 ModelScope 下载模型（Qwen 是阿里自家模型，ModelScope 必有）
uv pip install modelscope
.venv/bin/python -c \
    "from modelscope import snapshot_download; print(snapshot_download('Qwen/Qwen3.5-0.8B'))"
# 验证：du -sh ~/.cache/modelscope/models/Qwen--Qwen3.5-0.8B/ 应约 1.7G

# 2. 跑测试（模型走 ModelScope 缓存 + HF 镜像兜底）
unset http_proxy https_proxy all_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY
unset HF_HUB_OFFLINE
export VLLM_USE_MODELSCOPE=True
export HF_ENDPOINT=https://hf-mirror.com
.venv/bin/python -m pytest \
    "tests/basic_correctness/cpu_offload/test_cpu_offload.py::test_tower_weight_offloading[True]" \
    -v -s --timeout=600
```

> **已知超时（不致命）**：`try_get_safetensors_metadata`（`config.py:1097`）未检查
> `VLLM_USE_MODELSCOPE`，仍走 HF API 获取权重元数据，国内会超时 ~22s。但该函数
> `except Exception: return None` 兜底，超时后返回 `None` 继续运行，**不影响测试通过**。
> 想跳过这 22s 可加 `export HF_HUB_OFFLINE=1`（模型已在 ModelScope 缓存，不受影响）。

---

## 7. 测试运行与卡死排查

### 为什么慢

vLLM 的基础正确性测试（如 `tests/basic_correctness/` 下的 `compare_two_settings`）不是直接调 `LLM` 类，而是用 `RemoteOpenAIServer` 启动**真实的 `vllm serve` 子进程**：每组参数启 2 个 server（baseline + 待测），每个含 spawn worker → `torch.compile` 编译 → CUDA graph 捕获 → 健康检查。参数组合多时最多 16 次 server 启动。

测试默认用 **dummy 权重**（`VLLM_TEST_FORCE_LOAD_FORMAT`），所以卡住和模型大小无关，卡在 **torch.compile 编译**（小模型首次也要 1–3 分钟/次）。server 日志透传到 `sys.stdout`，但 **pytest 默认捕获输出**，不加 `-s` 看不到任何日志 → 看起来像卡死。

### 排查四步

**第一步：加 `-s` 看实时日志（最关键）**

```bash
export HF_HUB_OFFLINE=1            # 模型已下好，离线复用
export VLLM_LOGGING_LEVEL=INFO     # server 日志更详细
.venv/bin/python -m pytest tests/basic_correctness/cpu_offload/test_cpu_offload.py \
    -v -s --timeout=600
```

日志在动 = 没卡死，是正常编译。

**第二步：先 collect 看准确用例名，再单条精准运行**

`-k` 表达式容易写错（parametrize 的 id 用参数值如 `False-False`，不是参数名），常导致全被 deselect。先 collect：

```bash
.venv/bin/python -m pytest tests/basic_correctness/cpu_offload/test_cpu_offload.py \
    --collect-only -q
```

挑一个用**全名**跑（中括号必须用引号包裹）：

```bash
.venv/bin/python -m pytest \
    "tests/basic_correctness/cpu_offload/test_cpu_offload.py::test_cpu_offload[False-False]" \
    -v -s --timeout=600
```

**第三步：另开终端盯 GPU**

```bash
watch -n1 nvidia-smi
```

- 显存占用 + GPU util 波动 → 正在编译，没卡死，耐心等。
- 显存占满但 util 0 → 上个 server 的 worker 没退出，卡在 `_wait_for_gpu_memory_release`；`pkill -9 vllm` 清残留。
- 显存一直 0 → server 没起来，看 `-s` 日志报错。

**第四步：卡住位置对照**

| 卡在（看 `-s` 日志） | 原因 | 对策 |
|----------------------|------|------|
| `Compiling...` 很久 | torch.compile 首次编译 | 正常，首次 1–3 分钟/次 |
| `Capturing CUDA graph` | 图捕获慢 | 该组合没加 `--enforce-eager`；先跑带 enforce_eager 的组合 |
| `Server failed to start in time` | 健康检查超时 | 查端口/GPU，加大 `max_wait_seconds` |
| GPU 内存释放等待 | 上个 server 没退出 | `pkill -9 vllm` 后重跑 |
| `subprocess.run` 跑 bench 无输出 | `capture_output=True` 吞了输出 | 直接跑 `vllm bench startup` 命令绕开 |

---

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
| `build_rust.sh` | Rust 前端构建（`./build_rust.sh` 或 `--debug`） |
| `use_existing_torch.py` | 复用已有 PyTorch 安装 |
