# API 设计规范

本文档定义项目 HTTP API 的设计和兼容性规范。

核心原则：

> API 是稳定契约，不应随着数据库结构或内部实现随意变化。

> API 使用清晰的英文业务语义，不暴露数据库拼音命名和持久化细节。

> 具体业务输出统一优先使用 VO；统一 HTTP 响应包装默认使用 `ApiResponse<T>`。如果目标项目已有其他统一响应类型、历史 API 或序列化契约，以项目现有约定为准。

> 没有明确需求时，优先保持已有接口兼容。

模型职责与 Package 归属统一由应用分层规范定义：

- [应用分层与模型边界](../architecture/layering.md)

---

# 1. API 边界

API 层负责：

* 接收客户端请求；
* 表达业务输入；
* 返回业务结果；
* 参数结构校验；
* 对外错误表达；
* 维护接口兼容性。

API 不应暴露：

* 数据库表结构；
* Mapper；
* DO；
* 数据库拼音字段；
* 内部异常堆栈；
* 内部技术实现细节。

---

# 2. URL 设计

URL 应表达资源和业务语义。

推荐：

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
DELETE /places/{id}
```

已有项目存在明确统一约定时，优先保持现有风格。

---

# 3. 业务动作

无法自然表达为 CRUD 的业务动作，可以使用明确动作路径。

例如：

```text
POST /places/{id}/audit
POST /places/{id}/activate
POST /cases/{id}/submit
POST /cases/{id}/cancel
```

避免：

```text
POST /places/doSomething
POST /places/process
POST /places/handle
```

业务动作名称必须明确表达实际含义。

---

# 4. HTTP Method

原则上：

```text
GET
→ 查询

POST
→ 创建或业务命令

PUT
→ 整体或明确更新

PATCH
→ 局部更新

DELETE
→ 删除
```

不得为了实现方便把所有接口都设计成 POST。

---

# 5. GET 请求

GET 用于读取数据，原则上：

* 不改变业务状态；
* 不创建数据；
* 不产生持久化副作用。

禁止：

```text
GET /places/{id}/delete
GET /places/{id}/approve
```

这种通过查询请求修改数据的设计。

---

# 6. Request 模型

复杂接口应使用明确 Request 对象。

例如：

```java
public class PlaceAuditRequest {

    @NotNull
    private AuditResult result;

    @Size(max = 500)
    private String remark;
}
```

Request 名称应表达具体用途：

```text
PlaceCreateRequest
PlaceUpdateRequest
PlaceAuditRequest
CaseRegisterRequest
```

避免：

```text
PlaceRequest
CommonRequest
DataRequest
```

不要为了减少一个类大量使用：

```java
Map<String, Object>
```

表达普通业务请求。

如果一个 Request 同时承担多个完全不同接口的输入，应评估拆分。

Request 的模型职责和 Package 归属读取：

- [layering.md](../architecture/layering.md#91-request)

---

# 7. Request 与数据库隔离

Request 使用英文业务语义。

数据库：

```text
zjhm
rqsj
csbh
```

API：

```text
identityNumber
entryTime
placeCode
```

禁止直接设计：

```json
{
  "zjhm": "...",
  "rqsj": "...",
  "csbh": "..."
}
```

除非已有 API 契约明确要求兼容历史字段。

原则：

```text
Database（拼音）
      ↓
   防腐边界
      ↓
API（英文业务语义）
```

---

# 8. VO 与 ApiResponse<T>

具体业务接口输出统一优先使用 VO。

例如：

```java
public class PlaceDetailVO {

    private String id;

    private String placeName;

    private PlaceStatus status;
}
```

默认模型语义：

```text
Request
→ API 输入

VO
→ 具体业务视图输出

ApiResponse<T>
→ 通用 HTTP 响应包装
```

因此，具体业务视图模型优先使用：

```text
PlaceVO
PlaceDetailVO
PlaceStatsVO
PlaceTreeNodeVO
```

而不是新增：

```text
PlaceResponse
PlaceStatsResponse
place.response.*
```

统一 HTTP 响应包装默认使用：

```text
ApiResponse<T>
```

例如：

```text
ApiResponse<PlaceVO>
ApiResponse<PlaceDetailVO>
```

但 `ApiResponse<T>` 是本 Skill 的默认推荐，不是覆盖项目既有契约的强制要求。如果目标项目已经存在其他统一响应类型、固定序列化结构或已发布 API，必须优先遵循目标项目，不得为了套用本规范强制迁移。

禁止直接返回数据库 DO：

```java
@GetMapping("/{id}")
public PlaceDO detail(...) {
}
```

原因：

* 数据库字段可能泄漏；
* 内部字段可能被意外暴露；
* 数据库变化可能直接破坏 API；
* API 生命周期与数据库模型生命周期不同。

不得为了统一命名擅自重命名已经发布的历史 `*Response` API 模型；兼容性优先于规范迁移。

模型分类与 Package 归属详细读取：

- [layering.md](../architecture/layering.md#9-模型分类与-package-归属)

Java 模型实现方式读取：

- [java.md](../coding/java.md#3-模型对象的-java-实现)

---

# 9. 返回字段最小化

只返回客户端真正需要的数据。

不要因为 DO 中存在字段，就全部复制到 VO。

尤其注意：

* 内部主键；
* 删除标识；
* 内部状态；
* 数据权限字段；
* 审计字段；
* 密钥；
* Token；
* 内部备注；
* 技术字段。

API 字段由接口契约决定，不由数据库结构决定。

---

# 10. 参数校验

结构性参数校验使用 Bean Validation。

例如：

```java
@NotBlank
@Size(max = 128)
private String placeId;
```

常见：

```text
@NotNull
@NotBlank
@NotEmpty
@Size
@Min
@Max
@Pattern
```

Controller 使用：

```java
@Valid
```

或项目已有校验方式。

---

# 11. 结构校验与业务校验分离

结构校验：

```text
名称不能为空
长度不能超过 100
编号格式必须正确
```

适合 Bean Validation。

业务校验：

```text
只有待审核状态才能审核
当前用户是否有权操作
场所是否允许删除
案件是否已经提交
```

应放在 Service / Manager。

不要把复杂业务判断写成 Bean Validation 注解。

---

# 12. Path、Query、Body

Path 参数用于标识具体资源：

```text
/places/{id}
```

Query 参数用于过滤、分页、排序：

```text
/places?status=ACTIVE&pageNum=1&pageSize=20
```

Body 用于复杂业务输入：

```json
{
  "result": "APPROVED",
  "remark": "审核通过"
}
```

不要把大量复杂业务参数全部塞进 URL。

---

# 13. Query 对象

查询条件较多时，应使用 Query 对象。

例如：

```java
public class PlaceQuery {

    private String placeName;

    private PlaceStatus status;

    private String centerCode;
}
```

避免大量方法参数，也不要使用：

```java
Map<String, Object>
```

承载普通业务查询条件。

Query 的模型语义和 Package 归属读取：

- [layering.md](../architecture/layering.md#92-query)

Java 实现方式读取：

- [java.md](../coding/java.md)

---

# 14. 分页

分页接口应遵循项目统一分页结构。

例如：

```text
pageNum
pageSize
```

或者项目已有其他命名。

不得在同一个项目随意混用：

```text
page
pageIndex
current
pageNum
```

如果项目已经存在统一分页返回结构，应直接复用。

分页包装属于通用 HTTP 响应结构，不等同于具体业务 VO。

如果目标项目没有其他约定，分页业务数据可以作为 `ApiResponse<T>` 的数据部分返回；具体分页对象结构仍以项目约定为准。

---

# 15. 分页排序

分页必须具有稳定排序。

例如：

```text
createTime DESC
id DESC
```

不要依赖数据库默认返回顺序。

如果允许客户端指定排序字段，排序字段必须通过白名单映射。

禁止直接将用户输入拼接到 SQL。

---

# 16. 空值语义

API 必须明确：

```text
null
空字符串
空数组
字段不存在
```

分别代表什么。

集合结果通常优先返回：

```json
[]
```

而不是：

```json
null
```

但应遵循现有 API 契约，不要随意改变已有接口 Null 语义。

---

# 17. Boolean

Boolean 字段应使用明确业务名称。

推荐：

```text
enabled
deleted
editable
auditable
```

避免 API 中机械使用：

```text
isEnabled
isDeleted
```

具体仍应遵循当前项目已有序列化约定。

---

# 18. 枚举

API 中的枚举必须使用稳定值。

例如：

```text
PENDING
APPROVED
REJECTED
```

或者项目统一定义的稳定编码。

不要直接使用页面展示名称作为不可变接口协议，除非项目明确如此设计。

展示文本与业务枚举应适当分离。

不得自行创造新的枚举值、别名或兼容输入。

---

# 19. 时间字段

时间字段命名必须表达业务语义。

推荐：

```text
createTime
updateTime
entryTime
exitTime
auditTime
```

避免：

```text
time
date
sj
```

时间格式必须遵循项目统一约定。

涉及跨时区场景时，必须明确：

* 时间代表的时区；
* 是否使用 UTC；
* 是否包含 Offset。

---

# 20. ID

资源 ID 应保持稳定类型。

例如接口已经使用 String，就不要无明确需求改成 Long。

不要因为数据库主键类型变化就直接改变 API ID 类型。

API 与数据库之间允许存在转换。

---

# 21. 统一响应结构

本 Skill 默认使用：

```text
ApiResponse<T>
```

作为统一 HTTP 响应包装。

典型关系：

```text
业务数据
→ VO

HTTP 通用包装
→ ApiResponse<VO>
```

例如：

```text
ApiResponse<PlaceVO>
```

但具体实现必须优先检查目标项目。如果项目已经存在其他统一响应类型、固定字段结构、全局异常响应格式或已发布契约，应继续复用项目既有类型和语义。

原则：

> `ApiResponse<T>` 是默认推荐；项目已有统一约定时，以项目为主。

不得为了套用本 Skill：

* 在已有项目中平行创建第二套响应包装；
* 批量修改历史接口返回类型；
* 改变既有 JSON 字段结构；
* 破坏已有客户端兼容性。

---

# 22. 错误码

错误码应：

* 稳定；
* 可识别；
* 具有明确业务含义。

不要把异常文本本身当成稳定错误码。

例如：

```text
PLACE_NOT_FOUND
PLACE_STATUS_INVALID
PERMISSION_DENIED
```

或者项目已有数字错误码体系。

已有错误码不得在没有明确需求时改变含义。

---

# 23. 错误信息

错误信息应便于调用方理解，但不得暴露：

* SQL；
* 数据库表名；
* Java 堆栈；
* 内部服务器路径；
* Token；
* 密钥；
* 内部敏感信息。

禁止直接将：

```java
exception.getMessage()
```

无条件返回客户端。

---

# 24. HTTP 状态码

如果项目已有统一状态码策略，应遵循已有实现。

常见语义：

```text
200  请求成功
400  请求参数错误
401  未认证
403  无权限
404  资源不存在
409  状态冲突 / 资源冲突
500  服务端异常
```

不要在单个接口自行发明新的状态码使用方式。

Service / Manager 不负责直接选择 HTTP 状态码，具体边界读取 Spring 规范。

---

# 25. API 兼容性

API 是外部契约。

没有明确需求时，不得修改：

* URL；
* HTTP Method；
* Request 字段名称；
* VO / 已发布输出字段名称；
* 通用响应包装结构；
* 字段类型；
* Null 语义；
* 枚举值；
* 时间格式；
* 错误码；
* 分页结构。

规范命名不能成为破坏现有 API 的理由。

特别是已有接口没有使用 `ApiResponse<T>` 时，不得仅因为本 Skill 默认推荐该类型就擅自修改其响应契约。

---

# 26. 新增字段

VO 新增可选字段通常比：

```text
删除字段
重命名字段
修改字段类型
```

风险更低。

但新增字段前仍应考虑：

* 是否暴露敏感信息；
* 是否存在序列化影响；
* 客户端是否可能严格解析；
* 是否真的属于当前接口。

---

# 27. 删除或重命名字段

不得直接删除或重命名已发布字段。

例如：

```text
placeName
```

不要直接修改为：

```text
name
```

除非：

* 明确要求破坏兼容；
* 已有版本迁移方案；
* 已确认调用方影响。

---

# 28. API 版本

不要因为小修改就创建：

```text
/v2
/v3
```

只有发生无法兼容的重大契约变化时才评估版本升级。

优先通过兼容方式演进现有接口。

---

# 29. 幂等性

需要关注重复请求的接口包括：

* 提交；
* 审核；
* 支付；
* 状态流转；
* 外部回调；
* 重试可能发生的写操作。

必须判断重复调用是否会：

* 重复插入；
* 重复扣减；
* 重复发送；
* 重复改变状态。

根据业务可以使用：

* 唯一约束；
* 业务状态校验；
* 请求幂等键；
* 条件 UPDATE；
* 已处理记录。

不要看到 POST 就默认一定不需要幂等。

---

# 30. 权限

客户端传来的：

```text
userId
deptCode
tenantId
dataScope
```

不得默认可信。

用户身份和权限信息应优先来自服务端可信上下文，例如：

```text
currentOperator()
SecurityContext
```

或项目已有认证体系。

不得通过客户端参数绕过数据权限。

---

# 31. 敏感字段

涉及以下数据时，应特别检查：

* 身份证件；
* 手机号；
* 地址；
* 密码；
* Token；
* 密钥；
* 生物特征；
* 内部权限信息。

只返回业务实际需要的内容。

不得因为数据库里有完整数据就全部返回 API。

---

# 32. 批量接口

批量接口应明确：

* 最大数量；
* 是否部分成功；
* 是否整体事务；
* 单项错误如何返回；
* 是否允许重复；
* 是否需要幂等。

避免无上限：

```json
{
  "ids": [...]
}
```

导致单请求处理任意数量的数据。

---

# 33. 文件接口

上传接口应明确：

* 文件大小；
* 文件类型；
* 文件数量；
* 文件名处理；
* 存储方式；
* 权限；
* 安全检查。

不要直接信任客户端上传文件名和 MIME 类型。

文件下载接口应检查访问权限。

---

# 34. 内部接口

内部接口也应保持明确契约。

不要因为“只有内部系统调用”就：

* 返回 DO；
* 暴露数据库字段；
* 忽略参数校验；
* 忽略权限和数据边界。

内部接口同样可能长期演进。

---

# 35. 跨模块调用

同一个应用内部的模块调用，优先使用 Java Service / Facade。

不要为了模块间调用自己项目内部能力，就机械增加 HTTP API。

推荐：

```text
CaseService
    ↓
PlaceService / PlaceFacade
```

而不是：

```text
CaseService
    ↓
HTTP
    ↓
PlaceController
```

除非两个模块本身就是独立服务。

---

# 36. Controller 示例

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

这里的 `ApiResponse.success(...)` 仅表示统一响应包装的示例构造方式；具体静态方法名、构造方式、错误结构和序列化字段必须以目标项目实际实现为准。

如果项目已经存在其他统一响应类型、固定 Controller 返回风格或历史 API 契约，应继续沿用项目现有设计，不得为了套用本示例强制修改为 `ApiResponse<T>`。

Controller 不承载数据库查询、状态判断、事务或复杂模型组装。

具体分层规范读取：

- [layering.md](../architecture/layering.md)

---

# 37. Codex API 修改流程

新增或修改 API 前检查：

1. 项目是否已经存在类似接口。
2. 目标项目是否已有统一响应包装；已有则复用，没有明确约定时默认使用 `ApiResponse<T>`。
3. 涉及 Request / Query / DTO / BO / DO / VO 或 Package 时，是否按 `layering.md` 确定职责与归属。
4. 是否能够复用已有 Request / VO。
5. 具体业务输出是否正确使用 VO。
6. URL 和 HTTP Method 是否符合现有约定。
7. 是否会影响已有调用方。
8. 是否改变已有字段语义。
9. 是否需要参数校验。
10. 是否需要权限校验。
11. 是否存在幂等问题。
12. 是否泄漏数据库或内部模型。
13. 是否需要补充测试。

---

# 38. Codex API 检查

完成 API 修改后检查：

* Controller 是否直接调用 Mapper；
* Request / Query / DTO / BO / DO / VO 的职责和 Package 是否符合 `layering.md`；
* 具体业务输出是否使用 VO；
* 是否错误新增 `*Response` 作为具体业务视图模型；
* 项目没有其他约定时，新接口是否默认使用 `ApiResponse<T>`；
* 项目已有统一响应约定时，是否错误地为了改成 `ApiResponse<T>` 破坏现有契约；
* 是否使用 `Map<String, Object>` 代替业务对象；
* DO 是否直接作为接口输出；
* 数据库拼音是否泄漏到 API；
* 是否暴露无关或敏感字段；
* 参数校验是否完整；
* 权限是否依赖可信上下文；
* 分页是否保持统一；
* 排序是否安全；
* 错误码是否沿用现有体系；
* 是否改变已有 API 契约；
* 写接口是否需要考虑幂等；
* 是否存在没有明确需求的破坏性修改。

最终原则：

> API 表达稳定业务契约；模型职责与 Package 由分层规范统一定义；Request 管输入，VO 管具体业务输出，统一 HTTP 响应默认使用 `ApiResponse<T>`，但目标项目已有约定时以项目为主；数据库和内部实现不得直接泄漏到接口边界。
