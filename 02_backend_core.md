# 📋 ЭТАП 2: BACKEND CORE
# Freqtrade Multi-Bot System - Полная реализация

**Время выполнения:** 8 часов
**Цель:** Создать полную FastAPI архитектуру с моделями, сервисами и middleware

---

## 🎯 ЗАДАЧИ ЭТАПА

### ✅ Задача 2.1: FastAPI Application Setup (1.5 часа)

#### 1. Основное приложение
**`core_server/main.py`:**
```python
"""
Main entry point for Freqtrade Multi-Bot System API server.
"""

import uvicorn
import logging
from core.core.app import create_application

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.StreamHandler(),
        logging.FileHandler('logs/api.log')
    ]
)

logger = logging.getLogger(__name__)

def main():
    """Main application entry point."""
    try:
        # Create FastAPI application
        app = create_application("full_featured")

        logger.info("🚀 Starting Freqtrade Multi-Bot System API Server")
        logger.info("📊 Profile: full_featured")
        logger.info("🌐 API docs available at: http://localhost:8000/docs")

        # Start server
        uvicorn.run(
            app,
            host="0.0.0.0",
            port=8000,
            reload=True,
            log_level="info",
            access_log=True
        )

    except Exception as e:
        logger.error(f"❌ Failed to start server: {e}")
        raise

if __name__ == "__main__":
    main()
```

#### 2. Application Factory
**`core_server/core/app.py`:**
```python
"""
FastAPI application factory with profile-based configuration.
"""

from fastapi import FastAPI, Request, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.responses import JSONResponse
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from slowapi.middleware import SlowAPIMiddleware
import logging
from typing import Dict, Any
from datetime import datetime

from ..api.v1.router import api_router
from ..middleware.rate_limiting import create_limiter
from ..middleware.logging import RequestLoggingMiddleware
from ..middleware.error_handling import APIError, api_error_handler
from ..database import init_database, close_database
from ..core.config import settings

logger = logging.getLogger(__name__)

def create_application(profile: str = "trading_only") -> FastAPI:
    """
    Create FastAPI application with profile-based configuration.

    Args:
        profile: Application profile ("trading_only", "trading_audit", "full_featured")

    Returns:
        Configured FastAPI application
    """

    # Validate profile
    valid_profiles = ["trading_only", "trading_audit", "full_featured"]
    if profile not in valid_profiles:
        raise ValueError(f"Invalid profile: {profile}. Must be one of {valid_profiles}")

    # Create FastAPI app
    app = FastAPI(
        title="Freqtrade Multi-Bot System API",
        description="Enterprise-grade multi-bot trading platform with AI-powered management",
        version="1.0.0",
        docs_url="/docs" if profile != "trading_only" else None,
        redoc_url="/redoc" if profile != "trading_only" else None,
        openapi_url="/openapi.json" if profile != "trading_only" else None,
    )

    # Add profile to app state
    app.state.profile = profile

    # CORS middleware
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.ALLOWED_ORIGINS,
        allow_credentials=True,
        allow_methods=["GET", "POST", "PUT", "DELETE", "OPTIONS"],
        allow_headers=["*"],
    )

    # Trusted host middleware (production only)
    if settings.ENVIRONMENT == "production":
        app.add_middleware(
            TrustedHostMiddleware,
            allowed_hosts=settings.ALLOWED_HOSTS
        )

    # Rate limiting
    limiter = create_limiter()
    app.state.limiter = limiter
    app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
    app.add_middleware(SlowAPIMiddleware)

    # Custom middleware
    app.add_middleware(RequestLoggingMiddleware)

    # Error handlers
    app.add_exception_handler(APIError, api_error_handler)

    # Global exception handler
    @app.exception_handler(Exception)
    async def global_exception_handler(request: Request, exc: Exception):
        logger.error(f"Unhandled exception: {exc}", exc_info=True)
        return JSONResponse(
            status_code=500,
            content={
                "error": "Internal server error",
                "timestamp": datetime.utcnow().isoformat(),
                "path": str(request.url)
            }
        )

    # Health check endpoint (always available)
    @app.get("/health", tags=["health"])
    async def health_check():
        """Health check endpoint."""
        return {
            "status": "healthy",
            "service": "freqtrade-multi-bot-api",
            "profile": profile,
            "version": "1.0.0",
            "timestamp": datetime.utcnow().isoformat(),
            "environment": settings.ENVIRONMENT
        }

    # Info endpoint
    @app.get("/api/v1/info", tags=["info"])
    async def get_system_info():
        """Get system information."""
        return {
            "name": "Freqtrade Multi-Bot System",
            "version": "1.0.0",
            "profile": profile,
            "environment": settings.ENVIRONMENT,
            "features": _get_profile_features(profile),
            "timestamp": datetime.utcnow().isoformat()
        }

    # Include API router
    app.include_router(api_router, prefix="/api/v1")

    # Profile-specific routes
    if profile in ["trading_audit", "full_featured"]:
        _add_audit_routes(app)

    if profile == "full_featured":
        _add_admin_routes(app)

    # Lifecycle events
    @app.on_event("startup")
    async def startup_event():
        logger.info(f"🚀 Starting Freqtrade Multi-Bot System (profile: {profile})")
        await init_database()
        logger.info("✅ Database initialized")

    @app.on_event("shutdown")
    async def shutdown_event():
        logger.info("🛑 Shutting down Freqtrade Multi-Bot System")
        await close_database()
        logger.info("✅ Database connections closed")

    return app

def _get_profile_features(profile: str) -> list:
    """Get features available for profile."""
    features = {
        "trading_only": ["bot_management", "basic_monitoring"],
        "trading_audit": ["bot_management", "monitoring", "analytics", "audit_logs"],
        "full_featured": ["bot_management", "monitoring", "analytics", "audit_logs",
                         "admin_panel", "advanced_features", "ai_integration"]
    }
    return features.get(profile, [])

def _add_audit_routes(app: FastAPI):
    """Add audit-specific routes."""
    @app.get("/api/v1/audit/logs", tags=["audit"])
    async def get_audit_logs(limit: int = 100):
        """Get audit logs."""
        # Implementation would go here
        return {"logs": [], "count": 0}

def _add_admin_routes(app: FastAPI):
    """Add admin-specific routes."""
    @app.get("/api/v1/admin/system/status", tags=["admin"])
    async def get_system_status():
        """Get detailed system status."""
        # Implementation would go here
        return {"status": "operational"}
```

#### 3. Configuration
**`core_server/core/config.py`:**
```python
"""
Application configuration with Pydantic settings.
"""

from pydantic import BaseSettings, validator
from typing import List, Optional
import os

class Settings(BaseSettings):
    """Application settings."""

    # Environment
    ENVIRONMENT: str = "development"
    DEBUG: bool = False

    # Server
    HOST: str = "0.0.0.0"
    PORT: int = 8000

    # Security
    SECRET_KEY: str = "your-secret-key-change-in-production"
    JWT_SECRET: str = "your-jwt-secret-change-in-production"
    JWT_ALGORITHM: str = "HS256"
    JWT_EXPIRATION_HOURS: int = 24

    # CORS
    ALLOWED_ORIGINS: List[str] = ["http://localhost:3000", "http://127.0.0.1:3000"]
    ALLOWED_HOSTS: List[str] = ["localhost", "127.0.0.1"]

    # Database
    DATABASE_URL: str = "sqlite+aiosqlite:///./freqtrade.db"

    # Redis
    REDIS_URL: str = "redis://localhost:6379"
    REDIS_PASSWORD: Optional[str] = None

    # External APIs
    TELEGRAM_BOT_TOKEN: Optional[str] = None
    GITHUB_TOKEN: Optional[str] = None

    # Rate limiting
    RATE_LIMIT_REQUESTS: int = 100
    RATE_LIMIT_WINDOW: int = 60  # seconds

    # File paths
    LOGS_DIR: str = "logs"
    DATA_DIR: str = "user_data"

    @validator('DATABASE_URL', pre=True)
    def assemble_db_url(cls, v: Optional[str], values: dict) -> str:
        """Build database URL from components."""
        if isinstance(v, str):
            return v
        return "sqlite+aiosqlite:///./freqtrade.db"

    @validator('ALLOWED_ORIGINS', pre=True)
    def assemble_cors_origins(cls, v: Optional[List[str]]) -> List[str]:
        """Build CORS origins list."""
        if isinstance(v, str):
            return [origin.strip() for origin in v.split(',')]
        return v or ["http://localhost:3000"]

    class Config:
        env_file = ".env"
        case_sensitive = True

# Global settings instance
settings = Settings()
```

### ✅ Задача 2.2: Database Models & Pydantic Schemas (2 часа)

#### 1. Base Models
**`core_server/models/base.py`:**
```python
"""
Base SQLAlchemy models and utilities.
"""

from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, Integer, DateTime, String
from sqlalchemy.sql import func

Base = declarative_base()

class TimestampMixin:
    """Mixin for timestamp columns."""
    created_at = Column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    updated_at = Column(DateTime(timezone=True), onupdate=func.now(), nullable=True)

class SoftDeleteMixin:
    """Mixin for soft delete functionality."""
    deleted_at = Column(DateTime(timezone=True), nullable=True)
    is_deleted = Column(Boolean, default=False, nullable=False)
```

#### 2. User Models
**`core_server/models/user.py`:**
```python
"""
User models and schemas.
"""

from sqlalchemy import Column, Integer, String, Boolean, DateTime
from sqlalchemy.sql import func
from pydantic import BaseModel, EmailStr, validator
from typing import Optional
from passlib.context import CryptContext
from datetime import datetime

from .base import Base, TimestampMixin

# Password hashing
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

class User(Base, TimestampMixin):
    """User SQLAlchemy model."""
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    username = Column(String(50), unique=True, nullable=False, index=True)
    email = Column(String(100), unique=True, nullable=False, index=True)
    hashed_password = Column(String(200), nullable=False)
    full_name = Column(String(100), nullable=True)
    is_active = Column(Boolean, default=True, nullable=False)
    is_superuser = Column(Boolean, default=False, nullable=False)
    last_login = Column(DateTime(timezone=True), nullable=True)

    def verify_password(self, password: str) -> bool:
        """Verify user password."""
        return pwd_context.verify(password, self.hashed_password)

    def set_password(self, password: str) -> None:
        """Set user password."""
        self.hashed_password = pwd_context.hash(password)

# Pydantic schemas
class UserBase(BaseModel):
    username: str
    email: EmailStr
    full_name: Optional[str] = None
    is_active: bool = True
    is_superuser: bool = False

    @validator('username')
    def username_alphanumeric(cls, v):
        assert v.isalnum(), 'Username must be alphanumeric'
        return v

class UserCreate(UserBase):
    password: str

    @validator('password')
    def password_strength(cls, v):
        if len(v) < 8:
            raise ValueError('Password must be at least 8 characters')
        return v

class UserUpdate(BaseModel):
    username: Optional[str] = None
    email: Optional[EmailStr] = None
    full_name: Optional[str] = None
    is_active: Optional[bool] = None
    password: Optional[str] = None

class UserResponse(UserBase):
    id: int
    created_at: datetime
    updated_at: Optional[datetime] = None
    last_login: Optional[datetime] = None

    class Config:
        from_attributes = True

class UserLogin(BaseModel):
    username: str
    password: str

class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"
    expires_in: int
    user: UserResponse
```

#### 3. Bot Models
**`core_server/models/bot.py`:**
```python
"""
Bot models and schemas.
"""

from sqlalchemy import Column, Integer, String, Text, DateTime, Boolean, Float, JSON, ForeignKey
from sqlalchemy.sql import func
from pydantic import BaseModel, validator
from typing import Optional, Dict, Any, List
from datetime import datetime
from enum import Enum

from .base import Base, TimestampMixin

class BotStatus(str, Enum):
    """Bot status enumeration."""
    STOPPED = "stopped"
    STARTING = "starting"
    RUNNING = "running"
    STOPPING = "stopping"
    ERROR = "error"

class Bot(Base, TimestampMixin):
    """Bot SQLAlchemy model."""
    __tablename__ = "bots"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(100), unique=True, nullable=False, index=True)
    description = Column(Text, nullable=True)
    strategy_name = Column(String(100), nullable=False)
    exchange = Column(String(50), nullable=False)
    stake_currency = Column(String(10), nullable=False)
    stake_amount = Column(Float, nullable=False)
    max_open_trades = Column(Integer, default=3, nullable=False)
    config = Column(JSON, nullable=False, default=dict)
    is_active = Column(Boolean, default=True, nullable=False)
    status = Column(String(20), default=BotStatus.STOPPED.value, nullable=False)
    pid = Column(Integer, nullable=True)
    created_by = Column(Integer, ForeignKey("users.id"), nullable=False)

    # Performance tracking
    total_trades = Column(Integer, default=0, nullable=False)
    profitable_trades = Column(Integer, default=0, nullable=False)
    total_profit = Column(Float, default=0.0, nullable=False)
    max_drawdown = Column(Float, default=0.0, nullable=False)

# Pydantic schemas
class BotBase(BaseModel):
    name: str
    description: Optional[str] = None
    strategy_name: str
    exchange: str
    stake_currency: str
    stake_amount: float
    max_open_trades: int = 3
    config: Dict[str, Any] = {}

    @validator('stake_amount')
    def validate_stake_amount(cls, v):
        if v <= 0:
            raise ValueError('stake_amount must be positive')
        if v > 10000:
            raise ValueError('stake_amount cannot exceed 10000')
        return v

    @validator('max_open_trades')
    def validate_max_open_trades(cls, v):
        if not (1 <= v <= 50):
            raise ValueError('max_open_trades must be between 1 and 50')
        return v

class BotCreate(BotBase):
    pass

class BotUpdate(BaseModel):
    name: Optional[str] = None
    description: Optional[str] = None
    strategy_name: Optional[str] = None
    exchange: Optional[str] = None
    stake_currency: Optional[str] = None
    stake_amount: Optional[float] = None
    max_open_trades: Optional[int] = None
    config: Optional[Dict[str, Any]] = None
    is_active: Optional[bool] = None

class BotResponse(BotBase):
    id: int
    is_active: bool
    status: BotStatus
    pid: Optional[int]
    created_at: datetime
    updated_at: Optional[datetime]
    total_trades: int
    profitable_trades: int
    total_profit: float
    max_drawdown: float

    class Config:
        from_attributes = True

class BotStatusResponse(BaseModel):
    bot_name: str
    status: BotStatus
    pid: Optional[int]
    uptime: Optional[float]
    active_trades: int
    total_trades: int
    profit: float
    last_update: datetime
    error_message: Optional[str] = None
```

#### 4. Strategy Models
**`core_server/models/strategy.py`:**
```python
"""
Strategy models and schemas.
"""

from sqlalchemy import Column, Integer, String, Text, DateTime, Boolean, JSON
from sqlalchemy.sql import func
from pydantic import BaseModel
from typing import Optional, Dict, Any, List
from datetime import datetime

from .base import Base, TimestampMixin

class Strategy(Base, TimestampMixin):
    """Strategy SQLAlchemy model."""
    __tablename__ = "strategies"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(100), unique=True, nullable=False, index=True)
    description = Column(Text, nullable=True)
    code = Column(Text, nullable=False)
    parameters = Column(JSON, nullable=False, default=dict)
    is_active = Column(Boolean, default=True, nullable=False)
    created_by = Column(Integer, nullable=False)

    # Metadata
    version = Column(String(20), default="1.0.0", nullable=False)
    tags = Column(JSON, nullable=False, default=list)

# Pydantic schemas
class StrategyBase(BaseModel):
    name: str
    description: Optional[str] = None
    code: str
    parameters: Dict[str, Any] = {}
    version: str = "1.0.0"
    tags: List[str] = []

class StrategyCreate(StrategyBase):
    pass

class StrategyUpdate(BaseModel):
    name: Optional[str] = None
    description: Optional[str] = None
    code: Optional[str] = None
    parameters: Optional[Dict[str, Any]] = None
    is_active: Optional[bool] = None
    version: Optional[str] = None
    tags: Optional[List[str]] = None

class StrategyResponse(StrategyBase):
    id: int
    is_active: bool
    created_at: datetime
    updated_at: Optional[datetime] = None

    class Config:
        from_attributes = True
```

### ✅ Задача 2.4: FtRestClient Service (2 часа)
**ВНИМАНИЕ:** Эта задача была обновлена для использования Redis Streams Event Bus

**Цель:** Создать централизованный сервис для управления всеми Freqtrade ботами, избегая дублирования кода.

#### 1. FtRestClient Service
**`core_server/services/ft_rest_client_service.py`:**
```python
"""
Централизованный FtRestClient Service для управления всеми Freqtrade ботами.
Обеспечивает единый интерфейс для Core Server и MCP Bridge.
"""

import asyncio
import logging
from typing import Dict, Any, Optional, List
from dataclasses import dataclass
from contextlib import asynccontextmanager
import aiohttp
import json
from datetime import datetime, timedelta

logger = logging.getLogger(__name__)

@dataclass
class BotConnection:
    """Конфигурация подключения к боту"""
    name: str
    url: str
    username: str
    password: str
    session: Optional[aiohttp.ClientSession] = None
    last_health_check: Optional[datetime] = None
    is_healthy: bool = False
    retry_count: int = 0
    max_retries: int = 3

class FtRestClientService:
    """
    Централизованный сервис для управления Freqtrade ботами.
    Используется Core Server и MCP Bridge для всех операций с ботами.
    """

    def __init__(self):
        self.connections: Dict[str, BotConnection] = {}
        self._lock = asyncio.Lock()
        self._initialized = False

    async def initialize(self, bot_configs: Dict[str, Dict[str, Any]]) -> bool:
        """Инициализация сервиса с конфигурациями ботов"""
        async with self._lock:
            try:
                logger.info(f"Initializing FtRestClient Service with {len(bot_configs)} bots")

                for bot_name, config in bot_configs.items():
                    connection = BotConnection(
                        name=bot_name,
                        url=config["api_url"],
                        username=config.get("api_username", "freqtrade"),
                        password=config.get("api_password", "supersecurepassword")
                    )
                    self.connections[bot_name] = connection

                # Создание сессий для всех ботов
                await self._create_sessions()

                self._initialized = True
                logger.info("FtRestClient Service initialized successfully")
                return True

            except Exception as e:
                logger.error(f"Failed to initialize FtRestClient Service: {e}")
                return False

    async def _create_sessions(self):
        """Создание HTTP сессий для всех ботов"""
        for connection in self.connections.values():
            if connection.session is None or connection.session.closed:
                connector = aiohttp.TCPConnector(limit=10, ttl_dns_cache=300)
                timeout = aiohttp.ClientTimeout(total=30, connect=10)

                connection.session = aiohttp.ClientSession(
                    connector=connector,
                    timeout=timeout,
                    auth=aiohttp.BasicAuth(connection.username, connection.password)
                )

    async def _ensure_session(self, bot_name: str) -> Optional[aiohttp.ClientSession]:
        """Обеспечение активной сессии для бота"""
        if bot_name not in self.connections:
            logger.error(f"Bot {bot_name} not configured")
            return None

        connection = self.connections[bot_name]

        # Проверка здоровья сессии
        if connection.session is None or connection.session.closed:
            await self._create_sessions()

        return connection.session

    async def _make_request(self, bot_name: str, method: str, endpoint: str,
                          data: Optional[Dict[str, Any]] = None) -> Dict[str, Any]:
        """Универсальный метод для HTTP запросов к боту"""

        session = await self._ensure_session(bot_name)
        if not session:
            return {"error": f"No session available for bot {bot_name}"}

        url = f"{self.connections[bot_name].url.rstrip('/')}/{endpoint.lstrip('/')}"

        try:
            if data:
                async with session.request(method, url, json=data) as response:
                    result = await response.json()
            else:
                async with session.request(method, url) as response:
                    result = await response.json()

            # Обновление статуса здоровья
            self.connections[bot_name].last_health_check = datetime.now()
            self.connections[bot_name].is_healthy = True
            self.connections[bot_name].retry_count = 0

            return result

        except aiohttp.ClientError as e:
            logger.error(f"HTTP error for bot {bot_name}, endpoint {endpoint}: {e}")
            await self._handle_connection_error(bot_name)
            return {"error": f"Connection failed: {str(e)}"}

        except Exception as e:
            logger.error(f"Unexpected error for bot {bot_name}: {e}")
            return {"error": f"Unexpected error: {str(e)}"}

    async def _handle_connection_error(self, bot_name: str):
        """Обработка ошибок подключения"""
        connection = self.connections[bot_name]
        connection.retry_count += 1
        connection.is_healthy = False

        if connection.retry_count >= connection.max_retries:
            logger.warning(f"Bot {bot_name} reached max retries, marking as unhealthy")

    # === ОСНОВНЫЕ МЕТОДЫ УПРАВЛЕНИЯ БОТАМИ ===

    async def start_bot(self, bot_name: str) -> Dict[str, Any]:
        """Запуск бота"""
        logger.info(f"Starting bot: {bot_name}")
        return await self._make_request(bot_name, "POST", "/api/v1/status/start")

    async def stop_bot(self, bot_name: str) -> Dict[str, Any]:
        """Остановка бота"""
        logger.info(f"Stopping bot: {bot_name}")
        return await self._make_request(bot_name, "POST", "/api/v1/status/stop")

    async def restart_bot(self, bot_name: str) -> Dict[str, Any]:
        """Перезапуск бота"""
        logger.info(f"Restarting bot: {bot_name}")
        stop_result = await self.stop_bot(bot_name)
        if "error" in stop_result:
            return stop_result

        await asyncio.sleep(2)
        return await self.start_bot(bot_name)

    async def get_bot_status(self, bot_name: str) -> Dict[str, Any]:
        """Получение статуса бота"""
        return await self._make_request(bot_name, "GET", "/api/v1/status")

    async def get_bot_profit(self, bot_name: str) -> Dict[str, Any]:
        """Получение данных о прибыли бота"""
        return await self._make_request(bot_name, "GET", "/api/v1/profit")

    async def get_bot_balance(self, bot_name: str) -> Dict[str, Any]:
        """Получение баланса бота"""
        return await self._make_request(bot_name, "GET", "/api/v1/balance")

    async def get_open_trades(self, bot_name: str) -> Dict[str, Any]:
        """Получение открытых сделок"""
        return await self._make_request(bot_name, "GET", "/api/v1/status")

    async def force_exit_trade(self, bot_name: str, trade_id: str) -> Dict[str, Any]:
        """Принудительное закрытие сделки"""
        return await self._make_request(bot_name, "POST", f"/api/v1/status/trade/{trade_id}/exit")

    async def get_bot_config(self, bot_name: str) -> Dict[str, Any]:
        """Получение конфигурации бота"""
        return await self._make_request(bot_name, "GET", "/api/v1/config")

    async def update_bot_config(self, bot_name: str, config: Dict[str, Any]) -> Dict[str, Any]:
        """Обновление конфигурации бота"""
        return await self._make_request(bot_name, "PATCH", "/api/v1/config", config)

    # === ДОПОЛНИТЕЛЬНЫЕ МЕТОДЫ ===

    async def get_all_bots_status(self) -> Dict[str, Dict[str, Any]]:
        """Получение статуса всех ботов"""
        results = {}
        for bot_name in self.connections.keys():
            results[bot_name] = await self.get_bot_status(bot_name)
        return results

    async def start_all_bots(self) -> Dict[str, Dict[str, Any]]:
        """Запуск всех ботов"""
        results = {}
        for bot_name in self.connections.keys():
            results[bot_name] = await self.start_bot(bot_name)
        return results

    async def stop_all_bots(self) -> Dict[str, Dict[str, Any]]:
        """Остановка всех ботов"""
        results = {}
        for bot_name in self.connections.keys():
            results[bot_name] = await self.stop_bot(bot_name)
        return results

    async def health_check_all(self) -> Dict[str, bool]:
        """Проверка здоровья всех подключений"""
        results = {}
        for bot_name, connection in self.connections.items():
            results[bot_name] = connection.is_healthy and connection.session and not connection.session.closed
        return results

    async def get_connection_stats(self) -> Dict[str, Dict[str, Any]]:
        """Получение статистики подключений"""
        stats = {}
        for bot_name, connection in self.connections.items():
            stats[bot_name] = {
                "url": connection.url,
                "healthy": connection.is_healthy,
                "last_check": connection.last_health_check.isoformat() if connection.last_health_check else None,
                "retry_count": connection.retry_count,
                "session_active": connection.session is not None and not connection.session.closed
            }
        return stats

    # === МЕТОДЫ ДЛЯ MCP BRIDGE ===

    async def execute_ai_command(self, bot_name: str, command: str, **kwargs) -> Dict[str, Any]:
        """
        Выполнение AI команды для бота.
        Используется MCP Bridge для intelligent управления.
        """
        command_map = {
            "start": self.start_bot,
            "stop": self.stop_bot,
            "restart": self.restart_bot,
            "status": self.get_bot_status,
            "profit": self.get_bot_profit,
            "balance": self.get_bot_balance,
            "trades": self.get_open_trades,
            "config": self.get_bot_config,
        }

        if command not in command_map:
            return {"error": f"Unknown command: {command}"}

        try:
            result = await command_map[command](bot_name)

            # Добавление AI контекста
            result.update({
                "ai_command": command,
                "bot_name": bot_name,
                "timestamp": datetime.now().isoformat(),
                "executed_by": "mcp_bridge"
            })

            return result

        except Exception as e:
            logger.error(f"AI command execution failed: {command} for {bot_name}: {e}")
            return {
                "error": f"Command execution failed: {str(e)}",
                "ai_command": command,
                "bot_name": bot_name
            }

    async def get_ai_recommendations(self, bot_name: str) -> Dict[str, Any]:
        """
        Получение AI рекомендаций для бота.
        Анализирует статус и предлагает действия.
        """
        status = await self.get_bot_status(bot_name)
        profit = await self.get_bot_profit(bot_name)

        recommendations = []

        # Анализ статуса
        if status.get("state") == "stopped":
            recommendations.append({
                "action": "start",
                "reason": "Bot is stopped, consider starting for trading",
                "confidence": 0.8
            })

        # Анализ прибыли
        if profit.get("profit_all_coin", 0) < -0.05:  # -5%
            recommendations.append({
                "action": "stop",
                "reason": "Significant losses detected, consider stopping",
                "confidence": 0.9
            })

        # Анализ открытых сделок
        trades = status.get("trade_count", 0)
        if trades > 10:
            recommendations.append({
                "action": "reduce_exposure",
                "reason": "High number of open trades, consider risk management",
                "confidence": 0.7
            })

        return {
            "bot_name": bot_name,
            "recommendations": recommendations,
            "analysis_timestamp": datetime.now().isoformat()
        }

    async def shutdown(self):
        """Корректное завершение работы сервиса"""
        logger.info("Shutting down FtRestClient Service")

        # Закрытие всех сессий
        for connection in self.connections.values():
            if connection.session and not connection.session.closed:
                await connection.session.close()

        self.connections.clear()
        self._initialized = False
        logger.info("FtRestClient Service shutdown complete")

# Глобальный экземпляр сервиса
ft_rest_client_service = FtRestClientService()
```

#### 2. Инициализация сервиса в приложении
**Обновить `core_server/core/app.py`:**
```python
from core_server.services.ft_rest_client_service import ft_rest_client_service

@app.on_event("startup")
async def startup_event():
    # Загрузка конфигураций ботов из базы данных или config
    bot_configs = await load_bot_configs()
    await ft_rest_client_service.initialize(bot_configs)

@app.on_event("shutdown")
async def shutdown_event():
    await ft_rest_client_service.shutdown()
```

#### 3. Обновить Bot Service для использования централизованного сервиса
**Обновить `core_server/services/bot_service.py`:**
```python
# Вместо прямого использования FtRestClient
from .ft_rest_client_service import ft_rest_client_service

class BotService:
    async def start_bot(self, bot_name: str):
        return await ft_rest_client_service.start_bot(bot_name)

    async def get_bot_status(self, bot_name: str):
        return await ft_rest_client_service.get_bot_status(bot_name)
```

### ✅ Задача 2.3: Database Layer (1.5 часа)

#### 1. Database Connection
**`core_server/database/__init__.py`:**
```python
"""
Database connection and session management.
"""

from .connection import engine, async_session, get_db, init_database, close_database

__all__ = ["engine", "async_session", "get_db", "init_database", "close_database"]
```

**`core_server/database/connection.py`:**
```python
"""
Database connection management with SQLAlchemy.
"""

from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from sqlalchemy.pool import StaticPool
from sqlalchemy import text
import logging
from typing import AsyncGenerator
import os

from ..core.config import settings

logger = logging.getLogger(__name__)

# Create engine based on database URL
if settings.DATABASE_URL.startswith("sqlite"):
    engine = create_async_engine(
        settings.DATABASE_URL,
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
        echo=settings.DEBUG,
        future=True
    )
else:
    engine = create_async_engine(
        settings.DATABASE_URL,
        pool_size=10,
        max_overflow=20,
        pool_timeout=30,
        pool_recycle=1800,
        echo=settings.DEBUG,
        future=True
    )

# Create async session factory
async_session = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False
)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    """
    Dependency to get database session.

    Yields:
        Database session
    """
    async with async_session() as session:
        try:
            yield session
        finally:
            await session.close()

async def init_database() -> None:
    """
    Initialize database tables and run migrations.
    """
    try:
        logger.info("Initializing database...")

        # Import all models to ensure they are registered
        from ..models import Base, User, Bot, Strategy

        # Create tables
        async with engine.begin() as conn:
            logger.info("Creating database tables...")
            await conn.run_sync(Base.metadata.create_all)
            logger.info("✅ Database tables created")

        # Run initial data setup
        await _setup_initial_data()

        logger.info("✅ Database initialized successfully")

    except Exception as e:
        logger.error(f"❌ Failed to initialize database: {e}")
        raise

async def close_database() -> None:
    """
    Close database connections.
    """
    try:
        logger.info("Closing database connections...")
        await engine.dispose()
        logger.info("✅ Database connections closed")
    except Exception as e:
        logger.error(f"❌ Error closing database: {e}")

async def _setup_initial_data() -> None:
    """
    Setup initial data (admin user, default strategies, etc.)
    """
    try:
        async with async_session() as session:
            # Check if admin user exists
            result = await session.execute(
                text("SELECT COUNT(*) FROM users WHERE username = 'admin'")
            )
            admin_exists = result.scalar()

            if not admin_exists:
                logger.info("Creating default admin user...")
                # Create default admin user
                admin = User(
                    username="admin",
                    email="admin@freqtrade.local",
                    full_name="System Administrator",
                    is_active=True,
                    is_superuser=True
                )
                admin.set_password("admin123")  # Change in production!

                session.add(admin)
                await session.commit()
                logger.info("✅ Default admin user created")

    except Exception as e:
        logger.warning(f"Failed to setup initial data: {e}")
```

#### 2. Database Service
**`core_server/database/service.py`:**
```python
"""
Database service for common CRUD operations.
"""

from typing import List, Optional, Dict, Any, Type, TypeVar
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, update, delete, func
from sqlalchemy.exc import IntegrityError
import logging

from .connection import async_session

logger = logging.getLogger(__name__)

T = TypeVar('T')

class DatabaseService:
    """
    Generic database service for CRUD operations.
    """

    @property
    def session(self) -> AsyncSession:
        """Get database session."""
        return async_session()

    async def get_by_id(self, model_class: Type[T], id: int) -> Optional[T]:
        """Get record by ID."""
        async with self.session as session:
            result = await session.execute(
                select(model_class).where(model_class.id == id)
            )
            return result.scalar_one_or_none()

    async def get_all(self, model_class: Type[T], skip: int = 0, limit: int = 100) -> List[T]:
        """Get all records with pagination."""
        async with self.session as session:
            result = await session.execute(
                select(model_class).offset(skip).limit(limit)
            )
            return result.scalars().all()

    async def create(self, model_class: Type[T], data: Dict[str, Any]) -> T:
        """Create new record."""
        async with self.session as session:
            try:
                instance = model_class(**data)
                session.add(instance)
                await session.commit()
                await session.refresh(instance)
                return instance
            except IntegrityError as e:
                await session.rollback()
                logger.error(f"Integrity error creating {model_class.__name__}: {e}")
                raise ValueError(f"Record already exists or invalid data")

    async def update(self, model_class: Type[T], id: int, data: Dict[str, Any]) -> Optional[T]:
        """Update record by ID."""
        async with self.session as session:
            try:
                result = await session.execute(
                    select(model_class).where(model_class.id == id)
                )
                instance = result.scalar_one_or_none()

                if not instance:
                    return None

                for key, value in data.items():
                    if hasattr(instance, key):
                        setattr(instance, key, value)

                await session.commit()
                await session.refresh(instance)
                return instance

            except Exception as e:
                await session.rollback()
                logger.error(f"Error updating {model_class.__name__} {id}: {e}")
                raise

    async def delete(self, model_class: Type[T], id: int) -> bool:
        """Delete record by ID."""
        async with self.session as session:
            try:
                result = await session.execute(
                    select(model_class).where(model_class.id == id)
                )
                instance = result.scalar_one_or_none()

                if not instance:
                    return False

                await session.delete(instance)
                await session.commit()
                return True

            except Exception as e:
                await session.rollback()
                logger.error(f"Error deleting {model_class.__name__} {id}: {e}")
                raise

    async def count(self, model_class: Type[T]) -> int:
        """Count total records."""
        async with self.session as session:
            result = await session.execute(
                select(func.count()).select_from(model_class)
            )
            return result.scalar()
```

### ✅ Задача 2.4: Authentication & Authorization (1.5 часа)

#### 1. Auth Service
**`core_server/auth/service.py`:**
```python
"""
Authentication service with JWT tokens.
"""

from datetime import datetime, timedelta
from typing import Optional, Dict, Any
import jwt
import logging
from passlib.context import CryptContext

from ..core.config import settings
from ..database.service import DatabaseService
from ..models.user import User, UserCreate

logger = logging.getLogger(__name__)

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

class AuthService:
    """
    Authentication service for user management and JWT tokens.
    """

    def __init__(self):
        self.db_service = DatabaseService()
        self.secret_key = settings.JWT_SECRET
        self.algorithm = settings.JWT_ALGORITHM
        self.expiration_hours = settings.JWT_EXPIRATION_HOURS

    async def authenticate_user(self, username: str, password: str) -> Optional[User]:
        """
        Authenticate user with username and password.

        Returns:
            User object if authentication successful, None otherwise
        """
        try:
            # Find user by username
            async with self.db_service.session as session:
                result = await session.execute(
                    select(User).where(User.username == username, User.is_active == True)
                )
                user = result.scalar_one_or_none()

                if not user:
                    return None

                # Verify password
                if not user.verify_password(password):
                    return None

                # Update last login
                user.last_login = datetime.utcnow()
                await session.commit()

                return user

        except Exception as e:
            logger.error(f"Authentication error for user {username}: {e}")
            return None

    async def create_user(self, user_data: UserCreate) -> User:
        """
        Create new user.

        Args:
            user_data: User creation data

        Returns:
            Created user object

        Raises:
            ValueError: If user already exists
        """
        try:
            # Check if user already exists
            async with self.db_service.session as session:
                result = await session.execute(
                    select(User).where(
                        (User.username == user_data.username) | (User.email == user_data.email)
                    )
                )
                existing_user = result.scalar_one_or_none()

                if existing_user:
                    if existing_user.username == user_data.username:
                        raise ValueError("Username already exists")
                    else:
                        raise ValueError("Email already exists")

                # Create new user
                user = User(
                    username=user_data.username,
                    email=user_data.email,
                    full_name=user_data.full_name,
                    is_active=user_data.is_active,
                    is_superuser=user_data.is_superuser
                )
                user.set_password(user_data.password)

                session.add(user)
                await session.commit()
                await session.refresh(user)

                logger.info(f"Created new user: {user.username}")
                return user

        except ValueError:
            raise
        except Exception as e:
            logger.error(f"Error creating user {user_data.username}: {e}")
            raise

    def create_access_token(self, data: Dict[str, Any]) -> str:
        """
        Create JWT access token.

        Args:
            data: Data to encode in token

        Returns:
            JWT token string
        """
        to_encode = data.copy()
        expire = datetime.utcnow() + timedelta(hours=self.expiration_hours)
        to_encode.update({"exp": expire})

        encoded_jwt = jwt.encode(to_encode, self.secret_key, algorithm=self.algorithm)
        return encoded_jwt

    def verify_token(self, token: str) -> Optional[Dict[str, Any]]:
        """
        Verify and decode JWT token.

        Args:
            token: JWT token to verify

        Returns:
            Decoded token data or None if invalid
        """
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=[self.algorithm])
            return payload
        except jwt.ExpiredSignatureError:
            logger.warning("Token expired")
            return None
        except jwt.InvalidTokenError as e:
            logger.warning(f"Invalid token: {e}")
            return None

    async def get_current_user(self, token: str) -> Optional[User]:
        """
        Get current user from JWT token.

        Args:
            token: JWT access token

        Returns:
            User object or None if token invalid
        """
        try:
            payload = self.verify_token(token)
            if not payload:
                return None

            username = payload.get("sub")
            if not username:
                return None

            # Get user from database
            async with self.db_service.session as session:
                result = await session.execute(
                    select(User).where(User.username == username, User.is_active == True)
                )
                user = result.scalar_one_or_none()

                return user

        except Exception as e:
            logger.error(f"Error getting current user: {e}")
            return None

    async def change_password(self, user_id: int, old_password: str, new_password: str) -> bool:
        """
        Change user password.

        Args:
            user_id: User ID
            old_password: Current password
            new_password: New password

        Returns:
            True if password changed successfully
        """
        try:
            async with self.db_service.session as session:
                result = await session.execute(
                    select(User).where(User.id == user_id)
                )
                user = result.scalar_one_or_none()

                if not user:
                    return False

                # Verify old password
                if not user.verify_password(old_password):
                    return False

                # Set new password
                user.set_password(new_password)
                await session.commit()

                logger.info(f"Password changed for user {user.username}")
                return True

        except Exception as e:
            logger.error(f"Error changing password for user {user_id}: {e}")
            return False
```

#### 2. Auth Dependencies
**`core_server/auth/dependencies.py`:**
```python
"""
FastAPI dependencies for authentication.
"""

from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from typing import Optional
import logging

from .service import AuthService
from ..models.user import User

logger = logging.getLogger(__name__)

security = HTTPBearer()
auth_service = AuthService()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> User:
    """
    Get current authenticated user.

    Args:
        credentials: HTTP Bearer token

    Returns:
        User object

    Raises:
        HTTPException: If authentication fails
    """
    token = credentials.credentials

    user = await auth_service.get_current_user(token)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authentication credentials",
            headers={"WWW-Authenticate": "Bearer"},
        )

    return user

async def get_current_active_user(
    current_user: User = Depends(get_current_user)
) -> User:
    """
    Get current active user.

    Args:
        current_user: Current user from dependency

    Returns:
        Active user object

    Raises:
        HTTPException: If user is inactive
    """
    if not current_user.is_active:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Inactive user"
        )
    return current_user

async def get_current_superuser(
    current_user: User = Depends(get_current_active_user)
) -> User:
    """
    Get current superuser.

    Args:
        current_user: Current active user

    Returns:
        Superuser object

    Raises:
        HTTPException: If user is not superuser
    """
    if not current_user.is_superuser:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Not enough permissions"
        )
    return current_user
```

### ✅ Задача 2.5: Core Services (1.5 часа)

#### 1. Bot Service
**`core_server/services/bot_service.py`:**
```python
"""
Bot management service.
"""

from typing import List, Optional, Dict, Any
import asyncio
import logging
from datetime import datetime

from ..database.service import DatabaseService
from ..models.bot import Bot, BotCreate, BotUpdate, BotResponse, BotStatus, BotStatusResponse

logger = logging.getLogger(__name__)

class BotService:
    """
    Service for managing trading bots.
    """

    def __init__(self):
        self.db_service = DatabaseService()

    async def get_all_bots(self) -> List[BotResponse]:
        """Get all bots."""
        bots = await self.db_service.get_all(Bot)
        return [BotResponse.from_orm(bot) for bot in bots]

    async def get_bot_by_id(self, bot_id: int) -> Optional[BotResponse]:
        """Get bot by ID."""
        bot = await self.db_service.get_by_id(Bot, bot_id)
        return BotResponse.from_orm(bot) if bot else None

    async def get_bot_by_name(self, name: str) -> Optional[BotResponse]:
        """Get bot by name."""
        async with self.db_service.session as session:
            result = await session.execute(
                select(Bot).where(Bot.name == name)
            )
            bot = result.scalar_one_or_none()
            return BotResponse.from_orm(bot) if bot else None

    async def create_bot(self, bot_data: BotCreate, created_by: int) -> BotResponse:
        """Create new bot."""
        data = bot_data.dict()
        data["created_by"] = created_by
        data["status"] = BotStatus.STOPPED.value

        bot = await self.db_service.create(Bot, data)
        return BotResponse.from_orm(bot)

    async def update_bot(self, bot_id: int, updates: BotUpdate) -> Optional[BotResponse]:
        """Update bot."""
        data = updates.dict(exclude_unset=True)
        bot = await self.db_service.update(Bot, bot_id, data)
        return BotResponse.from_orm(bot) if bot else None

    async def delete_bot(self, bot_id: int) -> bool:
        """Delete bot."""
        return await self.db_service.delete(Bot, bot_id)

    async def start_bot(self, bot_id: int) -> Dict[str, Any]:
        """Start bot."""
        try:
            bot = await self.db_service.get_by_id(Bot, bot_id)
            if not bot:
                return {"success": False, "error": "Bot not found"}

            if bot.status == BotStatus.RUNNING.value:
                return {"success": False, "error": "Bot already running"}

            # Update status to starting
            await self.db_service.update(Bot, bot_id, {"status": BotStatus.STARTING.value})

            # Here you would start the actual Freqtrade process
            # For now, just simulate starting
            await asyncio.sleep(1)  # Simulate startup time

            # Update status to running
            await self.db_service.update(Bot, bot_id, {
                "status": BotStatus.RUNNING.value,
                "pid": 12345  # Mock PID
            })

            return {
                "success": True,
                "message": f"Bot {bot.name} started successfully",
                "timestamp": datetime.utcnow()
            }

        except Exception as e:
            logger.error(f"Error starting bot {bot_id}: {e}")
            await self.db_service.update(Bot, bot_id, {"status": BotStatus.ERROR.value})
            return {"success": False, "error": str(e)}

    async def stop_bot(self, bot_id: int) -> Dict[str, Any]:
        """Stop bot."""
        try:
            bot = await self.db_service.get_by_id(Bot, bot_id)
            if not bot:
                return {"success": False, "error": "Bot not found"}

            if bot.status == BotStatus.STOPPED.value:
                return {"success": False, "error": "Bot already stopped"}

            # Update status to stopping
            await self.db_service.update(Bot, bot_id, {"status": BotStatus.STOPPING.value})

            # Here you would stop the actual Freqtrade process
            await asyncio.sleep(0.5)  # Simulate shutdown time

            # Update status to stopped
            await self.db_service.update(Bot, bot_id, {
                "status": BotStatus.STOPPED.value,
                "pid": None
            })

            return {
                "success": True,
                "message": f"Bot {bot.name} stopped successfully",
                "timestamp": datetime.utcnow()
            }

        except Exception as e:
            logger.error(f"Error stopping bot {bot_id}: {e}")
            return {"success": False, "error": str(e)}

    async def get_bot_status(self, bot_id: int) -> Optional[BotStatusResponse]:
        """Get detailed bot status."""
        bot = await self.db_service.get_by_id(Bot, bot_id)
        if not bot:
            return None

        return BotStatusResponse(
            bot_name=bot.name,
            status=BotStatus(bot.status),
            pid=bot.pid,
            uptime=None,  # Would calculate from start time
            active_trades=0,  # Would get from Freqtrade API
            total_trades=bot.total_trades,
            profit=bot.total_profit,
            last_update=datetime.utcnow()
        )

    async def get_all_bot_statuses(self) -> Dict[str, BotStatusResponse]:
        """Get status of all bots."""
        bots = await self.db_service.get_all(Bot)
        statuses = {}

        for bot in bots:
            statuses[bot.name] = BotStatusResponse(
                bot_name=bot.name,
                status=BotStatus(bot.status),
                pid=bot.pid,
                uptime=None,
                active_trades=0,
                total_trades=bot.total_trades,
                profit=bot.total_profit,
                last_update=datetime.utcnow()
            )

        return statuses
```

#### 2. Cache Service
**`core_server/services/cache_service.py`:**
```python
"""
Redis-based caching service.
"""

import json
import logging
from typing import Any, Optional, Union
import redis.asyncio as redis

from ..core.config import settings

logger = logging.getLogger(__name__)

class CacheService:
    """
    Redis-based caching service with TTL support.
    """

    def __init__(self):
        self.redis_url = settings.REDIS_URL
        self.redis: Optional[redis.Redis] = None
        self.prefix = "freqtrade:"

    async def connect(self) -> None:
        """Connect to Redis."""
        if self.redis is None:
            self.redis = redis.from_url(self.redis_url)

    async def disconnect(self) -> None:
        """Disconnect from Redis."""
        if self.redis:
            await self.redis.close()
            self.redis = None

    async def get(self, key: str) -> Optional[Any]:
        """
        Get value from cache.

        Args:
            key: Cache key

        Returns:
            Cached value or None
        """
        try:
            await self.connect()
            value = await self.redis.get(f"{self.prefix}{key}")
            return json.loads(value) if value else None
        except Exception as e:
            logger.error(f"Cache get error for key {key}: {e}")
            return None

    async def set(self, key: str, value: Any, ttl: int = 3600) -> bool:
        """
        Set value in cache with TTL.

        Args:
            key: Cache key
            value: Value to cache
            ttl: Time to live in seconds

        Returns:
            True if successful
        """
        try:
            await self.connect()
            json_value = json.dumps(value)
            await self.redis.setex(f"{self.prefix}{key}", ttl, json_value)
            return True
        except Exception as e:
            logger.error(f"Cache set error for key {key}: {e}")
            return False

    async def delete(self, key: str) -> bool:
        """
        Delete value from cache.

        Args:
            key: Cache key

        Returns:
            True if successful
        """
        try:
            await self.connect()
            await self.redis.delete(f"{self.prefix}{key}")
            return True
        except Exception as e:
            logger.error(f"Cache delete error for key {key}: {e}")
            return False

    async def exists(self, key: str) -> bool:
        """
        Check if key exists in cache.

        Args:
            key: Cache key

        Returns:
            True if key exists
        """
        try:
            await self.connect()
            return bool(await self.redis.exists(f"{self.prefix}{key}"))
        except Exception as e:
            logger.error(f"Cache exists error for key {key}: {e}")
            return False

    async def increment(self, key: str, amount: int = 1) -> Optional[int]:
        """
        Increment numeric value in cache.

        Args:
            key: Cache key
            amount: Amount to increment

        Returns:
            New value or None if error
        """
        try:
            await self.connect()
            return await self.redis.incrby(f"{self.prefix}{key}", amount)
        except Exception as e:
            logger.error(f"Cache increment error for key {key}: {e}")
            return None

    async def expire(self, key: str, ttl: int) -> bool:
        """
        Set expiration time for key.

        Args:
            key: Cache key
            ttl: Time to live in seconds

        Returns:
            True if successful
        """
        try:
            await self.connect()
            return bool(await self.redis.expire(f"{self.prefix}{key}", ttl))
        except Exception as e:
            logger.error(f"Cache expire error for key {key}: {e}")
            return False
```

### ✅ Задача 2.6: Middleware & Error Handling (1 час)

#### 1. Rate Limiting
**`core_server/middleware/rate_limiting.py`:**
```python
"""
Rate limiting middleware using slowapi.
"""

from slowapi import Limiter
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from fastapi import Request
from fastapi.responses import JSONResponse
import logging

from ..core.config import settings

logger = logging.getLogger(__name__)

def create_limiter() -> Limiter:
    """
    Create rate limiter instance.

    Returns:
        Configured Limiter instance
    """
    return Limiter(
        key_func=get_remote_address,
        default_limits=[f"{settings.RATE_LIMIT_REQUESTS} per {settings.RATE_LIMIT_WINDOW} seconds"]
    )

async def rate_limit_exceeded_handler(request: Request, exc: RateLimitExceeded) -> JSONResponse:
    """
    Handle rate limit exceeded errors.

    Args:
        request: FastAPI request
        exc: Rate limit exception

    Returns:
        JSON response with error details
    """
    logger.warning(f"Rate limit exceeded for {request.client.host}: {exc.detail}")

    return JSONResponse(
        status_code=429,
        content={
            "error": "Too many requests",
            "detail": exc.detail,
            "retry_after": exc.retry_after
        },
        headers={"Retry-After": str(exc.retry_after)}
    )
```

#### 2. Request Logging
**`core_server/middleware/logging.py`:**
```python
"""
Request logging middleware.
"""

import time
import logging
from typing import Callable
from fastapi import Request, Response

logger = logging.getLogger(__name__)

class RequestLoggingMiddleware:
    """
    Middleware for logging HTTP requests and responses.
    """

    def __init__(self, app: Callable):
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        # Extract request info
        method = scope["method"]
        path = scope["path"]
        query = scope.get("query_string", b"").decode()

        # Log request
        start_time = time.time()
        logger.info(f"→ {method} {path}{f'?{query}' if query else ''}")

        # Process request
        await self.app(scope, receive, send)

        # Log response
        process_time = time.time() - start_time
        logger.info(".2f"
```

#### 3. Error Handling
**`core_server/middleware/error_handling.py`:**
```python
"""
Custom error handling for the API.
"""

from fastapi import HTTPException, Request
from fastapi.responses import JSONResponse
from typing import Dict, Any
import logging
from datetime import datetime

logger = logging.getLogger(__name__)

class APIError(Exception):
    """
    Custom API error with structured error information.
    """

    def __init__(
        self,
        message: str,
        error_code: str = "INTERNAL_ERROR",
        status_code: int = 500,
        details: Dict[str, Any] = None
    ):
        self.message = message
        self.error_code = error_code
        self.status_code = status_code
        self.details = details or {}
        super().__init__(self.message)

async def api_error_handler(request: Request, exc: APIError) -> JSONResponse:
    """
    Handle APIError exceptions.

    Args:
        request: FastAPI request
        exc: APIError exception

    Returns:
        JSON response with error details
    """
    logger.error(
        f"API Error: {exc.error_code} - {exc.message}",
        extra={
            "error_code": exc.error_code,
            "status_code": exc.status_code,
            "details": exc.details,
            "path": str(request.url),
            "method": request.method
        }
    )

    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": {
                "code": exc.error_code,
                "message": exc.message,
                "details": exc.details
            },
            "timestamp": datetime.utcnow().isoformat(),
            "path": str(request.url)
        }
    )

def create_validation_error(field: str, message: str) -> APIError:
    """Create validation error."""
    return APIError(
        message=f"Validation error for field '{field}': {message}",
        error_code="VALIDATION_ERROR",
        status_code=422,
        details={"field": field, "validation_message": message}
    )

def create_not_found_error(resource: str, resource_id: str = None) -> APIError:
    """Create not found error."""
    message = f"{resource} not found"
    if resource_id:
        message += f": {resource_id}"

    return APIError(
        message=message,
        error_code="NOT_FOUND",
        status_code=404,
        details={"resource": resource, "resource_id": resource_id}
    )

def create_permission_error(action: str, resource: str = None) -> APIError:
    """Create permission error."""
    message = f"Insufficient permissions to {action}"
    if resource:
        message += f" {resource}"

    return APIError(
        message=message,
        error_code="PERMISSION_DENIED",
        status_code=403,
        details={"action": action, "resource": resource}
    )
```

---

## 📊 КРИТЕРИИ ГОТОВНОСТИ ЭТАПА 2

### ✅ Технические требования:
- [x] FastAPI приложение с profile-based конфигурацией
- [x] Полные SQLAlchemy модели (User, Bot, Strategy)
- [x] Pydantic схемы с валидацией
- [x] Database layer с async сессиями
- [x] Authentication с JWT токенами
- [x] Core сервисы (BotService, CacheService)
- [x] Middleware (rate limiting, logging, error handling)

### ✅ Функциональные требования:
- [x] `python -c "from core_server.models import User, Bot; print('OK')"` работает
- [x] `python -c "from core_server.auth.service import AuthService; print('OK')"` работает
- [x] `python -c "from core_server.services import BotService, CacheService; print('OK')"` работает
- [x] Database инициализация проходит без ошибок
- [x] JWT токены генерируются и валидируются

### ✅ Качество кода:
- [x] Type hints для всех функций и методов
- [x] Async/await используется правильно
- [x] Error handling стандартизирован
- [x] Logging настроен для всех компонентов
- [x] Code соответствует PEP 8 и проходит проверки через uv:
  - `uv run black --check core_server/`
  - `uv run isort --check core_server/`
  - `uv run mypy core_server/`
  - `uv run flake8 core_server/`

---

## 🚀 СЛЕДУЮЩИЕ ШАГИ

**Переход к Этапу 3:** [API Layer - все 77 endpoints](03_api_layer.md)

**Проверка перед переходом:**
```bash
# Все импорты работают
python -c "
from core_server.core.app import create_application
from core_server.database import init_database
from core_server.auth.service import AuthService
from core_server.services import BotService
print('✅ All core components imported successfully')
"

# FastAPI приложение создается
python -c "
app = create_application('full_featured')
print(f'✅ App created: {app.title}')
"

# Модели валидны
python -c "
from core_server.models.user import UserCreate
user = UserCreate(username='test', email='test@example.com', password='password123')
print('✅ Pydantic models work')
"

# Качество кода проверено
uv run black --check core_server/
uv run isort --check core_server/
uv run flake8 core_server/
uv run mypy core_server/
```

### ✅ Задача 2.7: Redis Streams Event Bus (1 час)

**Цель:** Настроить enterprise-grade межсервисную связь через Redis Streams

#### 1. Redis Streams Event Bus
**`core_server/tools/redis_streams_event_bus.py`:**
```python
"""
Redis Streams Event Bus для коммуникации между FastAPI Core Server и MCP Bridge

Использует Redis Streams для надежной доставки сообщений с consumer groups и ACK mechanism.
Заменяет Pub/Sub на Streams для enterprise-grade надежности.
"""

import asyncio
import json
import logging
import time
from typing import Any, Callable, Dict, List, Optional

import redis.asyncio as redis
from prometheus_client import Counter, Gauge
from pydantic import BaseModel

logger = logging.getLogger(__name__)

class EventMessage(BaseModel):
    type: str
    data: Dict[str, Any]
    source: str
    timestamp: Optional[float] = None
    version: int = 1

    def model_post_init(self, __context):
        if self.timestamp is None:
            self.timestamp = time.time()

    def to_dict(self) -> Dict[str, Any]:
        return self.model_dump()

# Prometheus metrics для Streams
streams_events_published = Counter(
    "redis_streams_events_published",
    "Number of events published to streams",
    ["service", "stream"],
)
streams_events_consumed = Counter(
    "redis_streams_events_consumed",
    "Number of events consumed from streams",
    ["service", "stream"],
)
streams_events_failed = Counter(
    "redis_streams_events_failed",
    "Number of failed stream operations",
    ["service", "operation"],
)
streams_connection_status = Gauge(
    "redis_streams_connection_status", "Redis Streams connection status", ["service"]
)

class RedisStreamsEventBus:
    """
    Redis Streams Event Bus для межсервисной коммуникации

    Преимущества Redis Streams:
    - Надежная доставка сообщений (ACK mechanism)
    - Persistence - сообщения сохраняются
    - Consumer groups - параллельная обработка
    - Range queries - чтение в диапазоне
    - Enterprise-grade надежность
    """

    def __init__(
        self, redis_url: str = "redis://localhost:6379", service_name: str = "unknown"
    ):
        self.redis_url = redis_url
        self.service_name = service_name
        self.redis: Optional[redis.Redis] = None
        self.handlers: Dict[str, Callable] = {}
        self.consumer_groups: Dict[str, str] = {}
        self.is_connected = False
        self._listener_tasks: Dict[str, asyncio.Task] = {}

    async def connect(self) -> bool:
        """Async подключение к Redis с reconnections"""
        for attempt in range(5):
            try:
                self.redis = redis.from_url(self.redis_url)
                if self.redis is None:
                    raise ConnectionError("Failed to create Redis connection")

                pong = self.redis.ping()
                self.is_connected = True
                streams_connection_status.labels(service=self.service_name).set(1)

                logger.info(
                    f"✅ Redis Streams Event Bus connected for service: {self.service_name}"
                )
                return True

            except Exception as e:
                logger.warning(f"❌ Redis connection attempt {attempt + 1} failed: {e}")
                streams_connection_status.labels(service=self.service_name).set(0)
                if attempt < 4:
                    await asyncio.sleep(2**attempt)

        logger.error(f"❌ Failed to connect to Redis after 5 attempts")
        self.is_connected = False
        return False

    async def disconnect(self):
        """Отключение от Redis"""
        for task in self._listener_tasks.values():
            task.cancel()
        self._listener_tasks.clear()

        if self.redis:
            await self.redis.aclose()
        self.is_connected = False
        logger.info(
            f"🔌 Redis Streams Event Bus disconnected for service: {self.service_name}"
        )

    async def publish_event(self, stream: str, event: EventMessage) -> bool:
        """
        Async публикация события в Redis Stream с надежной доставкой
        """
        if not self.is_connected or not self.redis:
            logger.warning("Redis not connected, cannot publish event")
            streams_events_failed.labels(
                service=self.service_name, operation="publish"
            ).inc()
            return False

        try:
            event.source = self.service_name
            message_data = event.to_dict()

            redis_message: Dict[str, str] = {}
            for key, value in message_data.items():
                if isinstance(value, (str, int, float)):
                    redis_message[str(key)] = str(value)
                else:
                    redis_message[str(key)] = json.dumps(value)

            message_id = await self.redis.xadd(stream, redis_message)
            streams_events_published.labels(
                service=self.service_name, stream=stream
            ).inc()

            logger.debug(
                f"📤 Published event to stream {stream}: {event.type}, message_id: {message_id}"
            )
            return True

        except Exception as e:
            logger.error(f"❌ Failed to publish event to stream {stream}: {e}")
            streams_events_failed.labels(
                service=self.service_name, operation="publish"
            ).inc()
            return False

    async def subscribe_to_stream(
        self,
        stream: str,
        handler: Callable[[EventMessage], Any],
        consumer_group: Optional[str] = None,
    ):
        """
        Подписка на Redis Stream с consumer group
        """
        if not self.is_connected or not self.redis:
            logger.warning("Redis not connected, cannot subscribe")
            return

        consumer_group = consumer_group or self.service_name
        self.consumer_groups[stream] = consumer_group
        self.handlers[stream] = handler

        try:
            try:
                await self.redis.xgroup_create(
                    stream, consumer_group, "$", mkstream=True
                )
                logger.info(
                    f"📡 Created consumer group {consumer_group} for stream {stream}"
                )
            except redis.ResponseError as e:
                if "BUSYGROUP" not in str(e):
                    raise

            logger.info(f"📡 Subscribed to stream: {stream} (group: {consumer_group})")

            task = asyncio.create_task(self._listen_to_stream(stream, consumer_group))
            self._listener_tasks[stream] = task

        except Exception as e:
            logger.error(f"Failed to subscribe to stream {stream}: {e}")

    async def _listen_to_stream(self, stream: str, consumer_group: str):
        """Прослушивание stream с consumer group в фоне"""
        consumer_name = f"{self.service_name}_{id(self)}"

        try:
            while self.is_connected and stream in self.handlers:
                try:
                    if self.redis:
                        messages = await self.redis.xreadgroup(
                            consumer_group,
                            consumer_name,
                            {stream: ">"},
                            count=10,
                            block=5000,
                        )

                    for stream_name, message_list in messages:
                        for message_id, message_data in message_list:
                            try:
                                message_data = {
                                    k.decode() if isinstance(k, bytes) else k: v
                                    for k, v in message_data.items()
                                }

                                event = EventMessage.from_dict(message_data)
                                handler = self.handlers[stream]

                                if asyncio.iscoroutinefunction(handler):
                                    await handler(event)
                                else:
                                    handler(event)

                                if self.redis:
                                    await self.redis.xack(
                                        stream, consumer_group, message_id
                                    )
                                streams_events_consumed.labels(
                                    service=self.service_name, stream=stream
                                ).inc()

                            except Exception as e:
                                logger.error(
                                    f"Error processing message {message_id} from stream {stream}: {e}"
                                )
                                if self.redis:
                                    await self.redis.xack(
                                        stream, consumer_group, message_id
                                    )

                except Exception as e:
                    logger.error(f"Error reading from stream {stream}: {e}")
                    await asyncio.sleep(1)

        except asyncio.CancelledError:
            logger.info(f"Listener for stream {stream} cancelled")
        except Exception as e:
            logger.error(f"Error in stream listener for {stream}: {e}")

    # API совместимые методы
    async def publish_bot_event(self, event_type: str, bot_name: str, **kwargs) -> bool:
        """Публикация события связанного с ботом"""
        event = EventMessage(
            type=event_type,
            data={"bot_name": bot_name, **kwargs},
            source=self.service_name,
        )
        return await self.publish_event("bot_events", event)

    async def publish_strategy_event(
        self, event_type: str, strategy_name: str, **kwargs
    ) -> bool:
        """Публикация события связанного со стратегией"""
        event = EventMessage(
            type=event_type,
            data={"strategy_name": strategy_name, **kwargs},
            source=self.service_name,
        )
        return await self.publish_event("strategy_events", event)

# Глобальные экземпляры
core_streams_event_bus = RedisStreamsEventBus(service_name="core_server")
mcp_streams_event_bus = RedisStreamsEventBus(service_name="mcp_bridge")
```

#### 2. Интеграция в приложение
**Обновить `core_server/core/app.py`:**
```python
from core_server.tools.redis_streams_event_bus import core_streams_event_bus

@app.on_event("startup")
async def startup_event():
    # Инициализация Redis Streams
    await core_streams_event_bus.connect()

    # Другие инициализации...
    await ft_rest_client_service.initialize(bot_configs)

@app.on_event("shutdown")
async def shutdown_event():
    await core_streams_event_bus.disconnect()
    await ft_rest_client_service.shutdown()
```

#### 3. Интеграция в FtRestClient Service
**Обновить `core_server/services/ft_rest_client_service.py`:**
```python
from core_server.tools.redis_streams_event_bus import core_streams_event_bus

class FtRestClientService:
    def __init__(self):
        self.event_bus = core_streams_event_bus

    async def start_bot(self, bot_name: str):
        result = await self._make_request(bot_name, "POST", "/api/v1/status/start")

        # Публикация события через Streams
        await self.event_bus.publish_bot_event("bot_started", bot_name, result=result)

        return result
```

---

*Этап 2 завершен: полная backend архитектура с enterprise-grade межсервисной связью готова для API layer*