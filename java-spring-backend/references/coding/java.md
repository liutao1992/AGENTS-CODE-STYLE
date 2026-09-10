# Java 编码规范

本文档定义 Java 语言层面的实现规范。

本文负责回答：

> Java 类、方法、参数、集合、Null、异常、日志、注释和格式应该怎么写。

模型职责和 Package 归属读取：

- [应用分层与模型边界](../architecture/layering.md)

Spring、API、事务、并发、MyBatis 等专项行为由对应 reference 维护，本文不重复定义。

核心原则：

> 优先保证语义清晰、行为正确和项目一致性；规则用于降低理解成本和常见错误，不用于机械改造已有合理代码。

---

## 1. 命名

### 1.1 类名

类名使用 `UpperCamelCase`，表达真实职责：

```java
PlaceService
PlaceQuery
PlaceDetailVO
PlaceNotFoundException
```

避免无语义名称：

```text
CommonService
DataHandler
BizUtil
ProcessHelper
```

抽象类、接口、实现类只在职责真实存在时创建，不为了命名模板机械产生 `Interface + Impl`。

### 1.2 方法、字段、参数和局部变量

使用 `lowerCamelCase`，优先完整英文业务语义：

```java
identityNumber
expirationTime
queryPlaceDetail()
```

避免无行业共识的随意缩写、拼音、中文标识符以及拼音英文混合。

集合或类型语义需要表达时可使用：

```text
nameList
placeMap
workQueue
```

不要机械添加 `String`、`Integer` 等无价值类型后缀。

### 1.3 常量与魔法值

常量使用 `UPPER_SNAKE_CASE`：

```java
private static final int MAX_RETRY_COUNT = 3;
```

【强制】代码中不得直接出现带有业务或技术语义、但没有预先命名说明的魔法值。

所谓魔法值，是指某个固定字面量承担了状态、类型、阈值、超时、重试次数、缓存时间、业务编码等实际语义，但读者只能依赖上下文猜测它的含义。

避免：

```java
if ("1".equals(status)) {
    ...
}

if (retryCount >= 3) {
    ...
}

cache.put(key, value, 300);
```

应优先使用有名称的常量或 Enum：

```java
if (PlaceStatus.ENABLED.getCode().equals(status)) {
    ...
}

if (retryCount >= MAX_RETRY_COUNT) {
    ...
}

cache.put(key, value, CacheConsts.PLACE_DETAIL_TTL_SECONDS);
```

以下类型的值通常需要先命名再使用：

```text
业务状态 / 类型编码
权限 / 来源编码
缓存 TTL
超时时间
重试次数
批处理阈值
固定业务比例
具有业务意义的数字或字符串
```

“禁止魔法值”不等于把所有 Java 字面量机械抽成常量。没有独立业务 / 技术语义、含义由语言结构本身即可理解的值可以直接使用，例如：

```java
for (int i = 0; i < items.size(); i++) {
    ...
}

if (name.isEmpty()) {
    ...
}
```

是否提取常量的判断重点是：

> 这个值如果变化，维护者是否需要先知道“它代表什么”才能安全修改？

如果答案是“需要”，就不应以匿名字面量散落在代码中。

### 1.4 Package

Package 使用小写英文。职责词汇例如：

```text
controller
service
manager
mapper
client
adapter
request
dto
bo
domain
vo
```

`Query` 是模型语义，不要求对应独立 `query` Package；当前默认与 Request 一起放入 `request`。具体职责和 Package 归属统一读取 `layering.md`；项目物理目录读取 `project-structure.md`。

### 1.5 设计模式角色应体现在命名中

当模块 / Package、接口、类或方法**确实承担某种设计模式的明确角色**时，命名应尽量体现该模式或其角色语义，使阅读者不需要先展开实现就能够理解主要架构意图。

典型类型命名例如：

```text
PaymentStrategy
DefaultPaymentStrategy
NotificationFactory
StorageAdapter
PlaceBuilder
AuditHandler
AuditHandlerChain
ExportCommand
ExportCommandHandler
```

如果一个技术子模块或职责 Package 本身就是围绕某种模式组织，也可以使用能够直接表达模式角色的名称，例如：

```text
strategy
factory
adapter
handler
command
```

但业务模块仍优先表达业务能力，例如 `place`、`casecenter`；不能因为模块内部使用了一个 Strategy 就把整个业务模块机械改名为 `placeStrategy`。

方法名称优先体现该模式的典型职责动作，而不是无意义附加模式后缀。例如：

```text
Builder      → build(...)
Factory      → create(...) / createXxx(...)
Command      → execute(...)
Handler      → handle(...)
Visitor      → visit(...) / accept(...)
```

因此普通业务方法应避免没有上下文的 `handle()`、`execute()`；但如果所属类型已经明确是：

```text
PlaceAuditHandler
ExportCommand
```

那么：

```java
handler.handle(context);
command.execute();
```

可以准确表达设计模式角色，不属于无语义命名。

不要反过来为了使用模式名称而制造并不存在的模式。以下名称只有在对应职责真实存在时才使用：

```text
Strategy
Factory
Adapter
Builder
Handler
Command
Observer
Visitor
Template
Facade
```

例如只有一个普通分支判断、没有可替换算法族时，不要把类命名成 `XxxStrategy`；普通对象创建方法也不因为返回对象就自动创建 `XxxFactory`。

设计模式是否有真实必要性、是否属于过度设计统一读取 `layering.md`。

原则：

> 先确认模式和角色真实存在，再让命名暴露架构意图；名称用于解释设计，不用于伪造设计。

---

## 2. 类设计与 OOP

一个类应围绕一个明确职责设计。

创建新类前必须先搜索已有实现，避免：

* 无限增长的 `Utils`；
* 一个类混合多个无关职责；
* 为猜测中的未来扩展提前抽象；
* 只为形式创建接口、Factory、Strategy、Repository 包装。

原则：

> 先确定职责，再确定类名和 Package，最后决定是否真的需要新类。

### 2.1 覆写使用 `@Override`

覆写父类或接口方法时显式使用：

```java
@Override
```

### 2.2 静态成员通过类型名访问

推荐：

```java
Objects.equals(a, b);
PlaceConstants.MAX_NAME_LENGTH;
```

### 2.3 可见性从严

使用满足真实调用需求的最小可见范围，不为了调用方便把内部实现全部设为 `public`。

### 2.4 不机械创建 `Interface + Impl`

只有真实存在以下需求时才引入接口：

* 多实现；
* SPI；
* 可替换能力；
* 第三方隔离；
* 稳定模块边界。

普通单实现 Service 不因为“分层规范”自动创建：

```text
PlaceService
PlaceServiceImpl
```

### 2.5 构造器和访问器保持简单

构造器不承担数据库、HTTP/RPC、文件 IO、长耗时业务流程或隐式状态流转。

普通 Getter / Setter 不隐藏业务规则和副作用。需要业务约束时使用显式业务方法：

```java
changeStatus(...)
approve(...)
activate(...)
```

---

## 3. 模型对象的 Java 实现

Request、Query、DTO、BO、DO、VO 的职责由 `layering.md` 唯一维护；本文只规定 Java 实现。

项目没有其他稳定约定时，普通模型默认使用普通 `class`，不主动把模型改成 `record`。

普通无业务逻辑访问器优先使用：

```java
@Getter
@Setter
```

不机械使用 `@Data`。`@Data` 会同时影响：

```text
equals
hashCode
toString
构造器
```

只有确认这些行为符合对象语义时才使用。

### 3.1 JavaBean 命名

JavaBean 名称应直接表达模型职责，不再附加没有独立语义的泛化后缀。

推荐：

```text
PlaceCreateRequest
PlaceQuery
PlaceDTO
PlaceAuditBO
PlaceDO
PlaceDetailVO
```

其中 `PlaceQuery` 仍通过类名表达“查询条件”语义，但默认 Package 是 `<module>.request`，不要因为类名以 `Query` 结尾就机械创建独立 `query` Package。

避免在职责已经明确时再使用：

```text
PlaceBean
PlaceInfo
PlaceData
PlaceModel
CommonBean
```

如果目标项目已经稳定使用某类历史命名，应保持兼容，不为了本规则批量迁移。

JavaBean 属性继续遵循 `lowerCamelCase` 和完整英文业务语义，例如：

```java
private String placeCode;
private LocalDateTime entryTime;
private Boolean enabled;
```

布尔属性优先使用能够直接表达状态的名称，例如：

```text
enabled
deleted
editable
auditable
```

不要为了 JavaBean 风格机械把属性命名成 `isEnabled`、`isDeleted`。如果已有 Jackson、MyBatis、RPC 或历史 API 序列化契约，则以实际属性解析和兼容性要求为准。

原则：

> JavaBean 名称表达“它是什么职责”，字段名称表达“它承载什么业务语义”；不要用 `Bean / Info / Data / Model` 代替职责设计。

### 3.2 `@Builder` 与 `@NoArgsConstructor`

对于需要在 Java 代码中频繁组装、字段较多或使用多个 Setter 会明显降低可读性的普通模型，可以优先使用 Lombok `@Builder`。

例如避免：

```java
PlaceDetailVO detail = new PlaceDetailVO();
detail.setId(place.getId());
detail.setPlaceCode(place.getPlaceCode());
detail.setPlaceName(place.getPlaceName());
detail.setEnabled(place.getEnabled());
```

可以使用：

```java
PlaceDetailVO detail = PlaceDetailVO.builder()
        .id(place.getId())
        .placeCode(place.getPlaceCode())
        .placeName(place.getPlaceName())
        .enabled(place.getEnabled())
        .build();
```

对于需要保留标准 JavaBean 无参实例化能力的普通可变模型，可以同时提供 `@NoArgsConstructor`。典型包括项目通过无参构造 + Setter 完成绑定、映射或反序列化的模型。

类级 `@Builder` 与 `@NoArgsConstructor` 同时使用时，必须保证 Builder 存在可调用的全参数构造路径。推荐写法：

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class PlaceDetailVO {

    private String id;
    private String placeCode;
    private String placeName;
    private Boolean enabled;
}
```

这样：

```text
@NoArgsConstructor
→ 保留 JavaBean / 框架需要的无参实例化能力

@Builder
→ 简化代码中的对象组装，减少连续 Setter

@AllArgsConstructor(access = AccessLevel.PRIVATE)
→ 为类级 Builder 提供完整构造路径，同时避免无必要暴露 public 全参构造器
```

不要只为了“统一 Lombok 风格”给所有对象机械增加 `@Builder`。以下场景应先判断真实需要：

* 只有一两个字段且构造非常简单；
* 对象不应允许任意字段组合；
* 对象必须通过显式构造器或工厂方法建立不变式；
* Builder 会绕过必要的业务校验或状态约束；
* 目标项目已有其他稳定构造方式。

同样，`@NoArgsConstructor` 不是所有类的默认要求。不可变对象、必须在构造阶段满足不变式的对象，不应为了 JavaBean 形式无依据增加无参构造器。

对于 Request / Query / DTO / BO / DO / VO 等 POJO，不使用 `@Builder.Default` 设置属性默认值；具体默认值规则读取 3.5 节。

Builder 只负责对象构造，不替代业务行为。例如需要状态校验的：

```java
place.approve(operator);
```

不能仅为了使用 Builder 改成任意构造：

```java
PlaceDO.builder()
        .status(APPROVED)
        .build();
```

从而绕过已有业务状态流转。

原则：

> `@Builder` 用于让对象组装更清晰，`@NoArgsConstructor` 用于满足真实 JavaBean / 框架实例化需求；二者服务于构造便利性，不得破坏对象不变式、业务规则和既有框架契约。

### 3.3 不为了统一机械增加 Setter

如果对象应保持不可变或受控修改，只暴露需要的访问方式。

业务行为不能被普通 Setter 替代。

### 3.4 基本类型与包装类型

需要表达：

```text
未提供
未知
数据库 NULL
```

时使用包装类型；不存在 Null 语义的局部计算变量可以优先使用基本类型。

不得为了统一风格批量改变已有 API / 数据库 Null 语义。

### 3.5 POJO 不设置属性默认值

【强制】定义 Request、Query、DTO、BO、DO、VO 等 POJO 时，默认不在字段声明或 Builder 中设置属性默认值。

禁止新建：

```java
private Boolean enabled = true;
private Integer sortOrder = 0;
private String status = "PENDING";
private LocalDateTime createTime = LocalDateTime.now();
```

也禁止通过：

```java
@Builder.Default
private Boolean enabled = true;
```

把业务默认值隐藏进模型构造过程。

POJO 的职责是承载数据，不应在对象定义处悄悄决定：

```text
业务状态
默认开关
默认编码
当前时间
默认排序值
```

如果业务确实存在默认值，应由明确拥有该语义的边界处理，例如：

```text
Controller / 入站转换
→ 协议明确规定的请求默认语义

Service / Manager
→ 业务规则决定的默认状态或值

Database
→ 数据库负责且已明确约定的 DEFAULT
```

不要在多个边界重复设置同一默认值。

已有历史模型如果字段默认值已经构成稳定序列化、持久化或业务契约，不为了本规则无授权批量删除；新建模型或当前任务明确调整该模型语义时按本规则执行。

原则：

> POJO 只承载值，不隐藏业务默认决策；默认值由真正拥有该语义的边界显式产生。

### 3.6 `toString` 与敏感信息

不要为了调试机械输出完整对象。密码、Token、Secret、证件、生物特征等敏感字段不得通过自动 `toString()` 或日志泄漏。

---

## 4. 常量与枚举

### 4.1 常量按职责和功能分类

不要使用一个大而全的常量类维护整个项目的所有常量。

禁止长期增长的：

```text
Constants
CommonConstants
GlobalConstants
SystemConstants
```

如果一个常量只属于某个类的内部实现，优先直接作为该类的 `private static final` 成员。

跨类复用的常量按照真实功能和语义归类到职责明确的常量类。例如：

```text
CacheConsts
SystemConfigConsts
FileUploadConsts
PlaceConsts
```

典型关系：

```text
缓存相关常量
→ CacheConsts

系统配置相关常量
→ SystemConfigConsts

文件上传技术限制
→ FileUploadConsts

Place 模块稳定业务常量
→ PlaceConsts
```

常量类不能因为“很多地方都能用”就进入一个无边界的公共垃圾桶。业务常量优先跟随拥有其语义的业务模块；真正跨模块且稳定的技术常量才进入公共技术边界。

也不要为了“分类”把每一个常量都创建一个新类。分类粒度以能够用一个清楚的职责名称解释该组常量为准。

原则：

> 常量跟随语义所有者；宁可按职责形成少量清晰常量组，也不要建立一个全局常量仓库。

### 4.2 固定取值范围优先使用 Enum

如果一个变量的合法值只会在一个明确、固定、有限的范围内变化，优先使用 Enum 表达，而不是散落字符串或数字常量。

例如：

```java
public enum PlaceStatus {
    DRAFT,
    ENABLED,
    DISABLED
}
```

典型场景包括：

```text
业务状态
审核结果
固定业务类型
固定来源类型
有限操作类型
```

如果数据库或外部 API 已经使用稳定编码，可以由 Enum 显式承载和转换该编码，而不是因此继续在业务代码中散落：

```text
"0"
"1"
"PENDING"
"FORMAL"
```

Enum 不适用于实际上开放增长、由配置中心动态增加、由外部系统随时扩展且本应用不拥有全集的值域；这种场景应按真实契约建模，不为了使用 Enum 假装值域固定。

也不得自行发明目标项目不存在的业务状态、编码或兼容映射。

原则：

> 值域真正固定时用类型系统表达范围；值域由外部或配置动态决定时尊重真实契约。

### 4.3 数值字面量

`long` 字面量使用大写 `L`：

```java
1000L
```

涉及具有业务 / 技术语义的数值时，同时遵守 1.3 的魔法值规则。

---

## 5. 方法设计与可读性

方法应完成一个明确操作，优先使用业务语义明确的方法名、Guard Clause 和较浅嵌套。

推荐：

```text
auditPlace
registerCase
bindEquipment
```

避免：

```text
handle
process
doSomething
```

当所属类型已经明确承担 Handler、Command、Visitor 等设计模式角色时，`handle`、`execute`、`visit` 等典型模式动作可以是准确命名，具体读取 1.5 节。

### 5.1 控制参数数量

普通业务方法新建或显著修改时，默认不超过 **5 个参数**。这个数字是设计检查信号，不是“必须凑到 5 个以内”的形式目标。

参数较多时按以下顺序判断：

```text
方法是否承担过多职责？
        ↓
这些参数是否共同描述一次完整操作？
        ↓
来源、生命周期、信任边界是否一致？
        ↓
决定一个语义对象还是保持多个独立参数
```

如果一组参数共同描述一次完整操作，并且来源、生命周期和信任边界一致，优先封装为一个职责明确的对象。

例如：

```java
public String signEnvelope(
        RequestPayload payload,
        String password,
        String privateCertificate,
        String publicCertificate,
        String username,
        String ip,
        String userAgent) {
    ...
}
```

如果这些数据共同构成一次签名操作的完整内部输入，优先评估：

```java
public String signEnvelope(SignEnvelopeDTO signEnvelope) {
    ...
}
```

而不是为了减少数量机械拆成多个参数袋。

但来源或信任边界明显不同的数据不要为了“单参数”强行合并。例如：

```java
public void audit(PlaceAuditRequest request, Operator operator) {
    ...
}
```

这里：

```text
PlaceAuditRequest
→ 外部业务输入

Operator
→ 服务端可信调用者上下文
```

二者职责和信任来源不同，保持分离通常更清楚；不得把可信服务端身份机械塞入客户端 Request。

参数对象应使用真实语义模型，例如项目已有的：

```text
Request
Query
DTO
BO
Context
Command（项目已有此类模型体系时）
```

不要仅为了减少参数数量创建：

```text
XxxParam
CommonParam
Object[]
Map<String, Object>
```

这种无边界参数容器。

普通查询条件超过 **3 个** 时优先评估 Query；这是查询场景更具体的规则。

框架回调、接口覆写、第三方接口和稳定公共 API 的签名受外部契约约束时，不为满足参数数量规则破坏兼容性。

原则：

> 参数封装优先表达一次操作的完整语义；是否合并由职责、来源、生命周期和信任边界决定，不由数字本身决定。

### 5.2 方法长度

新建或显著修改的方法默认控制在约 **80 行** 内。超过时检查：

* 是否职责过多；
* 是否分支复杂；
* 是否抽象层次混杂；
* 是否存在重复逻辑；
* 是否存在能够独立命名的业务步骤。

80 行不是机械拆分命令。不要为了缩短方法制造大量无语义私有方法。

### 5.3 Guard Clause

错误或不满足条件的路径优先尽早返回/抛出，避免深层嵌套；但不要为了形式把简单代码拆得支离破碎。

---

## 6. Null、Optional 与返回契约

Null 语义应由契约明确，而不是由调用方猜测。

### 6.1 集合无结果优先空集合

对于“零到多条”的集合返回，默认使用空集合表达没有数据，不使用 `null`。

当下层已经具有明确非 Null 集合契约时，上层直接依赖契约，不机械增加：

```java
list == null ? new ArrayList<>() : list
```

```java
Optional.ofNullable(list).orElseGet(Collections::emptyList)
```

```java
if (list != null) {
    ...
}
```

如果第三方、遗留接口或其他来源确实允许返回 Null，应在最靠近来源的边界归一化一次，再向上提供稳定契约。

原则：

> 边界归一化一次，上层依赖稳定契约；不要每一层都假设上一层不可靠。

MyBatis `List<T>` 的具体契约读取 `mybatis.md`。

### 6.2 单对象返回按项目契约

单对象“不存在”可以由项目约定表达为：

```text
null
Optional
异常
```

不要把集合规则机械套到单对象。

### 6.3 Optional

`Optional` 主要用于明确表达返回值可能不存在，不作为所有字段、参数和集合的默认包装。

避免：

```text
Optional<List<T>>
```

仅用于表达“集合为空”这一普通情况。

### 6.4 不用默认值隐藏错误

没有业务契约时，不把：

```text
null → ""
null → 0
异常 → 默认成功结果
非法枚举 → 默认状态
```

作为所谓防御性编程。

---

## 7. 相等判断与 Boolean

对象相等优先：

```java
Objects.equals(a, b)
```

包装 Boolean 判断优先：

```java
Boolean.TRUE.equals(enabled)
Boolean.FALSE.equals(enabled)
```

避免对可能为 Null 的包装类型直接自动拆箱。

字符串业务比较使用 `equals`，不是 `==`。

---

## 8. 集合与泛型

不使用 Raw Type：

```java
List list;
Map map;
```

改为明确泛型。

集合实现根据真实使用选择，不为了“线程安全”默认使用并发集合；涉及多线程共享时读取 `concurrency.md`。

返回集合是否可变应遵循项目契约，不因为追求不可变性无授权改变已有调用语义。

避免循环内重复数据库/远程调用形成 N+1；涉及 SQL 时读取 `sql.md`。

---

## 9. BigDecimal 与精确数值

金额、比例等精确业务数值使用 `BigDecimal`，避免使用 `double` / `float` 承担精确业务语义。

构造小数优先：

```java
BigDecimal.valueOf(0.1)
new BigDecimal("0.1")
```

避免：

```java
new BigDecimal(0.1)
```

除法必须明确舍入和精度策略，不能依赖偶然结果。

BigDecimal 比较通常使用 `compareTo` 判断数值大小；如果业务关心 scale，则按真实契约使用 `equals`。

---

## 10. 时间

新代码优先使用 `java.time` 类型：

```text
Instant
LocalDate
LocalDateTime
OffsetDateTime
ZonedDateTime
Duration
```

具体选择由业务是否涉及时区决定。

不要通过字符串截取、手工毫秒计算表达日期边界；跨时区 API 和数据库行为必须遵循项目已有契约。

测试涉及当前时间时优先使用可控 `Clock` 或项目已有时间抽象。

---

## 11. 异常的 Java 实现

异常跨层职责读取 `error-handling.md`；本文只规定 Java 实现细节。

禁止空 catch：

```java
catch (Exception ex) {
}
```

转换异常时保留 cause：

```java
throw new StorageAccessException("Failed to load attachment", ex);
```

不要机械：

```text
catch
→ log
→ wrap
→ 每层重复
```

也不要捕获异常后返回无依据的 `null`、空集合、默认状态或成功结果。

使用资源时优先 `try-with-resources`。

---

## 12. 日志

优先使用目标项目统一日志框架和现有格式。

日志应记录对排查有价值的上下文，而不是完整 dump 所有对象。

参数化日志优先：

```java
log.info("place created, placeId={}", placeId);
```

避免不必要字符串拼接。

同一异常链通常只保留一次完整堆栈；异常应该在哪一层记录读取 `error-handling.md`。

禁止记录密码、Token、Cookie、Secret、私钥、完整证件、生物特征等敏感信息。

可预期业务失败不机械使用 `error` 级别。

---

## 13. 注释与 Javadoc

注释解释：

```text
为什么这样做
关键业务约束
非显然算法
外部契约
兼容原因
```

不要用注释重复代码字面含义。

公共 API、复杂算法或容易误用的能力可以使用 Javadoc；普通显然 Getter / Setter、简单私有方法不机械增加 Javadoc。

删除失效注释和长期注释掉的旧代码，历史由 Git 保存。

---

## 14. 格式与可读性

优先遵循目标项目已有 formatter、Checkstyle、Spotless、IDE 配置和附近代码风格。

如果项目没有更具体的自动格式化规则，使用以下默认格式。

### 14.1 相邻方法之间保留空行

类、接口、枚举等类型中的相邻方法声明或方法实现之间，默认保留一个空行，避免成员连续堆叠导致阅读困难。

避免：

```java
public interface PlaceQueryMapper {
    List<PlaceRecordDO> selectPage(PlaceQuery query);
    long count(PlaceQuery query);
    PlaceRecordDO selectDetail(@Param("id") String id, @Param("scope") String scope);
    PlaceStatsDO selectStats();
    List<PlaceTreeNodeDO> selectTree();
}
```

推荐：

```java
public interface PlaceQueryMapper {

    List<PlaceRecordDO> selectPage(PlaceQuery query);

    long count(PlaceQuery query);

    PlaceRecordDO selectDetail(@Param("id") String id, @Param("scope") String scope);

    PlaceStatsDO selectStats();

    List<PlaceTreeNodeDO> selectTree();
}
```

同一方法上的注解与方法声明属于一个整体，不在注解和方法之间插入无意义空行。

原则：

> 空行用于分隔独立成员和阅读单元，不为了压缩文件把多个方法声明连续堆在一起。

### 14.2 方法签名优先保持单行

方法声明能够在项目行宽内清晰表达时，优先保持单行。不要因为存在两个或三个普通参数、参数带简单注解，就机械拆成多行。

推荐：

```java
PlaceRecordDO selectDetail(@Param("id") String id, @Param("scope") String scope);
```

只有出现以下情况之一时再换行：

* 超过项目 formatter 或团队约定的行宽；
* 参数数量较多；
* 参数类型、泛型或注解较复杂；
* 单行已经明显降低可读性。

例如：

```java
int updateStatus(
        @Param("organizationCode") String organizationCode,
        @Param("placeCode") String placeCode,
        @Param("expectedStatus") String expectedStatus,
        @Param("targetStatus") String targetStatus);
```

一旦决定换行，默认一个参数一行，并保持统一缩进。方法调用、构造器调用遵循相同原则。

原则：

> 能清晰放一行就保持一行；只有真正过长或复杂时才换行，不为了“格式统一”机械增加垂直空间。

如果目标项目 formatter 对最大行宽、续行缩进、参数换行已有明确配置，以自动格式化结果为准，不与 formatter 对抗。

### 14.3 控制语句使用大括号

控制语句统一使用大括号，避免：

```java
if (condition) return;
```

在复杂业务代码中形成维护风险。

### 14.4 不扩大无关格式化范围

新增或修改代码本身应符合当前文件格式，但不要因为当前任务顺手格式化整个文件或模块，避免制造无关 diff。

---

## 15. Codex Java 修改检查

修改 Java 代码时检查：

1. 名称是否表达真实英文业务语义，JavaBean 是否使用职责明确的 Request / Query / DTO / BO / DO / VO 等命名，而不是泛化 `Bean / Info / Data / Model`；Query 是否因为类名被无依据拆到独立 `query` Package。
2. 真实使用 Strategy / Factory / Adapter / Builder / Handler / Command / Visitor 等设计模式时，类型、技术子模块和方法是否体现其模式角色；是否反过来为了名称伪造不需要的设计模式。
3. 是否存在带业务 / 技术语义的魔法值；需要说明含义的固定值是否已提取为职责明确的常量或 Enum。
4. 常量是否按功能和语义归类；是否把所有常量塞入 `Constants / CommonConstants / GlobalConstants` 等大而全常量类。
5. 固定有限值域是否适合 Enum；是否错误把动态配置或外部开放值域强行枚举化。
6. Request / Query / DTO / BO / DO / VO 等 POJO 是否设置了字段初始化值或 `@Builder.Default`，从而隐藏默认业务语义。
7. 新类是否有真实独立职责，是否已搜索现有实现。
8. 模型是否沿用项目 `class` / Lombok 风格，是否机械使用 `@Data` / `record`。
9. 使用 `@Builder` / `@NoArgsConstructor` 是否来自真实对象构造和框架实例化需求；类级 Builder 是否具有可用构造路径，是否破坏对象不变式或业务规则。
10. 方法参数是否过多；封装是否基于完整语义、来源、生命周期和信任边界，而不是凑参数数量。
11. 普通查询条件是否已经适合 Query。
12. 方法是否因职责混杂而过长，而不是仅根据行数机械拆分。
13. 已有非 Null 集合契约时是否仍存在重复 Null 防御。
14. 是否通过默认值或 fallback 掩盖错误。
15. Optional、泛型、BigDecimal、时间语义是否正确。
16. catch / throw 是否保留失败语义和 cause，是否重复记录异常。
17. 日志是否泄漏敏感数据。
18. 相邻方法之间是否保留清晰空行；方法签名是否能清晰单行时保持单行，只在真正过长或复杂时合理换行。
19. 格式和注释是否遵循项目已有机制且没有扩大无关 diff。

最终原则：

> Java 规范负责把已经确定的职责和契约实现得清晰可靠；架构、API、事务和业务语义由各自专项规范决定。
