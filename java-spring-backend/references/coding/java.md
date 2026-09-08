# Java 编码规范

本文档定义项目 Java 编码规范。

参考《阿里巴巴 Java 开发手册》，结合本项目实际情况制定。

模型的职责分类与 Package 归属属于架构边界，由应用分层规范统一维护：

- [应用分层与模型边界](../architecture/layering.md)

本文主要规定 Java 层面的实现方式，例如命名、类设计、Lombok、`class` / `record`、方法、集合、异常和日志等。

---

## 1. 命名

### 类

使用 `UpperCamelCase`：

```java
PlaceService
PlaceQuery
PlaceDetailVO
```

### 方法、字段、参数

使用 `lowerCamelCase`：

```java
placeName
identityNumber
queryPlaceDetail()
```

### 常量

使用：

```java
UPPER_SNAKE_CASE
```

例如：

```java
private static final int MAX_RETRY_COUNT = 3;
```

### 包

统一使用小写英文。

禁止：

* Java 标识符使用汉语拼音；
* 中文标识符；
* 拼音和英文混合；
* 含义不明确的缩写。

数据库可以使用拼音，但 Java 代码必须使用英文业务语义。

Package 的职责归属读取：

- [layering.md](../architecture/layering.md)

---

## 2. 类设计

一个类应具有明确职责。

禁止：

* 无限制扩大的 `Utils`；
* 一个类承担多个无关业务；
* 为未来可能出现的需求提前创建抽象；
* 仅为了形式创建无意义接口。

优先复用已有类和已有设计。

创建新类前，应先搜索是否存在职责相近的实现。

原则：

> 先确定类的职责，再确定类名和 Package，最后创建文件。

不得因为当前正在开发某个业务模块，就把所有新类机械放入该模块现有目录。

类、接口和 Package 的架构职责以及 SOLID 判断统一读取：

- [layering.md](../architecture/layering.md)

---

## 3. 模型对象的 Java 实现

Request、Query、DTO、BO、DO、VO 的职责、边界、命名语义与 Package 归属由应用分层规范统一定义：

- [模型分类与 Package 归属](../architecture/layering.md#9-模型分类与-package-归属)

本文只规定这些模型在 Java 层面的实现方式。

项目业务模型默认使用普通 Java `class`，不主动使用 Java `record`。

以下模型默认使用普通 Java class：

```text
DO
DTO
BO
VO
Query
Request
```

并使用 Lombok 生成无业务逻辑的 Getter / Setter。

除非当前模块已经明确统一使用 `record`，或者任务明确要求使用 `record`，否则不得为了减少样板代码自行将模型设计为 `record`。

推荐：

```java
@Getter
@Setter
public class PlaceDO {

    private String id;

    private String placeName;

    private PlaceStatus status;
}
```

禁止仅为了字段访问手工编写：

```java
public String getPlaceName() {
    return placeName;
}

public void setPlaceName(String placeName) {
    this.placeName = placeName;
}
```

### 3.1 优先使用 `@Getter` 和 `@Setter`

普通模型对象优先使用：

```java
@Getter
@Setter
```

不要机械使用：

```java
@Data
```

`@Data` 除 Getter / Setter 外，还会生成：

```text
equals
hashCode
toString
@RequiredArgsConstructor
```

这些行为可能影响：

* 对象相等性；
* `Set` / `Map` 行为；
* 日志输出；
* 敏感字段暴露；
* MyBatis / 持久化对象行为。

只有确认这些自动生成行为符合对象语义时，才可以使用 `@Data`。

### 3.2 不机械添加 Setter

如果对象不允许字段被任意修改，可以只使用：

```java
@Getter
```

例如：

```java
@Getter
public class AuditResultBO {

    private final String placeId;

    private final AuditStatus status;

    public AuditResultBO(
            String placeId,
            AuditStatus status) {
        this.placeId = placeId;
        this.status = status;
    }
}
```

不要为了统一 Lombok 而机械增加：

```java
@Setter
```

破坏对象原有的不变性。

### 3.3 业务行为不能被 Setter 替代

具有业务规则的方法应继续显式定义。

例如：

```java
public void changeStatus(PlaceStatus targetStatus) {
    if (!status.canTransitionTo(targetStatus)) {
        throw new PlaceStatusException();
    }

    this.status = targetStatus;
}
```

不要为了使用 Lombok 将其替换成：

```java
setStatus(...)
```

原则：

> Lombok 用于消除无业务价值的样板代码，不用于绕过业务规则。

Codex 创建或修改模型对象时必须：

1. 先按分层规范确定模型职责和 Package。
2. 普通 Getter / Setter 优先使用 Lombok。
3. 不新增无意义的手写 Getter / Setter。
4. 已使用 Lombok 的类，不重复手写普通访问器。
5. 不机械使用 `@Data`。
6. 不删除具有校验、转换或业务语义的方法。
7. 不为了生成 Setter 破坏对象原有的不变性。
8. 不主动使用 `record` 替代项目现有模型风格。

---

## 4. 方法设计

方法应完成一个明确操作。

优先：

* 业务语义明确的方法名；
* Guard Clause；
* 较浅的嵌套层级；
* 清晰的输入和输出。

推荐：

```java
auditPlace(...)
registerCase(...)
bindEquipment(...)
```

避免：

```java
handle(...)
process(...)
doSomething(...)
```

避免方法参数过多。

查询条件较多时（超过 3 个时），应封装为 Query 对象，避免使用过多方法参数或 `Map<String, Object>` 传递普通业务查询条件。

Query 的职责与 Package 归属读取：

- [layering.md](../architecture/layering.md#92-query)

---

## 5. equals 与 Null

不要在可能为空的对象上调用：

```java
value.equals(...)
```

推荐：

```java
Objects.equals(a, b);
```

或者：

```java
"ACTIVE".equals(status);
```

当空集合能够表达相同含义时，不返回 `null`：

```java
return Collections.emptyList();
```

---

## 6. Optional

`Optional` 主要用于表达“返回值可能不存在”。

避免用于：

* Entity / DO 字段；
* DTO 字段；
* 普通方法参数。

不要直接：

```java
optional.get();
```

优先：

```java
optional.orElseThrow(...);
```

---

## 7. 集合

声明优先使用接口：

```java
List<String> names = new ArrayList<>();
Map<String, PlaceDO> placeMap = new HashMap<>();
```

不要在增强 `for` 中直接进行不安全的删除。

需要删除时使用：

* Iterator；
* `removeIf`；
* 合适的集合 API。

用作 `Map Key` 或 `Set` 元素的对象，应正确实现：

```java
equals()
hashCode()
```

不要依赖未明确保证的集合遍历顺序。

---

## 8. BigDecimal

禁止：

```java
new BigDecimal(0.1)
```

推荐：

```java
new BigDecimal("0.1");
BigDecimal.valueOf(0.1);
```

数值比较优先：

```java
amount.compareTo(BigDecimal.ZERO)
```

除法必须明确考虑：

* scale；
* roundingMode。

---

## 9. 日期时间

新代码优先使用：

```text
LocalDate
LocalDateTime
Instant
Duration
ZonedDateTime
```

避免新增：

```text
Date
Calendar
SimpleDateFormat
```

不要通过格式化后的字符串比较时间。

业务涉及时区时必须明确时区语义。

---

## 10. 控制语句

`if`、`else`、`for`、`while` 等必须使用大括号。

推荐：

```java
if (place == null) {
    throw new PlaceNotFoundException(id);
}

if (!place.canAudit()) {
    throw new PlaceStatusException(id);
}
```

避免过深嵌套。

---

## 11. 异常

禁止：

```java
catch (Exception e) {
}
```

禁止：

* 吞异常；
* 使用异常完成普通流程控制；
* 无意义重复包装异常；
* 随意捕获 `Throwable`。

异常转换应发生在有明确抽象边界的位置。

Spring / HTTP 异常边界读取：

- [spring.md](spring.md)

---

## 12. 日志

禁止：

```java
System.out.println(...)
System.err.println(...)
exception.printStackTrace()
```

使用项目统一日志框架。

推荐参数化日志：

```java
log.info("Audit place successfully, placeId={}", placeId);
```

异常日志：

```java
log.error("Audit place failed, placeId={}", placeId, ex);
```

禁止输出：

* 密码；
* Token；
* Secret；
* 私钥；
* 敏感凭证。

---

## 13. 注释

注释重点解释：

* 为什么这样实现；
* 特殊业务规则；
* 不明显的技术限制；
* 重要设计取舍。

避免：

```java
// 设置名称
user.setName(name);
```

不要保留已经废弃的大段注释代码。

---

## 14. 不可变性

没有必要修改的数据尽量保持不可变。

局部变量在有助于表达意图时可以使用 `final`。

避免共享可变静态状态。

---

## 15. Codex Java 修改原则

Codex 修改 Java 代码时必须：

1. 先查看类似实现。
2. 延续当前模块命名和代码风格。
3. 优先复用已有工具和模型。
4. 不顺带重构无关代码。
5. 不为了“更优雅”改变已有架构。
6. 行为发生变化时检查是否需要测试。
7. 新增模型或调整 Package 时，先读取 `layering.md` 确定职责与归属。
8. 本文只负责模型的 Java 实现方式，不在这里重新定义 Request / Query / DTO / BO / DO / VO 的架构职责。
9. 模型对象的普通 Getter / Setter 优先使用 Lombok。
10. 不为了 Lombok 删除具有业务语义的方法。
11. 不机械使用 `@Data`。
12. 不主动使用 `record` 替代普通模型 class。
13. 不为了形式机械增加 DTO / BO / Converter 等中间模型。

最终原则：

> 架构规范决定“类是什么、放哪里”；Java 规范决定“类在 Java 中如何实现”。
