# Java 编码规范

本文档定义项目 Java 编码规范。

参考《阿里巴巴 Java 开发手册》的编程规约，并结合现代 Java、Spring Boot 项目实践和本 Skill 已有架构约定进行筛选与调整。

模型的职责分类与 Package 归属属于架构边界，由应用分层规范统一维护：

- [应用分层与模型边界](../architecture/layering.md)

本文主要规定 Java 层面的实现方式，例如命名、类设计、Lombok、`class` / `record`、常量、方法、集合、泛型、异常、日志和代码格式等。

原则：

> 优先保证语义清晰、行为正确和项目一致性；规范用于降低理解成本和常见错误，不用于机械改造已有合理代码。

---

## 1. 命名

命名应优先表达完整业务语义，做到见名知意。

### 1.1 类

使用 `UpperCamelCase`：

```java
PlaceService
PlaceQuery
PlaceDetailVO
```

抽象基类如果需要通过名称表达抽象角色，可以使用：

```text
AbstractXxx
BaseXxx
```

异常类使用能够表达异常语义的名称，并以 `Exception` 结尾：

```java
PlaceNotFoundException
PlaceStatusException
```

测试类优先使用被测试类型或行为名称，并遵循目标项目已有测试命名约定。

不要为了套用命名模板机械创建抽象类、接口或实现类。

---

### 1.2 方法、字段、参数和局部变量

统一使用 `lowerCamelCase`：

```java
placeName
identityNumber
queryPlaceDetail()
```

名称应尽量使用完整且明确的英文单词组合。

推荐：

```java
condition
identityNumber
expirationTime
```

避免无行业共识的随意缩写：

```java
condi
idNumStr
expTm
```

类型或集合语义需要体现在名称中时，可以放在词尾：

```java
startTime
workQueue
nameList
placeMap
```

但不要机械添加无意义类型后缀，例如：

```java
nameString
countInteger
```

---

### 1.3 常量

使用 `UPPER_SNAKE_CASE`：

```java
private static final int MAX_RETRY_COUNT = 3;
private static final Duration DEFAULT_TIMEOUT = Duration.ofSeconds(5);
```

常量名称应表达真实业务或技术语义，不为了缩短名称牺牲可读性。

---

### 1.4 Package

Package 统一使用小写英文，并使用明确、稳定的职责词汇。

例如：

```text
controller
service
manager
mapper
request
query
vo
common.mybatis.handler
```

Package 的职责归属读取：

- [layering.md](../architecture/layering.md)

不得仅为了满足“单数/复数”形式机械改动项目已有 Package。

---

### 1.5 命名禁止项

禁止：

* Java 标识符使用汉语拼音；
* 中文标识符；
* 拼音和英文混合；
* 含义不明确的缩写；
* 以下划线或 `$` 作为普通业务标识符的开始或结尾；
* 子类字段与父类字段使用容易混淆的同名定义；
* 不同作用域中大量复用相同名称导致语义不清。

数据库可以使用拼音，但 Java 代码必须使用英文业务语义。

---

### 1.6 数组、接口和枚举命名

数组类型使用：

```java
int[] values;
String[] names;
```

而不是：

```java
int values[];
```

接口方法无需重复声明隐含的：

```java
public abstract
```

例如：

```java
public interface PlaceProvider {
    PlaceVO getById(String id);
}
```

枚举成员使用常量风格：

```java
PENDING
APPROVED
REJECTED
```

枚举类型本身是否使用 `Enum` 后缀，以目标项目已有约定为准，不在本 Skill 中机械强制。

布尔属性优先使用能够直接表达状态的名称：

```java
enabled
deleted
editable
```

避免在可能受 JavaBean / Jackson / RPC 属性解析影响的模型中随意使用：

```java
isDeleted
isEnabled
```

但已发布 API 的字段名和序列化契约优先保持兼容。

---

## 2. 类设计与 OOP

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

### 2.1 覆写必须使用 `@Override`

覆写父类或实现接口方法时必须显式使用：

```java
@Override
```

这样可以让编译器帮助发现签名错误，也能清晰表达方法来源。

---

### 2.2 静态成员通过类名访问

静态字段和静态方法应通过类型名调用：

```java
PlaceConstants.MAX_NAME_LENGTH
Objects.equals(a, b)
```

避免：

```java
placeConstants.MAX_NAME_LENGTH
objects.equals(a, b)
```

---

### 2.3 访问范围从严

类、字段和方法使用满足实际调用需求的最小可见范围。

优先：

```text
private
→ 默认不对外暴露

protected / package-private
→ 只有真实继承或包内协作需求时使用

public
→ 明确属于外部调用边界时使用
```

不要为了调用方便把内部实现全部声明为 `public`。

工具类如果不应实例化，应禁止外部构造，例如使用私有构造器。

---

### 2.4 不机械创建 `Interface + Impl`

旧式规范中常见：

```text
XxxService
XxxServiceImpl
```

本项目不把它作为默认要求。

只有存在真实的：

* 多实现；
* SPI；
* 第三方隔离；
* 可替换能力；
* 稳定模块边界；

时才根据职责创建接口。

不得仅因为“Service 应该有接口”机械增加：

```text
PlaceService
PlaceServiceImpl
```

详细判断读取：

- [layering.md](../architecture/layering.md)

---

### 2.5 兼容与废弃

已经被其他模块或外部调用的公共方法、接口和类型，不得无明确需求随意修改签名或语义。

需要废弃公共能力时，优先使用：

```java
@Deprecated
```

并在适当位置说明替代方式。

新代码避免继续依赖已经明确废弃的 API；如果因兼容性必须保留，应说明原因。

---

### 2.6 构造器和访问器保持简单

构造器不应承载：

* 数据库访问；
* HTTP / RPC；
* 文件 IO；
* 长耗时业务流程；
* 隐式状态流转。

构造器可以完成建立对象不变式所必需的轻量校验和赋值。

普通 Getter / Setter 不应隐藏业务逻辑、远程调用或不可预期副作用。

如果修改属性需要业务规则，应使用明确业务方法，例如：

```java
changeStatus(...)
approve(...)
activate(...)
```

---

### 2.7 同类方法组织

重载方法和承担同一能力的方法应尽量相邻放置，减少阅读时的上下跳转。

类内方法可以优先按以下可读性顺序组织：

```text
对外主要方法
→ 内部辅助方法
→ 访问器或简单基础方法
```

但目标项目已有 formatter、IDE code arrangement 或明确风格时，以项目为准。

---

### 2.8 慎用 `clone`

`Object#clone()` 默认是浅拷贝，容易对可变引用字段产生误解。

需要复制对象时，优先使用职责清晰的：

* 构造器；
* 静态工厂；
* 显式复制方法；
* 项目已有映射能力。

不要为了少写字段复制机械使用 `clone()`。

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

---

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

---

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

---

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

---

### 3.4 基本类型与包装类型

模型字段需要表达“未提供”“未知”“数据库为 NULL”等额外语义时，优先使用包装类型：

```java
Integer
Long
Boolean
```

局部计算变量在不存在 Null 语义时，可以优先使用基本类型：

```java
int
long
boolean
```

不要因为统一风格批量把已有字段从基本类型改成包装类型，或反向修改；API、数据库 Null 语义和兼容性优先。

DO 字段的 Java 类型应能够正确表达数据库字段类型和 Null 语义。

---

### 3.5 不自行给模型字段增加默认业务值

Request、Query、DTO、BO、DO、VO 不应在没有明确契约时自行初始化业务默认值，例如：

```java
private LocalDateTime createTime = LocalDateTime.now();
private Boolean enabled = true;
private String status = "PENDING";
```

默认值必须来自明确的业务契约、数据库默认值或已有项目约定。

这与“不得自行创造业务规则”的整体原则一致。

---

### 3.6 `toString` 与敏感字段

不要为了调试方便机械为所有模型生成完整 `toString()`。

使用 Lombok `@Data`、`@ToString` 或手写 `toString()` 时，应检查是否可能输出：

* 密码；
* Token；
* 身份证件；
* 手机号；
* 密钥；
* 生物特征；
* 其他敏感字段。

日志可观测性不能以泄漏敏感数据为代价。

---

### 3.7 模型对象检查

Codex 创建或修改模型对象时必须：

1. 先按分层规范确定模型职责和 Package。
2. 普通 Getter / Setter 优先使用 Lombok。
3. 不新增无意义的手写 Getter / Setter。
4. 已使用 Lombok 的类，不重复手写普通访问器。
5. 不机械使用 `@Data`。
6. 不删除具有校验、转换或业务语义的方法。
7. 不为了生成 Setter 破坏对象原有的不变性。
8. 不主动使用 `record` 替代项目现有模型风格。
9. 不自行增加无依据的字段默认值。
10. 检查包装类型与数据库/API Null 语义是否一致。

---

## 4. 常量与枚举

### 4.1 禁止散落魔法值

具有业务或技术语义并会重复使用的固定值，不应散落在代码中。

避免：

```java
if (retryCount > 3) {
}

String cacheKey = "place:" + placeId;
```

如果 `3` 和 `"place:"` 具有稳定且明确的约束语义，应使用职责清晰的常量或配置。

但不要把：

```java
0
1
-1
```

等所有普通计算字面量机械提取成常量；只有存在独立语义或复用价值时才提取。

---

### 4.2 常量按职责归类

避免建立无限增长的：

```text
Constants
CommonConstants
GlobalConstants
```

常量应跟随实际职责归类，例如：

```text
CacheConstants
ValidationConstants
```

或者放在真正拥有该约束的类中。

---

### 4.3 `long` 字面量使用大写 `L`

推荐：

```java
long timeout = 1000L;
```

避免：

```java
long timeout = 1000l;
```

小写 `l` 容易与数字 `1` 混淆。

---

### 4.4 有限稳定取值可以使用枚举

当一个值存在明确、有限且稳定的业务取值集合时，可以使用枚举表达。

但不得为了“消除魔法字符串”自行创造业务状态、枚举值、兼容别名或状态流转。

已有 API / 数据库值和项目枚举体系优先。

---

## 5. 方法设计与可读性

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

---

### 5.1 控制参数数量和参数语义

普通业务方法的新建或显著修改，默认不超过 **5 个参数**。超过 5 个时，应先检查方法职责是否过多，并优先把天然属于同一语义的一组参数封装为职责明确的对象。

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

应优先评估：

```java
public String signEnvelope(
        RequestPayload payload,
        SignContext context) {
    ...
}
```

参数对象必须表达真实语义，可以根据职责使用：

```text
Request
Query
DTO
BO
Context
Command
```

等项目已有模型或明确对象；不得仅为了满足参数数量创建：

```text
XxxParam
CommonParam
Object[]
Map<String, Object>
```

这种无边界的参数容器。

参数数量未超过 5 个也不代表设计一定合理。尤其多个参数类型相同、只能依靠顺序区分时，例如：

```java
move(String sourceCode, String targetCode, String operatorCode, String tenantCode)
```

应检查是否存在天然的上下文或业务对象，降低调用时参数顺序错误的风险。

对于普通业务查询，查询条件超过 **3 个** 时仍优先评估封装为 Query 对象；这是查询场景更具体的默认规则，不因为“总参数未超过 5 个”就继续扩展长查询签名。

框架回调、覆写方法、第三方接口或已有稳定公共 API 的签名受外部契约约束时，以真实契约为准，不为了满足数量规则破坏兼容性。

原则：

> 参数数量用于提示职责和可调用性问题；封装必须形成真实语义边界，而不是把多个参数机械塞进另一个无意义对象。

Query 的职责与 Package 归属读取：

- [layering.md](../architecture/layering.md#92-query)

---

### 5.2 方法长度控制

新建或显著修改的方法，默认应控制在 80 行以内，优先通过清晰职责、Guard Clause 和有业务语义的方法抽取保持可读性。

方法超过 80 行时，应检查是否存在：

* 职责过多；
* 分支复杂；
* 抽象层次混杂；
* 重复逻辑；
* 可以独立命名的业务步骤。

但 80 行不是脱离上下文的机械拆分命令。若为了满足行数限制只能产生大量无语义的方法：

```java
step1();
step2();
step3();
```

或者拆分反而破坏局部逻辑完整性，不应仅为了凑行数进行形式化拆分。

原则：

> 默认控制在 80 行以内；超过时必须重新检查职责和可读性，拆分必须产生真实语义价值。

---

### 5.3 复杂条件使用有意义的中间变量

当条件包含大量：

```text
&&
||
!
多层方法调用
```

时，优先把关键语义提取成命名清晰的布尔变量或方法。

例如：

```java
boolean canAudit = place.isPending() && operator.hasAuditPermission();
if (!canAudit) {
    throw new PlaceAuditException();
}
```

不要为了减少一行代码把复杂逻辑全部塞进 `if`。

---

### 5.4 不在条件表达式中隐藏赋值

避免：

```java
if ((result = load()) != null) {
}
```

优先：

```java
Result result = load();
if (result != null) {
}
```

赋值应尽量成为可见、独立的代码动作。

---

### 5.5 循环体保持轻量

如果不依赖循环变量，应尽量把以下工作移出循环：

* 重复创建可复用对象；
* 重复解析固定配置；
* 重复获取相同资源；
* 不必要的异常包装；
* 可提前完成的固定计算。

但不要为微小、无证据的性能收益牺牲清晰度。

数据库循环访问引起的 N+1 问题读取 MyBatis / SQL 规范。

---

### 5.6 循环内字符串拼接

大量循环拼接字符串时优先使用：

```java
StringBuilder
```

例如：

```java
StringBuilder builder = new StringBuilder();
for (String name : names) {
    builder.append(name).append(',');
}
```

少量、非循环的普通字符串拼接可以继续使用 `+`，不需要机械改写。

---

## 6. equals、比较与 Null

### 6.1 Null 安全比较

不要在可能为空的对象上调用：

```java
value.equals(...)
```

推荐：

```java
Objects.equals(a, b);
```

或者常量调用：

```java
"ACTIVE".equals(status);
```

布尔包装类型需要明确判断时，可以使用：

```java
Boolean.TRUE.equals(enabled)
```

---

### 6.2 包装数值不要使用 `==` 比较值

例如：

```java
Integer a = 1000;
Integer b = 1000;
```

不要使用：

```java
if (a == b) {
}
```

判断包装对象的数值相等。

应根据 Null 语义使用：

```java
Objects.equals(a, b)
```

基本类型值比较不受此限制。

---

### 6.3 空集合优先于 Null，并依赖明确契约

当空集合能够准确表达“没有结果”时，集合返回值优先使用空集合，而不是 `null`。

例如：

```java
return Collections.emptyList();
```

或者：

```java
return List.of();
```

但更重要的是：**一旦下层方法已经具有明确的非 Null 集合契约，上层应直接依赖该契约，不再机械增加 Null 防御。**

例如下层明确保证：

```java
List<AssetDO> listAssets(...);
```

无结果返回空集合，则调用方可以直接：

```java
List<AssetDO> assets = assetRepository.listAssets(...);
return assets.stream()
        .map(...)
        .toList();
```

不要无依据增加：

```java
List<AssetDO> safeAssets = assets == null
        ? new ArrayList<>()
        : assets;
```

也不要为了“更安全”机械包装：

```java
Optional.ofNullable(assets)
        .orElseGet(Collections::emptyList);
```

或者：

```java
if (assets != null) {
    ...
}
```

这些写法会让调用方错误地认为正常契约可能返回 Null，并把防御代码扩散到每一层。

如果数据来源确实允许 Null，例如：

* 第三方 SDK；
* 外部 HTTP / RPC 响应；
* 遗留接口；
* 明确声明 nullable 的内部契约；

应在最靠近来源、最了解该技术差异的边界归一化一次：

```java
List<VendorItem> items = response.getItems();
return items == null ? List.of() : items;
```

然后由上层依赖稳定的非 Null 集合契约，不在 Manager / Service / Controller 继续重复：

```text
items == null ? emptyList : items
```

已有 API 或调用契约对 Null 具有明确独立语义时，不得擅自改变；单对象查询的 Null / Optional / 异常语义也应按其自身契约判断，不能套用集合规则。

原则：

> 先确认来源契约；集合无结果优先使用空集合；确实存在 Null 差异时在边界归一化一次，上层不层层猜测和兜底。

MyBatis 集合查询的具体规则读取：

- [mybatis.md](mybatis.md#61-集合查询的-null-契约)

---

### 6.4 `String#split` 后访问索引要确认长度

如果输入格式可能不完整，不要直接假设：

```java
String[] parts = value.split(":");
String second = parts[1];
```

应先确认输入契约和数组长度，避免 `ArrayIndexOutOfBoundsException`。

---

## 7. Optional

`Optional` 主要用于表达“返回值可能不存在”。

避免用于：

* DO / DTO / BO / VO / Request / Query 字段；
* 普通方法参数；
* 仅为了避免一次 Null 判断而机械包装。

不要直接：

```java
optional.get();
```

优先根据语义使用：

```java
optional.orElse(...)
optional.orElseGet(...)
optional.orElseThrow(...)
```

---

## 8. 集合与泛型

### 8.1 声明优先使用接口

推荐：

```java
List<String> names = new ArrayList<>();
Map<String, PlaceDO> placeMap = new HashMap<>();
```

除非确实需要具体实现特有能力，否则不要让业务接口暴露具体集合实现类型。

---

### 8.2 禁止 Raw Type

新代码不要使用没有泛型参数的集合：

```java
List list = new ArrayList();
Map map = new HashMap();
```

应使用明确泛型：

```java
List<String> names = new ArrayList<>();
Map<String, PlaceDO> places = new HashMap<>();
```

不要通过未受检查的 Raw Type 赋值绕过编译期类型安全。

---

### 8.3 使用 Diamond 语法

能够由编译器推断时优先使用：

```java
Map<String, PlaceDO> placeMap = new HashMap<>();
```

而不是重复：

```java
Map<String, PlaceDO> placeMap = new HashMap<String, PlaceDO>();
```

---

### 8.4 泛型边界遵循 PECS

设计泛型 API 时可以使用：

```text
Producer Extends
Consumer Super
```

例如：

```java
void copy(
        List<? super PlaceVO> target,
        List<? extends PlaceVO> source) {
}
```

只有当泛型边界确实提升类型安全和复用能力时才使用，不为了展示泛型技巧增加不必要复杂度。

---

### 8.5 不在增强 `for` 中直接修改集合结构

避免：

```java
for (String item : items) {
    if (shouldRemove(item)) {
        items.remove(item);
    }
}
```

需要删除时使用：

* `Iterator#remove()`；
* `removeIf()`；
* 合适的集合 API。

并发集合的修改还必须读取并发规范。

---

### 8.6 `equals` 与 `hashCode` 必须保持一致

覆写 `equals()` 时必须同时检查 `hashCode()`。

自定义对象作为：

* `HashMap` Key；
* `HashSet` 元素；

时，必须确保对象的相等性语义稳定且正确。

不要仅为了方便在所有模型上机械生成 `equals/hashCode`；应先确认对象语义。

---

### 8.7 注意集合视图和不可变集合

`subList()` 返回的是原集合的视图，修改原集合结构可能导致子列表操作出现异常或非预期行为。

不要把：

```java
list.subList(...)
```

强制转换为：

```java
ArrayList
```

`Collections.emptyList()`、`List.of()` 等不可变集合不能执行结构修改。

如果调用方需要可变集合，应明确创建副本：

```java
new ArrayList<>(source)
```

---

### 8.8 `Arrays.asList()` 不是普通可变 `ArrayList`

`Arrays.asList(array)` 返回固定大小的列表视图，不支持：

```java
add
remove
clear
```

需要独立可变集合时使用：

```java
new ArrayList<>(Arrays.asList(array))
```

现代 Java 项目也可以根据版本和语义使用合适的集合工厂方法。

---

### 8.9 集合转数组保持类型安全

优先使用类型安全的 `toArray`：

```java
String[] array = names.toArray(new String[0]);
```

或者在目标 Java 版本和项目风格允许时使用：

```java
String[] array = names.toArray(String[]::new);
```

不要先获取 `Object[]` 再强制转换为具体数组类型。

---

### 8.10 集合参数先确认 Null 契约

调用：

```java
addAll(...)
putAll(...)
```

等批量 API 前，应确认参数是否允许为 Null。

不要依赖集合实现抛出 NPE 来表达业务校验。

如果参数来源已经具有明确非 Null 集合契约，也不要为了“防御”再次机械包装为空集合；应直接依赖契约。

---

### 8.11 Map 遍历

同时需要 Key 和 Value 时，优先使用：

```java
for (Map.Entry<String, PlaceDO> entry : placeMap.entrySet()) {
}
```

或者：

```java
placeMap.forEach((key, value) -> {
});
```

不要为了获取 Value 遍历 Key 后再重复 `get()`，除非存在明确原因。

---

### 8.12 明确集合 Null 和顺序语义

不要假设所有 Map 都允许 Null Key / Value，也不要假设普通 `HashMap` / `HashSet` 存在稳定业务顺序。

如果业务依赖：

* 插入顺序；
* 排序顺序；
* Null 支持；
* 并发安全；

应显式选择能够保证该语义的数据结构。

集合本身是否允许为 Null，应优先由方法契约或数据来源边界明确，不在每个调用点各自猜测。

---

### 8.13 Comparator 必须满足比较契约

自定义 Comparator 必须正确处理：

* 小于；
* 等于；
* 大于；
* 传递性。

避免：

```java
return a.getId() > b.getId() ? 1 : -1;
```

这种忽略相等情况的实现。

优先使用：

```java
Comparator.comparing(...)
Comparator.comparingInt(...)
Comparator.comparingLong(...)
```

等标准 API 表达明确比较语义。

---

### 8.14 容量和去重按实际规模选择

能够合理估计大集合规模时，可以设置初始容量，减少不必要扩容。

但不要为了无法验证的微小性能收益到处手工计算容量。

有明确唯一性需求时优先考虑 `Set` 或键控结构，不要在大 List 中反复 `contains()` 实现低效去重。

---

## 9. 数值、浮点数与 BigDecimal

### 9.1 禁止从二进制浮点字面量直接构造 BigDecimal

禁止：

```java
new BigDecimal(0.1)
```

推荐：

```java
new BigDecimal("0.1");
BigDecimal.valueOf(0.1);
```

---

### 9.2 BigDecimal 比较使用 `compareTo`

如果业务只关心数值大小而不关心 scale，优先：

```java
amount.compareTo(BigDecimal.ZERO) > 0
```

不要误把：

```java
new BigDecimal("1.0")
new BigDecimal("1.00")
```

的 `equals()` 结果当成纯数值相等判断。

---

### 9.3 BigDecimal 除法明确精度规则

除法必须明确考虑：

* scale；
* roundingMode；
* 业务精度要求。

不得为了避免异常随意选择舍入方式。

---

### 9.4 浮点数不要直接用 `==` 判断业务等值

对于：

```java
float
double
Float
Double
```

涉及计算结果的业务等值判断时，不要依赖直接 `==` 或包装类 `equals()` 消除浮点误差。

根据业务选择：

* 允许误差范围；
* `BigDecimal`；
* 领域明确的舍入规则。

例如：

```java
double diff = 1e-9;
if (Math.abs(a - b) < diff) {
}
```

误差范围必须来自业务或计算精度要求，不应随意编造。

---

## 10. 日期时间

新代码优先使用：

```text
LocalDate
LocalDateTime
Instant
Duration
ZonedDateTime
DateTimeFormatter
```

避免新增：

```text
Date
Calendar
SimpleDateFormat
```

尤其不要把可变且线程不安全的 `SimpleDateFormat` 作为共享静态实例。

不要通过格式化后的字符串比较时间。

业务涉及时区时必须明确：

* 时间代表哪个时区；
* 是否使用 UTC；
* 是否需要 Offset；
* 数据库存储和 API 序列化如何约定。

---

## 11. 控制语句

### 11.1 必须使用大括号

`if`、`else`、`for`、`while`、`do` 等必须使用大括号。

推荐：

```java
if (place == null) {
    throw new PlaceNotFoundException(id);
}
```

避免：

```java
if (place == null)
    throw new PlaceNotFoundException(id);
```

---

### 11.2 优先 Guard Clause

异常、失败和提前退出条件优先放在前面处理，减少嵌套。

推荐：

```java
if (place == null) {
    throw new PlaceNotFoundException(id);
}

if (!place.canAudit()) {
    throw new PlaceStatusException(id);
}

performAudit(place);
```

不要为了消除 `if` 就机械引入 Strategy / State 等设计模式。

---

### 11.3 控制嵌套深度

连续多层 `if / else if / else` 会显著增加理解成本。

出现复杂嵌套时优先评估：

* Guard Clause；
* 提取具有明确语义的方法；
* 简化条件；
* 在存在真实稳定变化点时使用合适的设计模式。

不要仅根据“超过 3 层”机械重构。

---

### 11.4 `switch` 输入先确认 Null 语义

对可能来自外部输入的：

```java
String
Enum
```

进行 `switch` 前应确认 Null 是否可能出现，并根据契约处理。

不要假设：

```java
case "null"
```

能够处理真正的 Null。

对于现代 Java 的 switch expression，是否需要 `default` 应根据编译器穷尽性、枚举演进和业务兼容要求判断，不机械添加无意义分支。

---

### 11.5 优先正向、易读条件

避免多重取反：

```java
if (!(count >= limit)) {
}
```

优先：

```java
if (count < limit) {
}
```

但如果领域中否定名称本身是稳定语义，不为了形式进行难懂改写。

---

## 12. 代码格式

### 12.1 项目 formatter 优先

如果目标项目已经配置：

* Spotless；
* Checkstyle；
* IDE Code Style；
* EditorConfig；
* 其他格式化工具；

以项目已有自动化规则为准。

不得为了本文风格要求格式化无关文件或产生大面积无业务差异的变更。

---

### 12.2 缩进

没有项目特殊约定时，Java 代码统一使用 4 个空格缩进，禁止使用 Tab 字符作为代码缩进。

推荐：

```java
if (place != null) {
    audit(place);
}
```

项目存在 `.editorconfig`、formatter 或 IDE Code Style 时，以其配置为准。

---

### 12.3 单行注释格式

`//` 与注释正文之间有且仅有一个空格。

推荐：

```java
// 检查当前状态是否允许审核
if (!place.canAudit()) {
    throw new PlaceStatusException(place.getId());
}
```

避免：

```java
//检查当前状态
//  检查当前状态
```

---

### 12.4 类型强制转换格式

类型强制转换时，右括号与被转换的表达式之间不增加空格。

推荐：

```java
int second = (int)first + 2;
```

避免：

```java
int second = (int) first + 2;
```

如果目标项目 formatter 对该格式有明确不同约定，以 formatter 为准。

---

### 12.5 文件编码与换行

默认要求：

```text
UTF-8
LF（Unix 换行）
```

避免新文件使用平台相关编码或 CRLF，除非目标项目已有明确不同约定。

编码和换行应优先由：

```text
.editorconfig
Git attributes
IDE Code Style
formatter
```

统一约束，而不是依赖开发者手工保持。

---

### 12.6 方法行数

新建或显著修改的方法默认不超过 80 行。

超过 80 行时，应优先检查职责是否过多、分支是否复杂、是否存在可独立命名的业务步骤，并在有真实语义价值时进行拆分。

不得通过删除必要空行、压缩表达式或创建无语义方法来“满足 80 行”。

本规则与 [5.2 方法长度控制](#52-方法长度控制) 一致。

---

### 12.7 空行

不同逻辑、不同语义或不同业务步骤之间使用一个空行分隔，提高阅读性。

例如：

```java
PlaceDO place = placeMapper.getById(id);
if (place == null) {
    throw new PlaceNotFoundException(id);
}

validateAudit(place, request);

placeMapper.updateStatus(id, request.getStatus());
```

不要在同一逻辑块内部随意插入空行，也不要连续使用多个空行进行视觉分隔。

原则：

> 一个空行表达一次逻辑分组；不靠大量空行制造“层次感”。

---

### 12.8 不为了格式改业务

格式问题应由 formatter / IDE 自动化处理的，优先交给已有工具。

不要在一个业务修复中顺带：

* 重排整个类；
* 批量调整无关空行；
* 全文件 import 重排；
* 全项目格式化。

除非这些变化是当前任务或项目检查明确要求的一部分。

---

## 13. 异常

禁止：

```java
catch (Exception e) {
}
```

禁止：

* 吞异常；
* 使用异常完成普通流程控制；
* 无意义重复包装异常；
* 随意捕获 `Throwable`；
* 在 `finally` 中 `return` 覆盖正常返回或异常；
* 捕获异常后只打印日志却让调用方误认为操作成功。

异常转换应发生在有明确抽象边界的位置，并尽量保留根因：

```java
throw new BusinessException("Failed to load place", ex);
```

具体异常类型必须优先复用目标项目已有异常体系，不得为了示例创建平行异常层级。

异常跨层转换、记录和 Web/API 收口读取：

- [error-handling.md](../architecture/error-handling.md)

Spring 框架中的 Advice / Handler 实现读取：

- [spring.md](spring.md)

---

## 14. 日志

日志的目标是帮助定位问题、理解关键业务行为和支撑可观测性，不是记录代码执行的每一步。

原则：

> 每条日志都应回答“为什么需要记录、谁会看、看到后能做什么”；没有排查、审计、监控或业务价值的日志不要机械输出。

### 14.1 使用项目统一日志门面

业务代码不应直接依赖具体日志实现的 API，例如：

```text
Log4j
Logback
```

Spring Boot 项目默认优先通过 SLF4J 等项目统一日志门面记录日志，例如：

```java
private static final Logger log = LoggerFactory.getLogger(PlaceService.class);
```

如果项目统一使用 Lombok：

```java
@Slf4j
```

也可以沿用现有风格。

不要为了统一本文示例，在已有项目中机械替换日志字段名、Lombok 注解或日志门面实现。

禁止：

```java
System.out.println(...)
System.err.println(...)
exception.printStackTrace()
```

---

### 14.2 使用参数化日志

日志中的动态变量优先使用占位符：

```java
log.info("Audit place successfully, placeId={}, status={}", placeId, status);
```

避免：

```java
log.info("Audit place successfully, placeId=" + placeId + ", status=" + status);
```

也不要为了日志提前进行无必要的字符串拼接或复杂 `toString()`。

参数化日志可以减少在日志级别未开启时仍然进行字符串拼接的无意义开销，并使日志结构更统一。

---

### 14.3 昂贵日志参数先判断级别

对于简单变量的参数化日志，通常不需要机械包一层：

```java
if (log.isDebugEnabled()) {
}
```

但如果日志参数本身需要执行：

* 复杂对象构造；
* 大对象序列化；
* JSON 转换；
* 昂贵方法调用；
* 大集合格式化；

则在 `debug` / `trace` 日志前应先判断日志级别，避免日志未开启时仍执行昂贵计算。

例如：

```java
if (log.isDebugEnabled()) {
    log.debug("Place audit context={}", buildAuditDebugContext());
}
```

不要为了日志调用具有业务副作用的方法。

---

### 14.4 日志级别必须与影响匹配

默认语义可以按以下方式理解：

```text
ERROR
→ 系统逻辑错误、非预期异常、关键操作失败，需要关注或处理

WARN
→ 异常但可恢复、降级、重试、输入异常或值得关注的非致命问题

INFO
→ 关键业务节点、重要状态变化、启动/停止等有长期观察价值的信息

DEBUG / TRACE
→ 开发和故障诊断细节，不作为正常生产业务日志的主要载体
```

不要把正常业务分支、用户可预期操作结果或普通校验失败机械记录为 `error`，否则会造成无效告警。

同样不要把每个方法的进入、退出、每条查询结果都机械记录为 `info`。

生产环境的具体日志级别由项目配置决定；本 Skill 不自行修改生产日志级别。临时开启 `debug` / `trace` 应遵循项目运维流程，并注意日志量和敏感信息。

---

### 14.5 异常日志必须同时包含现场信息和堆栈

需要记录非预期异常时，日志应尽量包含两类信息：

```text
案发现场
+
异常堆栈
```

推荐：

```java
log.error(
        "Audit place failed, placeId={}, targetStatus={}",
        placeId,
        targetStatus,
        ex);
```

不要只记录：

```java
log.error(ex.getMessage());
```

这会丢失堆栈，也缺少业务上下文。

也不要直接把整个 Request、DO、用户对象或第三方响应 `toString()` 打进错误日志，应只选择排查所需且安全的关键字段。

异常在哪一层转换、在哪一层记录完整现场，读取：

- [异常处理与错误边界](../architecture/error-handling.md)

---

### 14.6 同一异常链避免重复打印

同一个失败链路通常只应在拥有足够上下文的位置记录一次完整异常堆栈。

禁止形成：

```text
Mapper log.error
    ↓
Manager log.error
    ↓
Service log.error
    ↓
ControllerAdvice log.error
```

导致同一异常出现多份几乎相同的堆栈。

如果当前层只是：

```text
不能恢复
不能增加新的诊断上下文
也不负责最终记录
```

则直接传播通常比 `catch + log + throw` 更合理。

---

### 14.7 谨慎控制日志量

高频路径中的日志必须特别谨慎，例如：

* 循环内逐条日志；
* 批量任务逐条 `info`；
* 高频接口每次输出大对象；
* 定时轮询每次都记录正常结果；
* 重试过程中重复输出完整堆栈。

大量无效日志会增加：

* 磁盘和日志平台成本；
* IO 压力；
* 检索噪声；
* 告警噪声；
* 真正错误的定位成本。

新增日志前应至少判断：

```text
这条日志是否有人消费？
是否能够帮助定位问题或审计行为？
是否可能高频触发？
是否需要每次都记录？
```

临时观察日志、灰度日志或问题排查日志完成使命后应及时清理或降级，不要永久遗留大量临时 `info` / `warn`。

如果项目已有采样、限流、聚合或指标系统，应优先使用已有机制；不要为了控制日志量自行引入新的日志基础设施。

---

### 14.8 扩展日志按用途分类

访问日志、审计日志、监控日志、统计日志和普通应用日志具有不同用途。

如果项目已经存在：

```text
access
audit
monitor
stats
security
```

等日志分类或独立 Appender，应沿用项目约定，不要全部混入普通业务日志，也不要自行创造第二套命名体系。

日志文件名称、Appender、滚动策略、采集路径等属于项目日志配置和运维边界，不应在业务代码中硬编码。

---

### 14.9 日志保留周期以项目合规与运维要求为准

日志保留周期不是 Java 业务代码自行决定的常量。

具体保留时间应由：

* 组织合规要求；
* 安全审计要求；
* 业务追溯周期；
* 存储成本；
* 日志平台策略；
* 运维配置；

共同确定。

不得仅因为通用编码手册给出了某个天数，就自行修改目标项目的日志保留策略。

涉及安全、审计、管理操作或敏感业务行为的日志，必须优先遵循目标项目及组织的合规要求。

---

### 14.10 日志语言以准确和一致为先

日志内容应优先满足：

* 团队能够快速理解；
* 关键术语一致；
* 便于搜索；
* 不产生歧义。

如果目标项目统一使用英文日志，应延续英文；如果复杂业务信息用中文表达更准确，也可以使用中文。

TCP、HTTP、RPC、SQL、MyBatis、TraceId 等专业术语保持原有英文表达，不做生硬翻译。

不要在同一模块中随意混用多套完全不同的日志措辞风格。

---

### 14.11 记录必要的关联标识

项目已有 TraceId、RequestId、任务 ID、业务主键等关联能力时，异常和关键业务日志应优先包含足以串联调用链的安全标识。

例如：

```java
log.warn("Import task partially failed, taskId={}, failedCount={}", taskId, failedCount);
```

如果 TraceId 等信息已经由 MDC、日志框架或链路追踪组件自动输出，不要在每条日志中重复手工拼接。

不要自行创造不存在的 TraceId、租户号或用户字段。

---

### 14.12 禁止输出敏感信息

日志中禁止直接输出：

* 密码；
* Token；
* Secret；
* 私钥；
* Session / Cookie 凭证；
* 完整身份证件；
* 完整银行卡号；
* 生物特征；
* 未脱敏手机号等个人敏感信息；
* 第三方认证凭证；
* 其他目标项目明确禁止记录的数据。

需要记录业务对象时，应显式选择必要字段并按项目脱敏规则处理，不得依赖整个对象的 `toString()`。

日志排障价值不能高于安全和隐私边界。

---

### 14.13 Codex 日志检查

新增或修改日志时检查：

1. 是否使用项目统一日志门面，而不是直接依赖具体日志实现。
2. 是否使用参数化日志而不是字符串拼接。
3. `debug` / `trace` 参数是否存在昂贵计算，必要时是否先判断级别。
4. 日志级别是否与实际影响一致。
5. 异常日志是否同时保留必要业务现场和异常堆栈。
6. 同一异常链是否在多个层重复打印完整堆栈。
7. 是否在循环、高频接口、批量任务中产生过量日志。
8. 是否把临时观察日志永久留在正常业务路径。
9. 是否遵循项目已有 access / audit / monitor 等日志分类。
10. 是否包含必要且安全的业务标识或链路标识。
11. 是否输出密码、Token、密钥、完整证件等敏感数据。
12. 是否无需求修改日志文件名、Appender、保留周期或生产日志级别。

最终原则：

> 日志要足够定位问题，但不能重复、泛滥或泄漏敏感信息；日志框架、级别、分类、采集和保留策略始终以目标项目现有约定和运维/合规要求为准。

---

## 15. 注释与 Javadoc

### 15.1 注释解释“为什么”和契约

注释重点解释：

* 为什么这样实现；
* 特殊业务规则；
* 不明显的技术限制；
* 重要设计取舍；
* 公共 API 的输入、输出和异常契约；
* 容易被误解的兼容性要求。

避免重复代码本身已经清楚表达的内容：

```java
// 设置名称
user.setName(name);
```

推荐解释真正原因：

```java
// 数据库字段沿用历史拼音命名，Java 层在 ResultMap 中统一转换为英文语义。
```

---

### 15.2 公共接口和抽象能力使用 Javadoc

对外公开、跨模块使用或作为扩展契约的类、接口、抽象方法，如果仅靠签名不能完整表达契约，应使用 Javadoc 说明：

* 功能和业务含义；
* 参数语义；
* 返回值语义；
* 可能抛出的异常；
* Null 约定；
* 副作用；
* 实现方需要遵守的约束。

例如：

```java
/**
 * 获取指定场所的可见详情。
 *
 * @param id 场所标识
 * @return 当前调用方可见的场所详情
 * @throws PlaceNotFoundException 场所不存在时抛出
 */
PlaceVO getById(String id);
```

对于普通私有方法或语义已经非常清晰的简单访问器，不机械添加无信息量 Javadoc。

---

### 15.3 方法内部注释格式

方法内部的单行说明注释放在被说明代码的上方，并使用：

```java
// 注释内容
```

多行块注释使用：

```java
/*
 * 注释内容
 */
```

并与周围代码保持正确缩进。

不要在一行复杂业务代码末尾堆积过长行尾注释。

---

### 15.4 枚举成员说明业务语义

业务枚举成员如果名称不能完整表达编码、兼容或业务含义，应为每个成员提供清晰说明。

例如：

```java
public enum AuditStatus {

    /** 等待审核。 */
    PENDING,

    /** 审核通过。 */
    APPROVED,

    /** 审核拒绝。 */
    REJECTED
}
```

不得通过注释自行创造项目不存在的业务状态或含义。

---

### 15.5 注释语言以准确为先

注释应使用团队能够准确理解的语言。

当中文能够更清楚表达复杂业务原因时，可以直接使用中文；TCP、HTTP、JVM、MyBatis 等专有名词保持原有英文术语，不做生硬翻译。

原则：

> 注释的目标是准确传递设计和业务含义，不是展示语言形式。

---

### 15.6 代码与注释必须同步

修改以下内容时，必须同步检查相关注释和 Javadoc：

* 参数；
* 返回值；
* 异常；
* 状态语义；
* 核心算法；
* 兼容逻辑；
* 调用约束。

禁止代码已经改变而注释仍描述旧行为。

过期注释比没有注释更容易误导维护者。

---

### 15.7 谨慎保留注释掉的代码

永久不用的代码应直接删除，由版本控制保存历史，不应长期留成大段：

```java
// old logic...
// old logic...
```

如果确实需要短期保留注释代码，必须明确说明保留原因和恢复条件；否则应删除。

---

### 15.8 注释适量且有信息量

好的命名和结构本身应具有自解释能力。

不要为了“有注释”而给每一行代码添加解释，也不要使用大量注释掩盖职责混乱或命名不清。

注释应准确反映：

* 设计思想；
* 代码背后的业务含义；
* 非显而易见的约束。

不机械要求每个类都填写作者和创建日期。此类历史信息优先由 Git 记录维护。

---

## 16. 不可变性

没有必要修改的数据尽量保持不可变。

局部变量在有助于表达意图时可以使用 `final`，但不要求所有局部变量机械添加 `final`。

成员字段如果在构造后不应重新赋值，应优先考虑：

```java
private final ...
```

避免共享可变静态状态。

需要线程安全时读取：

- [concurrency.md](../architecture/concurrency.md)

不要把并发规则重新复制到本文。

---

## 17. Codex Java 修改原则

Codex 修改 Java 代码时必须：

1. 先查看类似实现和项目已有格式化约定。
2. 延续当前模块合理的命名和代码风格。
3. 优先复用已有工具和模型。
4. 不顺带重构、重排或格式化无关代码。
5. 不为了“更优雅”改变已有架构。
6. 行为发生变化时检查是否需要测试。
7. 新增模型或调整 Package 时，先读取 `layering.md` 确定职责与归属。
8. 本文只负责模型的 Java 实现方式，不在这里重新定义 Request / Query / DTO / BO / DO / VO 的架构职责。
9. 模型对象的普通 Getter / Setter 优先使用 Lombok。
10. 不为了 Lombok 删除具有业务语义的方法。
11. 不机械使用 `@Data`。
12. 不主动使用 `record` 替代普通模型 class。
13. 不为了形式机械增加 DTO / BO / Converter 等中间模型。
14. 不机械创建 `Service + ServiceImpl`。
15. 普通业务方法参数默认不超过 5 个；超过时先检查职责并封装真实语义对象，查询条件超过 3 个优先评估 Query，不使用 `Map<String, Object>` 机械兜底。
16. 不引入无依据的魔法状态、默认值或枚举值。
17. 使用包装类型时检查 Null 语义，避免自动拆箱 NPE。
18. 使用集合时检查可变性、视图、泛型、顺序和 Null 契约；已有明确非 Null 集合契约时，不机械增加 `list == null`、`Optional.ofNullable(list)` 或空集合兜底；来源确实允许 Null 时在边界归一化一次。
19. 数值计算时检查精度、比较和舍入规则。
20. 控制复杂条件和嵌套，优先提高可读性而不是追求代码行数最少。
21. 新建或显著修改的方法默认控制在 80 行以内；超过时必须检查职责和拆分价值。
22. 遵守 4 空格、UTF-8、LF、注释间距和逻辑空行等默认格式；项目 formatter 存在时以项目配置为准。
23. 修改代码行为时同步检查注释和 Javadoc，禁止保留与实现不一致的旧注释。
24. 不长期保留无说明的注释代码，不机械添加作者、日期或无信息量注释。
25. 新增日志时检查日志门面、参数化输出、级别、异常现场、重复日志、日志量和敏感信息。
26. 不无需求修改项目日志级别、Appender、日志文件名、采集方式或保留周期。
27. 只报告和执行项目真实存在的 formatter、静态检查和测试。

最终原则：

> 架构规范决定“类是什么、放哪里”；Java 规范决定“类在 Java 中如何实现”。Java 编程风格优先追求清晰、类型安全、稳定契约、Null 安全、精度正确和维护成本可控，并始终以目标项目已有契约和合理一致性为优先。
