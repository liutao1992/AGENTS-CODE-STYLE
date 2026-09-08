# 异常处理与错误边界规范

本文档定义异常在 Mapper / Client / Manager / Service / Web / API 等边界之间如何传播、转换、记录和对外表达。

本文负责回答：

> 异常在哪里转换、哪里补充业务上下文、哪里记录完整现场、哪里转换为对外错误契约。

具体 Java `catch` / `throw` / `finally`、日志 API 和日志格式读取：

- [Java 编码](../coding/java.md)

Spring Web 异常处理机制读取：

- [Spring](../coding/spring.md)

HTTP 错误码、错误信息和响应契约读取：

- [API 设计](../api/api-design.md)

核心原则：

> 异常只在真正拥有“恢复、抽象转换、补充上下文、记录现场或对外表达”职责的边界处理，不让每一层都机械 `catch + log + wrap`。

---

## 1. 异常流转

典型流转：

```text
数据库 / 第三方系统
        ↓
Mapper / Client / Adapter
        ↓ 必要时隔离技术异常
Manager / 应用能力
        ↓ 必要时转换抽象语义
Service / 业务用例
        ↓ 补充业务上下文
Web / API 边界
        ↓ 转换为安全稳定的错误契约
客户端
```

不是每次失败都必须经过上述所有层。简单调用链可以直接：

```text
Mapper
  ↓
Service
  ↓
Web Error Handler
```

只有真实存在对应边界时才处理。

---

## 2. Mapper / DAO

Mapper / DAO 负责数据访问，不负责重复记录业务失败现场。

Spring + MyBatis 项目优先复用框架已有的数据访问异常翻译机制，不机械为每个方法增加：

```text
catch Exception
→ DAOException
```

只有存在真实技术隔离需求时才转换，例如：

* 项目已有统一 DataAccess 异常体系；
* 底层存储库不会被现有框架正确翻译；
* 上层不应直接依赖具体驱动或存储实现异常。

转换时必须保留原始 cause。

Mapper / DAO 通常不重复打印完整异常堆栈，因为上层通常拥有更有价值的业务上下文。

原则：

> 数据访问层负责数据访问和必要的技术异常隔离，不负责证明异常经过了这一层。

---

## 3. Client / Adapter

HTTP Client、RPC Client、SDK Adapter、对象存储 Client 等出站适配器应隔离第三方协议和供应商技术细节。

如果第三方异常不应成为上层稳定契约，可以在适配边界转换，例如：

```text
VendorSdkTimeoutException
        ↓
FaceRecognitionUnavailableException
```

转换的目标是让上层依赖项目自己的稳定语义，而不是供应商：

```text
异常类
状态码
Response 对象
SDK 类型
```

但不要把所有第三方异常都无差别包装成一个无法判断原因的通用异常。

原则：

> Client / Adapter 隔离技术协议；异常转换必须对应真实抽象变化。

---

## 4. Manager

Manager 与 Service 同进程时：

* 能恢复的异常可以在职责范围内处理；
* 抽象语义确实变化时可以转换；
* 无法处理时继续传播；
* 不因为“经过 Manager”就重复打印同一堆栈。

禁止机械：

```text
RuntimeException
    ↓
ManagerException
    ↓
ServiceException
```

如果 Manager 独立部署成为远程服务，它已经是独立应用边界，应按照独立服务的日志和 API / RPC 错误契约处理。

原则：

> 同进程 Manager 不制造无价值异常层级；独立部署时按独立应用边界处理。

---

## 5. Service / 应用边界

Service 最了解当前业务用例，因此是补充业务失败上下文的重要位置。

非预期失败最终应能够关联必要且安全的信息，例如：

```text
业务动作
资源标识
关键业务编号
当前处理阶段
TraceId / RequestId（项目存在时）
```

但这不意味着每个 Service 方法必须：

```java
catch (Exception ex) {
    log.error(..., ex);
    throw ex;
}
```

如果项目已经由统一异常处理器、AOP 或应用边界集中记录未处理异常，并且可以获得足够上下文，则 Service 不应重复记录相同堆栈。

可预期业务失败，例如资源不存在、状态不允许、权限不足，也不应机械按系统故障记录为 `error`。

日志级别和记录方式以目标项目已有日志规范为准。

原则：

> 在最了解业务上下文且能够避免重复的位置保留一次足够的失败现场。

---

## 6. Web / API 边界

异常不得以 Java 堆栈、内部异常类型、SQL、服务器路径或第三方技术细节直接暴露给客户端。

在 Spring MVC 中，“Web 层收口异常”不等于每个 Controller 手写 `try/catch`。

优先复用项目已有：

```text
@RestControllerAdvice
@ExceptionHandler
HandlerExceptionResolver
统一错误页面机制
```

普通 Controller 可以让异常继续传播到应用内部统一 Web 异常处理器；关键要求是异常必须在 HTTP 边界内被转换为稳定、安全的错误契约。

REST / Open API 通常需要稳定表达：

```text
错误码
错误信息
必要的请求追踪标识
```

具体结构和 `ApiResponse<T>` 等统一响应约定由 API 规范及目标项目已有契约决定。

原则：

> 异常可以在应用内部继续传播，但不能以内部技术异常的形式越过对外协议边界。

---

## 7. 异常转换必须保留根因

跨抽象边界转换异常时必须保留原始 cause。

推荐：

```java
throw new StorageAccessException(
        "Failed to load attachment",
        ex);
```

避免丢失根因：

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
    ↓
Manager log.error
    ↓
Service log.error
    ↓
ControllerAdvice log.error
```

造成重复堆栈、重复告警和日志噪声。

推荐思路：

```text
Mapper / Client / Manager
→ 必要时转换，不重复打印

Service / 应用边界
→ 补充业务上下文

统一异常处理器
→ 如果项目在这里集中记录，则上游不重复打印

Web / API
→ 转换为安全错误响应
```

最终记录位置以目标项目日志架构为准。

---

## 9. 敏感信息

异常和日志上下文不得直接记录或返回：

* 密码；
* Token；
* Cookie / Session 凭证；
* 私钥 / Secret；
* 完整身份证件；
* 生物特征；
* 未脱敏的敏感个人信息；
* 第三方认证凭证。

排查信息应使用必要的安全标识和摘要，不通过完整对象 `toString()` 暴露敏感字段。

---

## 10. 禁止事项

禁止吞异常：

```java
catch (Exception ex) {
}
```

禁止把失败伪装成成功或普通空值：

```java
catch (Exception ex) {
    log.error("failed", ex);
    return null;
}
```

禁止为了形式统一：

```text
每层 catch
每层 log
每层 wrap
```

禁止在没有真实抽象边界时平行创建：

```text
DAOException
ManagerException
ServiceException
```

禁止为了隐藏异常破坏：

* 事务回滚；
* API 错误语义；
* 调用方失败判断；
* 原始异常 cause。

---

## 11. Codex 检查

涉及异常处理时检查：

1. 当前层是否真的拥有恢复、转换、补充上下文、记录或对外表达职责。
2. 是否只是机械 `catch + log + throw`。
3. 异常转换是否对应真实抽象边界并保留 cause。
4. 同一异常链是否重复记录完整堆栈。
5. Mapper / Client 是否把底层技术异常无必要地泄漏到业务层。
6. Manager 是否制造无意义的异常包装层级。
7. Service / 应用边界是否能够保留足够业务上下文。
8. Web / API 是否通过统一机制收口，而不是每个 Controller 重复 `try/catch`。
9. 对外错误是否泄漏 SQL、堆栈、内部类型、路径或敏感信息。
10. catch 后是否通过返回 `null`、默认值等方式改变失败语义。
11. 是否影响事务回滚或调用方原有异常契约。

最终原则：

> 底层负责必要的技术异常隔离，业务边界补充真实业务语义，应用边界记录一次足够的失败现场，对外协议边界转换为安全稳定的错误契约。
