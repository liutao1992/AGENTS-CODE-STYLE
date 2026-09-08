# API 设计规范

本文档定义 HTTP API 的对外契约、兼容性和接口语义。

本文负责回答：

> URL、HTTP Method、Request / VO、统一响应、分页、错误码、兼容性、幂等和对外安全边界应该怎么设计。

应用分层、模型职责和 Package 归属读取：

- [应用分层与模型边界](../architecture/layering.md)

Spring MVC 注解、Validation 和 Advice 机制读取：

- [Spring](../coding/spring.md)

异常跨层流转读取：

- [异常处理与错误边界](../architecture/error-handling.md)

核心原则：

> API 是稳定契约，不应随着数据库结构或内部实现随意变化。

> 具体业务输出使用 VO；统一 HTTP 响应包装默认推荐 `ApiResponse<T>`。目标项目已有其他统一响应包装、历史 API 或序列化契约时，以项目现有约定为准。

---

## 1. API 边界

API 负责：

* 接收客户端输入；
* 表达业务请求和返回；
* 定义 HTTP 语义；
* 结构性参数校验；
* 对外错误表达；
* 维护接口兼容性。

API 不应暴露：

* Mapper / DAO；
* 数据库 DO；
* 数据库物理字段命名；
* Java 堆栈；
* 内部异常类型；
* 第三方 SDK 对象；
* 内部技术实现细节。

Controller 的架构职责不在本文重复定义，读取 `layering.md`。

---

## 2. URL、业务动作与 HTTP Method

URL 应表达资源和明确业务语义。

常见资源：

```text
/places
/cases
/equipments
```

常见资源操作：

```text
GET    /places
GET    /places/{id}
POST   /places
PUT    /places/{id}
PATCH  /places/{id}
DELETE /places/{id}
```

无法自然表达为 CRUD 的业务动作，可以使用明确动作路径：

```text
POST /places/{id}/audit
POST /places/{id}/activate
POST /cases/{id}/submit
POST /cases/{id}/cancel
```

避免：

```text
/places/doSomething
/places/process
/places/handle
```

HTTP Method 原则上：

```text
GET    → 查询
POST   → 创建或业务命令
PUT    → 整体或明确更新
PATCH  → 局部更新
DELETE → 删除
```

不得为了实现方便把所有接口机械设计成 POST。

已有项目存在稳定 URL / Method 风格时优先保持兼容。

---

## 3. GET 请求

GET 原则上用于读取：

* 不改变业务状态；
* 不创建持久化数据；
* 不产生业务写副作用。

禁止用 GET 实现删除、审核、提交等状态变更操作。

---

## 4. Request 模型

复杂接口输入使用职责明确的 Request，例如：

```text
PlaceCreateRequest
PlaceUpdateRequest
PlaceAuditRequest
CaseRegisterRequest
```

避免过宽：

```text
PlaceRequest
CommonRequest
DataRequest
```

不要为了少建一个类大量使用：

```java
Map<String, Object>
```

表达普通业务请求。

Request 的职责和 Package 统一读取：

- [layering.md](../architecture/layering.md#91-request)

Java 实现方式读取：

- [java.md](../coding/java.md)

---

## 5. 参数位置与结构校验

Path 参数用于定位资源：

```text
/places/{id}
```

Query 参数用于过滤、分页、排序：

```text
/places?status=ACTIVE&pageNum=1&pageSize=20
```

Body 用于复杂业务输入。

结构性校验适合 Bean Validation，例如：

```text
@NotNull
@NotBlank
@NotEmpty
@Size
@Min
@Max
@Pattern
```

业务校验，例如状态是否允许审核、当前用户是否有权操作，应由业务层处理，不通过复杂 Bean Validation 代替业务流程。

Spring 具体使用读取 `spring.md`。

---

## 6. Query 对象

查询条件较多时使用职责明确的 Query，例如：

```java
public class PlaceQuery {

    private String placeName;

    private PlaceStatus status;

    private String centerCode;
}
```

Query 的职责和 Package 统一读取：

- [layering.md](../architecture/layering.md#92-query)

不要使用 `Map<String, Object>` 代替普通业务查询模型。

---

## 7. 数据库与 API 隔离

API 使用 Java 英文业务语义，不直接暴露数据库物理字段命名。

例如数据库已有：

```text
zjhm
rqsj
csbh
```

API 对外可以表达为：

```text
identityNumber
entryTime
placeCode
```

已有历史 API 明确使用其他字段时优先兼容，不为了本规范无授权重命名。

数据库映射细节由 MyBatis 规范维护。

---

## 8. VO 与统一响应

具体业务输出使用 VO，例如：

```text
PlaceVO
PlaceDetailVO
PlaceStatsVO
PlaceTreeNodeVO
```

VO 的职责和 Package 读取：

- [layering.md](../architecture/layering.md#96-vo)

不要为了区分“业务输出”和“HTTP 输出”再增加一层职责相同的模型。只有输出职责、契约或数据语义真实发生变化时才进行必要转换。

统一 HTTP 响应默认推荐：

```text
ApiResponse<T>
```

典型：

```text
PlaceVO
   ↓
ApiResponse<PlaceVO>
```

`ApiResponse<T>` 是 HTTP 外层包装，不是具体业务输出模型，也不参与 Request / Query / DTO / BO / DO / VO 的模型职责分类。

`ApiResponse<T>` 是本 Skill 的默认推荐，不是覆盖目标项目现有 HTTP 包装契约的强制要求。

如果目标项目已经存在：

* 其他统一响应包装；
* 固定 JSON 字段结构；
* 全局异常响应格式；
* 已发布 API 契约；

必须继续复用项目已有 HTTP 契约和语义，不得为了本 Skill 平行创建第二套包装或批量修改历史接口。

禁止直接把数据库 DO 作为接口输出。

> 具体业务输出模型使用 VO；`ApiResponse<T>` 等统一 HTTP 包装只负责传输层响应结构。

---

## 9. 返回字段与敏感信息

只返回客户端真正需要的数据，不因为 DO 存在字段就全部复制到 VO。

特别检查：

* 内部状态和技术字段；
* 删除标识；
* 数据权限字段；
* 审计字段；
* 密码 / Token / Secret；
* 完整身份证件；
* 生物特征；
* 内部备注和服务器信息。

API 字段由接口契约决定，不由数据库结构决定。

---

## 10. 空值、Boolean、枚举、时间与 ID

API 必须保持稳定的 Null 语义：

```text
null
空字符串
空数组
字段不存在
```

集合结果通常优先 `[]`，但已有接口契约优先。

Boolean 字段使用明确业务名称，例如：

```text
enabled
deleted
editable
auditable
```

具体序列化名称以已有契约为准。

枚举值必须稳定，不自行创造别名、兼容值或状态。

时间字段名称和格式应明确业务语义；跨时区时明确 UTC / Offset / 时区约定。

资源 ID 已发布为 String / Long 等类型后，不因数据库内部类型变化随意改变 API 类型。

---

## 11. 分页与排序

分页参数和分页返回结构必须遵循项目统一约定。

例如项目已经使用：

```text
pageNum
pageSize
```

就不要在同一项目随意混入：

```text
page
pageIndex
current
```

分页结果必须具有稳定排序。

客户端可指定排序字段时，必须通过白名单映射，禁止直接把用户输入拼接到 SQL。

分页包装是通用接口结构，不属于具体业务 VO。

---

## 12. 错误码、错误信息与 HTTP Status

开放 API 的错误应形成稳定契约，例如：

```text
错误码
错误信息
必要的请求追踪标识
```

错误码应稳定、有明确语义，并复用目标项目已有体系。

禁止无依据自行创造错误码规则或改变已有错误码含义。

错误信息不得直接暴露：

* SQL；
* 数据库表名；
* Java 堆栈；
* Java 异常类；
* 内部服务器路径和地址；
* Token / Secret；
* 第三方内部异常细节。

不要把 `exception.getMessage()` 无条件作为客户端错误信息。

HTTP Status 策略以项目已有规则为准。Service / Manager 不负责选择 HTTP Status。

异常如何跨层传播读取：

- [error-handling.md](../architecture/error-handling.md)

Spring Advice 实现读取：

- [spring.md](../coding/spring.md)

---

## 13. API 兼容性

没有明确需求时，不得随意修改已发布：

* URL；
* HTTP Method；
* Request 字段；
* VO / 输出字段；
* 统一响应包装；
* 字段类型；
* Null 语义；
* 枚举值；
* 时间格式；
* 错误码；
* 分页结构。

规范命名不是破坏兼容的理由。

新增字段通常比删除、重命名、修改类型风险低，但仍应检查敏感信息、序列化和客户端兼容。

不要因为小修改机械创建 `/v2`、`/v3`；只有无法兼容的重大契约变化才评估 API 版本升级。

---

## 14. 幂等性

以下写接口需要根据真实业务评估重复请求：

```text
提交
审核
支付
状态流转
外部回调
可能重试的写操作
```

检查重复调用是否会造成：

* 重复插入；
* 重复扣减；
* 重复发送；
* 重复改变状态。

可根据现有业务约定使用唯一约束、状态校验、幂等键、条件 UPDATE、已处理记录等方式。

不要看到 POST 就默认“天然不幂等”，也不要自行创造幂等策略。

---

## 15. 认证、权限与客户端输入

客户端提交的：

```text
userId
deptCode
tenantId
dataScope
```

不得默认可信。

身份、租户和数据权限优先来自目标项目可信服务端上下文。

不得通过客户端参数绕过权限或扩大数据范围。

具体安全实现以目标项目已有安全规范和代码为准。

---

## 16. 批量、文件和内部接口

批量接口应明确已有业务约束，例如：

* 数量限制；
* 部分成功还是整体失败；
* 事务范围；
* 单项错误；
* 重复与幂等。

没有明确依据时不得自行发明固定批量上限。

文件上传需要考虑文件大小、类型、数量、文件名、存储、权限和安全检查；不得直接信任客户端文件名和 MIME 类型。

内部接口同样属于契约，不因为“内部使用”就直接暴露 DO、数据库字段或忽略权限边界。

同一应用内部跨模块调用的架构规则统一读取 `layering.md`，本文不重复维护 Service / Facade / Mapper 调用规则。

---

## 17. Controller 示例

命令类接口示例：

```java
@PostMapping("/{id}/audit")
public ApiResponse<Void> audit(
        @PathVariable
        @NotBlank
        String id,
        @Valid @RequestBody PlaceAuditRequest request) {

    placeService.audit(id, request, currentOperator());
    return ApiResponse.success();
}
```

查询类接口示例：

```java
@GetMapping("/{id}")
public ApiResponse<PlaceVO> detail(
        @PathVariable
        @NotBlank
        String id) {

    PlaceVO place = placeService.getById(id);
    return ApiResponse.success(place);
}
```

`ApiResponse.success(...)` 只是默认示例构造方式；具体方法名、字段结构和序列化契约以目标项目实际实现为准。

如果项目已有其他统一 HTTP 响应包装或历史 API，继续沿用项目现有设计。

Controller 的分层职责统一读取 `layering.md`，Spring 注解和 Validation 使用读取 `spring.md`。

---

## 18. Codex API 修改流程

新增或修改 API 前检查：

1. 是否已经存在类似接口或统一响应结构。
2. URL 和 HTTP Method 是否符合已有契约。
3. Request / Query / VO 是否能够复用已有模型。
4. 新增模型时是否按 `layering.md` 判断职责和 Package。
5. 具体业务输出是否使用 VO；统一响应是否遵循项目约定。
6. 是否无授权改变字段、Null、枚举、时间、分页或错误码语义。
7. 参数校验和业务校验是否位于正确边界。
8. 是否依赖可信身份 / 权限上下文。
9. 写接口是否存在真实幂等问题。
10. 是否泄漏数据库、内部技术对象或敏感信息。
11. 行为变化是否需要测试。

完成后检查：

* API 是否保持兼容；
* DO / 数据库物理字段是否泄漏；
* 具体业务输出是否使用项目约定的 VO；
* `ApiResponse<T>` 是否只是默认推荐而没有覆盖项目已有统一 HTTP 包装；
* 动态排序是否安全；
* 错误码和错误信息是否稳定、安全；
* 客户端身份、租户和数据范围参数是否被错误信任；
* 是否存在无依据的状态、错误码、批量阈值或兼容规则。

最终原则：

> API 规范只维护对外契约；具体业务输出使用 VO，统一 HTTP 包装与业务输出模型职责分离；分层、Spring 实现和异常流转分别由对应专项规范维护。
