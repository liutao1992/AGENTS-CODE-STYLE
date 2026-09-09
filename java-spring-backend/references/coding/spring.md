# Spring Boot 编码规范

本文档定义 Spring Framework / Spring Boot 的框架使用规范。

本文负责回答：

> Spring MVC、Bean Validation、依赖注入、Bean 生命周期、配置、Proxy、事务 API、异步注解和 Web 异常处理机制应该怎么使用。

本文不定义：

```text
Controller / Service / Manager / Mapper 的业务职责
HTTP URL / Method / VO / 统一响应契约
事务是否需要以及一致性范围
并发是否值得引入
异常跨层语义
```

这些分别读取：

- [分层](../architecture/layering.md)
- [API](../api/api-design.md)
- [事务](../architecture/transactions.md)
- [并发](../architecture/concurrency.md)
- [异常处理](../architecture/error-handling.md)

核心原则：

> Spring 负责容器、代理和协议框架机制，不替代业务分层和领域契约设计。

---

## 1. Spring MVC

Controller 是 HTTP 入站适配器，职责读取 `layering.md`。

按目标项目现有风格使用：

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

请求绑定根据真实 HTTP 契约选择：

```text
@PathVariable
@RequestParam
@RequestBody
@ModelAttribute
```

URL、HTTP Method、Request、VO、统一响应和兼容性由 `api-design.md` 决定；Spring 规范不重新定义这些契约。

### 1.1 Mapping 注解位置

本 Skill 不机械规定 `@RequestMapping` 只能放方法，也不机械要求必须放类上。

以下都可以是合理方式：

```java
@RestController
@RequestMapping("/places")
public class PlaceController {

    @GetMapping("/{id}")
    public Object detail(@PathVariable String id) {
        ...
    }
}
```

或：

```java
@RestController
public class PlaceController {

    @GetMapping("/places/{id}")
    public Object detail(@PathVariable String id) {
        ...
    }
}
```

选择依据：

* 项目已有风格；
* 公共前缀是否稳定；
* 最终 URL 是否容易定位；
* 是否存在继承 / 多级 Mapping 导致路径难以理解。

能够使用更具体映射注解时，优先具体注解，不把所有方法机械写成宽泛 `@RequestMapping`。

### 1.2 Controller 保持协议层简洁

Spring MVC Controller 中保留协议边界代码：

```text
参数绑定
Bean Validation
读取当前请求上下文
调用 Service
协议层返回
```

不在 Controller 中编写数据库访问、业务状态机、复杂业务拼装和业务事务。

当前用户、租户、部门等请求专用信息如何进入业务层读取 `layering.md`。

### 1.3 OpenAPI / Swagger

项目已经使用 OpenAPI / Swagger 时，应同步维护真实接口契约信息。

本 Skill 不统一要求：

```text
每个方法必须使用某个固定文档注解
文档注解必须填写作者姓名
```

作者与变更历史优先由 Git 维护。

---

## 2. Bean Validation

结构性约束优先使用项目已有 Bean Validation 体系：

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

嵌套对象需要级联校验时使用：

```java
@Valid
```

Bean Validation 适合：

```text
非空
长度
数值范围
格式
结构约束
```

业务状态、权限、跨数据源条件、数据库存在性等需要业务语义或外部数据的规则由业务层负责，不把完整业务流程塞入 `ConstraintValidator`。

### 2.1 避免重复结构校验

已经由可信入口通过 Bean Validation 等机制保证的同一结构约束，不在 Service / Manager 机械再写同义：

```text
null
blank
size
pattern
```

判断。

例如入口已经可靠保证 `placeCode` 非空，业务层不再仅为同一非空语义重复 `StringUtils.hasText`。

但 Service 存在多入口时，不能只因为某个 HTTP Controller 有 `@Valid` 就假设所有调用方都已校验。

优先检查：

1. 每个外部入口是否完成必要结构校验；
2. 是否存在项目统一方法级 Validation；
3. Service 方法是否明确承担公共输入契约。

缺少校验时修复真实缺失边界，不通过业务代码零散二次判断掩盖边界设计。

### 2.2 不用默认值掩盖非法输入

对于已经声明必填或固定结构的字段，没有契约时不要转换：

```text
null → ""
null → 0
blank → 默认编码
非法枚举 → 默认状态
```

默认行为必须来自真实需求或项目既有契约。

### 2.3 `@Validated`

需要方法级 Bean Validation 时，按目标项目已有方式使用 `@Validated`。

不要机械形成：

```text
Controller @Valid
+
Service @Validated / @Valid
+
Service 手写同义校验
```

三套完全相同约束。

---

## 3. 依赖注入

新代码优先构造器注入。

例如：

```java
@Service
@RequiredArgsConstructor
public class PlaceService {
    private final PlaceManager placeManager;
}
```

也可以显式构造器。

避免新增字段注入：

```java
@Autowired
private PlaceManager placeManager;
```

除非当前项目已有明确统一约定且任务不适合迁移。

不要为了改成构造器注入顺带批量修改无关历史类。

---

## 4. Spring Bean 生命周期

需要容器生命周期、依赖注入、代理或框架协作的组件交给 Spring 管理，例如：

```text
@Service
@Component
@Repository
@Controller / @RestController
@Configuration
```

普通 Request / Query / DTO / BO / DO / VO、值对象和纯 Java 算法类没有容器需求时不机械声明为 Bean。

业务代码不得自行 `new` 本应由 Spring 管理并依赖代理 / 生命周期的组件。

原则：

> 真实存在容器协作需求才成为 Spring Bean。

---

## 5. 配置

环境和可变配置进入目标项目已有配置体系。

结构化配置可以优先评估：

```java
@ConfigurationProperties
```

相比大量分散 `@Value` 更适合表达一组相关配置。

禁止在业务代码硬编码：

* 环境地址；
* 用户名 / 密码；
* Token / Secret；
* 私钥；
* 环境开关。

项目使用配置中心或其他绑定机制时沿用现有方案。

---

## 6. Spring Proxy

以下能力常通过 Spring Proxy 实现：

```text
@Transactional
@Async
@Cacheable
@CacheEvict
```

同一个对象内部直接调用带注解方法时，可能绕过代理。

例如：

```java
public void process() {
    audit();
}

@Transactional
public void audit() {
}
```

不能仅看到 `audit()` 上有注解就认定当前自调用路径经过事务代理。

使用代理能力时检查：

* Bean 是否由 Spring 管理；
* 调用是否经过代理；
* 方法可见性；
* JDK / CGLIB / AspectJ 等实际机制；
* 是否存在自调用。

不要为了让注解“生效”机械拆 Bean；先判断真实职责和项目代理机制。

---

## 7. Spring 事务实现机制

事务为什么需要、谁拥有一致性边界、传播 / 隔离 / 锁 / rollback 语义统一读取 `transactions.md`。

Spring 侧只负责两种实现方式。

### 7.1 `@Transactional`

适合清晰的方法级事务边界。

检查：

* 是否经过 Proxy；
* 是否存在自调用；
* `rollbackFor` / `noRollbackFor` 配置是否与事务规范和项目契约一致；
* 异常是否被吞掉导致非预期提交。

不要因为方法执行写 SQL 就机械添加 `@Transactional`。

### 7.2 `TransactionTemplate`

适合显式、局部事务代码块。

例如：

```java
transactionTemplate.executeWithoutResult(status -> {
    ...
});
```

Spring 机制注意：

* 它不依赖 `@Transactional` 方法代理；
* `rollbackFor` 不适用于 `TransactionTemplate`；
* 回调内异常被捕获并吞掉后，不会因为“曾发生异常”自动回滚；
* 需要显式恢复时才根据项目语义使用 `status.setRollbackOnly()`；
* 使用哪个 `PlatformTransactionManager`、传播、隔离、超时必须与项目配置一致。

`TransactionTemplate` 不决定代码应该位于 Service 还是 Manager。事务所有者由 `transactions.md` 的一致性边界决定。

原则：

> Spring 决定事务如何生效；事务规范决定事务是否存在、覆盖什么。

---

## 8. `@Async`

使用 `@Async` 时必须同时读取 `concurrency.md`。

Spring 侧检查：

* 是否经过 Proxy；
* 使用哪个 Executor；
* 异步异常由谁接收；
* SecurityContext / MDC / ThreadLocal 是否有项目级传播机制。

不得假设事务和请求上下文自动传播到异步线程。

---

## 9. Cache 注解

使用 `@Cacheable` / `@CacheEvict` 等能力时检查：

* 是否经过 Proxy；
* key 是否稳定；
* 缓存一致性语义是否已有业务依据；
* 是否因为自调用导致注解不生效。

不要为推测性能收益机械增加缓存。

---

## 10. Web 异常处理

项目已有统一 Web 异常处理时，普通 Controller 不重复手写：

```java
try {
    service.execute();
} catch (Exception ex) {
    ...
}
```

优先复用：

```text
@RestControllerAdvice
@ControllerAdvice
@ExceptionHandler
HandlerExceptionResolver
```

异常在哪层转换 / 记录读取 `error-handling.md`；HTTP 错误结构读取 `api-design.md`。

---

## 11. HTTP 语义止于 Web 边界

业务 Service / Manager 原则上不直接依赖：

```text
HttpStatus
ResponseEntity
HttpServletRequest
HttpServletResponse
```

这属于分层规则，详细读取 `layering.md`。

如果某个基础设施类本身就是 Web 技术组件，则按真实职责判断，不仅靠类型名称机械判定。

---

## 12. 不要过度使用 Spring

不要为了形式机械增加：

```text
@Component
@Service
@Bean
@Configuration
Event / Listener
AOP
自定义 Starter
```

Spring 解决容器与框架协作问题，不负责给所有 Java 类增加框架身份。

---

## 13. Codex Spring 检查

修改 Spring 代码时检查：

1. Controller 是否只承担协议边界职责。
2. Mapping 注解是否遵循项目已有机制，最终 URL 是否清楚。
3. Bean Validation 是否表达结构约束，是否在业务层机械重复。
4. 多入口时真正的校验边界是否完整。
5. 是否通过默认值掩盖非法输入。
6. 依赖注入和 Bean 生命周期是否合理。
7. 环境配置和凭证是否进入正确配置机制。
8. `@Transactional` / `@Async` / Cache 是否真正经过 Proxy。
9. 事务是否被 Spring API 形式反向决定分层；`TransactionTemplate` 是否只作为实现手段。
10. Web 异常是否复用统一 Advice / Handler。
11. 是否让业务层无必要依赖 HTTP 类型。
12. 是否为了 Spring 形式扩大无关改动。

最终原则：

> Spring reference 只维护框架机制；分层、API、事务、并发和异常的业务语义由各自 reference 唯一维护。