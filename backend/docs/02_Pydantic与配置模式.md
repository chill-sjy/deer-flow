# Python Pydantic 与配置模式学习笔记

> 作者视角：有 Java 多线程背景，通过对比建立 Python Pydantic 心智模型。

---

## 1. BaseModel 是什么

`BaseModel` 来自 **Pydantic** 库，是 Python 生态里最流行的数据校验框架。

**Java 类比：`BaseModel` ≈ Java Bean + `@Min/@Max` 校验注解 + Jackson 序列化，三合一。**

### Java 写法（需要大量手动代码）

```java
public class TitleConfig {
    private boolean enabled = true;

    @Min(1) @Max(20)
    private int maxWords = 6;

    @Min(10) @Max(200)
    private int maxChars = 60;

    // 还需要写 getter/setter/equals/hashCode...
    public void setMaxWords(int maxWords) {
        if (maxWords < 1 || maxWords > 20) throw new IllegalArgumentException(...);
        this.maxWords = maxWords;
    }
}
```

### Python Pydantic 写法（继承 BaseModel，自动完成上面所有事）

```python
from pydantic import BaseModel, Field

class TitleConfig(BaseModel):
    enabled: bool = True
    max_words: int = Field(default=6, ge=1, le=20)   # ge=大于等于, le=小于等于
    max_chars: int = Field(default=60, ge=10, le=200)
    model_name: str | None = None                     # 可以是 str 或 None
```

Pydantic 自动提供：
- **类型检查**：传入错误类型自动抛出 `ValidationError`
- **范围校验**：`ge/le/gt/lt` 等同于 `@Min/@Max`
- **序列化**：`.model_dump()` → dict，`.model_dump_json()` → JSON 字符串
- **反序列化**：`TitleConfig(**dict)` 或 `TitleConfig.model_validate(dict)`

---

## 2. Field() 常用参数对照表

| Pydantic Field 参数 | Java 注解等价 | 含义 |
|---|---|---|
| `default=6` | 字段默认值 | 不传时的默认值 |
| `ge=1` | `@Min(1)` | 大于等于（greater than or equal） |
| `le=20` | `@Max(20)` | 小于等于（less than or equal） |
| `gt=0` | `@Min(1)`（严格） | 严格大于 |
| `lt=100` | `@Max(99)`（严格） | 严格小于 |
| `description="..."` | `@Schema(description="...")` | 字段说明（API 文档用） |
| `default_factory=lambda: []` | 无直接等价 | 每次生成新默认值（避免共享引用） |

---

## 3. `**config_dict` 字典解包语法

`**` 是 Python 的**字典解包**运算符，把字典的每个 key-value 展开成关键字参数。

```python
config_dict = {"enabled": True, "max_words": 8, "max_chars": 50}

# 以下两行完全等价：
TitleConfig(**config_dict)
TitleConfig(enabled=True, max_words=8, max_chars=50)
```

**Java 类比**：没有直接等价语法，最接近的是用 `ObjectMapper` 从 Map 构造对象：

```java
// Java
Map<String, Object> configDict = Map.of("enabled", true, "maxWords", 8);
TitleConfig config = objectMapper.convertValue(configDict, TitleConfig.class);
```

### 常见使用场景

```python
# 场景1：从 JSON/字典构造配置对象
config = TitleConfig(**json.loads(config_json_str))

# 场景2：合并字典
defaults = {"enabled": True, "max_words": 6}
overrides = {"max_words": 10}
merged = {**defaults, **overrides}   # {"enabled": True, "max_words": 10}
# 类比 Java 的 Map.putAll()

# 场景3：函数调用时展开参数
def create_config(enabled, max_words, max_chars):
    ...

params = {"enabled": True, "max_words": 6, "max_chars": 60}
create_config(**params)   # 等价于 create_config(enabled=True, max_words=6, max_chars=60)
```

---

## 4. 全局配置单例模式

项目中每个 config 文件都有这个固定结构：

```python
# 全局单例
_title_config: TitleConfig = TitleConfig()   # 模块加载时创建默认配置

def get_title_config() -> TitleConfig:
    """读取当前配置"""
    return _title_config

def set_title_config(config: TitleConfig) -> None:
    """替换整个配置对象"""
    global _title_config
    _title_config = config

def load_title_config_from_dict(config_dict: dict) -> None:
    """从字典（通常来自 JSON/YAML）加载配置"""
    global _title_config
    _title_config = TitleConfig(**config_dict)
```

**Java 类比**：等同于 Spring 的 `@ConfigurationProperties` + 单例 Bean：

```java
@Configuration
@ConfigurationProperties(prefix = "title")
public class TitleConfig {
    private boolean enabled = true;
    private int maxWords = 6;
    // ...
}
// Spring 容器管理单例，通过 @Autowired 注入
```

Python 这里是用模块级变量（`_title_config`）手动管理单例，`global` 关键字声明"我要修改模块级变量"。

---

## 5. 类型注解备忘

项目中常见的 Python 类型注解与 Java 对照：

| Python 类型注解 | Java 等价 | 含义 |
|---|---|---|
| `str` | `String` | 字符串 |
| `int` | `int` / `Integer` | 整数 |
| `float` | `float` / `Double` | 浮点数 |
| `bool` | `boolean` / `Boolean` | 布尔值 |
| `str \| None` | `@Nullable String` / `Optional<String>` | 可以为 None |
| `list[str]` | `List<String>` | 字符串列表 |
| `dict[str, int]` | `Map<String, Integer>` | 字典/映射 |
| `Literal["a", "b"]` | `enum` | 限定为特定值之一 |
| `int \| float` | 无直接等价（类似 Union 类型） | 可以是 int 或 float |
