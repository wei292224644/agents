---
name: uv-package-manager
description: 掌握 uv 包管理器 —— 极速 Python 依赖管理、虚拟环境和现代化项目工作流。当你搭建 Python 项目、管理依赖或优化 Python 开发流程时使用。
---

# UV 包管理器

全面指南：使用 uv（Rust 编写的极速 Python 包安装器和解析器）进行现代 Python 项目管理和依赖工作流。

## 何时使用

- 快速搭建新 Python 项目
- 比 pip 更快地管理 Python 依赖
- 创建和管理虚拟环境
- 安装 Python 解释器版本
- 从 pip/pip-tools/poetry 迁移
- CI/CD 流水线加速
- 管理 monorepo 中的 Python 项目
- 使用 lockfile 实现可复现构建

## 核心概念

### 1. 什么是 uv？

- **极速包安装器**：比 pip 快 10-100 倍
- **Rust 编写**：利用 Rust 的性能优势
- **pip 替代方案**：兼容 pip 工作流
- **虚拟环境管理**：创建和管理 venv
- **Python 安装器**：下载和管理 Python 版本
- **解析器**：高级依赖解析
- **Lockfile 支持**：可复现安装

### 2. 关键特性

- 极快的安装速度
- 全局缓存，节省磁盘空间
- 兼容 pip、pip-tools、poetry
- 全面的依赖解析
- 跨平台支持（Linux、macOS、Windows）
- 无需 Python 即可安装
- 内置虚拟环境支持

### 3. UV vs 传统工具

| 工具 | UV 优势 |
|------|--------|
| pip | 快 10-100 倍，更优的解析器 |
| pip-tools | 更快、更简洁、更好的用户体验 |
| poetry | 更快、侵入性更低、更轻量 |
| conda | 更快，专注 Python |

## 安装

```bash
# macOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows (PowerShell)
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

# Homebrew (macOS)
brew install uv

# 验证安装
uv --version
```

## 快速开始

### 创建新项目

```bash
# 创建带虚拟环境的新项目
uv init my-project
cd my-project

# 或在当前目录初始化
uv init .

# 初始化会创建：
# - .python-version（Python 版本）
# - pyproject.toml（项目配置）
# - README.md
# - .gitignore
```

### 安装依赖

```bash
# 添加包（自动创建 venv 并更新 pyproject.toml）
uv add requests pandas

# 添加开发依赖
uv add --dev pytest ruff

# 安装所有依赖
uv sync

# 添加带版本约束的包
uv add "django>=4.0,<5.0"

# 添加可选依赖组
uv add --optional docs sphinx
```

## 虚拟环境管理

### 创建虚拟环境

```bash
# 创建虚拟环境
uv venv

# 指定 Python 版本
uv venv --python 3.12

# 自定义名称
uv venv my-env

# 指定路径
uv venv /path/to/venv
```

### 使用 uv run（推荐，无需手动激活）

```bash
# 运行 Python 脚本（自动激活 venv）
uv run python app.py

# 运行安装的 CLI 工具
uv run ruff format .
uv run pytest

# 指定 Python 版本运行
uv run --python 3.11 python script.py

# 传递参数
uv run python script.py --arg value
```

`uv run` 是推荐的日常工作方式 —— 无需手动激活虚拟环境，自动确保命令在正确的环境上下文中运行。

## 包管理

### 添加依赖

```bash
# 添加包
uv add requests

# 添加多个包
uv add numpy pandas matplotlib

# 添加开发依赖
uv add --dev pytest pytest-cov

# 从 git 添加
uv add git+https://github.com/user/repo.git

# 添加本地包（可编辑模式）
uv add -e ./local-package
```

### 移除依赖

```bash
uv remove requests
uv remove --dev pytest
```

### 升级依赖

```bash
# 升级指定包
uv add --upgrade requests

# 升级全部
uv sync --upgrade

# 查看哪些包有新版本
uv tree --outdated

# 仅更新 lockfile 不安装
uv lock --upgrade
```

### Lockfile 管理

```bash
# 生成 uv.lock
uv lock

# 更新 lockfile
uv lock --upgrade

# 只锁定不安装
uv lock --no-install

# 升级特定包
uv lock --upgrade-package requests
```

## Python 版本管理

### 安装 Python 版本

```bash
# 安装指定版本
uv python install 3.12

# 安装多个版本
uv python install 3.11 3.12 3.13

# 安装最新版本
uv python install

# 列出已安装版本
uv python list

# 查看所有可用版本
uv python list --all-versions
```

### 固定 Python 版本

```bash
# 为项目固定 Python 版本
uv python pin 3.12

# 这会创建/更新 .python-version 文件

# 为单次命令指定版本
uv --python 3.11 run python script.py
```

## 项目配置

### pyproject.toml 配置示例

```toml
[project]
name = "my-project"
version = "0.1.0"
description = "My awesome project"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "requests>=2.31.0",
    "pydantic>=2.0.0",
    "click>=8.1.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-cov>=4.1.0",
    "ruff>=0.2.0",
]
docs = [
    "sphinx>=7.0.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

### 迁移现有项目

```bash
# 从 requirements.txt 迁移
uv add -r requirements.txt

# 已有 pyproject.toml（如 poetry），直接用：
uv sync

# 导出到 requirements.txt
uv pip freeze > requirements.txt

# 导出含哈希的 requirements.txt
uv pip freeze --require-hashes > requirements.txt
```

## Docker 集成

```dockerfile
FROM python:3.12-slim

# 安装 uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

WORKDIR /app
COPY pyproject.toml uv.lock ./

# 仅安装生产依赖
RUN uv sync --frozen --no-dev

COPY . .

CMD ["uv", "run", "python", "app.py"]
```

## 常用工作流

### 日常开发

```bash
uv run ruff check --fix .   # lint 并自动修复
uv run ruff format .         # 格式化
uv run pytest -v             # 运行测试
uv run python app.py         # 启动应用
```

### CI/CD 流水线

```yaml
# GitHub Actions 示例
- name: Install uv
  uses: astral-sh/setup-uv@v4

- name: Install dependencies
  run: uv sync --frozen

- name: Run lint
  run: uv run ruff check .

- name: Run tests
  run: uv run pytest --cov
```

## 最佳实践

1. **使用 `uv run`** — 首选工作流，无需手动激活 venv
2. **提交 uv.lock** — 应用锁文件确保可复现构建
3. **CI 中使用 `--frozen`** — 防止依赖意外变更
4. **用 `uv add` 而非手动编辑** — 保持 pyproject.toml 和 lockfile 同步
5. **开发依赖用 `--dev`** — 与生产依赖分离
6. **迁移时渐进采用** — 先在 CI 中替换 pip，再在开发环境中替换

更高级的工作流（Docker 集成、lockfile 管理、性能优化、工具对比、常见工作流、故障排查、迁移指南和命令参考）参见 [references/advanced-patterns.md](references/advanced-patterns.md)
