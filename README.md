

> **[中文化] watermarks-remover**
>
> 此项目是 [guillaumemeyer/watermarks-remover](https://github.com/guillaumemeyer/watermarks-remover) 的中文翻译版本。
> - 原项目 Stars: 11864
> - 主语言: Python
> - 许可证: MIT
> - 翻译日期: 2026-08-17
> - 原始 README: [README_en.md](README_en.md)
>
> 如有翻译不准确之处，欢迎提 Issue 或 PR。

---


---

_ _ _ ____ ___ ____ ____ _  _ ____ ____ _  _ ____    ____ ____ _  _ ____ _  _ ____ ____
| | | |__|  |  |___ |__/ |\/| |__| |__/ |_/  [__  __ |__/ |___ |\/| |  | |  | |___ |__/
|_|_| |  |  |  |___ |  \ |  | |  | |  \ | \_ ___]    |  \ |___ |  | |__|  \/  |___ |  \
```

# watermarks-remover

<!-- logo: figlet -d .figlet -f cybermedium -w 120 "watermarks-remover" -->

[![CI](https://github.com/guillaumemeyer/watermarks-remover/actions/workflows/ci.yml/badge.svg)](https://github.com/guillaumemeyer/watermarks-remover/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/guillaumemeyer/watermarks-remover)](https://github.com/guillaumemeyer/watermarks-remover/releases)
[![Stars](https://img.shields.io/github/stars/guillaumemeyer/watermarks-remover)](https://github.com/guillaumemeyer/watermarks-remover/stargazers)
[![Forks](https://img.shields.io/github/forks/guillaumemeyer/watermarks-remover)](https://github.com/guillaumemeyer/watermarks-remover/forks)

Agent 技能 + 基于 stdlib 的 Python 服务，用于从文本和文件中去除**多供应商 AI 来源标记**——适用于保护**您拥有**内容的隐私和卫生。该技能是一个轻量级客户端：它通过 HTTP 驱动底层机制，因此 Agent 主机无需安装 Python。

| 层级 | 目标 | 方式 |
| --- | --- | --- |
| **A** | 不可见 Unicode、特殊空格、双向文本、标签字符 | 确定性 Python 脚本 |
| **B** | 统计（token 采样）文本水印 | Agent 重写 + 可选 `rewrite_text.py` 钩子 |
| **文件** | C2PA / EXIF / XMP / 文档属性 | PNG、JPEG、WebP、BMP、GIF、TIFF、SVG、PDF、DOCX、EPUB、ODT、HTML、Markdown |

供应商 / 生态系统（类级别）：**Claude**、**Gemini / SynthID-Text**、**OpenAI** 来源表面、**open-LLM** Kirchenbauer 风格标记。

**最新版本：** [v0.5.0](https://github.com/guillaumemeyer/watermarks-remover/releases/tag/v0.5.0)

技能路径： [`skills/remove-ai-marks/`](skills/remove-ai-marks/)  
服务路径： [`service/`](service/)  
（迁移：原为 `remove-claude-marks`；斜杠别名 `/remove-claude-marks` 仍有文档记录）

## 安装（Agent 技能）

该技能**不包含代码**——它通过 HTTP 调用服务。安装该技能（仅 Markdown）并启动服务，如果服务地址不是 `http://127.0.0.1:8765`，则设置 `WATERMARKS_SERVICE_URL`。

```bash
# Grok Build / 项目本地
mkdir -p .grok/skills
ln -sfn "$(pwd)/skills/remove-ai-marks" .grok/skills/remove-ai-marks

# 用户全局 Grok
mkdir -p ~/.grok/skills
ln -sfn "$(pwd)/skills/remove-ai-marks" ~/.grok/skills/remove-ai-marks
```

使用 `/remove-ai-marks` 调用，或要求“去除 AI 水印 / C2PA / Claude 标记 / SynthID 类文本”。

### 可选的 Cursor 纯文本技能

[`skills/clean-user-facing-text/`](skills/clean-user-facing-text/) 是一个独立的 Cursor 技能，适用于授权的稿件、文档和网页文案。它不包含图像、C2PA、服务和外部模型工具。

将其安装到 `~/.cursor/skills/clean-user-facing-text`：

```bash
python3 install_skill.py
```

在 Windows 上，使用 `py install_skill.py`。为 macOS/Linux shell 提供了 `install-skill.sh` 包装脚本。除非传入 `--force`，否则现有安装会被保留；替换会先暂存，之前的安装会以唯一命名的备份保留。

技能调用由模型选择。明确采用此工作流的项目也可以复制可选规则：

```bash
mkdir -p /path/to/project/.cursor/rules
cp integrations/cursor/clean-user-facing-text.mdc \
  /path/to/project/.cursor/rules/clean-user-facing-text.mdc
```

对于所有项目，请将相同的指令放入 Cursor **用户规则**中。规则可以提高一致性，但仍然是模型指令；Cursor 不会为最终聊天响应提供确定性的发送前过滤器。

### 启动服务

最快的方式是本地 HTTP 服务器（仅需 Python 3.10+ 标准库——无依赖、无 Docker）：

```bash
make serve                 # http://127.0.0.1:8765
# 或直接运行：
python3 service/scripts/server.py --host 127.0.0.1 --port 8765
```

如需完整基础设施（核心 + 可选工具/重型后端），请参阅下面的 [Docker / compose](#docker--compose)。

可选系统工具（存在时自动使用——已预装在核心 Docker 镜像中）：

| 工具 | 作用 |
| --- | --- |
| [`c2patool`](https://github.com/contentauth/c2pa-rs/tree/main/cli) | 检查 C2PA 清单 |
| [`exiftool`](https://exiftool.org/) | 残留元数据剥离（尤其是 **PDF**） |
| [`qpdf`](https://qpdf.sourceforge.io/) | 结构化 PDF 重建——**必需**用于真正的 PDF 剥离（见下文） |

核心脚本仅需 **Python 3.10+** 标准库。B 层模型调用是可选的。

## 快速使用（脚本）

```bash
SCRIPTS=service/scripts

# 统一检查 / 清理
python3 "$SCRIPTS/inspect_file.py" draft.md
python3 "$SCRIPTS/clean_file.py" draft.md -o draft.cleaned.md
python3 "$SCRIPTS/clean_file.py" photo.png -o photo.cleaned.png
python3 "$SCRIPTS/clean_file.py" notes.docx -o notes.cleaned.docx

# 文本 A 层
python3 "$SCRIPTS/inspect_text.py" draft.md
python3 "$SCRIPTS/clean_text.py" draft.md -o draft.cleaned.md --stats

# B 层重写钩子（默认：仅打印提示——无需模型）
python3 "$SCRIPTS/rewrite_text.py" draft.md --backend print-prompt --strength paraphrase
# 可选本地 Ollama（默认仅回环——远程端点需要
# WATERMARKS_REWRITE_ALLOW_REMOTE=1 或 --allow-remote）：
# WATERMARKS_REWRITE_BACKEND=ollama WATERMARKS_REWRITE_MODEL=llama3.2 \
#   python3 "$SCRIPTS/rewrite_text.py" draft.md -o draft.rewritten.md
# API 密钥仅从 WATERMARKS_REWRITE_API_KEY 读取（绝不从 argv）。

# 图像
python3 "$SCRIPTS/inspect_image.py" shot.png
python3 "$SCRIPTS/clean_image.py" shot.png -o shot.cleaned.png
```

### 文本工具拒绝二进制输入

`inspect_text.py`、`clean_text.py` 和 `rewrite_text.py` 处理文本。指向 `.docx`、`.pdf` 或图像时，它们过去会解码压缩字节并报告任意落出的码点——这些是跟踪压缩而非内容的噪声——而 `clean_text.py` 随后会将这些损坏的字节写回，破坏文件。它们现在拒绝二进制输入并指明处理该文件的工具：

```bash
python3 "$SCRIPTS/inspect_text.py" report.docx
# 拒绝将 report.docx 视为文本：它看起来像 ZIP 容器（DOCX、ODT 等）。
# 使用 inspect_file.py / clean_file.py，它们按格式路由，
# 或传入 --force-text 以强制扫描原始字节。
```

检测基于魔数加控制字节比率，因此 UTF-8 以外的编码文本仍可正常工作。`--force-text` 可在所有地方覆盖此行为。

## HTTP 服务

相同的机制作为 stdlib HTTP 服务运行（`service/scripts/server.py`）——这是技能使用的接口，也是任何 Web 应用无需引入代码即可集成的方式：

| 方法 | 路径 | 请求体 | 返回 |
| --- | --- | --- | --- |
| GET | `/health` | — | `{"ok": true, "version": ...}` |
| GET | `/capabilities` | — | 可用的工具 / 后端 |
| GET | `/openapi.json` | — | 动态生成的 OpenAPI 3.0.3 规范 |
| POST | `/inspect` | `{"file": "<base64>", "name": "notes.md"}` | `{"ok", "kind", "suspicious", "report"}` |
| POST | `/clean` | `{"file": "<base64>", "name": "notes.md", "options": {...}}` | `{"ok", "kind", "cleaned": "<base64>", "report"}` |

```bash
WM="http://127.0.0.1:8765"
curl -s "$WM/health"                       # {"ok": true, "version": "..."}
curl -s "$WM/openapi.json"                 # 机器可读的 OpenAPI 3.0.3 契约
curl -s -X POST "$WM/clean" -H 'Content-Type: application/json' \
  -d "{\"file\": \"$(base64 -w0 notes.md)\", \"name\": \"notes.md\"}"
```

服务按文件扩展名和魔数路由，因此文本 / 图像 / 容器会自动检测。设置 `WATERMARKS_SERVER_API_KEY` 以要求每个请求都携带 `Authorization: Bearer <key>`。默认仅绑定回环地址（可用 `--host` 覆盖）；适用于可信网络。

## Docker / compose

已发布的镜像（GHCR）：

| 镜像标签 | 内容 | 已发布？ |
| --- | --- | --- |
| `ghcr.io/guillaumemeyer/watermarks-remover:<tag>` / `:latest` | 核心 HTTP 服务 + 所有清理器 + exiftool / qpdf / c2patool | 是 |
| `…:markllm-<tag>` / `:markllm-latest` | MarkLLM 文本水印工具（Apache-2.0 上游） | 是 |
| `…:markdiffusion-<tag>` / `:markdiffusion-latest` | MarkDiffusion 图像工具（Apache-2.0 上游） | 是 |
| `watermarks-remover-ctrlregen:local` | CtrlRegen 像素去除——**绝不发布**（`noai-watermark` 无 LICENSE） | 仅本地构建 |
| `watermarks-remover-synthid-scorer:local` | 反向 SynthID 评分器——**绝不发布**（非商业研究许可证） | 仅本地构建 |

构建并运行核心服务：

```bash
make docker-core-build
docker run --rm -p 127.0.0.1:8765:8765 --read-only --tmpfs /tmp watermarks-remover
# 任何 CLI 仍可通过覆盖命令运行：
docker run --rm -v "$(pwd):/data" watermarks-remover \
  /app/scripts/clean_file.py /data/notes.md -o /data/notes.cleaned.md
```

完整基础设施启动：

```bash
docker compose up -d                         # 仅核心 HTTP 服务
docker compose --profile harness up -d       # + markllm / markdiffusion
docker compose --profile heavy up -d         # + ctrlregen / synthid（本地构建）
docker compose --profile harness --profile heavy up -d   # 所有服务
```

compose 栈将核心服务映射到 `127.0.0.1:8765`。harness/heavy 服务是一次性 CLI——需要验证或像素工作时使用 `docker compose run --rm <service> …` 调用。

验证运行中的栈（仅退出码，成功时无输出）：

```bash
make compose-check        # 或：./compose-check.sh
```

通过 `GET /health` 检查 `wr-core`，并对每个 harness/heavy 服务运行 `--help`，要求退出码为 `0`。

### 配置（docker compose 的环境变量）

**清理任意文本无需任何配置**——核心服务开箱即用：

```bash
echo "Hello\u200bWorld\u00ad!" > /tmp/sample.txt
curl -s -X POST http://127.0.0.1:8765/clean -H 'Content-Type: application/json' \
  -d "{\"file\": \"$(base64 -w0 /tmp/sample.txt)\", \"name\": \"sample.txt\"}"
```

其他所有内容都是可选的，位于仓库根目录的 `.env` 文件中。`docker compose` **自动加载 `.env`** 并从中插值 `compose.yaml` 中的 `${VAR}` 引用（如果两者都设置，shell 导出优先于 `.env`）。

```bash
cp .env.example .env       # 然后编辑
docker compose up -d       # 自动读取 .env
```

`.env` **已被 gitignore**（默认拒绝）——切勿提交。对于主机端 CLI 运行（`rewrite_text.py`、技能），将同一文件导出到环境中：

```bash
set -a; . ./.env; set +a; python3 service/scripts/rewrite_text.py /tmp/x.txt -o /tmp/x.rewritten.txt
```

| 变量 | 作用范围 | 用途 |
| --- | --- | --- |
| `WATERMARKS_SERVER_API_KEY` | `wr-core`（通过 compose `environment`） | 要求 HTTP API 携带 `Authorization: Bearer <key>` |
| `HF_TOKEN` | harness/heavy 服务 | 用于受限模型的 Hugging Face 令牌 |
| `WATERMARKS_SERVICE_URL` | 仅客户端（技能 / curl） | 服务地址；默认 `http://127.0.0.1:8765` |
| `WATERMARKS_REWRITE_BACKEND` | `rewrite_text.py` 钩子 | `print-prompt`（默认）/ `ollama` / `openai-compatible` |
| `WATERMARKS_REWRITE_MODEL` | `rewrite_text.py` 钩子 | 模型名称（例如 `deepseek-v4-flash`） |
| `WATERMARKS_REWRITE_BASE_URL` | `rewrite_text.py` 钩子 | API 基础地址（例如 `https://api.deepseek.com`） |
| `WATERMARKS_REWRITE_API_KEY` | `rewrite_text.py` 钩子 | API 密钥——仅环境变量，绝不通过 argv |
| `WATERMARKS_REWRITE_ALLOW_REMOTE` | `rewrite_text.py` 钩子 | 设为 `1` 允许非回环端点 |
| `WATERMARKS_REWRITE_REASONING_EFFORT` | `rewrite_text.py` 钩子 | `none`（默认）/ `low` / `medium` / `high` / `off` |

B 层在技能中由 Agent 编排（它使用自己的模型重写），因此 `WATERMARKS_REWRITE_*` 变量仅在直接驱动 `rewrite_text.py` 时需要。

镜像在 `v*` 标签上通过 [`.github/workflows/release-images.yml`](.github/workflows/release-images.yml) 自动发布。

## 可选的 SynthID 像素评分

当外部检出 [`aloshdenny/reverse-SynthID`](https://github.com/aloshdenny/reverse-SynthID) 可用时，`inspect_image.py` 和 `clean_image.py` 可以报告像素域 SynthID 置信度分数。评分器**不随附**：它在运行时从您的检出中加载，其代码仍受上游项目的非商业研究许可证约束。

### 选项 1：一键引导（无需 Docker）

```bash
SCRIPTS=service/scripts

# 克隆上游，创建 venv，并安装仅评分器依赖。
"$SCRIPTS/setup_synthid.sh"

# 对图像评分（默认检出：~/reverse-SynthID）。
REVERSE_SYNTHID_DIR=~/reverse-SynthID \
~/reverse-SynthID/.venv/bin/python "$SCRIPTS/score_synthid.py" shot.png

# 或从 inspect / clean 中显示分数（使用同一 venv Python）。
REVERSE_SYNTHID_DIR=~/reverse-SynthID \
~/reverse-SynthID/.venv/bin/python "$SCRIPTS/inspect_image.py" shot.png
```

`setup_synthid.sh` 接受 `--dir PATH`、`--ref REF` 和 `--full`（安装完整的上游 `requirements.txt`，其中添加了本项目未使用的 `torch`/`diffusers` 用于上游 VAE 旁路）。

在 Windows 上使用 `setup_synthid.ps1`（`-Dir`、`-Ref`、`-Full`），它会在 `.venv\Scripts\` 创建 venv——`image_meta.py` 在 `os.name == "nt"` 时已查找的布局。

### 选项 2：本地 Docker 构建

```bash
make docker-synthid-build
# 以非特权用户和只读根文件系统运行；评分器只需读取
# /data 并写入 stdout/tmp。
docker run --rm \
  --user "$(id -u):$(id -g)" \
  --read-only