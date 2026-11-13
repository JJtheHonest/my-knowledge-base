---
title: Pydantic 模型字段与普通 Python 类属性的区别
type: note
permalink: my-kb/notes/pydantic-模型字段与普通-python-类属性的区别
tags:
- python
- pydantic
---

# Pydantic 模型字段与普通 Python 类属性的区别

## 核心结论

继承 Pydantic `BaseModel` 的类通常不需要自己定义 `__init__()`。Pydantic 根据带类型注解的字段声明，负责字段初始化、数据解析和校验；实例创建后，可以通过 `self.字段名` 访问字段。

这不是所有 Python 类都具备的行为。普通类中直接在类体里赋值，通常定义的是类属性；实例能访问类属性，不等于实例独立拥有它。

## Pydantic 模型字段

```python
from pydantic import BaseModel, Field

class EnhancedMemory(BaseModel):
    all_findings: list[str] = Field(default_factory=list)
    agent_findings: dict[str, list[str]] = Field(default_factory=dict)
    original_task: str = ""

memory = EnhancedMemory(original_task="分析报告")
print(memory.original_task)  # 分析报告
print(memory.all_findings)   # []
```

- `all_findings: list[str]` 声明字段名称和类型。
- 默认值用于未传入该字段的情况；没有默认值的普通模型字段通常是必填字段。
- Pydantic 初始化时解析并校验输入，设置实例字段。
- 方法内部用 `self.all_findings` 访问当前实例的字段。
- 显式标注为 `ClassVar` 的属性是类变量，不属于模型字段。

## 普通类中的类属性与实例属性

```python
class NormalMemory:
    all_findings: list[str] = []

a = NormalMemory()
b = NormalMemory()
a.all_findings.append("发现 A")
print(b.all_findings)  # ["发现 A"]
```

这里两个实例都通过属性查找访问同一个类属性列表。类型注解本身不会将类属性转换成实例属性。

要让每个实例拥有独立的列表，通常在 `__init__()` 中赋值：

```python
class NormalMemory:
    def __init__(self):
        self.all_findings: list[str] = []
```

注意：`self.all_findings.append(...)` 修改已找到的列表；`self.all_findings = []` 则会为当前普通实例设置同名实例属性，遮蔽类属性。

## 可变默认值

Pydantic 允许下面的写法：

```python
class Memory(BaseModel):
    all_findings: list[str] = []
```

对于这里的列表、字典等不可哈希默认值，Pydantic 会为每个实例深拷贝默认值，因此不同实例不会共享这个默认列表。

更明确的写法是 `Field(default_factory=list)` 或 `Field(default_factory=dict)`：创建实例且未传入字段时，调用工厂函数创建新容器。这里传的是函数 `list`，不是调用结果 `list()`。

## 额外初始化：model_post_init

Pydantic v2 可以通过 `model_post_init()`，在模型字段初始化完成后执行额外逻辑，例如创建线程锁：

```python
import threading
from typing import Any
from pydantic import BaseModel, Field, PrivateAttr

class EnhancedMemory(BaseModel):
    all_findings: list[str] = Field(default_factory=list)
    _lock: Any = PrivateAttr()

    def model_post_init(self, __context: Any) -> None:
        self._lock = threading.Lock()
```

`_lock` 显式声明为私有属性，不参与普通字段校验，也不会作为模型字段出现在 `model_dump()` 中。每个实例初始化时创建自己的锁。

也可以直接写 `_lock: Any = PrivateAttr(default_factory=threading.Lock)`，如果没有其他初始化逻辑，就不必为创建锁单独定义 `model_post_init()`。

## Observations

- [fact] BaseModel 根据字段声明处理初始化和校验，通常不需要自定义 __init__。
- [fact] 普通 Python 类的类体赋值定义类属性，实例能够访问它，但并不自动拥有独立副本。
- [fact] Pydantic 为列表和字典这类不可哈希默认值创建实例级深拷贝。
- [insight] 判断属性是否由实例独立持有，需要结合普通类或框架的初始化行为，不能只看是否能通过 self 访问。
- [decision] 新容器默认值优先使用 Field(default_factory=...)，内部锁等非字段状态使用 PrivateAttr。

## 来源

根据关于 EnhancedMemory、BaseModel 和实例字段初始化的对话整理；适用于 Pydantic v2。

## Relations

- relates_to [[LangGraph State：TypedDict 与 Pydantic 的选择]]
