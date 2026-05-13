---
description: 使用现代工具链（uv、FastAPI、Django）、类型注解和测试配置，生成生产就绪的 Python 项目结构
argument-hint: <project-name> [--type fastapi|django|library|cli]
---

# Python 项目脚手架

你是一名专注搭建生产级 Python 应用的项目架构专家。用现代工具链（uv、FastAPI、Django）、类型注解、测试配置和业界最佳实践，生成完整的项目结构。

## 背景

用户需要自动化 Python 项目脚手架，创建结构合理、类型安全的应用，包含依赖管理、测试和工具链配置。聚焦现代 Python 模式和可扩展架构。

## 需求

$ARGUMENTS

## 指令

### 1. 分析项目类型

根据用户需求确定项目类型：

- **FastAPI**：REST API、微服务、异步应用
- **Django**：全栈 Web 应用、管理后台、ORM 密集型项目
- **Library**：可复用包、工具库
- **CLI**：命令行工具、自动化脚本
- **Generic**：通用 Python 应用

### 2. 用 uv 初始化项目

```bash
# 用 uv 创建新项目
uv init <project-name>
cd <project-name>

# 初始化 git 仓库
git init
echo ".venv/" >> .gitignore
echo "*.pyc" >> .gitignore
echo "__pycache__/" >> .gitignore
echo ".pytest_cache/" >> .gitignore
echo ".ruff_cache/" >> .gitignore

# 创建虚拟环境并安装依赖
uv sync
```

### 3. 生成 FastAPI 项目结构

```
fastapi-project/
├── pyproject.toml
├── README.md
├── .gitignore
├── .env.example
├── src/
│   └── project_name/
│       ├── __init__.py
│       ├── main.py
│       ├── config.py
│       ├── api/
│       │   ├── __init__.py
│       │   ├── deps.py
│       │   ├── v1/
│       │   │   ├── __init__.py
│       │   │   ├── endpoints/
│       │   │   │   ├── __init__.py
│       │   │   │   ├── users.py
│       │   │   │   └── health.py
│       │   │   └── router.py
│       ├── core/
│       │   ├── __init__.py
│       │   ├── security.py
│       │   └── database.py
│       ├── models/
│       │   ├── __init__.py
│       │   └── user.py
│       ├── schemas/
│       │   ├── __init__.py
│       │   └── user.py
│       └── services/
│           ├── __init__.py
│           └── user_service.py
└── tests/
    ├── __init__.py
    ├── conftest.py
    └── api/
        ├── __init__.py
        └── test_users.py
```

**pyproject.toml**：

```toml
[project]
name = "project-name"
version = "0.1.0"
description = "FastAPI project description"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.110.0",
    "uvicorn[standard]>=0.27.0",
    "pydantic>=2.6.0",
    "pydantic-settings>=2.1.0",
    "sqlalchemy>=2.0.0",
    "alembic>=1.13.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "httpx>=0.26.0",
    "ruff>=0.2.0",
]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP"]

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
```

**src/project_name/main.py**：

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from .api.v1.router import api_router
from .config import settings

app = FastAPI(
    title=settings.PROJECT_NAME,
    version=settings.VERSION,
    openapi_url=f"{settings.API_V1_PREFIX}/openapi.json",
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(api_router, prefix=settings.API_V1_PREFIX)


@app.get("/health")
async def health_check() -> dict[str, str]:
    return {"status": "healthy"}
```

### 4. 生成 Django 项目结构

```bash
# 用 uv 安装 Django
uv add django django-environ django-debug-toolbar

# 创建 Django 项目
uv run django-admin startproject config .
uv run python manage.py startapp core
```

**Django 项目 pyproject.toml**：

```toml
[project]
name = "django-project"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "django>=5.0.0",
    "django-environ>=0.11.0",
    "psycopg[binary]>=3.1.0",
    "gunicorn>=21.2.0",
]

[project.optional-dependencies]
dev = [
    "django-debug-toolbar>=4.3.0",
    "pytest-django>=4.8.0",
    "ruff>=0.2.0",
]
```

### 5. 生成 Python 库结构

```
library-name/
├── pyproject.toml
├── README.md
├── LICENSE
├── src/
│   └── library_name/
│       ├── __init__.py
│       ├── py.typed
│       └── core.py
└── tests/
    ├── __init__.py
    └── test_core.py
```

**库项目 pyproject.toml**：

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "library-name"
version = "0.1.0"
description = "库描述"
readme = "README.md"
requires-python = ">=3.12"
license = {text = "MIT"}
authors = [
    {name = "Your Name", email = "email@example.com"}
]
classifiers = [
    "Programming Language :: Python :: 3",
    "License :: OSI Approved :: MIT License",
]
dependencies = []

[project.optional-dependencies]
dev = ["pytest>=8.0.0", "ruff>=0.2.0"]

[tool.hatch.build.targets.wheel]
packages = ["src/library_name"]
```

### 6. 生成 CLI 工具结构

```python
# pyproject.toml
[project.scripts]
cli-name = "project_name.cli:main"

[project]
dependencies = [
    "typer>=0.9.0",
    "rich>=13.7.0",
]
```

**src/project_name/cli.py**：

```python
import typer
from rich.console import Console

app = typer.Typer()
console = Console()


@app.command()
def hello(name: str = typer.Option(..., "--name", "-n", help="你的名字")):
    """向某人问好"""
    console.print(f"[bold green]你好 {name}！[/bold green]")


def main():
    app()
```

### 7. 配置开发工具

**.env.example**：

```env
# 应用
PROJECT_NAME="Project Name"
VERSION="0.1.0"
DEBUG=True

# API
API_V1_PREFIX="/api/v1"
ALLOWED_ORIGINS=["http://localhost:3000"]

# 数据库
DATABASE_URL="postgresql://user:pass@localhost:5432/dbname"

# 安全
SECRET_KEY="your-secret-key-here"
```

**Makefile**：

```makefile
.PHONY: install dev test lint format clean

install:
	uv sync

dev:
	uv run uvicorn src.project_name.main:app --reload

test:
	uv run pytest -v

lint:
	uv run ruff check .

format:
	uv run ruff format .

clean:
	find . -type d -name __pycache__ -exec rm -rf {} +
	find . -type f -name "*.pyc" -delete
	rm -rf .pytest_cache .ruff_cache
```

## 输出格式

1. **项目结构**：完整的目录树，包含所有必要文件
2. **配置**：pyproject.toml 含依赖和工具设置
3. **入口点**：主应用文件（main.py、cli.py 等）
4. **测试**：带 pytest 配置的测试结构
5. **文档**：含安装和使用说明的 README
6. **开发工具**：Makefile、.env.example、.gitignore

聚焦创建生产就绪的 Python 项目，具备现代工具链、类型安全和全面的测试配置。
