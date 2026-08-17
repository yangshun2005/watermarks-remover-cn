# 部署：Docker 中的 CLI + API

状态：构想 / 提议
日期：2026-08-14
分支上下文：`refactor/format-dispatch`

## 目标

通过提供一个同时具备 **CLI** 和持久化 **HTTP API** 的 Docker 容器，并支持通过本地镜像构建进行自托管，来简化 watermarks-remover 的部署。主要动机：减少主机配置的繁琐（Python 版本、exiftool/qpdf/c2patool、操作系统差异），并通过共享的多用户服务向非 Python 用户开放该工具。

## 决策（已与作者确认）

- 范围：**v1 中的全部功能** —— 核心（Layer A + 文件元数据）、Layer B 重写以及 GPU 像素移除（CtrlRegen）。
- 分发：**全部为本地构建**（不发布到镜像仓库），遵循仓库中关于绝不捆绑受限上游代码的许可证安全规则。
- CLI + API 重构**一同**作为一个变更提交。
- 认证：v1 中使用来自环境变量的**单个共享 Bearer 密钥**（后续再支持按消费者分配密钥）。

## 架构 —— 3 个镜像

| 镜像 | 内容 | 模式 |
| --- | --- | --- |
| `watermarks-remover-core` | 摘要固定的 `python:3.14-slim` + exiftool/qpdf/c2patool + 核心脚本 + FastAPI 服务器 | 默认：**CLI**（`docker run wm-core clean_file.py ...`）。`serve` 子命令：**HTTP API** |
| `watermarks-remover-ctrlregen` | 现有 GPU 镜像（不变，本地构建，许可证安全） | 异步像素移除任务的**工作节点** |
| `watermarks-remover-markllm` / synthid | 现有的可选镜像 | **不在服务范围内**。MarkLLM 是验证工具，不是面向用户的功能；SynthID 评分器是后续可选边车 |

理由：单一的全能镜像会是一个依赖 GPU 的巨型镜像，并且会与绝不捆绑受限上游代码的规则冲突。重型后端保持为独立的边车，由 API 进行调度。

## 阶段

### 阶段 0 —— 可调用的核心重构（风险最高）

将路由中的 argv 解析与业务逻辑分离，以便 API 可以在进程内运行它们：

- `inspect_file.py`、`clean_file.py`、`inspect_text.py`、`clean_text.py`、
  `rewrite_text.py`、`inspect_image.py`、`clean_image.py`、`audit_dir.py`、
  `audit_website.py`
- 暴露稳定的接口，例如 `clean_bytes(data, filename, opts) -> (report, out_bytes)`
- 复用 `format_dispatch.py` 的分类逻辑和 `common.py` 的防护措施
  （字节上限、二进制嗅探、安全写入）
- **不变式：CLI 行为逐字节一致**；现有的 60 多个测试是安全网。

### 阶段 1 —— 核心镜像

`Dockerfile.core` 采用现有的加固风格：摘要固定的基础镜像、非特权用户、固定版本的 pip、apt 安装 exiftool/qpdf/c2patool。入口点根据第一个参数（`serve` 或脚本名称）进行分发。Makefile 目标：
`docker-core-build`、`docker-core-help`、`serve-core`。

### 阶段 2 —— API 服务器（FastAPI + uvicorn，位于核心镜像中）

- `POST /v1/inspect` —— multipart 文件 -> JSON 报告（发现结果 + 置信度）
- `POST /v1/clean` —— multipart 文件 -> 流式返回清理后的文件
- `POST /v1/rewrite` —— 文本 + 后端配置 -> 重写后的文本；默认仅限回环，
  远程端点仅通过服务器环境变量配置（绝不通过请求传入）
- `POST /v1/clean/pixel` —— 异步：SQLite 作业记录 -> 返回 `job_id`；工作节点
  获取该任务
- `GET /v1/jobs/{id}`、`GET /v1/health`（报告哪些可选后端可达）
- 安全：
  - 来自环境变量的单个共享 `Bearer` 密钥（遵循仅环境变量密钥的规则）
  - 文件始终作为上传流进入每个请求的临时目录 —— 绝不使用
    客户端提供的文件系统路径，从而保证符号链接/安全写入防护有效
  - 每个请求复用 `common.py` 的字节上限
  - 速率限制；通过反向代理实现文档化的 TLS 终止
- 作业存储在共享卷上的 SQLite 中；工作节点轮询数据库（无需新协议；GPU
  工作本身是串行化的）。

### 阶段 3 —— Compose + 文档

`docker-compose.yml`（核心 + ctrlregen 工作节点，共享卷用于作业/模型
缓存）、README 章节（快速入门、认证、安全说明、CLI 模式示例）、
Makefile `up`/`down`。

### 阶段 4 —— 安全审查

- SSRF（重写后端、audit_website）
- 上传大小上限
- 密钥处理（仅环境变量，绝不记录日志）
- 并发/线程池边界
- 工作节点崩溃时进行中的作业的恢复

## 验证

- `make test`（重构）
- `make smoke`（CLI 不变）
- 新增测试：通过 FastAPI 测试客户端测试 API 端点、认证拒绝、
  上限强制执行、使用模拟工作节点的异步作业生命周期、`docker run wm-core ...`
  对测试夹具的冒烟测试。

## 待定事项

- MarkLLM 不包含在服务中（如有异议请提出）
- CtrlRegen 工作节点通信 = 共享卷 + SQLite 轮询（简单，无需新基础设施）
- SynthID 评分器 = 可选的未来边车，不属于 v1