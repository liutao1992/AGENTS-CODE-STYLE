# Spring Boot 编码规范

本文档定义 Spring Boot 框架使用规范。

本文负责回答：

> Spring Bean、依赖注入、Validation、配置、Proxy、Web 异常处理等框架能力应该怎么使用。

本文不重复定义 Controller / Service / Manager / Mapper 的架构职责，也不重复 HTTP 契约、事务设计和并发设计。

相关规范：

- [应用分层与模型边界](../architecture/layering.md)
- [异常处理与错误边界](../architecture/error-handling.md)
- [事务](../architecture/transactions.md)
- [并发](../architecture/concurrency.md)
- [API 设计](../api/api-design.md)
- [Java 编码](java.md)

核心原则：

> Spring 用于管理应用组件、依赖和框架边界，不替代业务分层设计。

> 具体业务输出优先使用 VO；统一 HTTP 响应包装默认使用 `ApiResponse<T>`，但目标项目已有响应契约时以项目为准。详细规则由 API 规范维护。

---

## 1. Controller 的 Spring 使用

Controller 属于 HTTP 入站适配器，职责边界统一读取：

- [layering.md](../architecture/layering.md#3-controller--web-层)

Spring MVC 中按项目现有风格使用：

```text
@RestController
@Controller
@RequestMapping
@GetMapping
@PostMapping
@PutMapping
@PatchMapping
@DeleteMapping
```

请求参数根据 HTTP 契约使用：

```text
@PathVariable
@RequestParam
@RequestBody
@ModelAttribute
```

命令类接口默认示例：

```java
@PostMapping("/{id}/audit")
public ApiResponse<Void> audit(
        @PathVariable String id,
        @Valid @RequestBody PlaceAuditRequest request) {

    placeService.audit(id, request, currentOperator());
    return ApiResponse.success();
}
```

查询类接口默认示例：

```java
@GetMapping("/{id}")
public ApiResponse<PlaceVO> detail(@PathVariable String id) {
    PlaceVO place = placeService.getById(id);
    return ApiResponse.success(place);
}
```

这里的 `ApiResponse.success(...)` 只表示本 Skill 的默认示例。具体返回类型、构造方法、JSON 字段和历史契约以目标项目已有实现为准，不得为了示例强制迁移。

URL、HTTP Method、Request / VO、统一响应和兼容性统一读取：

- [api-design.md](../api/api-design.md)

### 1.1 路由注解位置遵循项目一致性

本 Skill 不机械规定 `@RequestMapping` 只能放在方法上，也不机械要求必须放在类上。

以下两种方式都可以是合理的：

```java
@RestController
@RequestMapping("/places")
public class PlaceController {

    @GetMapping("/{id}")
    public ApiResponse<PlaceVO> detail(@PathVariable String id) {
        ...
    }
}
```

以及：

```java
@RestController
public class PlaceController {

    @GetMapping("/places/{id}")
    public ApiResponse<PlaceVO> detail(@PathVariable String id) {
        ...
    }
}
```

选择依据是：

* 目标项目已有风格；
* URL 是否容易搜索和定位；
* 公共前缀是否真实稳定；
* 是否会因为继承、组合或多级 Mapping 造成难以理解的最终路径。

同一模块应保持合理一致，不为了个人偏好批量迁移已有 Controller。

能够使用更具体的映射注解时，优先：

```text
@GetMapping
@PostMapping
@PutMapping
@PatchMapping
@DeleteMapping
```

而不是所有方法统一写宽泛的 `@RequestMapping` 再配置 method。

URL 是否采用资源式 REST、业务动作路径或兼容历史接口属于 API 契约，统一由 `api-design.md` 判断；Spring 规范不强制一种 URL 流派。

### 1.2 Controller 保持协议层简洁

Controller 方法应以 Spring MVC 边界代码为主，例如：

```text
参数绑定
Bean Validation
获取当前请求调用者上下文
调用 Service
协议层响应包装
```

不在 Controller 中编写业务状态流转、复杂数据拼装、数据库访问或业务事务。

当前用户、部门、租户等请求绑定上下文如果业务需要，应按 `layering.md` 的边界规则取得并显式传递；不要让 Service 为了获取当前请求用户反向依赖 Web 请求对象。

### 1.3 OpenAPI / Swagger 文档遵循项目现有机制

项目已经使用 OpenAPI / Swagger 注解或自动文档时，应同步维护真实接口描述、参数和返回契约。

但本 Skill 不统一要求：

```text
每个 Controller 方法必须存在某个特定文档注解
文档描述中必须填写作者姓名
```

作者和变更历史优先由 Git 记录；接口文档只保留对调用方有长期价值的契约信息。

项目存在文档门禁、注解要求或代码生成约束时，以项目已有配置为准。

---

## 2. Bean Validation

接口结构性校验优先使用项目已有 Bean Validation 体系，例如：

```text
@NotNull
@NotBlank
@NotEmpty
@Size
@Min
@Max
@DecimalMin
@Pattern
```

Request 需要级联校验时使用：

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

Bean Validation 适合表达结构约束：

```text
不能为空
长度范围
数值范围
格式约束
```

业务状态、权限、数据存在性、跨字段业务语义以及需要访问数据库或外部系统才能确认的约束，由业务层处理。

不要为了复用校验逻辑，把复杂业务流程塞进自定义 `ConstraintValidator`。

接口校验语义读取：

- [api-design.md](../api/api-design.md)

### 2.1 避免重复结构校验

已经由可信入站边界完成的结构性约束，不应在 Service / Manager 中再次使用手写 `if`、`StringUtils`、`Objects` 等方式机械重复同义校验。

例如 Request 已经声明：

```java
public class PlaceCreateRequest {

    @NotBlank
    private String placeCode;
}
```

Controller 已经触发校验：

```java
@PostMapping
public ApiResponse<Void> create(
        @Valid @RequestBody PlaceCreateRequest request) {

    placeService.create(request);
    return ApiResponse.success();
}
```

则普通业务流程中不要再次写：

```java
if (!StringUtils.hasText(request.getPlaceCode())) {
    throw new BusinessException("场所编号不能为空");
}
```

也不要把同一个必填约束改写成：

```java
Objects.requireNonNull(request.getPlaceCode());
```

除非当前调用路径并未经过上述可信校验边界，或者这里存在与入站结构校验不同的独立契约。

原则：

> 一个约束由最合适的边界负责；不要为了“防御性编程”在多个层重复表达完全相同的前置条件。

### 2.2 禁止用默认值掩盖无效输入

对于已经声明必填或非空的字段，不得通过兜底值把非法输入悄悄转换成另一个合法值。

避免：

```java
String placeCode = StringUtils.hasText(request.getPlaceCode())
        ? request.getPlaceCode()
        : "";
```

以及没有业务依据的：

```java
String placeCode = StringUtils.hasText(request.getPlaceCode())
        ? request.getPlaceCode()
        : DEFAULT_PLACE_CODE;
```

类似：

```text
null → ""
null → 0
blank → 默认编码
非法枚举 → 默认状态
```

都可能把结构错误转换成新的业务语义。

只有需求、既有契约或项目稳定实现明确规定默认行为时才能使用默认值；不得由 Agent 为了避免异常自行创造。

### 2.3 Service 仍然负责业务校验

“不重复 Bean Validation”不代表 Service 不做校验。

业务层仍应负责真实业务规则，例如：

```text
@NotBlank placeCode
→ 入站结构校验

@Size(max = 128)
→ 入站结构校验

只有 PENDING 状态允许审核
→ Service / Manager 业务校验

场所编号是否已存在
→ Service / Manager + 数据库能力

当前操作人是否有权限
→ 业务 / 权限边界

数据库必须唯一
→ UNIQUE Constraint
```

重点是区分：

```text
结构有效性
!=
业务有效性
```

### 2.4 多入口调用时先补齐入口校验

Service 可能同时被以下入口调用：

```text
HTTP Controller
RPC Endpoint
Message Consumer
Scheduled Task
其他模块 Service / Facade
```

因此不能仅因为某个 HTTP Controller 使用了 `@Valid`，就假设所有调用方都一定完成相同结构校验。

出现多入口时，优先判断：

1. 每个外部入口是否已经在自己的边界完成必要结构校验；
2. 项目是否已有统一的方法级 Validation 机制；
3. Service 方法本身是否明确承担公共输入契约。

如果确实需要方法级校验，可以按项目现有方式评估：

```java
@Service
@Validated
public class PlaceService {

    public void create(@Valid PlaceCreateRequest request) {
        ...
    }
}
```

但不要形成：

```text
Controller @Valid
+
Service @Valid
+
Service 手写 StringUtils.hasText
```

三套完全相同的机械校验。

原则：

> 缺少校验时修复真正缺失的入口或公共契约；不要通过业务代码中的零散二次校验弥补不清晰的调用边界。

---

## 3. 依赖注入

新代码优先使用构造器注入。

使用 Lombok 时可以：

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

除非目标模块已有明确统一约定且当前任务不适合迁移。

构造器注入有利于：

* 依赖显式；
* 不可变字段；
* 单元测试；
* 发现循环依赖。

不要为了使用构造器注入顺带批量改造无关历史类。

---

## 4. Spring Bean 生命周期与组件边界

需要由 Spring 管理生命周期和依赖关系的应用组件交给容器管理，例如：

```text
@Service
@Component
@Repository
@Controller / @RestController
@Configuration
```

业务代码不得自行 `new` 本应由 Spring 管理的 Service、Manager、Client、Component 或 Configuration。

普通数据模型和纯 Java 值对象不受此限制。

不要为了“Spring 化”机械把所有类声明为 Bean。无状态纯函数、普通转换对象、Request / Query / DTO / BO / DO / VO 等没有容器需求时不应自动进入 Spring 容器。

原则：

> 只有真实存在生命周期、依赖注入、代理或容器协作需求时才交给 Spring 管理。

---

## 5. 配置

环境和可变配置应进入目标项目已有配置体系。

Spring Boot 常见：

```text
application.yml
application-{profile}.yml
```

结构化配置优先考虑：

```java
@ConfigurationProperties
```

相比大量分散的：

```java
@Value("${...}")
```

更适合表达一组相关配置。

禁止在业务代码硬编码：

* 环境地址；
* 用户名密码；
* Token / Secret；
* 私钥；
* 环境相关开关。

已有项目使用其他配置中心或配置绑定方式时，以项目现有机制为准。

---

## 6. Spring Proxy 与自调用

以下常见能力通常依赖 Spring Proxy：

```text
@Transactional
@Async
@Cacheable
@CacheEvict
```

同一个对象内部直接调用带这些注解的方法时，可能绕过代理。

例如：

```java
public void process() {
    audit();
}

@Transactional
public void audit() {
}
```

不能仅因为 `audit()` 上存在 `@Transactional` 就认为自调用一定经过事务代理。

使用代理能力时必须检查：

* Bean 是否由 Spring 管理；
* 调用是否经过代理；
* 方法可见性和代理方式；
* 是否存在自调用；
* 项目是否有 AspectJ 或其他不同机制。

不要为了让注解“生效”机械拆 Bean，应先根据实际职责和项目代理方案判断。

---

## 7. `@Transactional`

`@Transactional` 是 Spring 提供的事务声明机制，不是事务设计本身。

是否需要事务、事务放在哪个一致性边界、隔离级别、传播和锁如何选择，统一读取：

- [transactions.md](../architecture/transactions.md)

Spring 侧重点只检查：

* 注解是否实际经过代理；
* 自调用是否导致失效；
* 异常是否被吞掉导致非预期提交；
* 配置的传播 / rollback 规则是否与事务规范一致。

不要因为方法执行 INSERT / UPDATE / DELETE 就机械添加 `@Transactional`。

`rollbackFor` 应根据目标项目异常体系和真实回滚语义决定，不设置“所有事务必须统一写 `rollbackFor = Exception.class`”之类的通用硬规则。

---

## 8. `@Async` 与异步能力

使用：

```text
@Async
CompletableFuture
Executor
ThreadPoolTaskExecutor
```

时必须读取：

- [concurrency.md](../architecture/concurrency.md)

Spring 侧重点包括：

* `@Async` 是否经过代理；
* 使用哪个 Executor；
* 异步异常如何处理；
* SecurityContext / MDC / ThreadLocal 是否有项目级传播机制。

不得假设 Spring 事务或线程上下文自动传播到异步线程。

---

## 9. Web 异常处理

项目存在统一 Web 异常处理机制时，普通 Controller 不应重复手写：

```java
try {
    service.execute();
} catch (Exception ex) {
    ...
}
```

优先复用项目已有：

```text
@RestControllerAdvice
@ControllerAdvice
@ExceptionHandler
HandlerExceptionResolver
```

职责关系：

```text
Service / 应用代码
       ↓
业务或技术异常
       ↓
统一 Web 异常处理器
       ↓
HTTP 错误契约
```

异常在哪一层转换、哪里记录完整现场，读取：

- [error-handling.md](../architecture/error-handling.md)

错误码、错误信息、HTTP Status 和响应结构读取：

- [api-design.md](../api/api-design.md)

禁止在每个 Controller 创建一套局部错误响应体系。

---

## 10. HTTP 语义止于入站边界

业务 Service / Manager 原则上不应直接依赖：

```text
HttpStatus
ResponseEntity
HttpServletRequest
HttpServletResponse
```

业务层表达业务语义；Web 层负责 HTTP 表达。

这是架构边界规则，详细判断统一读取：

- [layering.md](../architecture/layering.md)

如果某个底层组件确实属于 Web 基础设施，应按真实职责判断，而不是仅靠类型名称机械判定。

---

## 11. 不要过度使用 Spring

不要为了“框架统一”机械增加：

```text
@Component
@Service
@Bean
@Configuration
Event / Listener
AOP
自定义 Starter
```

也不要仅为了依赖注入给一个纯数据或纯算法类增加 Spring 身份。

原则：

> Spring 解决容器和框架协作问题，不负责给所有 Java 类增加一层框架包装。

---

## 12. Codex Spring 修改流程

修改 Spring 代码时：

1. 先按 `layering.md` 确认当前类真实职责。
2. 查看当前模块已有 Spring 注解、路由注解位置和依赖注入风格。
3. Controller/API 契约读取 `api-design.md`，不在 Spring 规范重复推导；不机械禁止类级 `@RequestMapping`，也不为了个人偏好迁移路由风格。
4. 参数校验区分结构校验和业务校验；已有可信 Bean Validation 时不在 Service / Manager 机械重复同义校验。
5. 检查必填字段是否被 `""`、`0`、默认编码或默认状态等无依据兜底掩盖。
6. 多入口调用时确认真正缺失的是哪个入口校验或公共方法契约，不使用零散 `StringUtils` 判断代替边界设计。
7. Controller 是否只保留协议边界所需逻辑，当前请求调用者上下文是否按分层规则传递。
8. OpenAPI / Swagger 是否沿用项目已有文档机制，不机械要求作者注解。
9. 新增 Bean 前确认确实需要 Spring 生命周期、依赖注入或代理能力。
10. 使用 `@Transactional`、`@Async`、缓存等代理能力时检查实际代理边界。
11. Web 异常优先复用统一 Advice / Handler，并读取 `error-handling.md`。
12. 不硬编码环境配置和敏感凭证。
13. 修改后执行目标项目已有相关测试和静态检查。

检查重点：

* Controller 是否遵循项目已有 Spring MVC 风格；
* 路由注解位置是否与模块保持一致，是否存在难以理解的多级 Mapping；
* 是否机械规定只能 GET / POST 或机械反对项目已有 REST 风格；
* `@Valid` / Bean Validation 是否用于结构性约束；
* 已完成 Bean Validation 的字段是否又在业务层进行同义 `null` / blank / size 校验；
* 是否通过 `StringUtils.hasText(...) ? value : defaultValue` 等方式掩盖本应拒绝的无效输入；
* 是否把结构校验和业务校验混为一谈；
* 多入口场景是否遗漏真正的入口校验；
* Controller 是否包含业务逻辑、数据库访问或复杂业务数据拼装；
* 是否让 Service / Manager 直接读取请求专用上下文；
* 是否新增字段注入；
* 是否自行 `new` Spring 管理组件；
* 是否创建无必要 Spring Bean；
* 是否硬编码环境配置；
* `@Transactional` / `@Async` / `@Cacheable` 是否可能因自调用绕过代理；
* 是否无依据强制所有事务设置统一 `rollbackFor`；
* Web 异常是否重复在 Controller 手工处理；
* Service / Manager 是否无必要依赖 HTTP 类型；
* 是否为了 Spring 形式顺带改造无关代码。

最终原则：

> 分层规范决定组件职责，API 规范决定 HTTP 契约，事务和并发专项规范决定行为边界；Spring 规范只负责这些设计在 Spring 框架中的正确实现。结构性约束由合适的可信边界统一保证，业务层不机械二次校验，也不通过无依据默认值掩盖非法输入。
