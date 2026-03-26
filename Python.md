# Python 高级面试文档（10年经验）

> 适用场景：P7/P8 级别后端开发、架构师、Python 方向面试  
> 覆盖范围：语言核心 · 并发编程 · 内存管理 · 设计模式 · 性能优化 · 实战场景

---

## 目录

1. [Python 核心语言特性](#一python-核心语言特性)
2. [数据结构与内置类型](#二数据结构与内置类型)
3. [面向对象编程](#三面向对象编程)
4. [函数式编程与高级特性](#四函数式编程与高级特性)
5. [并发与异步编程](#五并发与异步编程)
6. [内存管理与垃圾回收](#六内存管理与垃圾回收)
7. [性能优化](#七性能优化)
8. [常用标准库与第三方库](#八常用标准库与第三方库)
9. [实战场景题](#九实战场景题)
10. [高频追问与陷阱题](#十高频追问与陷阱题)

---

## 一、Python 核心语言特性

### 1.1 Python 的核心设计哲学

```python
import this  # 输出 The Zen of Python

# 关键原则：
# - 优美胜于丑陋（Beautiful is better than ugly）
# - 显式胜于隐式（Explicit is better than implicit）
# - 简单胜于复杂（Simple is better than complex）
# - 可读性很重要（Readability counts）
```

### 1.2 动态类型与强类型

```python
# Python 是动态强类型语言
x = 10          # int
x = "hello"     # str（动态类型：变量类型可变）

# 强类型：不同类型不会隐式转换
print(1 + "2")  # ❌ TypeError
print(1 + 2)    # ✅ 3
print("1" + "2")# ✅ "12"

# 类型注解（Python 3.5+，仅提示，不强制）
def greet(name: str) -> str:
    return f"Hello, {name}"
```

### 1.3 一切皆对象

```python
# Python 中一切都是对象，包括函数、类、模块
def foo():
    pass

print(type(foo))          # <class 'function'>
print(type(int))          # <class 'type'>
print(type(type))         # <class 'type'>（type 是自身的实例）

# 函数也是对象，可以赋值、传递、返回
funcs = [print, len, type]
for f in funcs:
    print(f.__name__)
```

### 1.4 可变与不可变对象

| 不可变（Immutable） | 可变（Mutable） |
|---|---|
| `int`, `float`, `bool` | `list` |
| `str` | `dict` |
| `tuple` | `set` |
| `frozenset` | 自定义类（默认） |
| `bytes` | `bytearray` |

```python
# 不可变对象的"修改"实际上创建新对象
a = "hello"
id_before = id(a)
a += " world"
print(id(a) == id_before)  # False（新对象）

# 可变对象陷阱：默认参数
def append_item(item, lst=[]):   # ❌ 危险！默认参数只创建一次
    lst.append(item)
    return lst

print(append_item(1))  # [1]
print(append_item(2))  # [1, 2]  ← 预期是 [2]！

def append_item_safe(item, lst=None):  # ✅ 正确写法
    if lst is None:
        lst = []
    lst.append(item)
    return lst
```

### 1.5 作用域与 LEGB 规则

```python
# LEGB: Local → Enclosing → Global → Built-in
x = "global"

def outer():
    x = "enclosing"
    def inner():
        x = "local"
        print(x)  # local
    inner()
    print(x)      # enclosing

outer()
print(x)          # global

# global 与 nonlocal
def counter():
    count = 0
    def increment():
        nonlocal count   # 修改外层函数变量
        count += 1
        return count
    return increment

c = counter()
print(c())  # 1
print(c())  # 2
```

---
## 二、数据结构与内置类型

### 2.1 列表（list）

```python
# 列表推导式（比 map/filter 更 Pythonic）
squares = [x**2 for x in range(10) if x % 2 == 0]

# 嵌套列表推导
matrix = [[1,2,3],[4,5,6],[7,8,9]]
flat = [x for row in matrix for x in row]

# 常用操作时间复杂度
# append(x)      O(1) 均摊
# insert(i, x)   O(n)
# pop()          O(1)
# pop(i)         O(n)
# x in list      O(n)
# list[i]        O(1)
# list.sort()    O(n log n) Timsort

# 切片（不修改原列表）
lst = [1, 2, 3, 4, 5]
print(lst[1:4])    # [2, 3, 4]
print(lst[::-1])   # [5, 4, 3, 2, 1]  逆序
print(lst[::2])    # [1, 3, 5]  步长2
```

### 2.2 字典（dict）

```python
# Python 3.7+ dict 有序（插入顺序）
d = {"a": 1, "b": 2, "c": 3}

# 常用操作时间复杂度（平均）
# d[key]          O(1)
# d[key] = val    O(1)
# key in d        O(1)
# del d[key]      O(1)

# 字典推导式
squared = {k: v**2 for k, v in d.items()}

# 合并字典（Python 3.9+）
d1 = {"a": 1}
d2 = {"b": 2}
merged = d1 | d2          # {'a': 1, 'b': 2}
d1 |= d2                  # 原地合并

# setdefault vs get
d.setdefault("key", []).append(1)  # 不存在则初始化，存在则直接操作
val = d.get("missing", "default")  # 不存在返回默认值

# defaultdict
from collections import defaultdict
graph = defaultdict(list)
graph["A"].append("B")   # 无需判断 key 是否存在
```

### 2.3 集合（set）

```python
# 集合基于哈希表，元素唯一、无序
s1 = {1, 2, 3, 4}
s2 = {3, 4, 5, 6}

print(s1 & s2)   # 交集 {3, 4}
print(s1 | s2)   # 并集 {1, 2, 3, 4, 5, 6}
print(s1 - s2)   # 差集 {1, 2}
print(s1 ^ s2)   # 对称差 {1, 2, 5, 6}

# 去重（保持顺序用 dict.fromkeys）
lst = [1, 3, 2, 1, 3]
unique = list(dict.fromkeys(lst))  # [1, 3, 2]（保持顺序）
unique_set = list(set(lst))        # 顺序不保证
```

### 2.4 collections 模块

```python
from collections import Counter, deque, OrderedDict, namedtuple, ChainMap

# Counter：计数器
words = ["apple", "banana", "apple", "cherry", "banana", "apple"]
cnt = Counter(words)
print(cnt.most_common(2))  # [('apple', 3), ('banana', 2)]

# deque：双端队列（O(1) 两端操作）
dq = deque([1, 2, 3], maxlen=5)
dq.appendleft(0)   # 左端添加
dq.rotate(1)       # 旋转

# namedtuple：命名元组
Point = namedtuple("Point", ["x", "y"])
p = Point(3, 4)
print(p.x, p.y)    # 可按名称访问

# Python 3.3+：SimpleNamespace
from types import SimpleNamespace
obj = SimpleNamespace(x=1, y=2)
```

### 2.5 字符串高级操作

```python
# f-string（Python 3.6+，推荐）
name, score = "Alice", 98.5
print(f"{name} 得了 {score:.1f} 分")       # 保留1位小数
print(f"{1000000:,}")                       # 千位分隔符: 1,000,000
print(f"{'hello':^20}")                     # 居中对齐，宽度20

# str.join 比 + 拼接效率高
parts = ["a", "b", "c"]
result = "".join(parts)     # O(n)，推荐
# result = ""
# for p in parts: result += p  # O(n²)，每次创建新对象

# 正则表达式
import re
pattern = re.compile(r"\d+")     # 预编译（复用时更高效）
matches = pattern.findall("abc 123 def 456")  # ['123', '456']
```

---
## 三、面向对象编程

### 3.1 类的底层机制

```python
class MyClass:
    class_var = 0          # 类变量（所有实例共享）

    def __init__(self, x):
        self.x = x         # 实例变量

    def method(self):      # 实例方法（self 是实例）
        return self.x

    @classmethod
    def class_method(cls): # 类方法（cls 是类本身）
        return cls.class_var

    @staticmethod
    def static_method():   # 静态方法（无 self/cls）
        return "static"

# 实例属性查找顺序：
# 实例 __dict__ → 类 __dict__ → 父类 __dict__（MRO 顺序）
```

### 3.2 魔术方法（Dunder Methods）

```python
class Vector:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __repr__(self):          # repr(obj) / 调试用
        return f"Vector({self.x}, {self.y})"

    def __str__(self):           # str(obj) / print 用
        return f"({self.x}, {self.y})"

    def __add__(self, other):    # v1 + v2
        return Vector(self.x + other.x, self.y + other.y)

    def __eq__(self, other):     # v1 == v2
        return self.x == other.x and self.y == other.y

    def __hash__(self):          # hash(v)，使对象可放入 set/dict
        return hash((self.x, self.y))

    def __len__(self):           # len(v)
        return 2

    def __getitem__(self, idx):  # v[0]
        return (self.x, self.y)[idx]

    def __iter__(self):          # for i in v
        yield self.x
        yield self.y

    def __bool__(self):          # bool(v) / if v
        return bool(self.x or self.y)

    def __enter__(self):         # with v as ...
        return self

    def __exit__(self, *args):   # with 块结束
        pass
```

### 3.3 继承与 MRO

```python
class A:
    def method(self): return "A"

class B(A):
    def method(self): return "B"

class C(A):
    def method(self): return "C"

class D(B, C):   # 多继承
    pass

# MRO（方法解析顺序）：C3 线性化算法
print(D.__mro__)   # D → B → C → A → object
print(D().method()) # "B"

# super() 遵循 MRO 顺序
class B2(A):
    def method(self):
        return "B2->" + super().method()
```

### 3.4 描述符与属性

```python
class TypedProperty:
    """描述符：类型检查属性"""
    def __set_name__(self, owner, name):
        self.name = name

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return obj.__dict__.get(self.name)

    def __set__(self, obj, value):
        if not isinstance(value, int):
            raise TypeError(f"{self.name} must be int")
        obj.__dict__[self.name] = value

class Person:
    age = TypedProperty()

    def __init__(self, age):
        self.age = age   # 触发 __set__

p = Person(25)
p.age = "old"    # ❌ TypeError


# property 装饰器（更常用的描述符封装）
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("Radius cannot be negative")
        self._radius = value

    @property
    def area(self):
        import math
        return math.pi * self._radius ** 2
```
### 3.5 dataclass（Python 3.7+）

```python
from dataclasses import dataclass, field
from typing import List

@dataclass(order=True, frozen=True)   # order=True 自动生成比较方法
class Product:
    name: str
    price: float
    tags: List[str] = field(default_factory=list)

    def __post_init__(self):           # 初始化后的验证
        if self.price < 0:
            raise ValueError("Price cannot be negative")

p1 = Product("iPhone", 5999.0, ["手机", "苹果"])
p2 = Product("iPad", 3999.0)
print(p1 > p2)   # True（按字段顺序比较）
```

---

## 四、函数式编程与高级特性

### 4.1 装饰器

```python
import functools
import time

# 基础装饰器
def timer(func):
    @functools.wraps(func)   # 保留原函数元信息
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"{func.__name__} 耗时: {time.time()-start:.3f}s")
        return result
    return wrapper

# 带参数的装饰器
def retry(max_times=3, exceptions=(Exception,)):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for i in range(max_times):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if i == max_times - 1:
                        raise
                    print(f"第{i+1}次失败: {e}，重试...")
        return wrapper
    return decorator

@retry(max_times=3, exceptions=(ConnectionError,))
def fetch_data(url: str):
    pass

# 类装饰器
class Singleton:
    _instances = {}

    def __call__(self, cls):
        @functools.wraps(cls)
        def get_instance(*args, **kwargs):
            if cls not in self._instances:
                self._instances[cls] = cls(*args, **kwargs)
            return self._instances[cls]
        return get_instance
```

### 4.2 生成器与迭代器

```python
# 生成器函数（惰性求值，节省内存）
def fibonacci():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

fib = fibonacci()
print([next(fib) for _ in range(10)])  # [0,1,1,2,3,5,8,13,21,34]

# 生成器表达式（比列表推导更省内存）
total = sum(x**2 for x in range(1_000_000))   # 不会创建完整列表

# yield from（委托生成器，Python 3.3+）
def chain(*iterables):
    for it in iterables:
        yield from it

# 自定义迭代器
class Range:
    def __init__(self, stop):
        self.stop = stop
        self.current = 0

    def __iter__(self):
        return self

    def __next__(self):
        if self.current >= self.stop:
            raise StopIteration
        val = self.current
        self.current += 1
        return val
```

### 4.3 上下文管理器

```python
from contextlib import contextmanager, suppress

# 使用 contextmanager 装饰器
@contextmanager
def managed_resource(name):
    print(f"获取资源: {name}")
    try:
        yield name              # yield 前是 __enter__，后是 __exit__
    except Exception as e:
        print(f"处理异常: {e}")
        raise
    finally:
        print(f"释放资源: {name}")

with managed_resource("db_conn") as conn:
    print(f"使用 {conn}")

# suppress：忽略特定异常
with suppress(FileNotFoundError):
    open("不存在的文件.txt")

# 多个上下文管理器
with open("a.txt") as f1, open("b.txt") as f2:
    pass
```

### 4.4 闭包与函数工厂

```python
def make_multiplier(n):
    def multiplier(x):
        return x * n   # 捕获外部变量 n（闭包）
    return multiplier

double = make_multiplier(2)
triple = make_multiplier(3)
print(double(5))   # 10
print(triple(5))   # 15

# 闭包陷阱：循环变量捕获
# ❌ 错误：所有函数共享同一个 i（循环结束后 i=9）
funcs = [lambda: i for i in range(10)]
print(funcs[0]())  # 9（不是 0！）

# ✅ 正确：通过默认参数捕获当前值
funcs = [lambda i=i: i for i in range(10)]
print(funcs[0]())  # 0
```

### 4.5 元类（Metaclass）

```python
# 元类：类的类（控制类的创建过程）
class SingletonMeta(type):
    _instances = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Database(metaclass=SingletonMeta):
    def __init__(self, url):
        self.url = url

db1 = Database("mysql://localhost")
db2 = Database("postgres://localhost")
print(db1 is db2)   # True（单例）

# __init_subclass__（Python 3.6+，更简单的钩子）
class Plugin:
    _registry = {}

    def __init_subclass__(cls, plugin_name=None, **kwargs):
        super().__init_subclass__(**kwargs)
        if plugin_name:
            Plugin._registry[plugin_name] = cls

class AuthPlugin(Plugin, plugin_name="auth"):
    pass

print(Plugin._registry)  # {'auth': <class 'AuthPlugin'>}
```

---
## 五、并发与异步编程

### 5.1 GIL（全局解释器锁）

```
GIL（Global Interpreter Lock）：
  CPython 中的互斥锁，同一时刻只允许一个线程执行 Python 字节码

影响：
  ✅ I/O 密集型任务：线程在等待 I/O 时会释放 GIL，多线程有效
  ❌ CPU 密集型任务：GIL 导致多线程无法真正并行，用多进程代替

结论：
  I/O 密集型 → 多线程 / asyncio
  CPU 密集型 → 多进程（multiprocessing）/ C 扩展
```

### 5.2 多线程

```python
import threading
from concurrent.futures import ThreadPoolExecutor
import time

# 基础线程
def worker(name, delay):
    print(f"线程 {name} 开始")
    time.sleep(delay)
    print(f"线程 {name} 结束")

threads = []
for i in range(3):
    t = threading.Thread(target=worker, args=(f"T{i}", i*0.1))
    threads.append(t)
    t.start()

for t in threads:
    t.join()

# ThreadPoolExecutor（推荐，更高级）
def fetch_url(url: str) -> str:
    import urllib.request
    with urllib.request.urlopen(url) as resp:
        return resp.read()[:100]

urls = ["http://example.com"] * 5

with ThreadPoolExecutor(max_workers=5) as executor:
    futures = [executor.submit(fetch_url, url) for url in urls]
    results = [f.result() for f in futures]

# 线程安全：Lock / RLock / Event / Semaphore
lock = threading.Lock()
counter = 0

def safe_increment():
    global counter
    with lock:          # 自动 acquire/release
        counter += 1
```

### 5.3 多进程

```python
from concurrent.futures import ProcessPoolExecutor
import multiprocessing as mp

def cpu_bound(n: int) -> int:
    """CPU 密集型任务"""
    return sum(i**2 for i in range(n))

# ProcessPoolExecutor
if __name__ == "__main__":
    numbers = [10**6, 10**6, 10**6, 10**6]

    with ProcessPoolExecutor(max_workers=mp.cpu_count()) as executor:
        results = list(executor.map(cpu_bound, numbers))

    # 进程间通信
    q = mp.Queue()
    p = mp.Process(target=lambda: q.put(cpu_bound(10**6)))
    p.start()
    p.join()
    print(q.get())
```

### 5.4 asyncio 异步编程

```python
import asyncio
import aiohttp

# 基础 async/await
async def fetch(session: aiohttp.ClientSession, url: str) -> str:
    async with session.get(url) as resp:
        return await resp.text()

async def main():
    urls = [
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/1",
    ]

    async with aiohttp.ClientSession() as session:
        # 并发执行（~1s，非串行 3s）
        tasks = [fetch(session, url) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)

    return results

asyncio.run(main())

# asyncio 工具
async def with_timeout():
    try:
        result = await asyncio.wait_for(
            asyncio.sleep(10),
            timeout=2.0          # 超时控制
        )
    except asyncio.TimeoutError:
        print("超时！")

# 异步生成器
async def async_range(n):
    for i in range(n):
        await asyncio.sleep(0)   # 让出控制权
        yield i

async def consume():
    async for i in async_range(5):
        print(i)

# 异步上下文管理器
class AsyncDB:
    async def __aenter__(self):
        await asyncio.sleep(0.1)  # 模拟连接
        return self

    async def __aexit__(self, *args):
        await asyncio.sleep(0.1)  # 模拟断开
```

### 5.5 并发方案选型

| 场景 | 推荐方案 | 说明 |
|---|---|---|
| I/O 密集 + 少量并发 | `threading` | 简单，适合传统同步代码 |
| I/O 密集 + 高并发 | `asyncio` | 单线程事件循环，高吞吐 |
| CPU 密集型计算 | `multiprocessing` | 绕过 GIL，真正并行 |
| 混合场景 | `asyncio` + `ProcessPoolExecutor` | 异步 + CPU 并行 |
| 简单任务池 | `ThreadPoolExecutor` / `ProcessPoolExecutor` | 高层 API，推荐 |

---
## 六、内存管理与垃圾回收

### 6.1 引用计数

```python
import sys

a = [1, 2, 3]
print(sys.getrefcount(a))  # 2（a + getrefcount 参数）

b = a
print(sys.getrefcount(a))  # 3（a, b, getrefcount）

del b
print(sys.getrefcount(a))  # 2

# 引用计数降为 0 时立即释放内存
# ⚠️ 无法处理循环引用
```

### 6.2 垃圾回收（GC）

```python
import gc

# Python GC：引用计数 + 标记清除（处理循环引用）
# 分代回收：三代（0代最频繁，2代最少）

# 循环引用示例
class Node:
    def __init__(self, val):
        self.val = val
        self.next = None

a = Node(1)
b = Node(2)
a.next = b
b.next = a   # 循环引用
del a, b     # 引用计数不为 0，但 GC 会回收

gc.collect()            # 手动触发 GC
gc.get_threshold()      # (700, 10, 10) 默认阈值

# __del__ 方法（析构函数，不保证调用时机）
class Resource:
    def __del__(self):
        print("资源释放")   # GC 时调用，不推荐依赖此方法
```

### 6.3 内存优化技巧

```python
# __slots__：禁用 __dict__，减少内存占用
class Point:
    __slots__ = ("x", "y")   # 减少每个实例约 50% 内存

    def __init__(self, x, y):
        self.x = x
        self.y = y

# 内存视图（避免复制）
data = bytearray(b"hello world")
view = memoryview(data)
print(view[0:5].tobytes())   # b'hello'（无复制）

# 使用生成器而非列表
import sys
lst = [i for i in range(10000)]
gen = (i for i in range(10000))
print(sys.getsizeof(lst))    # ~87,632 bytes
print(sys.getsizeof(gen))    # 128 bytes

# weakref：弱引用（不增加引用计数，不阻止回收）
import weakref

class Cache:
    pass

obj = Cache()
ref = weakref.ref(obj)   # 弱引用
print(ref())             # <Cache object>
del obj
print(ref())             # None（已被回收）
```

---
## 七、性能优化

### 7.1 性能分析工具

```python
# cProfile：函数级别性能分析
import cProfile
cProfile.run("sum(range(10**6))", sort="cumtime")

# line_profiler：行级别（需安装 line_profiler）
# @profile
# def slow_function():
#     ...

# timeit：精确测量小代码片段
import timeit
t = timeit.timeit("'-'.join(str(n) for n in range(100))", number=10000)
print(f"{t:.3f}s")

# memory_profiler：内存分析（需安装）
# @profile
# def memory_hungry():
#     return [i for i in range(10**6)]
```

### 7.2 常见性能优化手段

```python
# 1. 使用局部变量（局部变量查找比全局快）
import math
def fast_sin(values):
    sin = math.sin    # 缓存为局部变量
    return [sin(v) for v in values]

# 2. 列表推导 vs map vs for 循环（推导最快）
data = range(10000)
# 列表推导：最快
result1 = [x**2 for x in data]
# map：接近，但需 list() 包装
result2 = list(map(lambda x: x**2, data))

# 3. 字符串拼接用 join
parts = ["a"] * 10000
"".join(parts)   # O(n)，远快于 += 循环

# 4. 集合/字典用于成员检查（O(1) vs O(n)）
big_list = list(range(100000))
big_set = set(range(100000))

# 慢：99999 in big_list  → O(n)
# 快：99999 in big_set   → O(1)

# 5. 避免在循环中重复计算
data = [1, 2, 3, 4, 5]
n = len(data)           # ✅ 只计算一次
for i in range(n):
    pass
```

### 7.3 NumPy / 向量化运算

```python
import numpy as np

# 向量化（避免 Python 循环）
a = np.array([1, 2, 3, 4, 5], dtype=np.float64)
b = np.array([2, 3, 4, 5, 6], dtype=np.float64)

# ✅ 向量化（C 层面运算，极快）
result = a * b + a ** 2

# ❌ Python 循环（慢 100x+）
result_slow = [a[i] * b[i] + a[i]**2 for i in range(len(a))]

# 广播机制
matrix = np.ones((3, 4))
row = np.array([1, 2, 3, 4])
result = matrix + row    # row 自动广播到 (3,4)
```

### 7.4 Cython / C 扩展

```python
# 使用 Cython 加速计算密集型代码（.pyx 文件）
# 也可使用 ctypes / cffi 调用 C 库

# numba：JIT 编译（数值计算）
from numba import jit

@jit(nopython=True)
def fast_sum(arr):
    result = 0.0
    for x in arr:
        result += x
    return result

# lru_cache：缓存（避免重复计算）
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)
```

---
## 八、常用标准库与第三方库

### 8.1 标准库常用模块

```python
# os / pathlib：文件路径操作
from pathlib import Path

p = Path("/home/user/data")
p.mkdir(parents=True, exist_ok=True)
files = list(p.glob("*.csv"))          # 文件匹配

# json：序列化
import json
data = {"name": "Alice", "score": 98.5}
json_str = json.dumps(data, ensure_ascii=False, indent=2)
obj = json.loads(json_str)

# logging：日志
import logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    handlers=[
        logging.StreamHandler(),
        logging.FileHandler("app.log")
    ]
)
logger = logging.getLogger(__name__)
logger.info("启动成功")

# datetime：日期时间
from datetime import datetime, timedelta, timezone
now = datetime.now(tz=timezone.utc)
tomorrow = now + timedelta(days=1)
fmt = now.strftime("%Y-%m-%d %H:%M:%S")

# itertools：迭代工具
import itertools
groups = itertools.groupby(sorted([1,1,2,2,3]), key=lambda x: x)
chain = itertools.chain([1,2], [3,4], [5,6])   # 合并迭代器
combos = list(itertools.combinations([1,2,3], 2))  # [(1,2),(1,3),(2,3)]
```

### 8.2 常用第三方库

| 库 | 用途 | 核心 API |
|---|---|---|
| `requests` | HTTP 客户端 | `get/post/session` |
| `aiohttp` | 异步 HTTP | `ClientSession` |
| `pydantic` | 数据校验/序列化 | `BaseModel` |
| `SQLAlchemy` | ORM/SQL工具包 | `Session, select` |
| `FastAPI` | 高性能 Web 框架 | `@app.get/post` |
| `celery` | 分布式任务队列 | `@app.task` |
| `redis-py` | Redis 客户端 | `StrictRedis` |
| `pandas` | 数据分析 | `DataFrame, Series` |
| `numpy` | 数值计算 | `ndarray` |
| `pytest` | 测试框架 | `@pytest.fixture` |

### 8.3 Pydantic 数据校验

```python
from pydantic import BaseModel, Field, validator
from typing import Optional, List
from datetime import datetime

class OrderItem(BaseModel):
    product_id: str
    quantity: int = Field(gt=0, description="数量必须大于0")
    price: float = Field(ge=0)

class Order(BaseModel):
    order_id: str
    user_id: str
    items: List[OrderItem]
    total: float
    created_at: datetime = Field(default_factory=datetime.now)
    status: str = "pending"

    @validator("total")
    def total_must_be_positive(cls, v):
        if v <= 0:
            raise ValueError("total must be positive")
        return round(v, 2)

    class Config:
        json_encoders = {datetime: lambda v: v.isoformat()}

# 自动类型转换 + 校验
order = Order(
    order_id="ORD-001",
    user_id="U-123",
    items=[{"product_id": "P1", "quantity": 2, "price": 99.9}],
    total=199.8
)
print(order.json(ensure_ascii=False))
```

---
## 九、实战场景题

### 场景 1：实现一个线程安全的 LRU 缓存

```python
from collections import OrderedDict
import threading

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()
        self.lock = threading.Lock()

    def get(self, key: int) -> int:
        with self.lock:
            if key not in self.cache:
                return -1
            self.cache.move_to_end(key)   # 移到末尾（最近使用）
            return self.cache[key]

    def put(self, key: int, value: int) -> None:
        with self.lock:
            if key in self.cache:
                self.cache.move_to_end(key)
            self.cache[key] = value
            if len(self.cache) > self.capacity:
                self.cache.popitem(last=False)  # 删除最久未使用（头部）

# 使用 functools.lru_cache 装饰器（内置实现，线程安全）
from functools import lru_cache

@lru_cache(maxsize=256)
def expensive_computation(n: int) -> int:
    return sum(range(n))
```

### 场景 2：异步任务调度器

```python
import asyncio
import heapq
from typing import Callable, Any

class AsyncScheduler:
    def __init__(self):
        self._tasks = []
        self._counter = 0

    def schedule(self, delay: float, func: Callable, *args) -> None:
        run_at = asyncio.get_event_loop().time() + delay
        heapq.heappush(self._tasks, (run_at, self._counter, func, args))
        self._counter += 1

    async def run(self):
        while self._tasks:
            run_at, _, func, args = heapq.heappop(self._tasks)
            now = asyncio.get_event_loop().time()
            if run_at > now:
                await asyncio.sleep(run_at - now)
            if asyncio.iscoroutinefunction(func):
                await func(*args)
            else:
                func(*args)
```

### 场景 3：装饰器实现 API 限流

```python
import time
import functools
from collections import deque
import threading

def rate_limit(max_calls: int, period: float):
    """滑动窗口限流装饰器"""
    def decorator(func):
        calls = deque()
        lock = threading.Lock()

        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            with lock:
                now = time.time()
                # 移除窗口外的调用记录
                while calls and calls[0] <= now - period:
                    calls.popleft()

                if len(calls) >= max_calls:
                    sleep_time = period - (now - calls[0])
                    raise RuntimeError(
                        f"Rate limit exceeded. Retry after {sleep_time:.2f}s"
                    )
                calls.append(now)
            return func(*args, **kwargs)
        return wrapper
    return decorator

@rate_limit(max_calls=5, period=1.0)  # 每秒最多 5 次
def api_call(endpoint: str):
    return f"调用 {endpoint}"
```

---

## 十、高频追问与陷阱题

### Q1：`is` 和 `==` 的区别？

> - `==`：比较**值**是否相等（调用 `__eq__`）
> - `is`：比较**内存地址**是否相同（即是否是同一对象）
>
> ```python
> a = [1, 2, 3]
> b = [1, 2, 3]
> print(a == b)   # True（值相同）
> print(a is b)   # False（不同对象）
>
> # 小整数缓存（-5 ~ 256）和字符串驻留（intern）
> x = 256; y = 256
> print(x is y)   # True（缓存）
> x = 257; y = 257
> print(x is y)   # False（可能，CPython 实现细节）
>
> # 正确用法：None / True / False 用 is 比较
> if result is None:
>     pass
> ```

---

### Q2：深拷贝与浅拷贝的区别？

> ```python
> import copy
>
> original = [[1, 2], [3, 4]]
>
> # 浅拷贝：外层新对象，内层仍是引用
> shallow = copy.copy(original)
> shallow[0].append(99)
> print(original)   # [[1, 2, 99], [3, 4]]  ← 原对象受影响！
>
> # 深拷贝：递归复制所有层次
> deep = copy.deepcopy(original)
> deep[0].append(88)
> print(original)   # [[1, 2, 99], [3, 4]]  ← 原对象不受影响
>
> # list[:] / list.copy() / dict.copy() 都是浅拷贝
> ```

---
### Q3：Python 中如何实现单例模式？

> ```python
> # 方法一：模块级变量（最 Pythonic）
> # singleton.py 中定义对象，import 它即可（模块只加载一次）
>
> # 方法二：__new__
> class Singleton:
>     _instance = None
>     def __new__(cls, *args, **kwargs):
>         if cls._instance is None:
>             cls._instance = super().__new__(cls)
>         return cls._instance
>
> # 方法三：元类
> class SingletonMeta(type):
>     _instances = {}
>     def __call__(cls, *args, **kwargs):
>         if cls not in cls._instances:
>             cls._instances[cls] = super().__call__(*args, **kwargs)
>         return cls._instances[cls]
>
> # 方法四：装饰器
> def singleton(cls):
>     instances = {}
>     @functools.wraps(cls)
>     def get_instance(*args, **kwargs):
>         if cls not in instances:
>             instances[cls] = cls(*args, **kwargs)
>         return instances[cls]
>     return get_instance
> ```

---

### Q4：`*args` 和 `**kwargs` 的使用？

> ```python
> def func(*args, **kwargs):
>     print(args)    # tuple
>     print(kwargs)  # dict
>
> func(1, 2, 3, name="Alice", age=25)
> # (1, 2, 3)
> # {'name': 'Alice', 'age': 25}
>
> # 仅关键字参数（* 后面的参数必须以关键字传入）
> def strict(a, b, *, keyword_only):
>     pass
>
> strict(1, 2, keyword_only=3)   # ✅
> strict(1, 2, 3)                # ❌ TypeError
>
> # 解包操作
> lst = [1, 2, 3]
> d = {"x": 1, "y": 2}
> func(*lst, **d)
> ```

---

### Q5：`__new__` 和 `__init__` 的区别？

> - `__new__`：**创建**对象（分配内存），返回新实例，静态方法
> - `__init__`：**初始化**对象（设置属性），返回 None
>
> ```python
> class MyClass:
>     def __new__(cls, *args, **kwargs):
>         print(f"创建实例: {cls}")
>         instance = super().__new__(cls)
>         return instance
>
>     def __init__(self, value):
>         print(f"初始化: {value}")
>         self.value = value
>
> # 使用场景：不可变类型子类化
> class PositiveInt(int):
>     def __new__(cls, value):
>         if value <= 0:
>             raise ValueError("Must be positive")
>         return super().__new__(cls, value)   # int 是不可变的，必须在 __new__ 中传值
> ```

---

### Q6：如何避免内存泄漏？

> 常见原因及解决方案：
>
> 1. **全局变量/类变量持有大对象** → 及时置 None 或使用 `weakref`
> 2. **循环引用** → Python GC 会处理，但可手动调用 `gc.collect()`
> 3. **未关闭的文件/连接** → 使用 `with` 语句确保资源释放
> 4. **无限增长的缓存** → 使用 `lru_cache(maxsize=N)` 或 TTL 缓存
> 5. **线程泄漏** → 使用线程池而非手动创建无限线程
> 6. **闭包持有大对象** → 检查闭包捕获的变量

---

### Q7：Python 的 GIL 什么时候会释放？

> GIL 会在以下情况释放：
> 1. **I/O 操作**：文件读写、网络请求等（系统调用期间释放）
> 2. **time.sleep()**：睡眠期间释放
> 3. **每执行 100 个字节码**（Python 3.2+ 改为每 5ms 检查一次）
> 4. **C 扩展**：可以在 C 代码中手动释放 GIL（如 NumPy 的矩阵运算）
>
> 因此，多线程在 I/O 密集场景下依然有效，CPU 密集场景需用多进程。

---

## 附录：面试前必看清单

- [ ] 理解 Python 的动态类型、强类型和一切皆对象
- [ ] 熟悉可变与不可变对象的区别及常见陷阱（默认参数、闭包变量）
- [ ] 能写出完整的装饰器（带参数、保留元信息）
- [ ] 理解生成器和迭代器的工作原理及内存优势
- [ ] 掌握 GIL 的影响及多线程/多进程/asyncio 的适用场景
- [ ] 了解 Python 的内存管理（引用计数 + 分代 GC）
- [ ] 熟悉 `__slots__`、`weakref`、`lru_cache` 等内存优化手段
- [ ] 能实现常见设计模式（单例、装饰器、工厂）
- [ ] 了解 MRO（C3 线性化）和 `super()` 的工作原理
- [ ] 熟悉 `dataclass`、`pydantic` 等现代 Python 工具

---

*文档版本：Python 3.11+ 适用 ｜ 最后更新：2025 年*
