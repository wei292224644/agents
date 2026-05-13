---
name: python-type-safety
description: Python 类型安全：类型标注、泛型、Protocol、严格类型检查。适用于添加类型注解、实现泛型类、定义结构化接口、配置 mypy/pyright 等场景。
---

# Python 类型安全

利用 Python 的类型系统在静态分析阶段捕获错误。类型注解既是可执行的文档，也能被工具自动验证。

## 适用场景

- 为已有代码添加类型标注
- 创建泛型、可复用的类
- 使用 Protocol 定义结构化接口
- 配置 mypy 或 pyright 进行严格检查
- 理解类型收窄（type narrowing）和类型守卫
- 构建类型安全的 API 和库

## 核心概念

### 1. 类型注解

为函数参数、返回值和变量声明期望的类型。

### 2. 泛型

编写可复用代码，同时在不同类型之间保留类型信息。

### 3. Protocol

无需继承即可定义结构化接口（带类型安全的鸭子类型）。

### 4. 类型收窄

使用守卫和条件判断在代码块中收窄类型。

## 快速开始

```python
def get_user(user_id: str) -> User | None:
    """返回类型明确表达了"可能不存在"的语义。"""
    ...

# 类型检查器强制你处理 None 的情况
user = get_user("123")
if user is None:
    raise UserNotFoundError("123")
print(user.name)  # 此处类型检查器知道 user 是 User 类型
```

## 基础模式

### 模式 1：为所有公开接口添加类型注解

每个公开的函数、方法和类都应有类型注解。

```python
def get_user(user_id: str) -> User:
    """根据 ID 获取用户。"""
    ...

def process_batch(
    items: list[Item],
    max_workers: int = 4,
) -> BatchResult[ProcessedItem]:
    """并发处理条目。"""
    ...

class UserRepository:
    def __init__(self, db: Database) -> None:
        self._db = db

    async def find_by_id(self, user_id: str) -> User | None:
        """找到返回 User，未找到返回 None。"""
        ...

    async def find_by_email(self, email: str) -> User | None:
        ...

    async def save(self, user: User) -> User:
        """保存并返回带生成 ID 的 User。"""
        ...
```

在 CI 中使用 `mypy --strict` 或 `pyright` 尽早捕获类型错误。对于已有项目，通过逐模块覆盖配置逐步启用严格模式。

### 模式 2：使用现代 Union 语法

Python 3.10+ 提供了更简洁的 union 语法。

```python
# 推荐（3.10+）
def find_user(user_id: str) -> User | None:
    ...

def parse_value(v: str) -> int | float | str:
    ...

# 旧式写法（仍然有效，3.9 兼容需要）
from typing import Optional, Union

def find_user(user_id: str) -> Optional[User]:
    ...
```

### 模式 3：使用守卫进行类型收窄

通过条件判断让类型检查器收窄类型。

```python
def process_user(user_id: str) -> UserData:
    user = find_user(user_id)

    if user is None:
        raise UserNotFoundError(f"用户 {user_id} 不存在")

    # 此处类型检查器知道 user 是 User，而非 User | None
    return UserData(
        name=user.name,
        email=user.email,
    )

def process_items(items: list[Item | None]) -> list[ProcessedItem]:
    # 过滤并收窄类型
    valid_items = [item for item in items if item is not None]
    # valid_items 现在是 list[Item]
    return [process(item) for item in valid_items]
```

### 模式 4：避免 `Any`，优先使用声明式类

用 `dataclass`、`Pydantic` 模型、`TypedDict` 或 `NamedTuple` 替代 `dict[str, Any]`。声明式类让数据结构在类型层面显式化，并能启用静态验证。

```python
# 不推荐：不透明的 dict 丢失了所有类型信息
def enroll_student(course_id: str, student_data: dict[str, Any]) -> dict[str, Any]:
    ...

# 推荐：声明式类型明确描述数据结构和约束
from dataclasses import dataclass

@dataclass
class EnrollRequest:
    student_name: str
    student_email: str
    course_id: str

@dataclass
class Enrollment:
    enrollment_id: str
    student_name: str
    student_email: str
    course_id: str
    enrolled_at: str

def enroll(req: EnrollRequest) -> Enrollment:
    ...
```

**选择合适的声明式工具：**

| 工具 | 适用场景 |
|------|----------|
| `dataclass` | 纯数据载体，无需验证 |
| `Pydantic`（BaseModel） | 需要验证、序列化、API schema |
| `TypedDict` | 轻量级键值结构（如 JSON 数据、kwargs 参数包） |
| `NamedTuple` | 不可变的小型命名元组 |

```python
# TypedDict：适用于类 JSON dict 的轻量场景
from typing import TypedDict

class EventPayload(TypedDict):
    event_type: str
    user_id: str
    timestamp: str

def handle_event(payload: EventPayload) -> None:
    print(payload["user_id"])  # 类型安全的键访问

# NamedTuple：不可变的紧凑记录
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float
    z: float = 0.0
```

**`Any` 在以下场景仍可接受：**

- 结构无法在静态阶段确定的数据（如运行时用户自定义 schema）
- 与无类型标注的第三方库交互
- 正在逐步迁移类型的存量代码边界

### 模式 5：泛型类

创建类型安全的可复用容器。

```python
from typing import TypeVar, Generic

T = TypeVar("T")
E = TypeVar("E", bound=Exception)

class Result(Generic[T, E]):
    """表示成功值或错误。"""

    def __init__(
        self,
        value: T | None = None,
        error: E | None = None,
    ) -> None:
        if (value is None) == (error is None):
            raise ValueError("value 和 error 必须有且仅有一个被设置")
        self._value = value
        self._error = error

    @property
    def is_success(self) -> bool:
        return self._error is None

    @property
    def is_failure(self) -> bool:
        return self._error is not None

    def unwrap(self) -> T:
        """获取值或抛出错误。"""
        if self._error is not None:
            raise self._error
        return self._value  # type: ignore[return-value]

    def unwrap_or(self, default: T) -> T:
        """获取值或返回默认值。"""
        if self._error is not None:
            return default
        return self._value  # type: ignore[return-value]

# 使用时保留类型信息
def parse_config(path: str) -> Result[Config, ConfigError]:
    try:
        return Result(value=Config.from_file(path))
    except ConfigError as e:
        return Result(error=e)

result = parse_config("config.yaml")
if result.is_success:
    config = result.unwrap()  # 类型：Config
```

## 高级模式

### 模式 6：泛型 Repository

创建类型安全的数据访问模式。

```python
from typing import TypeVar, Generic
from abc import ABC, abstractmethod

T = TypeVar("T")
ID = TypeVar("ID")

class Repository(ABC, Generic[T, ID]):
    """泛型 Repository 接口。"""

    @abstractmethod
    async def get(self, id: ID) -> T | None:
        """根据 ID 获取实体。"""
        ...

    @abstractmethod
    async def save(self, entity: T) -> T:
        """保存并返回实体。"""
        ...

    @abstractmethod
    async def delete(self, id: ID) -> bool:
        """删除实体，返回是否删除成功。"""
        ...

class UserRepository(Repository[User, str]):
    """使用字符串 ID 的 User 具体 Repository。"""

    async def get(self, id: str) -> User | None:
        row = await self._db.fetchrow(
            "SELECT * FROM users WHERE id = $1", id
        )
        return User(**row) if row else None

    async def save(self, entity: User) -> User:
        ...

    async def delete(self, id: str) -> bool:
        ...
```

### 模式 7：带边界的 TypeVar

将泛型参数限制为特定类型。

```python
from typing import TypeVar
from pydantic import BaseModel

ModelT = TypeVar("ModelT", bound=BaseModel)

def validate_and_create(model_cls: type[ModelT], data: dict) -> ModelT:
    """从 dict 创建经过验证的 Pydantic 模型。"""
    return model_cls.model_validate(data)

# 适用于任何 BaseModel 子类
class User(BaseModel):
    name: str
    email: str

user = validate_and_create(User, {"name": "Alice", "email": "a@b.com"})
# user 类型为 User

# 类型错误：str 不是 BaseModel 的子类
result = validate_and_create(str, {"name": "Alice"})  # 报错！
```

### 模式 8：使用 Protocol 实现结构化类型

无需继承即可定义接口。

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Serializable(Protocol):
    """任何可序列化/反序列化为 dict 的类。"""

    def to_dict(self) -> dict:
        ...

    @classmethod
    def from_dict(cls, data: dict) -> "Serializable":
        ...

# User 无需继承即可满足 Serializable 协议
class User:
    def __init__(self, id: str, name: str) -> None:
        self.id = id
        self.name = name

    def to_dict(self) -> dict:
        return {"id": self.id, "name": self.name}

    @classmethod
    def from_dict(cls, data: dict) -> "User":
        return cls(id=data["id"], name=data["name"])

def serialize(obj: Serializable) -> str:
    """适用于任何 Serializable 对象。"""
    return json.dumps(obj.to_dict())

# 正常运行 — User 匹配协议
serialize(User("1", "Alice"))

# 通过 @runtime_checkable 支持运行时检查
isinstance(User("1", "Alice"), Serializable)  # True
```

### 模式 9：常用 Protocol 模式

定义可复用的结构化接口。

```python
from typing import Protocol

class Closeable(Protocol):
    """可关闭的资源。"""
    def close(self) -> None: ...

class AsyncCloseable(Protocol):
    """可异步关闭的资源。"""
    async def close(self) -> None: ...

class Readable(Protocol):
    """可读取的对象。"""
    def read(self, n: int = -1) -> bytes: ...

class HasId(Protocol):
    """具有 ID 属性的对象。"""
    @property
    def id(self) -> str: ...

class Comparable(Protocol):
    """支持比较的对象。"""
    def __lt__(self, other: "Comparable") -> bool: ...
    def __le__(self, other: "Comparable") -> bool: ...
```

### 模式 10：类型别名

创建有意义的类型名称。

**注意：** `type Alias = ...` 语句语法（PEP 695）在 **Python 3.12** 中引入，而非 3.10。对于需要支持更早版本（包括 3.10/3.11）的项目，使用 `TypeAlias` 注解（PEP 613，Python 3.10 起可用）。

```python
# Python 3.12+ type 语句（PEP 695）
type UserId = str
type UserDict = dict[str, Any]

# Python 3.12+ 泛型 type 语句（PEP 695）
type Handler[T] = Callable[[Request], T]
type AsyncHandler[T] = Callable[[Request], Awaitable[T]]
```

```python
# Python 3.10-3.11 写法（用于更广泛的兼容性）
from typing import TypeAlias
from collections.abc import Callable, Awaitable

UserId: TypeAlias = str
Handler: TypeAlias = Callable[[Request], Response]
```

```python
# 使用
def register_handler(path: str, handler: Handler[Response]) -> None:
    ...
```

### 模式 11：Callable 类型

为函数参数和回调标注类型。

```python
from collections.abc import Callable, Awaitable

# 同步回调
ProgressCallback = Callable[[int, int], None]  # (current, total)

# 异步回调
AsyncHandler = Callable[[Request], Awaitable[Response]]

# 带命名参数的回调（使用 Protocol）
class OnProgress(Protocol):
    def __call__(
        self,
        current: int,
        total: int,
        *,
        message: str = "",
    ) -> None: ...

def process_items(
    items: list[Item],
    on_progress: ProgressCallback | None = None,
) -> list[Result]:
    for i, item in enumerate(items):
        if on_progress:
            on_progress(i, len(items))
        ...
```

## 配置

### 严格模式检查清单

`mypy --strict` 合规要求：

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_ignores = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
no_implicit_optional = true
```

渐进式推进目标：
- 所有函数参数有类型注解
- 所有返回值有类型注解
- 类属性有类型注解
- **杜绝 `dict[str, Any]`**，用 dataclass / Pydantic / TypedDict / NamedTuple 替代；`Any` 仅在真正动态数据或与无类型第三方代码交互时使用
- 泛型集合使用类型参数（`list[str]` 而非 `list`）

对于已有代码库，通过 `# mypy: strict` 逐文件启用，或在 `pyproject.toml` 中配置逐模块覆盖。

## 最佳实践总结

1. **为所有公开 API 加注解** — 函数、方法、类属性
2. **使用 `T | None`** — 现代 union 语法优于 `Optional[T]`
3. **在 CI 中运行严格类型检查** — `mypy --strict`
4. **使用泛型** — 在可复用代码中保留类型信息
5. **定义 Protocol** — 用结构化类型定义接口
6. **收窄类型** — 使用守卫帮助类型检查器
7. **给 TypeVar 加边界** — 将泛型限制在有意义的类型范围内
8. **创建类型别名** — 为复杂类型起有意义的名字
9. **用声明式类替代 `Any`** — dataclass、Pydantic、TypedDict、NamedTuple 优先于 `dict[str, Any]`；`Any` 仅在真正动态数据或与无类型第三方代码交互时使用
10. **用类型即文档** — 类型是可被工具强制执行的最佳文档
