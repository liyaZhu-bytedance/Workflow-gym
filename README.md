<p align="center">
  <strong>Open WorkflowGym</strong>
</p>

<p align="center">
  面向桌面 GUI 工作流的 Agent 采样与评测框架
</p>

<p align="center">
  <a href="#quick-start">Quick Start</a> •
  <a href="#1-项目背景">背景</a> •
  <a href="#2-安装与运行模式">安装</a> •
  <a href="#3-配置说明">配置</a> •
  <a href="#4-云采样环境搭建">云搭建</a> •
  <a href="#5-常见问题">FAQ</a>
</p>

<p align="center">
  <a href="https://opensource.org/licenses/Apache-2.0">
    <img src="https://img.shields.io/badge/License-Apache%202.0-blue.svg" alt="License">
  </a>
  <img src="https://img.shields.io/badge/python-%3E%3D3.10-blue.svg" alt="Python">
  <img src="https://img.shields.io/badge/platform-Ubuntu%20%7C%20Windows-lightgrey.svg" alt="Platform">
</p>

---

## Quick Start

**30 秒本地验证**（无需云资源、无需模型 Key）：

```bash
git clone https://github.com/<your-org>/open_workflowgym.git
cd open_workflowgym

python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp open_workflowgym/.env.example .env

ENABLE_REAL_CLOUD=false python main.py \
  --dry-run local \
  --no-real-feishu \
  --source-csv-path data/tasks.csv \
  --player dummy \
  --stepNum 1 \
  --max-workers 1 \
  --output-root output/smoke_run
```

也可运行内置冒烟测试：`python -m open_workflowgym.smoke_test`

冒烟通过后，再按下方 [4. 云采样环境搭建](#4-云采样环境搭建) 配置火山 ECS 与 Player Key，开始真实采样。

| 路径 | 适合谁 | 需要准备 |
|------|--------|----------|
| **A. 本地冒烟** | 验证安装与 CSV 流程 | 仅 Python |
| **B. 云采样（推荐）** | 真实 Agent 评测 | 火山 ECS + `RUNNER_*` + 任务 CSV |
| **C. 飞书协作（可选）** | 内部任务表同步 | `APP_ID` / `APP_SECRET` |

---

## 1. 项目背景

**Open WorkflowGym** 是一个面向桌面 GUI 工作流的 Agent **大规模采样与自动评测**框架。它在云桌面（Ubuntu / Windows）上驱动多模态大模型，通过 `pyautogui` 逐步操作真实桌面应用，自动记录轨迹、产物文件与判分结果，适用于工作流任务的批量采样调度、云实例管理、轨迹落盘与多类型自动判分。

### 架构一览

```mermaid
flowchart LR
  CSV[data/tasks.csv] --> Scheduler[main.py 调度器]
  Scheduler --> ECS[火山 ECS 云桌面]
  ECS --> Agent[Player 多模态模型]
  Agent --> Traj[轨迹 / 截图 / 产物]
  Traj --> TOS[VM tosutil 上传 + presign]
  TOS --> Scorer[本地判分]
  Scorer --> Report[report/samples.csv]
```

三层组成：

| 层级 | 角色 | 关键组件 |
|------|------|----------|
| **调度层** | 本地 Linux / macOS | Python 项目、`main.py` |
| **云桌面层** | 火山引擎 ECS 实例池 | Ubuntu / Windows + Cloud Assistant |
| **模型层** | 外部 API | Player（采样 Agent）、Judger（判分，按需） |

### 项目结构

仓库根目录即项目根目录（含 `main.py`），主要模块如下：

| 路径 | 说明 |
|------|------|
| `main.py` | 调度入口 |
| `agents/` | Player 适配器（GPT、Kimi、Claude 等） |
| `envs/` | 火山 ECS 实例池、RunCommand、TOS 上传 |
| `judgers/` | 自动判分（规则、Rubrics、image-llm） |
| `sampler.py` / `scheduler.py` | 单 sample 采样循环与任务调度 |
| `data/tasks.csv` | 开源默认任务表 |
| `tools/` | 实例分配等运维脚本（可选） |
| `tests/` | 单元测试与 E2E |
| `open_workflowgym/.env.example` | 环境变量模板 |

更多设计细节见 [ARCHITECTURE_RISKS.md](ARCHITECTURE_RISKS.md)，贡献流程见 [CONTRIBUTING.md](CONTRIBUTING.md)。

---

## 2. 安装与运行模式

### 环境要求

- Python >= 3.10
- 推荐 Linux / macOS 作为调度机
- 云采样需：火山引擎 ECS、`AK` / `SK`
- **开源默认无需飞书**：任务从本地 CSV 读取，结果写入本地 CSV

### 安装

```bash
git clone https://github.com/<your-org>/open_workflowgym.git
cd open_workflowgym

python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

cp open_workflowgym/.env.example .env
# 编辑 .env 后统一加载
source .env
```

### 三种运行模式

| 模式 | 典型命令 | 需要云 | 说明 |
|------|----------|--------|------|
| **本地冒烟** | `ENABLE_REAL_CLOUD=false` + `--dry-run local --no-real-feishu --player dummy` | 否 | 验证调度与落盘，不调用模型 |
| **本地 CSV 云采样** | `ENABLE_REAL_CLOUD=true` + 本地 `data/tasks.csv` | 是 | **开源推荐默认路径** |
| **飞书模式（可选）** | 配置 `APP_ID` / `APP_SECRET` + `SOURCE_FEISHU_URL` | 是 | 从飞书读任务、写结果；可加 `--no-real-feishu` 强制本地 CSV |

### 数据读写（开源默认）

| 方向 | 路径 | 说明 |
|------|------|------|
| 任务输入 | `data/tasks.csv` | 可通过 `SOURCE_CSV_PATH` 覆盖 |
| 结果输出 | `output/<run>/report/samples.csv` | 每次运行自动生成 |

---

## 3. 配置说明

以下配置块可全部写在 `.env` 中，执行 `source .env` 一次加载。完整变量见 [open_workflowgym/.env.example](open_workflowgym/.env.example)。

### 3.1 火山引擎（云采样必填）

```bash
AK=your_volcengine_access_key
SK=your_volcengine_secret_key
VOLCENGINE_REGION=cn-guangzhou
ENABLE_REAL_CLOUD=true   # 本地冒烟时设为 false
```

在 [API 访问密钥管理](https://console.volcengine.com/iam/keymanage/) 申请。Windows 任务额外需要：

```bash
WINDOWS_LOGIN_PASSWORD=your_password
```

### 3.2 Player（采样 Agent）— 统一 `RUNNER_*` 三元组

所有 Player（`gpt`、`kimi` 等）**优先读取同一套环境变量**，切换 `--player` 时通常只需改 `PLAYER` 和对应模型参数：

| 配置项 | 环境变量 | 说明 |
|--------|----------|------|
| API Key | `RUNNER_API_KEY` | 必填（各 Player 另有专属变量作兼容回退） |
| 接口地址 | `RUNNER_BASE_URL` | 可选，各 Player 有内置默认值 |
| 模型名 | `RUNNER_MODEL` | 可选，各 Player 有内置默认值 |

```bash
PLAYER=gpt
RUNNER_API_KEY=sk-your-api-key
# RUNNER_BASE_URL=https://api.openai.com/v1
# RUNNER_MODEL=gpt-5.4
```

切换 Kimi 示例：

```bash
PLAYER=kimi
RUNNER_API_KEY=sk-your-moonshot-key
# RUNNER_BASE_URL=https://api.moonshot.cn/v1
# RUNNER_MODEL=kimi-k2.6
```

| `--player` | 默认 Model | 兼容环境变量（回退） |
|------------|------------|----------------------|
| `gpt`（**CLI 默认**） | `gpt-5.4` | `OPENAI_API_KEY` |
| `gpt54-mini-native` | `gpt-5.4-mini` | `OPENAI_API_KEY` |
| `kimi` / `kimi_native` | `kimi-k2.6` | `MOONSHOT_API_KEY`、`KIMI_NATIVE_MODEL` |
| `kimi2.6` | `kimi-k2.6` | 同上（独立适配器） |
| `claude` | `anthropic/claude-sonnet-4.6` | `CLAUDE_API_KEY` |
| `gemini3.1` | `gemini-3-pro-preview-new` | `GEMINI31_API_KEY` |
| `seed2p0` / `seed_2p0` / `seed2.0` | `doubao-seed-2-1-pro-260628` | `SEED2P0_API_KEY`、`ARK_API_KEY` |
| `psm` / `psm2.1` | `Seed-XGPT-Server-Model` | 内网 PSM，需 `--psm` |
| `dummy` | — | 仅 dry-run |

内网环境如需代理，可设置 `OPENAI_PROXY` 或 `GPT_NATIVE_PROXY`（默认为空，不使用代理）。

### 3.3 裁判员（Judger）— 按需配置

采样结束后按任务 CSV 的 `metric_type` 自动选择判分策略。凡涉及 LLM 判分（`规则判分+LLM-as-judge`、`Rubrics`、`image-llm` 等），需配置：

| 配置项 | 环境变量 |
|--------|----------|
| API Key | `JUDGE_API_KEY` |
| 接口地址 | `JUDGE_BASE_URL` |
| 模型名 | `JUDGE_MODEL` |

```bash
JUDGE_API_KEY=your_judge_api_key
JUDGE_BASE_URL=https://your-compatible-endpoint/v1
JUDGE_MODEL=your-judge-model-name
```

网关需兼容 **OpenAI Chat Completions**。若任务全部为 `规则判分`（产物文件字节比对），**无需配置 Judger**。

### 3.4 TOS 产物上传（文件类任务推荐）

Agent 在 VM 上产出的文件，由 **VM 侧 tosutil** 上传并 presign 为 24 小时 HTTPS URL；**本地判分通过 urllib 下载**，正常情况**调度机无需安装 tosutil**。

```
VM 产物 → tosutil cp → tosutil presign → HTTPS URL
                                              ↓
本地判分 ← urllib 下载到 production/  ←────────┘
```

| 配置项 | 环境变量 | 说明 |
|--------|----------|------|
| 上传桶 | `TOS_UPLOAD_BUCKET` | 默认 `tos://open-workflowgym-sample` |
| TOS 凭证 | `TOS_AK` / `TOS_SK` | 未设时回退 `AK` / `SK` |
| 公网访问 | `TOS_PUBLIC_HOST` | 可选；未 presign 成功时生成 HTTPS 直链 |
| Presign 有效期 | `TOS_PRESIGN_VALID_PERIOD` | 默认 `24h` |
| 连通性探测前缀 | `TOS_VERIFY_PREFIX` | 默认 `.workflowgym-healthcheck/`，init 时 `tosutil ls` 仅列该前缀，避免扫全桶 |

仅当 URL 仍为 `tos://...`（presign 失败且无公网 host）时，本地判分才会尝试调用 `tosutil cp` 兜底。

### 3.5 采样参数

```bash
SOURCE_CSV_PATH=data/tasks.csv
SAMPLE_NUM=1          # 每个任务采样次数
STEP_NUM=400          # 每个 sample 最大步数
MAX_WORKERS=2         # 跨任务并发，建议 ≤ ECS 实例数
```

### 3.6 任务 CSV 列说明

| 列名 | 说明 |
|------|------|
| `id` | 任务唯一 ID |
| `题目` | 任务 prompt |
| `image` | ECS 镜像 ID（Ubuntu 填 `*_gui` 采样镜像） |
| `输入文件路径` / `输出文件路径` | VM 内目录 |
| `final_answer文件名` | 期望产物文件名 |
| `metric_type` | `规则判分` / `Rubrics` / `image-llm` 等 |
| `启动方式` | 如 `桌面` |

示例见 [data/tasks.csv](data/tasks.csv)。

### 3.7 输出目录

```
output/sample_gpt_1_step_400_202607151733/
├── report/
│   └── samples.csv          # 结果汇总
├── workflowgym_state.db     # 断点恢复
├── 258/
│   └── sample_1/
│       ├── traj.json        # 完整轨迹
│       ├── score.json       # 判分结果
│       ├── step_*.jpg       # 逐步截图
│       ├── production/      # Agent 产物（判分下载）
│       └── gt_production/   # 标准答案（判分下载）
└── ...
```

按一次 `Ctrl+C` 有序退出；连续两次强制中断。

### 3.8 常用命令行参数

| 参数 | 环境变量 | 说明 | 默认值 |
|------|----------|------|--------|
| `--source-csv-path` | `SOURCE_CSV_PATH` | 本地任务 CSV | `data/tasks.csv` |
| `--id` | `WORKFLOWGYM_TASK_ID` | 指定任务 ID（可空格分隔多个） | 全部 |
| `--sampleNum` | `SAMPLE_NUM` | 每任务采样次数 | `1` |
| `--stepNum` | `STEP_NUM` | 每 sample 最大步数 | `400` |
| `--player` | `PLAYER` | 采样 Agent | `gpt` |
| `--judger` | `JUDGER` | 判分元数据标识 | `gemini3.1` |
| `--max-workers` | `MAX_WORKERS` | 跨任务并发 | `2` |
| `--instance-ids` | `INSTANCE_IDS` | 指定 ECS 实例 | 自动发现 |
| `--no-real-feishu` | — | 强制本地 CSV | — |
| `--dry-run` | `DRY_RUN` | 本地验证 | — |
| `--resume-running-samples` | `RESUME_RUNNING_SAMPLES` | 断点续跑 | `false` |

---

## 4. 云采样环境搭建

以下从**零**搭建可跑通批量云采样的环境。命令均在含 `main.py` 的项目根目录执行。

### 推荐首次运行流程

```bash
# 1. 本地冒烟（必做）
ENABLE_REAL_CLOUD=false python main.py \
  --dry-run local --no-real-feishu --player dummy --stepNum 1 --max-workers 1

# 2. 单任务试跑（示例：Kimi）
python main.py --id 258 --player kimi --stepNum 40 --max-workers 1

# 3. 确认 output/.../score.json 正常后批量采样
python main.py --player kimi --max-workers 2
```

### 4.1 配置 `.env`

**最小配置（规则判分 + 单任务）：**

```bash
AK=...
SK=...
VOLCENGINE_REGION=cn-guangzhou
ENABLE_REAL_CLOUD=true

PLAYER=gpt
RUNNER_API_KEY=sk-your-api-key

SOURCE_CSV_PATH=data/tasks.csv
MAX_WORKERS=1
```

**完整配置（LLM 判分 + 文件产物上传）：** 在最小配置基础上增加 `JUDGE_*` 与 `TOS_UPLOAD_BUCKET`（`TOS_AK`/`TOS_SK` 可与 `AK`/`SK` 相同）。

### 4.2 创建 ECS 实例池

实例名前缀需与目标 OS 一致（可通过 `WORKFLOWGYM_INSTANCE_NAME_PREFIX` 配置，默认 `open_workflow_gym`）：

| 平台 | 实例名示例 | 说明 |
|------|------------|------|
| Ubuntu | `open_workflow_gym_ubuntu` | Linux GUI 任务 |
| Windows | `open_workflow_gym_windows` | Windows 桌面任务 |

```bash
# Ubuntu（示例：5 台）
python tools/allocate_instances.py \
  --instance-name open_workflow_gym_ubuntu \
  --count 5 \
  --wait

# Windows（需密码）
python tools/allocate_instances.py \
  --instance-name open_workflow_gym_windows \
  --count 5 \
  --password your_password \
  --wait
```

要求：状态 **RUNNING**、已安装 **Cloud Assistant**、`--max-workers` ≤ 实例数。

已有实例可直接指定：`INSTANCE_IDS=i-aaa,i-bbb`

### 4.3 （Ubuntu）烘焙 GUI 采样镜像

Ubuntu 任务 CSV 的 `image` 列需 `*_gui` 后缀镜像。若仅有基础镜像：

```bash
bash scripts/run_gui_image_bake_background.sh
tail -f output/gui_image_bake/gui_bake_latest.log
```

烘焙完成后将 `image-xxx_gui` 写入任务 CSV。Windows 任务通常使用预置镜像，可跳过。

### 4.4 启动采样

**本地冒烟：**

```bash
ENABLE_REAL_CLOUD=false python main.py \
  --dry-run local \
  --no-real-feishu \
  --player dummy \
  --stepNum 1 \
  --max-workers 1 \
  --output-root output/smoke_run
```

**云环境批量采样：**

```bash
python main.py --stepNum 400 --sampleNum 1 --player gpt --max-workers 2
```

**指定单任务调试：**

```bash
python main.py \
  --id 258 \
  --stepNum 40 \
  --player gpt \
  --instance-ids i-xxxxxxxx \
  --output-root output/task258_run
```

### 4.5 检查清单

1. [ ] `pip install -r requirements.txt` 完成
2. [ ] `.env` 已配置并 `source .env`
3. [ ] ECS 实例 RUNNING，Cloud Assistant 可用
4. [ ] 任务 CSV `image` 列有效（Ubuntu 需 `*_gui`）
5. [ ] `--max-workers` ≤ 实例数；Windows 已设 `WINDOWS_LOGIN_PASSWORD`
6. [ ] 本地冒烟通过
7. [ ] LLM 判分任务已配 `JUDGE_*`；文件产物任务 VM presign 正常

---

## 5. 常见问题

| 现象 | 可能原因 | 处理 |
|------|----------|------|
| `production保存失败: tosutil`（本地） | `tos_url` 为 `tos://` 而非 HTTPS | 检查 VM presign；配置 `TOS_PUBLIC_HOST`；确认 VM 已产出文件 |
| `no model permission`（Judger 401） | `JUDGE_*` 无权限或模型名错误 | 配置有效 `JUDGE_API_KEY` / `JUDGE_MODEL`；或先用 `规则判分` 任务验证 |
| 采样 `last_step=2` 就结束 | Agent 过早返回 `finished` | 增大 `--stepNum`；换 Player；检查 prompt |
| VM 初始化很慢（~10min） | 快速初始化失败，回退完整初始化 | 镜像预装 tosutil；检查 Cloud Assistant |
| `unknown agent` | `PLAYER` 名称错误或 `.env` 未加载 | 确认 `source .env`；Player 名见上表 |
| Windows 任务无法登录 | 缺桌面密码 | 设置 `WINDOWS_LOGIN_PASSWORD` |
| 全部任务 score=0 | 产物缺失或判分服务不可用 | 看 `score.json` 的 `error` / `vm_output_listing` |

### 可选工具说明

`tools/` 下部分脚本（飞书表格同步、镜像 ID 匹配等）面向内部协作流程，**开源用户可忽略**，不影响核心采样链路。

`docs/xdotool_pyautogui_mapping.md` 记录了 xdotool 与 pyautogui 动作映射，供 Ubuntu 桌面任务调试参考。

---

## License

Apache License 2.0 — 详见 [LICENSE](LICENSE)。
