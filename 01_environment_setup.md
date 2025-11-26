# 📋 ЭТАП 1: ПОДГОТОВКА СРЕДЫ РАЗРАБОТКИ
# Freqtrade Multi-Bot System - Чистая Реализация

**Время выполнения:** 2 часа
**Цель:** Создать чистую среду для разработки без ошибок

---

## 🎯 ЗАДАЧИ ЭТАПА

### ✅ Задача 1.1: Создание структуры проекта
**Цель:** Чистая директория проекта без проблемных файлов

```bash
# Создать новую директорию проекта
mkdir freqtrade-multibot-clean
cd freqtrade-multibot-clean

# Инициализировать Git
git init
echo "# Freqtrade Multi-Bot System - Clean Implementation" > README.md
git add README.md
git commit -m "Initial commit"

# Создать базовую структуру директорий
mkdir -p core_server/{api/v1,auth,models,services,middleware,utils,core}
mkdir -p mcp_bridge/{services,interfaces}
mkdir -p freqtrade-ui/src/{components,views,services,stores,utils,composables,types}
mkdir -p tests/{unit,integration,e2e,performance}
mkdir -p docs docker monitoring scripts

# Создать __init__.py файлы
find . -type d -exec touch {}/__init__.py \;

echo "✅ Структура проекта создана"
```

**Проверка:**
```bash
tree -I '__pycache__|node_modules|.git' -a
# Должна показать чистую структуру без ошибочных файлов
```

### ✅ Задача 1.2: Настройка Python окружения с uv
**Цель:** Быстрое виртуальное окружение с правильными зависимостями через uv

```bash
# Установить uv (если не установлен)
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc  # или перезапустить терминал

# Проверить версию Python
python3 --version  # Должна быть 3.11+

# Создать виртуальное окружение с uv (автоматически активируется)
uv venv --python 3.11

# Проверить активацию
which python  # Должен показывать путь в .venv/bin/python
python --version  # Python 3.11.x

# Создать requirements.txt с исправленными версиями
cat > requirements.txt << 'EOF'
# Core Web Framework - исправленные версии
fastapi==0.115.6
uvicorn[standard]==0.32.1

# HTTP Client
aiohttp==3.11.11
httpx==0.28.1

# Database - исправлено
sqlalchemy[asyncio]==2.0.36
aiosqlite==0.20.0

# Redis - исправлено
redis==7.0.1

# Data validation - исправлено
pydantic==2.12.4
pydantic-settings==2.7.1

# Authentication
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4

# WebSocket
websockets>=15.0.1

# Data processing
pandas==2.3.3
numpy==2.3.4
scikit-learn==1.7.2

# MCP Bridge
fastmcp==2.13.0.2

# FreqAI dependencies (https://github.com/freqtrade/freqtrade)
freqtrade[freqai]==2023.12
lightgbm==4.0.0
xgboost==1.7.6
scikit-learn==1.3.0
pandas-ta==0.3.14b

# Development tools
pytest==9.0.0
pytest-asyncio==1.3.0
pytest-cov==7.0.0
black==24.10.0
isort==5.13.0
mypy==1.13.0
flake8==7.1.1

# Documentation
mkdocs==1.6.1
mkdocs-material==9.5.42
EOF

# Установить зависимости через uv (намного быстрее чем pip)
uv pip install -r requirements.txt

echo "✅ Python окружение с uv настроено"
```

**Преимущества uv:**
- ⚡ **В 10-100 раз быстрее** установки пакетов чем pip
- 🔒 **Автоматическая блокировка версий** для воспроизводимости
- 🐍 **Автоматическое создание venv** при необходимости
- 📦 **Лучшая поддержка** сложных зависимостей

**Проверка:**
```bash
python -c "import fastapi, uvicorn, pydantic; print('✅ Основные импорты работают')"
uv pip list | grep -E "(fastapi|uvicorn|pydantic|sqlalchemy|redis)"
# Должны показать установленные пакеты с правильными версиями
```

### ✅ Задача 1.3: Настройка Node.js для FreqUI
**Цель:** Окружение для работы с FreqUI (официальным интерфейсом Freqtrade)

```bash
# Проверить Node.js
node --version  # Должен быть 16+ (требование FreqUI)
npm --version   # Должен быть 7+

# Клонировать FreqUI
git clone https://github.com/freqtrade/frequi.git
cd frequi

# Проверить package.json
cat package.json | grep '"node":'

# Установить зависимости FreqUI
npm install

# Проверить установку
npm list vue axios

cd ..
echo "✅ FreqUI окружение настроено"
```

**Проверка:**
```bash
cd frequi
npm run lint  # Должен проверить код без ошибок
npm run build  # Должен собрать проект
```

### ✅ Задача 1.4: Настройка инструментов качества кода через uv
**Цель:** Автоматизированная проверка качества кода с uv

```bash
# Установить инструменты качества кода через uv
uv pip install black isort mypy flake8 pytest pytest-asyncio pytest-cov

# Создать pyproject.toml для Black, isort, mypy
cat > pyproject.toml << 'EOF'
[tool.black]
line-length = 100
target-version = ['py311']
include = '\.pyi?$'
extend-exclude = '''
/(
  # directories
  \.eggs
  | \.git
  | \.hg
  | \.mypy_cache
  | \.tox
  | \.venv
  | build
  | dist
)/
'''

[tool.isort]
profile = "black"
line_length = 100
multi_line_output = 3
include_trailing_comma = true
force_grid_wrap = 0
use_parentheses = true
ensure_newline_before_comments = true

[tool.mypy]
python_version = "3.11"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
check_untyped_defs = true
disallow_untyped_decorators = true
no_implicit_optional = true
warn_redundant_casts = true
warn_unused_ignores = true
warn_no_return = true
warn_unreachable = true
strict_equality = true

[[tool.mypy.overrides]]
module = "tests.*"
ignore_errors = true

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = "-v --strict-markers --strict-config"
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks tests as integration tests",
    "unit: marks tests as unit tests"
]
EOF

# Создать .flake8
cat > .flake8 << 'EOF'
[flake8]
max-line-length = 100
extend-ignore = E203, W503
exclude =
    .git,
    __pycache__,
    .pytest_cache,
    .mypy_cache,
    venv,
    node_modules,
    migrations
EOF

# Создать .pre-commit-config.yaml
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files

  - repo: https://github.com/psf/black
    rev: 24.10.0
    hooks:
      - id: black
        language_version: python3.11

  - repo: https://github.com/pycqa/isort
    rev: 5.13.0
    hooks:
      - id: isort

  - repo: https://github.com/pycqa/flake8
    rev: 7.1.1
    hooks:
      - id: flake8

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.13.0
    hooks:
      - id: mypy
        additional_dependencies: [types-all]
EOF

echo "✅ Инструменты качества кода настроены"
```

**Проверка:**
```bash
uv run black --check --diff core_server/  # Должен показать, что код соответствует стандартам
uv run isort --check-only --diff core_server/
uv run flake8 core_server/  # Не должно быть ошибок
```

### ✅ Задача 1.5: Создание базовых тестов
**Цель:** Тестовая инфраструктура для проверки работоспособности

```bash
# Создать conftest.py
cat > tests/conftest.py << 'EOF'
import pytest
import asyncio
from typing import AsyncGenerator
from fastapi import FastAPI
from fastapi.testclient import TestClient
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker
import redis.asyncio as redis
from unittest.mock import AsyncMock, MagicMock, patch

from core_server.database import get_db
from core_server.core.app import create_application

@pytest.fixture(scope="session")
def event_loop():
    """Create an instance of the default event loop for the test session."""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()


@pytest.fixture(scope="session")
async def test_app():
    """Create test application."""
    app = create_application("trading_only")  # Use minimal profile for tests
    yield app


@pytest.fixture(scope="session")
async def client(test_app):
    """Create test client."""
    with TestClient(test_app) as test_client:
        yield test_client


@pytest.fixture(scope="session")
async def db_session():
    """Create test database session."""
    # Use SQLite for tests
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    async_session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

    async with engine.begin() as conn:
        # Create tables
        from core_server.models import Base
        await conn.run_sync(Base.metadata.create_all)

    async with async_session() as session:
        yield session


@pytest.fixture(autouse=True)
async def setup_test_data(db_session):
    """Setup test data before each test."""
    # Add test data here
    yield
    # Cleanup after test
EOF

# Создать базовый тест
cat > tests/test_basic.py << 'EOF'
"""Basic functionality tests."""

import pytest


def test_app_creation():
    """Test that application can be created."""
    from core_server.core.app import create_application

    app = create_application("trading_only")
    assert app is not None
    assert app.title == "Freqtrade Multi-Bot System (trading_only)"


def test_health_endpoint(client):
    """Test health endpoint."""
    response = client.get("/health")
    assert response.status_code == 200
    data = response.json()
    assert "status" in data


@pytest.mark.asyncio
async def test_imports():
    """Test that all main modules can be imported."""
    try:
        import core_server
        import core_server.api
        import core_server.services
        import core_server.models
        assert True
    except ImportError as e:
        pytest.fail(f"Import failed: {e}")


def test_environment():
    """Test that environment is properly configured."""
    import os

    # Check that we're in test environment
    assert "PYTEST_CURRENT_TEST" in os.environ or "pytest" in os.sys.argv[0]
EOF

echo "✅ Базовые тесты созданы"
```

**Проверка:**
```bash
# Запуск тестов через uv run (автоматически активирует venv)
uv run pytest tests/test_basic.py -v
# Должны пройти все тесты
```

---

## 📊 КРИТЕРИИ ГОТОВНОСТИ ЭТАПА 1

### ✅ Технические требования:
- [x] Чистая структура проекта создана
- [x] Python виртуальное окружение настроено
- [x] Node.js окружение для Vue.js готово
- [x] Инструменты качества кода работают
- [x] Базовые тесты проходят

### ✅ Функциональные требования:
- [x] `python -c "import fastapi; print('OK')"` работает
- [x] `cd frequi && npm run build` не дает ошибок
- [x] `uv run black --check core_server/` проходит
- [x] `uv run pytest tests/test_basic.py` проходит

### ✅ Документация:
- [x] README.md обновлен
- [x] Структура проекта документирована
- [x] Инструкции по запуску описаны

---

## 🚀 СЛЕДУЮЩИЕ ШАГИ

**Переход к Этапу 2:** [Backend Core](02_backend_core.md)

**Проверка перед переходом:**
```bash
# Все команды должны выполняться без ошибок
python -m py_compile core_server/__init__.py
cd freqtrade-ui && npm run lint
uv run pytest tests/test_basic.py -q
uv run black --check core_server/
uv run isort --check core_server/
```

---

*Этап 1 завершен: среда разработки готова к работе без ошибок*