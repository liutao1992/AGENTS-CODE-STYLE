# Java 编码规范

本文档定义项目 Java 编码规范。

参考《阿里巴巴 Java 开发手册》，结合本项目实际情况制定。

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

---

## 3. 模型分类与 Package 归属

项目中的数据模型必须根据实际职责分类。

默认模型体系：

```text
Request
→ 接口输入模型

Query
→ 查询条件模型

DTO
→ 应用内部数据传输模型

BO
→ 业务处理模型

DO
→ 持久化数据模型

VO
→ 视图输出模型
```

不同模型职责不同，不得因为“都是保存字段的 Java 类”就统一放入 `dto`。

---

### 3.1 Request

Request 表达接口输入。

例如：

```java
PlaceCreateRequest
PlaceUpdateRequest
PlaceAuditRequest
```

默认 Package：

```text
<module>.request
```

例如：

```text
place.request.PlaceCreateRequest
place.request.PlaceUpdateRequest
place.request.PlaceAuditRequest
```

Request 主要用于：

```text
HTTP Request
      ↓
Controller
      ↓
Service
```

Request 可以包含：

```java
@NotBlank
@Size
@NotNull
@Min
```

等接口结构性校验。

Request 不承担：

* 数据库存储职责；
* 查询结果职责；
* 视图输出职责；
* 复杂业务行为。

---

### 3.2 Query

Query 表达查询条件。

例如：

```java
PlaceQuery
PersonQuery
CaseQuery
```

默认 Package：

```text
<module>.query
```

例如：

```text
place.query.PlaceQuery
```

Query 可以在：

```text
Controller
    ↓
Service
    ↓
Mapper
```

之间传递查询条件。

Query 不属于 DTO，也不得因为 Mapper 使用它就放入：

```text
<module>.mapper
```

例如禁止：

```text
place.dto.PlaceQuery
place.mapper.PlaceQuery
```

应使用：

```text
place.query.PlaceQuery
```

---

### 3.3 DTO

DTO 表达应用内部的数据传输。

默认 Package：

```text
<module>.dto
```

DTO 只有在确实存在内部数据传输需求时才创建。

例如：

```text
Service
   ↓
Manager

模块 A
   ↓
模块 B 的应用接口
```

之间存在独立的数据传输模型时，可以使用 DTO。

不得把 DTO 当作所有数据对象的统一目录。

禁止机械创建：

```text
PlaceCreateDTO
PlaceResponseDTO
PlaceQueryDTO
```

仅仅因为这些类包含数据字段。

如果一个模块没有真正的 DTO 需求：

```text
<module>.dto
```

可以不存在。

---

### 3.4 BO

BO 表达业务处理过程中形成的业务对象。

默认 Package：

```text
<module>.bo
```

例如：

```java
AuditResultBO
PlaceRegistrationBO
```

只有业务逻辑确实需要一个独立于 Request、Query、DO、VO 的中间业务模型时，才创建 BO。

不得机械执行：

```text
Request
   ↓
DTO
   ↓
BO
   ↓
DO
   ↓
BO
   ↓
VO
```

简单业务允许：

```text
Request
   ↓
Service
   ↓
DO
   ↓
VO
```

原则：

> 没有真实职责，就不要增加中间模型。

---

### 3.5 DO

DO 表达数据库持久化数据。

例如：

```java
PlaceDO
PersonDO
CaseDO
```

默认 Package：

```text
<module>.domain
```

例如：

```text
place.domain.PlaceDO
```

DO 使用 Java 英文业务语义。

数据库：

```text
csxx
csbh
csmc
```

Java：

```text
PlaceDO
placeCode
placeName
```

数据库与 Java 之间通过 MyBatis 显式映射：

```text
Database（拼音）
       ↓
Mapper / ResultMap
       ↓
DO（英文）
```

DO 不要求机械复制数据库表名。

例如数据库表：

```text
ryxx
```

Java 可以使用：

```java
PersonDO
```

而不是：

```java
RyxxDO
```

涉及数据库映射时读取：

- [mybatis.md](mybatis.md)

---

### 3.6 VO

VO 表达提供给视图层或客户端的输出数据。

例如：

```java
PlaceVO
PlaceStatsVO
PlaceTreeNodeVO
```

默认 Package：

```text
<module>.vo
```

例如：

```text
place.vo.PlaceVO
place.vo.PlaceStatsVO
place.vo.PlaceTreeNodeVO
```

典型调用：

```text
Service
   ↓
VO
   ↓
Controller
   ↓
HTTP Response
```

具体业务返回模型统一优先使用 VO 表达。

例如：

```text
PlaceResponse
```

如果它实际表达的是场所接口返回数据，应命名为：

```text
PlaceVO
```

并放入：

```text
place.vo
```

不应放入：

```text
place.dto
```

也不应为了 HTTP 返回专门建立：

```text
place.response
```

业务视图模型统一使用：

```text
VO
```

表达。

原则：

> VO 表达视图数据，Response 是 HTTP 概念，不作为本项目业务模型分类体系。

---

### 3.7 通用模型

只有真正与具体业务模块无关，并且能够被多个模块复用的模型，才能进入 `common`。

例如：

```java
PageResponse<T>
```

如果它只是统一分页返回结构，不包含 Place、Case 等具体业务语义，可以放入：

```text
common.web
```

或项目已有统一公共 Web 模型目录。

而：

```java
PlaceVO
PlaceStatsVO
PlaceTreeNodeVO
```

具有明确 Place 业务语义，因此必须保留在：

```text
place.vo
```

不得因为多个 Controller 都可能使用，就移动到 `common`。

原则：

> 通用结构进入 common，具体业务模型留在业务模块。

---

### 3.8 模型归属判断

创建模型前必须先判断：

```text
这个对象是什么？
        ↓
接口输入？
→ Request

查询条件？
→ Query

内部数据传输？
→ DTO

业务处理中间对象？
→ BO

数据库持久化？
→ DO

视图输出？
→ VO
```

不得根据：

```text
它只是保存字段
```

判断：

```text
它就是 DTO
```

也不得根据：

```text
当前正在开发 Place
```

推导：

```text
所有模型都放 place.dto
```

Package 由模型职责决定。

---

### 3.9 创建模型前必须搜索

新增：

```text
Request
Query
DTO
BO
DO
VO
```

之前必须先搜索：

1. 是否已经存在相同或类似模型；
2. 当前模块是否已经存在统一 Package；
3. 是否能够复用已有模型；
4. 当前模型实际承担什么职责；
5. 是否真的需要新的模型类型。

禁止先把模型统一创建到：

```text
dto
```

然后再根据使用方式调整。

创建流程：

```text
确定职责
    ↓
Request / Query / DTO / BO / DO / VO？
    ↓
搜索已有模型
    ↓
判断是否需要新增
    ↓
确定 Package
    ↓
创建文件
```

---

## 4. Lombok 与模型对象

项目业务模型默认使用普通 Java class，不主动使用 Java `record`。

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

### 优先使用 `@Getter` 和 `@Setter`

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

### 不机械添加 Setter

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

### 业务行为不能被 Setter 替代

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

1. 普通 Getter / Setter 优先使用 Lombok。
2. 不新增无意义的手写 Getter / Setter。
3. 已使用 Lombok 的类，不重复手写普通访问器。
4. 不机械使用 `@Data`。
5. 不删除具有校验、转换或业务语义的方法。
6. 不为了生成 Setter 破坏对象原有的不变性。
7. 不主动使用 `record` 替代项目现有模型风格。

---

## 5. 方法设计

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

---

## 6. equals 与 Null

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

## 7. Optional

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

## 8. 集合

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

## 9. BigDecimal

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

## 10. 日期时间

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

## 11. 控制语句

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

## 12. 异常

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

## 13. 日志

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

## 14. 注释

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

## 15. 不可变性

没有必要修改的数据尽量保持不可变。

局部变量在有助于表达意图时可以使用 `final`。

避免共享可变静态状态。

---

## 16. 代码修改原则

Codex 修改 Java 代码时必须：

1. 先查看类似实现。
2. 延续当前模块命名和代码风格。
3. 优先复用已有工具和模型。
4. 不顺带重构无关代码。
5. 不为了“更优雅”改变已有架构。
6. 行为发生变化时检查是否需要测试。
7. 新增模型前先判断 Request / Query / DTO / BO / DO / VO 职责。
8. Package 根据模型职责确定，不统一放入 `dto`。
9. 业务输出模型统一使用 VO，不机械创建 `*Response` 业务模型。
10. 模型对象的普通 Getter / Setter 优先使用 Lombok。
11. 不为了 Lombok 删除具有业务语义的方法。
12. 不机械使用 `@Data`。
13. 不主动使用 `record` 替代普通模型 class。
14. 不为了形式机械增加 DTO / BO / Converter 等中间模型。

最终原则：

> 先确定职责，再确定模型类型和 Package；Request 管输入，Query 管查询，DTO 管内部传输，BO 管业务处理，DO 管持久化，VO 管视图输出。

