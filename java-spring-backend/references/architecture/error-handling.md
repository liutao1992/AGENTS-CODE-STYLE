# 异常处理与错误边界规范

本文档定义异常在 Mapper / Client / Manager / Service / Web / API 等边界之间如何传播、转换、记录和对外表达。

本文负责回答：

> 异常在哪里转换、哪里补充业务上下文、哪里记录完整现场、哪里转换为对外错误契约。

Java `catch` / `throw` / 日志 API 读取：

- [Java](../coding/java.md)

Spring Web 异常处理机制读取：

- [Spring](../coding/spring.md)

HTTP 错误契约读取：

- [API](../api/api-design.md)

核心原则：

> 异常只在真正拥有恢复、抽象转换、补充上下文、记录现场或对外表达职责的边界处理，不让每一层机械 `catch + log + wrap`。

---

## 1. 典型异常流转

```text
Database / External System
        ↓
Mapper / Client / Adapter
        ↓ 必要时隔离技术异常
Manager（可选）
        ↓ 必要时转换应用能力语义
Service
        ↓ 补充业务用例上下文
Web / API Boundary
        ↓ 转换为安全稳定错误契约
Client
```

不是每次失败都必须经过所有层。简单链路可以：

```text
Mapper → Service → Web Error Handler
```

只有真实存在对应边界时才处理。

---

## 2. Mapper / DAO

Mapper / DAO 负责数据访问，不负责重复记录业务失败现场。

Spring + MyBatis 项目优先复用框架已有数据访问异常翻译机制，不为每个 Mapper 方法机械增加：

```text
catch Exception
→ DAOException
```

只有真实技术隔离需求时才转换，并保留原始 cause。

Mapper 通常不重复打印完整异常堆栈，因为上层通常拥有更有价值的业务上下文。

原则：

> Mapper 负责数据访问和必要技术异常隔离，不负责证明异常经过这一层。

---

## 3. Client / Adapter

HTTP Client、RPC Client、SDK Adapter、对象存储 Client 等出站适配器应隔离供应商和协议技术细节。

例如：

```text
VendorSdkTimeoutException
        ↓
FaceRecognitionUnavailableException
```

转换目标是让上层依赖项目自己的稳定语义，而不是供应商：

```text
异常类
状态码
Request / Response 类型
SDK 类型
```

不要把所有第三方异常无差别包装成一个无法判断原因的通用异常。

原则：

> Client / Adapter 的异常转换对应真实技术抽象边界。

---

## 4. Manager

Manager 是当前应用进程内的可选应用能力层。

它可以：

* 在职责范围内恢复可恢复失败；
* 抽象语义确实变化时转换异常；
* 无法处理时继续传播；
* 为原子应用能力保留必要上下文。

不要因为异常“经过 Manager”就机械：

```text
RuntimeException
→ ManagerException
→ ServiceException
```

如果某项应用能力未来被拆成远程服务，它已经成为**另一个应用边界**。调用方应通过 Client / Adapter 访问该远程契约；远程服务内部再按自己的 Service / Manager / Mapper / Web 或 RPC 边界处理异常。

不要继续把远程服务理解成当前应用里的“独立部署 Manager”。

原则：

> Manager 表示当前应用内的应用能力；跨进程以后就是新的应用边界和外部调用契约。

---

## 5. Service

Service 最了解当前业务用例，是补充业务失败上下文的重要位置。

必要且安全的上下文可能包括：

```text
业务动作
资源标识
关键业务编号
当前处理阶段
TraceId / RequestId（项目存在时）
```

但这不意味着每个 Service 方法都要：

```java
catch (Exception ex) {
    log.error(..., ex);
    throw ex;
}
```

如果项目已有统一异常处理器、AOP 或应用边界记录未处理异常并能够获得足够上下文，Service 不重复记录相同堆栈。

可预期业务失败也不机械按系统故障记录为 `error`。

原则：

> 在最了解业务上下文且能够避免重复的位置保留一次足够失败现场。

---

## 6. Web / API 边界

Java 堆栈、内部异常类型、SQL、服务器路径和供应商技术细节不得直接越过对外协议边界。

在 Spring MVC 中，优先复用：

```text
@RestControllerAdvice
@ExceptionHandler
HandlerExceptionResolver
```

而不是每个 Controller 手写 `try/catch`。

对外错误码、错误信息、HTTP Status 和统一响应结构由 `api-design.md` 维护。

原则：

> 应用内部可以传播异常；对外协议边界必须转换成安全、稳定的错误契约。

---

## 7. 异常转换必须保留 cause

推荐：

```java
throw new StorageAccessException(
        "Failed to load attachment",
        ex);
```

避免：

```java
throw new StorageAccessException(
        "Failed to load attachment");
```

禁止无意义逐层包装：

```text
SQLException
→ DAOException
→ ManagerException
→ ServiceException
→ ApiException
```

原则：

> 转换异常是为了隔离抽象，不是为了体现调用层级。

---

## 8. 同一异常链通常只记录一次完整堆栈

避免：

```text
Mapper log.error
→ Manager log.error
→ Service log.error
→ ControllerAdvice log.error
```

导致重复告警和日志噪声。

推荐：

```text
Mapper / Client / Manager
→ 必要时转换，不机械打印完整堆栈

Service / 应用边界
→ 必要时补充业务上下文

统一异常处理器
→ 如果项目在这里集中记录，则上游不重复
```

具体日志位置服从目标项目现有日志架构。

---

## 9. 不吞异常、不伪装成功

禁止：

```java
catch (Exception ex) {
}
```

也禁止没有业务契约时：

```java
catch (Exception ex) {
    log.error("failed", ex);
    return null;
}
```

或返回：

```text
空集合
0
false
默认对象
默认状态
```

把系统失败转换成正常业务结果。

异步 fallback 的恢复语义读取 `concurrency.md`。

---

## 10. 事务与异常

捕获、转换或吞掉异常可能改变事务回滚。

涉及事务时必须确认：

* 当前异常是否应该触发回滚；
* 转换后的异常是否仍符合回滚契约；
* 是否因为 catch 后不再抛出导致事务提交；
* `TransactionTemplate` 回调是否无意吞掉失败。

详细规则读取 `transactions.md`。

---

## 11. 敏感信息

异常和日志不得直接记录或返回：

* 密码；
* Token；
* Cookie / Session 凭证；
* 私钥 / Secret；
* 完整身份证件；
* 生物特征；
* 未脱敏敏感个人信息；
* 第三方认证凭证。

不要通过完整对象 `toString()` 暴露敏感字段。

---

## 12. Codex 异常检查

涉及异常时检查：

1. 当前层是否真的拥有恢复、转换、补充上下文、记录或对外表达职责。
2. 是否只是机械 `catch + log + throw`。
3. 异常转换是否对应真实抽象变化并保留 cause。
4. 同一异常链是否重复记录完整堆栈。
5. Mapper / Client 是否无必要泄漏底层技术异常。
6. Manager 是否制造无意义异常层级。
7. Service 是否保留了必要且安全的业务上下文。
8. Web / API 是否通过统一边界收口。
9. 是否泄漏 SQL、堆栈、路径、供应商细节或敏感信息。
10. catch 后是否通过默认值 / fallback 改变失败语义。
11. 是否影响事务回滚或调用方原有异常契约。

最终原则：

> 底层做必要技术隔离，业务边界补充真实业务语义，应用边界记录一次足够现场，对外协议边界转换成安全稳定错误契约。