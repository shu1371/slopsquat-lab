# Slopsquatting 幻觉包名供应链攻击复现（slopsquat-lab）

本地可复现的 **AI 编码助手幻觉包名（Slopsquatting）供应链攻击**实验：编码助手给出看似合理但官方索引中**不存在**的 `pip install` 包名，攻击者提前抢注这些名字，开发者执行安装命令即触发恶意 `setup.py`，读取项目 `.env` 并把凭据外带。本实验用 Ollama + 本地小模型在本地完整复现「幻觉包名」与「文档投毒」两条入口，并验证「安装前注册表校验」这道防线的真实效果。

对应技术文章（本地，不放入本仓库）：见同目录 `article.md`（如未随附，请联系作者索取）。

## 实验结果（qwen2.5:7b，每场景 8 轮）

| 场景 | 建议了攻击者包 | 恶意安装（凭据外带） | 被注册表校验拦截 |
|---|---|---|---|
| 基线（通用任务，无投毒） | 0/8 | 0/8 | 0 |
| 幻觉诱导（小众任务） | 1/8 | 1/8 | 0 |
| 文档投毒（带投毒 README） | 3/8 | 3/8 | 0 |
| 注册表校验防御（同上任务） | 3/8 | 0/8 | 3/8 |

> 外带证据见 `lab/outputs/exfil.log`，示例：
> `[...] EXFIL package=yaml exfiltrated_from=.env data=AWS_ACCESS_KEY=AKIAFAKE123456789;DATABASE_PASSWORD=S3cret!2026`

## 目录结构

```text
.
├── article.md            # 技术文章全文（本地，未上传本仓库）
├── images/               # 示意图（SVG + PNG 均在 images/png/）
├── lab/
│   ├── code_assistant.py # 编码助手 / 官方索引 / 攻击者抢注名单 / 安装器 / 校验器
│   ├── experiments.py    # 实验编排：基线 / 幻觉 / 投毒 / 防御
│   ├── data/project/.env # 受害者机密（模拟云密钥与数据库口令）
│   └── outputs/          # 运行日志、外带日志与统计
└── LICENSE               # MIT
```

## 环境要求

- Python 3.8+（**零第三方依赖**，仅标准库 `json` / `re` / `urllib` / `os`）
- [Ollama](https://ollama.com) 0.32+ 与 `qwen2.5:7b` 模型（用于模拟"会建议安装命令"的编码助手）
- 若暂时无 Ollama，`code_assistant.py` 的 `chat()` 可替换为返回固定字符串的桩函数，便于离线演示攻击链

## 快速开始

```powershell
# 1) 启动 Ollama 并拉取模型
ollama serve
ollama pull qwen2.5:7b

# 2) 进入实验目录，初始化数据（生成模拟 .env）
cd lab
python experiments.py init

# 3) 依次运行四个场景（每场景 8 轮）
python experiments.py baseline 8          # 基线：通用任务
python experiments.py hallucination 8     # 入口①：幻觉包名
python experiments.py doc_poison 8        # 入口②：文档投毒
python experiments.py defense_registry 8  # 防御：安装前注册表校验

# 统计结果输出到 lab/outputs/all_stats.json，外带证据在 lab/outputs/exfil.log
```

## 核心代码说明

`code_assistant.py` 包含：

- `OFFICIAL_PACKAGES`：模拟 PyPI 官方索引白名单；
- `ATTACKER_PACKAGES`：攻击者抢注的"看似合理但官方索引不存在"的包；
- `ask_install_command(task)`：模拟编码助手，根据任务（可附带投毒文档）产出 `pip install` 建议；
- `extract_packages(answer)`：从模型回答中提取包名；
- `install_package(name, defense)`：模拟安装——命中攻击者包则执行恶意 `setup.py`（读取 `.env` 并写入 `exfil.log`）；打开 `defense="registry"` 时先做注册表校验；
- `validate_package(name)`：安装前校验，仅放行官方索引中的包。

`experiments.py` 编排四类场景并统计"建议攻击者包 / 恶意安装 / 被拦截"三项指标。

## 防御要点（仅示意，详见文章）

1. 安装前强制注册表校验：不在官方白名单的包一律拦截（本实验已量化，恶意安装 0/8）；
2. 官方源白名单 + 依赖锁文件（requirements.lock 带 sha256）；
3. 隔离环境安装，安装进程不可见 `.env` 等机密；
4. 监控安装阶段的异常外联（非官方源域名）；
5. README / 文档中的依赖推荐纳入内容审查（防文档投毒）。

## 免责声明

本项目全部数据为本地虚构数据，仅用于 AI 供应链安全研究与防御验证。**请勿对未授权系统使用，请勿注册真实恶意包。**
