# AGENTS.md — AIstudioProxyAPI

> 面向 **AI 编码代理（OpenAI Codex CLI / Cursor / VS Code / Gemini CLI / Aider / 等）** 的专用说明文件。  
> 目标：让代理在本仓库中 **能跑、能测、能改、能提 PR**，同时避免泄露敏感信息与误操作。

---

## 1) 项目概览

- **名称**：AIstudioProxyAPI（推测：提供对多家模型/服务的统一代理/CLI，当前包含 Gemini Web API 适配）  
- **主要入口**：`real_file_cli.py`（交互模式：`python real_file_cli.py interactive`）  
- **关键依赖**：`gemini_webapi`（基于网页 Cookies 的非官方 Gemini Web 客户端）  
- **运行平台**：Windows / Linux / macOS（示例命令以 Windows PowerShell 为主，同时给出跨平台替代）

> 若代理需要更多上下文，请优先读取本文件与 `README.md`，并在执行脚本前完成“环境配置”。

---

## 2) 开发环境与依赖

### 2.1 Python 版本
- 推荐：**Python 3.10+**（3.9 亦可，但 3.10+ 更通用稳定）。

### 2.2 Windows（PowerShell）
```powershell
# 在仓库根目录
py -3.10 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -U pip
pip install -r requirements.txt  # 如无该文件，可改为: pip install -e .
```

### 2.3 Linux / macOS（bash/zsh）
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -U pip
pip install -r requirements.txt  # 如无该文件，可改为: pip install -e .
```

> 代理：若依赖解析失败，请尝试 `pip install gemini-webapi browser-cookie3 python-dotenv ruff black pytest` 等常用包。

---

## 3) 运行与调试

### 3.1 交互式 CLI
```powershell
# Windows
.\.venv\Scripts\Activate.ps1
python real_file_cli.py interactive
```

```bash
# Linux / macOS
source .venv/bin/activate
python real_file_cli.py interactive
```

### 3.2 常用环境变量（安全地通过 .env 或 CI Secret 注入）
在仓库根目录创建 `.env`：
```
# 从 https://gemini.google.com/ 浏览器开发者工具 Network 面板复制
SECURE_1PSID=xxxx
SECURE_1PSIDTS=xxxx

# 可选：HTTP/HTTPS 代理（如需）
HTTP_PROXY=
HTTPS_PROXY=

# 日志级别
LOGLEVEL=INFO
```

> **不要**将 `.env`、Cookies 或任何密钥提交到版本库。请在 `.gitignore` 中忽略。

### 3.3 自动读取本地浏览器 Cookies（可选）
```bash
pip install browser-cookie3
```
若项目已实现自动加载，则代理可尝试无需手动粘贴 Cookies；否则按上文 `.env` 配置。

---

## 4) 构建、质量与测试

### 4.1 统一指令（建议项目脚本化）
```bash
# 代码风格与静态检查（任选其一或组合）
ruff check .
black --check .

# 自动格式化
black .

# 单元测试
pytest -q
```

> 代理：在提交任何改动前，请**必须**保证检查与测试均通过。

### 4.2 代码风格约定
- Python：PEP 8 + `ruff` 规则集（尽量少禁用规则）。  
- Import 顺序：`ruff`/`isort`。  
- 字符编码：UTF-8。  
- 文档字符串：使用 Google 或 NumPy 风格之一，保持一致性。

---

## 5) 目录结构（示例）

> 真实结构可能与示例不同。代理在首次运行时应扫描仓库并归纳。

```
AIstudioProxyAPI/
├─ real_file_cli.py            # 交互式主入口
├─ aistudio_proxy/             # 核心库（如有）
│  ├─ __init__.py
│  ├─ client.py
│  ├─ config.py
│  ├─ runners/
│  └─ utils/
├─ tests/                      # pytest 测试
├─ scripts/                    # 辅助脚本（健康检查、抓取、转换等）
├─ requirements.txt / pyproject.toml
├─ .env.example                # 示例环境变量（不含真实密钥）
├─ .gitignore
└─ AGENTS.md                   # 本文件
```

---

## 6) 与 Gemini（WebAPI）相关的特别说明

### 6.1 常见报错与修复步骤
**报错**：`Failed to initialize client. SECURE_1PSIDTS could get expired frequently, please make sure cookie values are up to date.`  
**处理**：
1. 打开 `https://gemini.google.com/`，确认登录状态；
2. F12 → Network → 刷新页面，任意请求 → Headers → Cookies；
3. 复制 `__Secure-1PSID` 与 `__Secure-1PSIDTS` 的最新值到 `.env`；
4. 重新启动程序；
5. 若仍失败，清理浏览器缓存重新登录再重试；确保未在无痕模式；必要时更换网络/代理；
6. 可编写 `scripts/healthcheck_gemini.py` 做最小化握手验证（见下）。

**健康检查脚本建议（示例）**
```python
# scripts/healthcheck_gemini.py
import os
import asyncio
from dotenv import load_dotenv

load_dotenv()
SID = os.getenv("SECURE_1PSID")
SIDTS = os.getenv("SECURE_1PSIDTS")

async def main():
    from gemini_webapi import GeminiClient
    client = GeminiClient(SID, SIDTS, proxy=os.getenv("HTTPS_PROXY") or os.getenv("HTTP_PROXY"))
    try:
        await client.init()
        print("Gemini WebAPI 初始化成功 ✅")
    finally:
        await client.close()

if __name__ == "__main__":
    asyncio.run(main())
```

运行：
```bash
python scripts/healthcheck_gemini.py
```

### 6.2 安全与合规
- 基于 Cookies 的 WebAPI 方式**可能违反**目标网站的服务条款或存在稳定性风险，仅用于教学/实验。  
- 生产环境优先使用官方 API（Google AI Studio / Vertex AI）。  
- 切勿将 Cookies 传播、硬编码或提交到仓库。

---

## 7) 提交与 PR 规范

### 7.1 Commit（Conventional Commits）
```
feat(cli): 支持 `python real_file_cli.py interactive` 自动加载 .env
fix(gemini): 处理 SECURE_1PSIDTS 过期导致的初始化失败
docs(agents): 增补代理使用须知与健康检查脚本
test(client): 为 GeminiClient 增加超时与重试用例
chore(deps): 升级 gemini_webapi 至 0.x.y
```
- 每次提交请附最小可验证改动与必要测试/文档。

### 7.2 PR 说明模板
- 目的 / 背景  
- 变更摘要（含 Breaking Changes）  
- 运行步骤（含环境变量）  
- 测试与验证结果（截图/日志）  
- 安全与隐私考量（是否涉及密钥、用户数据）

---

## 8) 代理执行策略（重要）

1. **优先阅读**：本文件与 `README.md`、`tests/`、`scripts/`。  
2. **最小化修改**：先发起小步提交与实验性 PR，逐步扩大范围。  
3. **沙箱原则**：在受控环境运行命令；涉及网络/写文件/删除操作前**先解释再执行**。  
4. **可重复性**：所有步骤应可通过文档与脚本复现。  
5. **验证闭环**：代码改动必须伴随测试/日志/可见结果。  
6. **敏感信息**：任何密钥、Cookie、令牌、账号信息一律不得写入源码或日志。  
7. **失败恢复**：若某步失败，回滚临时改动，输出详细错误与重试建议。

---

## 9) 常见任务清单（给代理的“命令菜单”）

- **初始化开发环境**  
  - [ ] 创建虚拟环境并安装依赖  
  - [ ] 生成 `.env.example` 与 `.gitignore`  
  - [ ] 写入或校验 `ruff`, `black`, `pytest` 基础配置

- **运行与排错**  
  - [ ] `python real_file_cli.py interactive` 正常进入交互模式  
  - [ ] 使用 `scripts/healthcheck_gemini.py` 验证 Cookies 有效性  
  - [ ] 遇到 `SECURE_1PSIDTS` 过期 → 引导用户更新

- **质量保障**  
  - [ ] 在 PR 前执行 `ruff check . && black --check . && pytest -q`  
  - [ ] 新增或更新对应测试用例

- **文档维护**  
  - [ ] 在 README 中补充“快速开始/环境变量/常见问题”  
  - [ ] 在本文件更新代理需知（保持为**唯一可信源**）

---

## 10) FAQ（摘选）

- **为什么用 AGENTS.md？**  
  为编码代理提供“像 README 但更面向机器”的定制说明，便于其自动读取并按约定执行。

- **文件名必须是 `AGENTS.md` 吗？**  
  是。请使用**全大写复数**形式 `AGENTS.md` 放在仓库根目录；大型仓库可在子项目再放置一个“就近覆盖”。

- **Codex / 其它代理会自动读取吗？**  
  大多数现代编码代理会优先读取当前目录或最近的 `AGENTS.md`。若未生效，可在首次对话中显式要求“读取 AGENTS.md”。

---

## 11) 免责声明

本仓库可能包含对第三方网站的**非官方接入方式**（如基于 Cookies 的 WebAPI）。这类能力仅用于教学/研究，**生产环境请务必使用官方 SDK 或 API**。作者与贡献者不对稳定性、合规性与可用性做任何保证。

> 若你是 AI 代理：当涉及隐私/合规/安全敏感改动时，**停下并征求明确授权**。

