# Руководство по добавлению нового функционала

## Обзор процесса

После полного рефакторинга система имеет четкую модульную архитектуру. Добавление нового функционала следует принципам **SOLID**, **dependency injection** и **comprehensive testing**.


## 📋 Шаги добавления функционала

### 1. Планирование

#### 1.1 Определите тип функционала
- **API эндпоинт**: Новый REST endpoint
- **Сервис**: Бизнес-логика (cache, database, reliability)
- **MCP инструмент**: AI интеграция
- **UI компонент**: Frontend функциональность

#### 1.2 Оцените влияние
- Какие модули затрагиваются?
- Нужны ли новые зависимости?
- Требуется ли миграция БД?
- Влияние на безопасность?

### 2. Реализация

#### 2.1 Добавление API эндпоинта

**Шаг 1: Создайте схему в `core_server/api/schemas.py`**
```python
class NewFeatureRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    value: int = Field(..., ge=0, le=1000)

class NewFeatureResponse(BaseModel):
    id: int
    name: str
    value: int
    created_at: str
```

**Шаг 2: Добавьте эндпоинт в `core_server/api/routes.py`**
```python
@router.post("/new-feature", response_model=NewFeatureResponse)
async def create_new_feature(request: NewFeatureRequest):
    """Create new feature"""
    validate_input(request.dict(), ["name", "value"])

    # Используйте сервис для бизнес-логики
    result = await new_feature_service.create_feature(request)

    return NewFeatureResponse(**result)
```

**Шаг 3: Создайте сервис в `core_server/services/new_feature_service.py`**
```python
from ..interfaces import INewFeatureService

class NewFeatureService(INewFeatureService):
    def __init__(self, cache_service, db_service):
        self.cache = cache_service
        self.db = db_service

    async def create_feature(self, request):
        # Бизнес-логика здесь
        # Используйте cache для производительности
        # Используйте db для хранения
        pass
```

#### 2.2 Добавление MCP инструмента

**Шаг 1: Добавьте инструмент в `mcp_bridge/bridge.py`**
```python
@app.tool()
async def new_ai_feature(param1: str, param2: int) -> str:
    """
    New AI-powered feature.

    Args:
        param1: Description of parameter 1
        param2: Description of parameter 2

    Returns:
        JSON result of the operation
    """
    try:
        # Используйте core_server API
        result = await call_core_server_api("/new-feature", {
            "param1": param1,
            "param2": param2
        })
        return json.dumps(result)
    except Exception as e:
        return json.dumps({"error": str(e)})
```

**Шаг 2: Обновите документацию в `docs/api/MCP_TOOLS_DOCUMENTATION.md`**
```markdown
#### `new_ai_feature(param1: str, param2: int) -> str`
New AI-powered feature with advanced capabilities.

**Parameters:**
- `param1` (str): Description
- `param2` (int): Description

**Returns:**
- JSON result
```

#### 2.3 Добавление сервиса

**Шаг 1: Создайте интерфейс в `core_server/interfaces/`**
```python
class IServiceInterface(ABC):
    @abstractmethod
    async def method_name(self, param: str) -> Dict[str, Any]:
        pass
```

**Шаг 2: Реализуйте сервис**
```python
class ServiceImplementation(IServiceInterface):
    def __init__(self, dependency1, dependency2):
        self.dep1 = dependency1
        self.dep2 = dependency2

    async def method_name(self, param: str) -> Dict[str, Any]:
        # Implementation
        pass
```

**Шаг 3: Зарегистрируйте в DI контейнере**
```python
# В server.py или отдельном container.py
service_instance = ServiceImplementation(dep1, dep2)
```

### 3. Тестирование

#### 3.1 Unit тесты

**Создайте тесты в `tests/unit/test_new_feature.py`**
```python
import pytest
from unittest.mock import AsyncMock

class TestNewFeatureService:
    @pytest.mark.asyncio
    async def test_create_feature_success(self):
        # Arrange
        mock_cache = AsyncMock()
        mock_db = AsyncMock()
        service = NewFeatureService(mock_cache, mock_db)

        # Act
        result = await service.create_feature(valid_request)

        # Assert
        assert result["id"] is not None
        mock_cache.set.assert_called_once()
        mock_db.save.assert_called_once()

    @pytest.mark.asyncio
    async def test_create_feature_validation_error(self):
        # Test input validation
        pass
```

#### 3.2 Integration тесты

**Добавьте в `tests/test_basic.py`**
```python
def test_new_feature_endpoint(client):
    response = client.post("/api/new-feature", json={
        "name": "test",
        "value": 100
    })
    assert response.status_code == 200
    data = response.json()
    assert "id" in data
```

#### 3.3 MCP тесты

**Добавьте в `tests/test_mcp_tools.py`**
```python
# Test new MCP tool
result = await new_ai_feature("test", 123)
assert "result" in result
```

### 4. Документация

#### 4.1 API документация
FastAPI автоматически генерирует OpenAPI спецификацию. Добавьте описания:
```python
@router.post("/new-feature", 
             summary="Create New Feature",
             description="Creates a new feature with validation and caching")
```

#### 4.2 Кодовая документация
```python
def create_feature(self, request) -> Dict[str, Any]:
    """
    Create a new feature with business logic validation.

    Args:
        request: Validated request object

    Returns:
        Dict containing created feature data

    Raises:
        ValidationError: If business rules are violated
        DatabaseError: If persistence fails
    """
```

#### 4.3 Обновите руководства
- **docs/user/USER_GUIDE.md**: Добавьте описание для пользователей
- **docs/developer/DETAILED_TECHNICAL_SPECIFICATION.md**: Обновите архитектуру

### 5. Безопасность

#### 5.1 Валидация входных данных
```python
def validate_input(data: dict, required_fields: list):
    for field in required_fields:
        if field not in data:
            raise HTTPException(400, f"Missing required field: {field}")
    # Additional validation logic
```

#### 5.2 Авторизация
```python
@router.post("/admin-feature")
async def admin_only_feature(current_user = Depends(get_current_user)):
    if current_user.role != "admin":
        raise HTTPException(403, "Admin access required")
```

#### 5.3 Аудит
```python
await audit_service.log_action(
    user_id=current_user.id,
    action="create_feature",
    resource_type="feature",
    resource_id=result["id"]
)
```

### 6. Деплоймент

#### 6.1 Обновите Docker
```dockerfile
# Добавьте новые зависимости в Dockerfile
RUN uv pip install new-dependency==1.0.0
```

#### 6.2 Миграции БД
```python
# Создайте миграцию в alembic/versions/
def upgrade():
    op.create_table('new_features',
        sa.Column('id', sa.Integer(), primary_key=True),
        sa.Column('name', sa.String(100), nullable=False),
        sa.Column('value', sa.Integer(), nullable=False)
    )
```

#### 6.3 Environment variables
```bash
# Добавьте в .env
NEW_FEATURE_API_KEY=your-api-key
NEW_FEATURE_TIMEOUT=30
```

### 7. Мониторинг

#### 7.1 Метрики
```python
# Добавьте в Prometheus метрики
NEW_FEATURE_REQUESTS = Counter('new_feature_requests_total', 'New feature requests')
NEW_FEATURE_ERRORS = Counter('new_feature_errors_total', 'New feature errors')
```

#### 7.2 Health checks
```python
# Добавьте в /health endpoint
"new_feature_service": await check_new_feature_service()
```

### 8. Code Review и QA

#### 8.1 Pre-commit hooks
```bash
# Запустите quality checks
black core_server/ mcp_bridge/
isort core_server/ mcp_bridge/
mypy core_server/ mcp_bridge/
pytest tests/ --cov=core_server --cov=mcp_bridge/
```

#### 8.2 Code review checklist
- [ ] SOLID принципы соблюдены
- [ ] Dependency injection используется
- [ ] Тесты написаны (unit + integration)
- [ ] Документация обновлена
- [ ] Безопасность проверена
- [ ] Производительность протестирована
- [ ] Логирование добавлено

### 9. Релиз

#### 9.1 Version bump
```python
# Обновите версию в core_server/server.py
__version__ = "1.1.0"
```

#### 9.2 Changelog
```markdown
## [1.1.0] - 2025-11-18
### Added
- New feature functionality
- MCP tool for AI integration
- Enhanced security validation

### Changed
- Improved caching strategy
- Updated API responses

### Fixed
- Bug in existing functionality
```

#### 9.3 Deployment
```bash
# Создайте тег и push
git tag v1.1.0
git push origin v1.1.0

# Deploy to production
docker-compose up -d --build
```

## 📋 Шаблоны для быстрого старта

### Шаблон нового сервиса
```python
# services/new_service.py
from ..interfaces import IServiceInterface

class NewService(IServiceInterface):
    def __init__(self, cache_service, db_service, config):
        self.cache = cache_service
        self.db = db_service
        self.config = config

    async def business_method(self, data):
        # Cache check
        cache_key = f"new_service:{data['id']}"
        cached = await self.cache.get(cache_key)
        if cached:
            return cached

        # Business logic
        result = await self._process_data(data)

        # Cache result
        await self.cache.set(cache_key, result, ttl=300)

        return result

    async def _process_data(self, data):
        # Private method for business logic
        pass
```

### Шаблон нового API эндпоинта
```python
# api/routes.py
@router.post("/new-resource",
             response_model=NewResourceResponse,
             summary="Create New Resource",
             tags=["resources"])
async def create_new_resource(
    request: NewResourceRequest,
    current_user: User = Depends(get_current_user),
    service: NewService = Depends(get_new_service)
):
    """Create a new resource with validation and caching."""
    try:
        # Validate input
        validate_input(request.dict(), ["name", "value"])

        # Check permissions
        if not await check_permissions(current_user, "create_resource"):
            raise HTTPException(403, "Insufficient permissions")

        # Execute business logic
        result = await service.create_resource(request)

        # Broadcast real-time update via WebSocket
        await websocket_manager.broadcast({
            "type": "resource_created",
            "resource_id": result["id"],
            "user_id": current_user.id,
            "timestamp": datetime.now().isoformat()
        })

        # Audit action
        await audit_service.log_action(
            user_id=current_user.id,
            action="create_resource",
            resource_type="resource",
            resource_id=result["id"]
        )

        return NewResourceResponse(**result)

    except ValidationError as e:
        raise HTTPException(400, str(e))
    except Exception as e:
        logger.error(f"Error creating resource: {e}")
        raise HTTPException(500, "Internal server error")
```

### Шаблон WebSocket эндпоинта
```python
# api/routes.py
@router.websocket("/ws/new-feature")
async def websocket_new_feature_endpoint(websocket: WebSocket):
    """
    WebSocket endpoint for real-time new-feature updates.
    """
    await websocket_manager.connect(websocket)
    try:
        # Send welcome message
        await websocket.send_json({
            "type": "welcome",
            "feature": "new-feature",
            "message": "Connected to New Feature WebSocket",
            "timestamp": datetime.now().isoformat()
        })

        while True:
            data = await websocket.receive_text()
            message = json.loads(data)

            # Handle feature-specific messages
            if message.get("type") == "subscribe_updates":
                # Handle subscription to feature updates
                await websocket.send_json({
                    "type": "subscribed",
                    "feature": "new-feature",
                    "updates_enabled": True,
                    "timestamp": datetime.now().isoformat()
                })
            elif message.get("type") == "get_status":
                # Send current feature status
                status = await get_feature_status()
                await websocket.send_json({
                    "type": "status",
                    "data": status,
                    "timestamp": datetime.now().isoformat()
                })

    except WebSocketDisconnect:
        websocket_manager.disconnect(websocket)
        logger.info("WebSocket client disconnected from new-feature")
    except Exception as e:
        logger.error(f"WebSocket error in new-feature: {e}")
        websocket_manager.disconnect(websocket)
```

### Шаблон нового MCP инструмента
```python
# mcp_bridge/bridge.py
@app.tool()
async def new_ai_tool(
    parameter1: str = Field(..., description="Description of parameter 1"),
    parameter2: int = Field(..., ge=0, le=100, description="Parameter 2 (0-100)")
) -> str:
    """
    New AI-powered tool for advanced operations.

    This tool provides AI-enhanced functionality for complex operations,
    with intelligent error handling and result formatting.

    Args:
        parameter1: Primary input parameter
        parameter2: Secondary numeric parameter

    Returns:
        JSON-formatted result with operation details

    Raises:
        ToolError: If operation fails with specific error details
    """
    try:
        # Input validation
        if not parameter1 or len(parameter1) > 1000:
            return json.dumps({
                "error": "Invalid parameter1",
                "details": "Must be non-empty string, max 1000 chars"
            })

        # Call core server API
        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{CORE_SERVER_URL}/api/new-operation",
                json={
                    "param1": parameter1,
                    "param2": parameter2
                },
                headers={"Authorization": f"Bearer {SERVICE_TOKEN}"}
            ) as response:
                if response.status == 200:
                    result = await response.json()
                    return json.dumps({
                        "success": True,
                        "result": result,
                        "timestamp": datetime.now().isoformat()
                    })
                else:
                    error_text = await response.text()
                    return json.dumps({
                        "error": f"API call failed: {response.status}",
                        "details": error_text
                    })

    except Exception as e:
        logger.error(f"MCP tool error: {e}")
        return json.dumps({
            "error": "Tool execution failed",
            "details": str(e)
        })
```

## 🎯 Best Practices

### Архитектурные принципы
1. **Single Responsibility**: Каждый модуль отвечает за одну задачу
2. **Dependency Injection**: Зависимости передаются извне
3. **Interface Segregation**: Используйте интерфейсы для decoupling
4. **Error Handling**: Comprehensive error handling на всех уровнях

### Код качества
1. **Type Hints**: Всегда используйте типизацию
2. **Docstrings**: Документируйте все публичные методы
3. **Logging**: Логируйте важные операции и ошибки
4. **Validation**: Валидируйте входные данные на всех уровнях

### Тестирование
1. **Unit Tests**: Для каждой функции/метода
2. **Integration Tests**: Для взаимодействия компонентов
3. **Performance Tests**: Для критичных операций
4. **Coverage**: Поддерживайте >80% coverage

### Безопасность
1. **Input Validation**: Всегда валидируйте входные данные
2. **Authentication**: Проверяйте аутентификацию
3. **Authorization**: Проверяйте разрешения
4. **Audit**: Логируйте все действия

### Производительность
1. **Caching**: Используйте кэширование для дорогих операций
2. **Async**: Все I/O операции асинхронны
3. **Connection Pooling**: Для БД и внешних API
4. **WebSocket Broadcasting**: Используйте для real-time обновлений вместо polling
5. **Monitoring**: Отслеживайте производительность и WebSocket connections

Следуя этому руководству, вы сможете добавлять новый функционал, сохраняя качество, надежность и поддерживаемость системы! 🚀