---
name: python-test-quality
description: 评估 Python 测试的质量，识别无意义测试、过度 mock、实现细节测试、维护成本过高的测试等问题。Use PROACTIVELY when reviewing test code, evaluating whether tests are worth writing, debugging flaky tests that fail on refactors, or when test coverage is high but confidence is low. Also use when the user asks "should I test this?", "why do my tests break so easily?", or suspects tests are written just for coverage metrics.
---

# Python Test Quality

判断测试是不是"为了写而写"。核心原则：测试是安全网，不是枷锁。好的测试在业务逻辑变时会挂，在重构实现时不应该挂。

审阅时同时参考：
- [[python-testing-patterns]] — 具体怎么写 pytest、fixture、mock
- [[code-review-excellence]] — review 流程和 severity 标签体系

## When to Use This Skill

- 审阅测试代码，判断测试是否有价值
- 测试频繁挂掉但业务逻辑没有变化
- 覆盖率很高但对代码质量没信心
- 决定某个函数/场景是否需要测试
- 重构后测试大面积失败
- 用户问"这个要不要测""测试为什么这么脆弱"

## Review Dimensions

### 1. 测试行为，不是实现（Test Behavior, Not Implementation）

**核心问题：重构内部实现后，测试会挂吗？**

**检查点：**
- assert 的是**输出/副作用**，还是**内部状态、调用次数、私有方法**
- 测试名说的是"发生了什么"，还是"什么被调用了"
- 改了变量名、提取了私有方法、换了循环方式，测试是否还能通过

**反例 vs 正例：**

```python
# BAD：测试实现细节。换了内部存储方式测试就挂。
def test_user_service():
    service = UserService()
    user = service.create_user({"email": "a@b.com"})
    assert service._users[0].email == "a@b.com"  # 依赖内部结构
    assert service.db.execute.call_count == 1    # 依赖调用次数

# GOOD：测试行为。内部怎么存不重要，重要的是用户能查到。
def test_create_user_makes_it_retrievable():
    service = UserService()
    user = service.create_user({"email": "a@b.com"})
    found = service.get_user(user.id)
    assert found.email == "a@b.com"
```

**标记示例：**
```markdown
🟡 [important] 这个测试 assert 了 `_internal_state` 和 `mock_call_count`，重构内部实现时一定会挂。建议改成 assert 输出行为。

🟢 [nit] 测试名 `test_process_calls_validate` 说的是"调用了什么"，而不是"验证了什么"。建议改成 `test_rejects_invalid_email`。
```

### 2. Mock 的边界（Mocking Boundaries）

**核心问题：mock 之后，测试还测了真实的东西吗？**

**检查点：**
- mock 的是**外部依赖**（HTTP、数据库、文件系统），还是**自己的业务逻辑**
- 一个测试里是否 mock 了超过 3 个内部组件
- mock 的返回值是否和真实行为相差太远（比如 mock 永远返回成功，掩盖了真实失败路径）
- patch 的目标是否是"where it's used"而不是"where it's defined"

**反例 vs 正例：**

```python
# BAD：mock 了自己的业务逻辑，测的是假东西。
@patch("myapp.services.UserValidator.validate")
def test_create_user(mock_validate):
    mock_validate.return_value = True
    result = create_user({"email": "a@b.com"})
    assert result is not None

# GOOD：mock 外部依赖，保留自己的业务逻辑。
@patch("myapp.services.send_email")
def test_create_user_sends_welcome_email(mock_send):
    create_user({"email": "a@b.com"})
    mock_send.assert_called_once_with(to="a@b.com", template="welcome")
```

**标记示例：**
```markdown
🟡 [important] 这个测试 mock 了 `UserService.create_user` 本身，那测的是什么？建议 mock 外部 HTTP 调用，让被测代码真实执行。

🟡 [important] 这里 mock 了 5 个内部组件，测试已经和被测代码完全脱节。建议精简 mock，或改用集成测试。
```

### 3. 覆盖率的真相（Coverage Truth）

**核心问题：这个测试是为了验证逻辑，还是为了刷覆盖率数字？**

**检查点：**
- 纯数据访问（getter/setter/dataclass 属性）是否有测试
- 一行 `return` 的简单函数是否有测试
- 不可能发生的路径（比如前置条件已经保证不会触发的分支）是否有测试
- 测试失败时是否能直接定位到业务逻辑问题

**不值得测的情况：**
- Pydantic model 的字段校验（库已经测过了）
- 纯 dataclass 的定义
- 简单的类型转换/格式封装函数
- 配置类的默认值

**标记示例：**
```markdown
🟢 [nit] `get_user_name()` 只是 `return self.name`，这个测试对质量没有增量。建议删除，把精力放在业务规则上。

🟡 [important] 这个分支 `if x < 0` 在调用方已经通过 Pydantic 保证了 `x > 0`，这里永远走不到。测试它只是为了覆盖率，建议删除或加注释说明是防御性代码。
```

### 4. 测试的维护成本（Maintenance Cost）

**核心问题：维护这个测试的成本，是否高于它带来的价值？**

**检查点：**
- 设置代码是否比测试本身还长
- 是否用了复杂的 fixture 嵌套（3 层以上）才能跑一个简单测试
- 测试之间是否共享可变状态（一个挂了影响其他）
- 测试数据是否是硬编码的，每次改业务规则就要改 10 处测试
- 是否为了测一个私有方法，用了反射或复杂的 mock 链

**标记示例：**
```markdown
🟡 [important] 这个测试需要 50 行 fixture 设置才能跑，但被测逻辑只有 5 行。说明被测代码的依赖设计有问题，或测试粒度不对。

🟡 [important] 3 个测试共享 `users` 列表，测试 B 依赖测试 A 的副作用。一旦单独跑 B 就挂。每个测试应该独立设置数据。
```

### 5. 测试即文档（Tests as Documentation）

**核心问题：不看源码，能从测试里读出业务规则吗？**

**检查点：**
- 测试名是否完整描述了场景和预期：`test_<action>_<condition>_<expected>`
- 测试失败时，错误信息是否直接告诉开发者哪条业务规则被打破了
- 测试数据是否用了有业务意义的名字，而不是 `user1`、`data2`
- 是否通过测试名就能了解系统支持的边界和约束

**反例 vs 正例：**

```python
# BAD：名字没说出规则，数据没有业务含义
def test_user_1():
    result = calculate_discount(user="u1", amount=100)
    assert result == 10

# GOOD：名字即规则，数据有含义
def test_vip_user_gets_10_percent_discount():
    vip_user = User(tier="gold", orders=[Order(amount=100)])
    discount = calculate_discount(vip_user)
    assert discount == 10  # 10% of 100
```

**标记示例：**
```markdown
🟢 [nit] 测试名 `test_case_3` 没有业务含义。建议改成 `test_rejects_order_when_inventory_empty`。

💡 [suggestion] 用 `pytest.param(id="insufficient_funds")` 给参数化测试起有业务含义的 ID，失败时能直接定位场景。
```

## Severity Labels

沿用 [[code-review-excellence]] 的体系：

| 标签 | 含义 | 是否阻塞 |
|------|------|----------|
| 🔴 [blocking] | 测试在欺骗你（mock 了被测逻辑本身、测了不可能路径） | 是 |
| 🟡 [important] | 测试脆弱/维护成本高/价值低 | 否，但需讨论 |
| 🟢 [nit] | 命名/数据可读性可以优化 | 否 |
| 💡 [suggestion] | 替代方案或补充场景 | 否 |
| 📚 [learning] | 测试设计模式分享 | 否 |
| 🎉 [praise] | 测试设计得好 | — |

## Quick Review Checklist

审阅测试时快速过一遍：

- [ ] **行为**：assert 的是输出还是内部实现？重构后会不会挂？
- [ ] **Mock**：mock 的是外部依赖还是自己的逻辑？mock 链是否过长？
- [ ] **价值**：这个测试验证的是业务规则，还是一行 return？是为了覆盖率吗？
- [ ] **成本**：设置代码是否比测试还长？测试之间是否独立？
- [ ] **文档**：测试名能否直接读出业务规则？失败时能否定位问题？

## When NOT to Test

不是所有代码都需要测试。以下情况可以考虑跳过：

- 纯数据定义（Pydantic model、dataclass、enum）
- 简单的透传/委托函数（除非为了集成测试的需要）
- 第三方库的 wrapper（测试你的代码，不要测试库）
- 配置和常量定义

## Related Skills

- [[python-testing-patterns]] — pytest、fixture、mock、参数化的具体写法
- [[python-code-review]] — 代码质量审阅框架
- [[code-review-excellence]] — review 流程、沟通技巧和 severity 标签
- [[python-anti-patterns]] — 代码反模式（测试中的坏代码也属于代码）
