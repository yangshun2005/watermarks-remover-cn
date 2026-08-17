# 为 watermarks-remover 做贡献

感谢你帮助保持技能的准确性和清理工具的可靠性。本项目是一个小型 Python 技能（`skills/remove-ai-marks/`）加上测试——聚焦的 Pull Request 合并速度最快。

## 谁能做什么

| 操作 | 谁 |
| --- | --- |
| 开启 Issue | 任何人 |
| 建议发布版本 | 任何人（使用 **发布版本建议** Issue 模板） |
| 开启 Pull Request | 任何人（Fork 仓库） |
| 批准并合并 Pull Request | 仅维护者（`@guillaumemeyer`） |

`main` 分支受保护。任何更改都需要通过 Pull Request、通过 **CI** 检查（`test`），并在合并前获得代码所有者的批准审查。只有维护者才能给予该批准。非管理员直接推送到 `main` 分支会被阻止。

如需在不修改代码的情况下建议发布版本：请开启一个 **发布版本建议** Issue。

## 前提条件

- **Python 3.10+**（技能脚本仅使用标准库；可选的改写后端使用 HTTP 连接本地 Ollama / OpenAI 兼容端点）
- 在仓库根目录下：开启 PR 前应通过 `python3 -m pytest -q`
- 可选的手动文件检查工具：[`c2patool`](https://github.com/contentauth/c2pa-rs/tree/main/cli)、[`exiftool`](https://exiftool.org/)（用于 PDF）

## 目录结构

| 路径 | 作用 |
| --- | --- |
| `skills/remove-ai-marks/SKILL.md` | Agent 技能入口（工作流程、伦理）——通过 HTTP 远程客户端 |
| `skills/remove-ai-marks/references/` | 供应商、标记类别、矩阵、伦理 |
| `service/scripts/` | A/B 层钩子 + 图像/容器清理器 + `server.py` HTTP 服务 |
| `service/Dockerfile*` | 容器镜像（核心 + 可选后端） |
| `compose.yaml` | 本地全栈启动 |
| `tests/` | Pytest 测试套件和夹具 |
| `.github/workflows/ci.yml` | CI 任务 `test` |
| `.github/workflows/release-images.yml` | 在 `v*` 标签上发布 GHCR 镜像 |

## 分层（在哪里改什么）

1. **A 层（Unicode / 格式控制）**——`service/scripts/` 下的确定性脚本（`text_unicode.py`、`clean_text.py`、`inspect_text.py`）。优先使用 `tests/fixtures/` 中的夹具编写测试。
2. **B 层（统计改写）**——`SKILL.md` 中的指南以及可选的 `rewrite_text.py`（默认打印提示；ollama / openai 兼容）。不捆绑模型。保持伦理意识。
3. **文件（C2PA / EXIF / XMP / 属性）**——`image_meta.py`（PNG/JPEG）、`container_meta.py`（SVG/PDF/DOCX/ODT/HTML/MD）、统一的 `inspect_file.py` / `clean_file.py`。保留文档正文 / 像素；仅移除溯源元数据。

## 更改检查清单

- [ ] 行为与 `SKILL.md` / `references/removal-matrix.md` 相关部分一致
- [ ] 在 `tests/` 下更新或添加单元测试
- [ ] `python3 -m pytest -q` 通过
- [ ] 如果影响用户可见行为，更新文档（README 和/或技能参考资料）
- [ ] 不包含与修复或功能无关的附带重构

## Pull Request 期望

- 保持聚焦并匹配现有风格（优先标准库脚本、清晰的 CLI 标志）
- 除非必要且已脱敏，否则不要提交密钥、私人用户文件或大型二进制夹具
- 遵守 `references/ethics.md`：此工具仅用于用户拥有的内容

有问题？请开启一个 Issue，描述输入类型（文本 / 图像 / 文档）以及哪一层失败或缺失。

## 社区

- [行为准则](CODE_OF_CONDUCT.md)——项目中的预期行为
- [安全政策](SECURITY.md)——如何私下报告漏洞
- [Bug 报告](.github/ISSUE_TEMPLATE/bug_report.md)和[功能请求](.github/ISSUE_TEMPLATE/feature_request.md)模板