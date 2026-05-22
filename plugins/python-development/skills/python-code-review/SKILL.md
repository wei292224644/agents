---
name: python-code-review
description: 审阅 Python 代码的冗余、不合理判断、不必要逻辑和重复造轮子问题。Use PROACTIVELY when the user asks to review Python code, check a PR for quality, find unnecessary logic, or evaluate whether code follows Python best practices. Also use when the user wants a second opinion on Python implementation choices, suspects over-engineering, or asks "is this Pythonic?"
---

# Python Code Review

对 Python 代码做高质量的审阅。核心关注四个维度：冗余、不合理判断、不必要逻辑、重复造轮子。

审阅时同时参考：
- [[python-anti-patterns]] — 具体反模式和修复方式
- [[code-review-excellence]] — review 流程、反馈技巧和 severity 标签体系

## When to Use This Skill

- 审阅 Python PR 或代码片段
- 检查代码是否存在过度设计或冗余
- 评估是否可以用标准库/第三方库替代手写逻辑
- 做代码走查（code walkthrough）
- 用户问"这段代码有问题吗""这样写规范吗"

## Review Dimensions

审阅时从以下四个维度逐一检查，按 [[code-review-excellence]] 的 severity 标签标记问题。

### 1. 代码冗余（Redundancy）

**检查点：**
- 同一逻辑在多处重复出现（retry、timeout、验证、转换）
- 过早抽象：只有 1-2 处相似就强行提取公共函数/基类
- 没必要的包装层：wrapper 函数只是透传参数，没增加价值
- 死代码：定义了但从未调用的函数/变量/import
- 冗余 boilerplate：手写 `__init__`/`__repr__`/`__eq__` 而不是用 `@dataclass`
- **参数过多**：函数参数超过 5 个，调用时容易传错且难以扩展

**标记示例：**
```markdown
🟡 [important] `validate_email()` 在 3 个 handler 里重复实现，建议集中到 `validators.py`。

🟢 [nit] `get_user_by_id()` 只是 `repo.get(id)` 的透传，没必要包装。

🟡 [important] `create_order()` 有 7 个位置参数，建议用 `CreateOrderInput` dataclass 封装，避免调用时传错顺序。
```

### 2. 不合理判断（Faulty Logic）

**检查点：**
- `bare except:` 或 `except Exception:` 吞掉所有异常
- 过度防御：对不可能为空的值做 `if x is not None`，或对内部数据做冗余校验
- 边界条件遗漏：空列表、零值、None、极值未处理
- 逻辑矛盾：前面 assert 了条件，后面又做同样判断
- 错误的布尔判断：`if x == True` 而不是 `if x`
- **可变默认参数**：`def f(items=[])` 会导致所有调用共享同一个对象
- **非 Pythonic 的判断**：用 `if key in d: value = d[key]`（LBYL）而不是 `try/except KeyError`（EAFP）；用 `type(x) == Foo` 而不是 `isinstance(x, Foo)`
- **日志 f-string**：`logger.info(f"...")` 无论日志级别是否开启都会执行格式化，应该用 `%s` 或 `extra`
- **`is` 滥用**：用 `is` 比较字符串、整数等值类型（依赖 CPython 实现细节），只应该用 `is` 比较 `None`/`True`/`False`/单例
- **`print` 代替 `logging`**：生产代码用 `print()` 输出，无法分级、过滤和重定向
- **错误的数据结构**：用 `list` 做频繁成员检测（`x in my_list` 是 O(n)）而不是 `set`（O(1)）
- **`@lru_cache` 滥用**：缓存不可哈希参数、mutable 对象、大数据且永不释放，导致内存泄漏或运行时异常

**标记示例：**
```markdown
🔴 [blocking] `except Exception: pass` 会吞掉 KeyboardInterrupt 和业务异常，至少改成捕获具体异常。

🟡 [important] 这里已经用 Pydantic 校验过输入了，再手动检查 `if data.get("email")` 是过度防御。

🟡 [important] `def add_item(item, items=[])` 的可变默认参数会导致所有调用共享同一个列表。改用 `items=None`。

🟢 [nit] 日志用 f-string 会导致每次调用都格式化，即使日志不输出。建议用 `logger.info("User %s logged in", user_id)`。

🟡 [important] 用 `is` 比较字符串 `"active"` 依赖 CPython 的字符串驻留机制，不是语言保证。改用 `==`。

🟡 [important] `ALLOWED_HOSTS` 是列表，成员检测是 O(n)。数据量上去会慢，建议改成 `set`。

🟡 [important] `@lru_cache` 缓存了 dict 参数，会抛 `TypeError: unhashable type: 'dict'`。要么转成 tuple，要么不用 cache。
```

### 3. 不必要逻辑（Unnecessary Complexity）

**检查点：**
- 为了"扩展性"引入抽象，但当前需求完全不需要
- 可以用简单数据结构（dict、set）解决，却用了类继承/工厂/注册模式
- 函数里混入了不属于当前层级的职责（I/O 混入业务逻辑）
- 嵌套过深（3 层以上）或函数超过 50 行且不分拆
- 用复杂表达式代替可读的正统写法
- **魔术数字/字符串**：硬编码的阈值、状态码、timeout、魔法值没有提取为命名常量
- **手工资源管理**：用 `try/finally` 关闭文件/连接，而不是 `with` 语句或 `contextlib`
- **模块顶层副作用**：在模块全局作用域执行 I/O（数据库连接、API 调用、文件读取），导致导入时触发副作用

**标记示例：**
```markdown
🟡 [important] 这个 Factory + 装饰器注册模式对 3 个 formatter 来说太重了，一个字典映射更直接。参考 [[python-design-patterns]] Pattern 1。

🟢 [nit] 三重列表推导可以拆成带中间变量的两步，可读性更好。

🟡 [important] `db = create_engine(...)` 在模块顶层执行，导入时就会连接数据库。建议用工厂函数延迟初始化。
```

### 4. 重复造轮子（Reinventing the Wheel）

**检查点：**
- 标准库已提供的功能自己重写（`json`、`pathlib`、`itertools`、`collections`、`functools`）
- 常用第三方库（如 `pydantic`、`httpx`、`typer`、`rich`）已解决的问题手写实现
- 语言特性没利用上：还在用 `if key in d: d[key] += 1` 而不是 `collections.Counter`
- 正则表达式处理简单字符串操作（能用 `str.startswith`/`split` 就不用 `re`）
- **没用到现代语法**：还在手写 `__init__`/`__repr__` 而不是 `@dataclass`；手写枚举值而不是 `enum.Enum`；手工拼接路径而不是 `pathlib`

**标记示例：**
```markdown
🟡 [important] 手动解析 JSON 字符串用 `json.loads()` 即可，不要用 `eval()` 或字符串 split。

💡 [suggestion] 这里的分组统计可以用 `collections.defaultdict(list)` 或 `itertools.groupby` 替代手写循环。
```

## Severity Labels

沿用 [[code-review-excellence]] 的体系：

| 标签 | 含义 | 是否阻塞 |
|------|------|----------|
| 🔴 [blocking] | 必须修复（bug、安全风险、明显错误） | 是 |
| 🟡 [important] | 应该修复（设计问题、可维护性） | 否，但需讨论 |
| 🟢 [nit] | 建议优化（风格、可读性） | 否 |
| 💡 [suggestion] | 替代方案参考 | 否 |
| 📚 [learning] | 教育性评论，不强制修改 | 否 |
| 🎉 [praise] | 做得好的地方 | — |

## Quick Review Checklist

审阅每段代码时快速过一遍：

- [ ] **冗余**：有没有重复逻辑？死代码？wrapper 是否有价值？手写 boilerplate 能不能用 dataclass？参数是否超过 5 个？
- [ ] **判断**：bare except？边界处理？过度防御？可变默认参数？日志用 f-string？`is` 比较值类型？list 做成员检测？lru_cache 滥用？
- [ ] **复杂度**：过度设计？单一职责？嵌套太深？魔术数字/字符串？资源管理用 `with`？模块顶层有 I/O 副作用？
- [ ] **轮子**：标准库/常用库能直接做？Pythonic 写法？dataclass/enum/pathlib 等现代特性？

## Review Output Format

输出结构参考 [[code-review-excellence]] 的 PR Review Comment Template：

```markdown
## 审阅概览

[一句话总结整体质量]

## 亮点

- 🎉 [praise] [具体表扬]

## 需要关注的问题

🔴 [blocking] [问题描述 + 建议修复方式]
🟡 [important] [问题描述 + 建议修复方式]

## 建议

💡 [suggestion] [替代方案]
🟢 [nit] [小优化]

## 结论

✅ 批准 / 💬 有建议 / 🔄 需要修改
```

## Related Skills

- [[python-anti-patterns]] — 常见反模式和修复代码示例
- [[code-review-excellence]] — review 流程、沟通技巧和模板
- [[python-design-patterns]] — KISS、SRP、Rule of Three 等设计原则
- [[python-code-style]] — PEP 8、ruff、mypy 等风格和类型规范
