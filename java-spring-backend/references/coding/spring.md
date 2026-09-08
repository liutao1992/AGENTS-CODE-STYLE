# Spring Boot 编码规范

本文档定义 Spring Boot 相关编码规范。

相关规范：

- [应用分层](../architecture/layering.md)
- [事务](../architecture/transactions.md)
- [并发](../architecture/concurrency.md)
- [API 设计](../api/api-design.md)
- [Java 编码与模型](java.md)

核心原则：

> Controller 管接口，Service 管业务流程，Manager 管复用、适配和原子操作。

> HTTP 语义不得向 Service / Manager 扩散。

> 具体业务接口输出统一优先使用 VO；Response 仅用于项目级通用 HTTP 响应包装。

> 不根据推测自行创造业务状态、编码、默认值或兼容规则。

---

## 1. Controller

Controller 只负责接口边界。

主要职责：

* 接收 HTTP 请求；
* Path / Query / Body 参数绑定；
* 基础参数校验；
* 获取当前登录用户和请求上下文；
* 调用 Service；
* 返回项目统一响应。

推荐：

```java
@PostMapping("/{id}/audit")
public void audit(
        @PathVariable String id,
        @Valid @RequestBody PlaceAuditRequest request) {

    placeService.audit(id, request, currentOperator());
}
```

禁止：

```text
Controller → Mapper
```

Controller 不负责：

* SQL；
* 数据库查询编排；
* 事务；
* 状态流转；
* 复杂业务校验；
* 复杂模型组装；
* 跨模块业务流程。

原则：

> Controller 保持轻量，只表达 HTTP 接口边界。

---

## 2. Service

Service 负责业务用例和业务流程编排。

主要职责：

* 执行业务流程；
* 执行业务校验；
* 调用 Manager；
* 简单场景下直接调用 Mapper；
* 组织业务输入和输出；
* 协调多个业务能力。

方法名称应使用明确业务语言。

推荐：

```java
audit(...)
register(...)
approve(...)
reject(...)
activate(...)
```

避免：

```java
handle(...)
process(...)
doSomething(...)
execute(...)
```

Service 不应成为 HTTP、SQL、第三方协议和业务流程的混合层。

---

## 3. Service 不依赖 HTTP 语义

Service / Manager 原则上不得直接依赖：

```text
HttpStatus
ResponseEntity
HttpServletRequest
HttpServletResponse
```

禁止把以下方式作为普通业务失败的默认表达：

```java
throw new ApiException(
        HttpStatus.NOT_FOUND,
        "场所不存在");
```

业务失败应优先使用项目已有业务异常体系，例如：

```java
throw new PlaceNotFoundException(id);
```

或者：

```java
throw new BusinessException("场所不存在");
```

HTTP 状态码和错误响应由 Web 层统一转换：

```text
Service / Manager
       ↓
业务异常
       ↓
@RestControllerAdvice
       ↓
HTTP Response
```

如果项目已经存在统一异常体系，必须优先沿用，不得自行创建平行异常体系。

原则：

> Service 表达业务失败，不表达 HTTP 失败。

---

## 4. 不得自行创造业务规则

Service / Manager 中新增以下内容前，必须先搜索已有定义：

* 状态值；
* 类型值；
* 枚举；
* 业务编码；
* 默认值；
* 状态流转；
* 编号生成规则；
* 兼容输入；
* 特殊业务常量。

例如不得在没有明确依据时自行添加：

```java
"pending"
"approved"
"rejected"
"enabled"
"disabled"
"1"
"0"
```

也不得自行决定：

```text
场所编号 = CS-XXXXXXXX
```

之类的业务编号格式。

新增业务规则前应优先检查：

1. 现有 Java 代码；
2. Enum / Constant；
3. 类似业务实现；
4. Request / VO；
5. 数据库已有值；
6. 测试；
7. API 契约；
8. Git 历史。

如果没有明确依据：

> 不得自行编造规则。

可以说明缺少业务定义，但不能代替业务方做决定。

---

## 5. 业务常量与枚举

不要在 Service 中大量散落魔法字符串。

如果项目已经存在对应枚举或常量，必须复用。

例如：

```java
AuditStatus.APPROVED
AuditStatus.REJECTED
```

优于：

```java
"approved"
"rejected"
```

但不得为了“更规范”擅自创建新的枚举并修改已有 API 或数据库语义。

原则：

> 先复用现有定义；只有职责明确并且当前任务需要时才新增抽象。

---

## 6. Manager

Manager 为可选层。

适用于：

* 多 Mapper 组合；
* 可复用的数据操作；
* 多表一致性操作；
* 第三方服务适配；
* 通用数据组装；
* 缓存等技术能力封装；
* 可复用原子操作。

例如：

```text
PlaceService
     ↓
PlaceManager
   ↙       ↘
PlaceMapper AuditMapper
```

不要因为：

```text
Service → Mapper
```

就机械增加 Manager。

也不要因为 Service 代码稍多，就立即拆分：

```text
Manager
Converter
Assembler
Factory
Builder
```

原则：

> 真实职责出现后再抽取，不为了形式增加层级。

详细规则读取：

- [layering.md](../architecture/layering.md)

---

## 7. Service 与 Mapper

简单业务允许：

```text
Controller
    ↓
Service
    ↓
Mapper
```

例如：

```java
public PlaceVO getById(String id) {
    PlaceDO place = placeMapper.getById(id);
    ...
}
```

但 Service 不应把 Mapper 当成无边界的数据工具类。

涉及：

* 多表原子操作；
* 公共数据访问能力；
* 可复用复杂组合；

时，应判断是否适合引入 Manager。

跨业务模块禁止直接依赖对方 Mapper。

例如避免：

```text
CaseService
    ↓
PlaceMapper
```

优先：

```text
CaseService
    ↓
PlaceService / PlaceFacade
```

---

## 8. 模型转换

简单模型转换允许在 Service 中完成。

例如：

```java
PlaceVO.from(place);
```

或者少量字段赋值。

不要为了几行转换代码机械创建：

```text
Converter
Assembler
Factory
Builder
```

当转换具有以下特点时，可以考虑独立组件：

* 逻辑复杂；
* 多处复用；
* 涉及多个模型；
* 本身形成明确职责。

原则：

> 简单转换就地完成，复杂且可复用时再抽取。

---

## 9. Request / VO / 通用 Response

接口输入使用明确的 Request；具体业务接口输出统一优先使用 VO。

典型调用：

```text
HTTP Request
     ↓
PlaceAuditRequest
     ↓
Controller
     ↓
Service
     ↓
PlaceVO
     ↓
Controller
     ↓
HTTP Response
```

数据库 DO：

```text
PlaceDO
```

不得因为方便直接暴露给客户端。

本项目模型语义：

```text
Request
→ 接口输入

VO
→ 具体业务视图输出

Response
→ 项目级通用 HTTP 响应包装概念
```

例如具体业务返回模型优先：

```text
PlaceVO
PlaceStatsVO
PlaceTreeNodeVO
```

而不是新建：

```text
PlaceResponse
PlaceStatsResponse
place.response.*
```

通用响应包装应复用目标项目已有类型。本 Skill 不固定统一响应类名，也不使用具体项目的响应包装类型作为示例。

不得为了统一命名擅自修改已经发布的公共 API；现有历史 `*Response` 模型仅在当前任务明确要求或兼容性允许时迁移。

详细模型规则读取：

- [java.md](java.md)
- [api-design.md](../api/api-design.md)

---

## 10. Bean Validation

接口结构性校验优先使用 Bean Validation。

例如：

```java
@NotNull
@NotBlank
@NotEmpty
@Size
@Min
@Max
@DecimalMin
```

Controller 请求对象需要时使用：

```java
@Valid
```

例如：

```java
public class PlaceCreateRequest {

    @NotBlank
    @Size(max = 128)
    private String placeName;
}
```

Bean Validation 负责：

```text
字段不能为空
长度限制
数值范围
格式约束
```

业务规则仍由 Service / Manager 负责。

例如：

```text
场所名称不能为空
→ Bean Validation

只有待审核状态允许审核
→ Service / Manager
```

不要使用自定义 Bean Validation 注解承载复杂业务流程。

---

## 11. 输入标准化

简单、无业务歧义的输入标准化可以在接口或业务边界统一处理，例如去除明确无意义的首尾空格。

但不得自行进行可能改变业务语义的转换。

例如，不要未经定义自动将：

```text
pass
approved
PASS
1
```

全部解释成同一个业务状态。

这种兼容规则属于 API / 业务契约，必须有明确依据。

原则：

> 标准化可以消除格式差异，但不能创造新的业务语义。

---

## 12. 依赖注入

统一优先使用构造器注入。

推荐使用 Lombok：

```java
@Service
@RequiredArgsConstructor
public class PlaceService {

    private final PlaceManager placeManager;
}
```

也可以使用显式构造器。

避免新增字段注入：

```java
@Autowired
private PlaceManager placeManager;
```

除非当前模块已有明确统一约定需要保持。

---

## 13. Bean 生命周期

由 Spring 管理的对象统一交给 Spring 容器管理。

业务代码不得自行：

```java
new SomeService(...)
new SomeManager(...)
```

创建本应由 Spring 管理的 Service、Manager、Repository、Component、Configuration 等组件。

普通 Request、Query、DTO、BO、DO、VO 等数据对象不受此限制。

---

## 14. 配置

配置优先放入：

```text
application.yml
application-{profile}.yml
```

结构化配置优先使用：

```java
@ConfigurationProperties
```

避免大量分散：

```java
@Value("${...}")
```

禁止在代码中硬编码：

* 环境地址；
* 密钥；
* Token；
* 用户名密码；
* 环境相关开关。

---

## 15. 事务

事务由数据一致性需求决定。

Service 方法存在：

```text
INSERT
UPDATE
DELETE
```

不代表必须添加：

```java
@Transactional
```

新增事务前必须判断：

1. 是否存在多个写操作必须整体成功或回滚；
2. 是否存在明确的一致性快照要求；
3. 是否需要数据库锁；
4. 是否存在查询后修改的并发一致性要求。

普通快照读默认不显式开启事务。

单个 INSERT / UPDATE / DELETE 也不因为前后存在普通查询就机械开启事务。

详细规则必须读取：

- [transactions.md](../architecture/transactions.md)

---

## 16. 查询后修改与并发

对于：

```text
SELECT
   ↓
业务判断
   ↓
UPDATE
```

不得认为仅添加：

```java
@Transactional
```

就自动保证并发正确性。

根据业务需要评估：

* 条件 UPDATE；
* 乐观锁；
* 唯一约束；
* 悲观锁；
* 合适的事务隔离级别。

例如状态流转可以评估：

```sql
UPDATE ...
SET zt = #{targetStatus}
WHERE id = #{id}
  AND zt = #{expectedStatus}
```

并根据更新行数判断结果。

详细规则读取：

- [transactions.md](../architecture/transactions.md)

---

## 17. 异步与并发

涉及：

```text
@Async
CompletableFuture
Executor
ThreadPoolTaskExecutor
```

必须读取：

- [concurrency.md](../architecture/concurrency.md)

不得因为多个操作“看起来可以同时执行”就自动改为并发。

特别禁止：

> 在已有事务中，为了加速数据库操作而直接将 Mapper 调用拆到多个线程。

---

## 18. Spring Proxy

使用以下 Spring 代理能力时：

```text
@Transactional
@Async
@Cacheable
```

必须注意代理边界。

同一个对象内部直接调用带这些注解的方法时，可能绕过 Spring Proxy。

不得仅因为方法上存在注解，就默认对应能力一定生效；应检查实际 Bean 调用关系。

---

## 19. 异常处理

Controller 不应重复：

```java
try {
    ...
} catch (...) {
    ...
}
```

项目存在统一异常处理机制时，应使用现有 `@RestControllerAdvice` 或全局异常处理方案。

职责应保持：

```text
Service
   ↓
业务异常
   ↓
Global Exception Handler
   ↓
HTTP 状态 + 错误响应
```

禁止：

* Controller 到处重复异常转换；
* Service 直接拼装 HTTP 错误响应；
* 为单个模块创建另一套错误响应体系；
* 将数据库异常详情直接返回客户端。

---

## 20. API 兼容性

没有明确需求时，不得修改：

* URL；
* HTTP Method；
* Request 字段；
* VO / 已发布输出字段；
* 通用响应包装结构；
* 字段类型；
* Null 语义；
* 枚举值；
* 错误码；
* 时间格式；
* 分页结构。

不得为了使后端实现更方便，擅自改变已有 API 契约。

详细规则读取：

- [api-design.md](../api/api-design.md)

---

## 21. 不要过度使用 Spring

不要为了“Spring 化”而机械增加：

```text
Component
Service
Bean
Configuration
Event
Listener
AOP
```

普通纯 Java 能力无需自动变成 Spring Bean。

原则：

> Spring 用于管理有生命周期、有依赖关系的应用组件，不是所有 Java 类都需要进入容器。

---

## 22. Codex 修改流程

修改 Spring 相关代码时：

1. 判断当前修改属于 Controller、Service、Manager 还是基础设施。
2. 阅读当前模块类似实现。
3. 检查项目已有异常、枚举、常量和业务规则。
4. 不自行创造状态、编码、默认值或兼容语义。
5. 检查是否真的需要 Manager。
6. 检查是否真的需要事务。
7. 检查是否引入 HTTP 语义到 Service / Manager。
8. 检查具体业务输出是否正确使用 VO，是否直接暴露 DO。
9. 检查是否存在不必要的 Spring Bean 或抽象。
10. 检查完整调用链。
11. 执行相关测试。

---

## 23. Codex 检查

修改 Spring 代码后检查：

### Controller

* Controller 是否过重；
* Controller 是否直接访问 Mapper；
* Controller 是否包含业务流程；
* 参数校验是否放在正确位置；
* 是否重复实现异常处理。

### Service / Manager

* Service 是否表达明确业务行为；
* Service / Manager 是否直接依赖 `HttpStatus`、`ResponseEntity` 等 HTTP 类型；
* 是否自行创造业务状态、编码、默认值或编号格式；
* 是否存在大量散落的魔法字符串；
* 是否优先复用了已有 Enum / Constant；
* 是否无意义增加 Manager；
* 是否出现过度 Converter / Assembler / Factory 抽象。

### 数据与模型

* DO 是否直接作为 API 输出；
* 具体业务输出是否使用 VO；
* 是否错误新增 `*Response` 作为业务视图模型；
* Request / Query / DTO / BO / DO / VO 是否职责明确；
* 输入标准化是否改变业务语义。

### Spring 基础设施

* 是否优先使用构造器注入；
* 是否新增字段注入；
* 是否自行 `new` Spring 管理对象；
* 是否硬编码环境配置；
* 是否错误依赖 Spring Proxy 自调用。

### 事务与并发

* 是否因为存在写操作就机械增加事务；
* 普通快照读是否无意义增加事务；
* 是否错误认为 `@Transactional` 自动解决查询后更新的并发问题；
* 是否因为性能猜测引入异步；
* 是否在事务中直接并行 Mapper 操作。

### 兼容性

* 是否修改已有 API 契约；
* 是否新建平行异常体系；
* 是否重复实现已有 Spring 基础设施。

最终原则：

> Controller 只处理接口边界，Service 表达业务流程，Manager 承担真正需要复用或原子化的能力；具体业务输出使用 VO，HTTP 语义止于 Web 层，业务规则必须有依据，事务和并发必须有真实需求。