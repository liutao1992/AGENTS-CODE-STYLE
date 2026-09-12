# 业务规则与用例边界

本文档借鉴《架构整洁之道》对 Business Rules 的划分，但按当前 Skill Pack 的 Spring Boot 分层方式落地。

本文负责回答：

> 一条业务规则应该属于稳定的核心业务规则，还是具体应用用例流程；什么时候值得把规则收口到有行为的业务对象，什么时候继续由 Service / Manager 编排；输入输出模型是否真的需要再隔一层。

相关规范：

- [应用分层与模型边界](layering.md)
- [项目与业务模块目录](project-structure.md)
- [Java 编码](../coding/java.md)
- [事务](transactions.md)

核心原则：

> 业务逻辑本身也有层级。越不依赖 HTTP、数据库、框架和具体入口，且越能跨用例长期保持稳定的规则，越接近核心业务规则；具体应用流程负责组织这些规则和外部能力。

> 借鉴的是职责判断，不是目录模板。不得为了套用 Clean Architecture 机械创建 `Entity`、`UseCase`、`Repository`、`Command`、`Result` 或额外转换层。

---

## 1. 先区分两类业务规则

### 1.1 核心业务规则

如果更换：

```text
HTTP → RPC
Web → 自动设备
MyBatis → Rabbit-SQL
PostgreSQL → 其他持久化方式
```

规则仍然成立，它通常更接近业务本身。

例如：

```text
只有待入库案件才能执行入库
已归档案件不能再次入库
柜位没有剩余容量时不能继续占用
贷款利息按明确业务公式计算
```

这类规则往往与某个业务概念的状态、不变量、计算或允许的行为高度相关。

### 1.2 应用用例规则

如果规则描述的是：

```text
当前系统为了完成一次用户目标
需要按什么顺序读取数据
调用哪些业务能力
写入哪些记录
发送哪些通知
返回什么结果
```

它更接近应用用例流程。

例如“案件入库”可能需要：

```text
查询案件
→ 查询柜位
→ 检查当前柜位
→ 执行案件入库规则
→ 占用柜位
→ 创建入库记录
→ 持久化
→ 返回结果
```

原则：

> 核心业务规则回答“业务对象自己必须始终遵守什么”；应用用例回答“当前系统为了完成这个业务目标，需要组织哪些步骤”。

---

## 2. 当前 Skill 中 Service 默认承担 Use Case 角色

本 Skill Pack 不要求新增 `*UseCase` 类。

默认仍然允许：

```text
Controller / 其他入站适配器
        ↓
      Service
        ↓
Manager / Mapper / Client
```

其中 Service 本身就是应用用例边界：

```text
Service
→ 表达当前业务目标
→ 执行业务校验
→ 组织核心业务规则
→ 协调 Mapper / Client / Manager
→ 决定当前用例的事务一致性范围
```

只有在目标项目已经采用 Use Case / Application Service 风格，或者单个用例已经形成能够独立命名、独立变化和独立测试的稳定职责时，才评估专门的：

```text
StoreCaseUseCase
ApproveCaseUseCase
```

不要只做：

```text
PlaceService.store(...)
↓ 改名
StorePlaceUseCase.execute(...)
```

但职责和依赖完全没变。

原则：

> `Use Case` 在本 Skill 中首先是一种职责，不默认是一种类名。

---

## 3. 稳定不变量可以收口到有行为的业务对象

如果同一条稳定业务规则与对象状态强相关，并且在多个用例中都必须成立，可以评估把它收口为行为，而不是让调用者重复：

```java
if (caseInfo.getStatus() != CaseStatus.PENDING_STORAGE) {
    throw new BusinessException("当前案件不能入库");
}
caseInfo.setCabinetId(cabinetId);
caseInfo.setStatus(CaseStatus.STORED);
```

如果项目已经存在独立的行为业务模型，更清晰的表达可能是：

```java
caseInfo.store(cabinetId);
```

适合收口的典型规则：

```text
状态迁移
对象自身不变量
稳定业务计算
对象在任何入口下都必须遵守的行为约束
```

但只有真实收益时才这样做。

以下情况不因为“富领域模型更好”就创建新对象：

* 只是简单 CRUD；
* 规则只在一个很小的用例中出现；
* 新增业务对象会产生大量无意义 DO ↔ Domain 转换；
* 目标项目没有独立领域模型且当前结构清晰；
* 所谓行为只是 getter / setter 的重新包装；
* 无法从需求、测试或稳定代码确认这条不变量真实存在。

原则：

> 优先消除真实重复和可绕过的不变量，不以“消灭贫血模型”为目标制造新的模型层。

---

## 4. Clean Architecture 的 Entity 不等于当前持久化 DO

这里必须区分概念。

Clean Architecture 中讨论的 Entity 更接近：

```text
关键业务数据
+
关键业务规则
```

而当前 Skill Pack 的：

```text
*DO
<module>.domain
```

默认仍然表示**数据库持久化模型**。

因此不得因为本文出现 `Entity` 概念，就机械执行：

```text
给所有 DO 加业务方法
把 <module>.domain 自动解释成 Clean Architecture Entity
把每张表包装成一个富领域对象
新增 Repository 再包装现有 Mapper
```

如果项目确实采用独立领域模型，可以存在：

```text
持久化 DO
↔
行为业务对象
```

但转换必须有职责差异和真实价值。

如果没有独立领域模型，稳定业务规则继续放在职责正确的 Service / Manager 中，也优于为了架构形式增加一层无价值映射。

原则：

> `DO` 是持久化职责；Clean Architecture `Entity` 是业务规则职责。名字相似或 Package 叫 `domain` 都不能证明二者等价。

---

## 5. 核心业务对象不负责 I/O 和应用流程

即使项目采用行为业务对象，也不要把所有逻辑塞进去。

核心业务对象可以表达：

```text
store
archive
borrow
approveTransition
calculateAmount
ensureAvailable
```

但不应自己负责：

```text
查数据库
保存自己
开启事务
发送 HTTP / RPC
调用第三方 SDK
写消息队列
发通知
读取当前 Web 用户
构造 HTTP Response
```

避免：

```java
caseInfo.store();
caseInfo.saveDatabase();
caseInfo.sendMessage();
caseInfo.notifyPolice();
```

这些外部协作由 Service / Manager / Mapper / Client 等边界承担。

原则：

> 业务对象维护自身规则；应用层组织对象之间以及对象与外部系统之间的协作。

---

## 6. 依赖方向保持从应用流程指向核心规则

如果项目存在独立行为业务对象，推荐依赖关系是：

```text
Controller / Consumer / RPC
          ↓
       Service
   （应用用例）
          ↓
   核心业务对象
```

同时 Service 还可以依赖：

```text
Manager
Mapper / DAO
Client / Adapter
```

核心业务对象原则上不反向依赖：

```text
Controller
Service / UseCase
Spring MVC
MyBatis / MyBatis-Plus
Rabbit-SQL
PostgreSQL
第三方 SDK
HTTP Request / Response
```

如果一个所谓“核心业务对象”必须知道 SQL、当前 Controller、Spring Bean 或 Vendor Response 才能工作，说明技术细节已经反向进入业务核心，应重新判断边界。

---

## 7. Service 应该读起来像业务流程，而不是数据库脚本

对于复杂用例，Service 高层代码优先表达：

```text
加载必要业务对象
→ 检查当前用例需要的业务条件
→ 调用稳定业务行为
→ 协调其他对象或外部能力
→ 持久化结果
→ 形成业务输出
```

例如概念上：

```java
CaseDO caseDO = caseMapper.getById(caseId);
CabinetDO cabinetDO = cabinetMapper.getById(cabinetId);

// 这里根据项目是否有独立行为模型，选择调用业务行为或在应用层执行规则。

storageRecordMapper.insert(record);
```

重点不是必须创建 `Case` / `Cabinet` Entity，而是不要让 Service 退化成：

```text
查表
→ if 魔法状态
→ set 字段
→ update
→ 再查表
→ 再拼协议对象
```

并把真正的业务语义隐藏在数据库字段操作里。

---

## 8. 需要数据库当前状态的规则仍由应用层协调

不是所有“业务规则”都适合塞进单个对象。

例如：

```text
编码是否唯一
当前用户是否有权限
柜位当前剩余容量是否足够
数据库中是否已有未完成记录
两个对象是否必须共同更新
```

这些规则依赖：

```text
数据库当前状态
其他对象
调用者上下文
事务一致性
```

应由 Service / Manager 在正确事务边界内协调，再调用对象自身规则。

不要为了让 Entity “纯粹”而让它自己访问 Mapper / Repository / Client。

事务范围继续读取 `transactions.md`。

---

## 9. 输入输出模型按语义隔离，不按层数机械复制

Clean Architecture 强调应用用例输入输出与外部协议模型解耦，这个方向可以借鉴，但当前 Skill 不要求固定链路：

```text
Request
→ Command
→ DTO
→ UseCase
→ Result
→ VO
```

是否新增应用输入 / 输出模型，先判断有没有真实语义差异。

适合独立模型的情况：

* HTTP / RPC / Message 多入口需要复用同一个应用用例；
* 外部 Request 带有协议字段，但应用层不应感知；
* 服务端可信上下文不能由客户端 Request 提供；
* 一个用例需要稳定的内部输入契约，且与当前接口模型明显不同；
* 对外 VO 与核心业务对象包含的数据和生命周期明显不同。

不需要额外转换层的情况：

* Request / Query 已经只是清晰的业务输入数据，没有协议对象泄漏；
* 只有一个入口，新增 Command 与 Request 字段和语义完全相同；
* Result 与 VO 只是同字段复制；
* 转换只增加样板代码，没有隔离任何变化。

具体业务输出仍优先使用 VO；DO 和核心业务对象都不直接作为公共 HTTP 输出。

原则：

> 边界模型由语义差异产生，不由“每经过一层就必须换一个对象”产生。

---

## 10. 规则应该放哪里的判断流程

遇到业务判断时按顺序问：

```text
这条规则在更换 HTTP / UI / 数据库实现后仍然成立吗？
        ↓
否 → 更可能是协议、应用流程或技术规则
        ↓
是
        ↓
它是否是某个业务概念自身的状态迁移、不变量或稳定计算？
        ↓
是 → 如果存在真实复用 / 防绕过价值，评估收口到行为业务对象
        ↓
否
        ↓
它是否依赖数据库当前状态、权限、多个对象、外部系统或事务？
        ↓
是 → Service / Manager 协调
        ↓
否 → 结合当前项目已有职责放置，不为了分类新建层级
```

再检查：

```text
这条规则是否已经在多个入口 / 用例重复？
是否有人可以绕过它直接 set 状态？
提取后是否让业务意图更清晰？
是否会因此引入无价值映射和额外层？
```

---

## 11. 不把“贫血模型”本身当成缺陷

以下代码形态不能单独证明存在问题：

```text
DO 只有字段
Service 中存在业务判断
没有 Entity 类
没有 UseCase 类
没有 Repository 接口
```

只有出现具体风险时才需要调整，例如：

```text
同一状态不变量在多个 Service 重复实现且已经出现语义漂移
多个调用者可以绕过关键状态检查直接修改对象
一个 Service 同时承担业务流程、协议转换、SQL、第三方 SDK 和状态规则
核心业务对象反向依赖 Spring / Mapper / Client
外部 API 直接暴露持久化或核心业务内部模型，导致边界被外部需求污染
```

原则：

> 评估真实职责和变化风险，不对“贫血模型 / 富领域模型”做流派式打分。

---

## 12. Codex 实施检查

涉及业务规则、状态流转或领域对象时检查：

1. 当前规则来自明确需求、现有契约、测试或稳定实现，而不是 Agent 自行发明。
2. 是否先区分核心业务规则和应用用例流程。
3. 同一稳定不变量是否已经在多个入口 / Service 重复。
4. 如果提取行为对象，是否真正封装状态迁移、不变量或稳定计算，而不是包装 setter。
5. 是否错误把持久化 DO 当成 Clean Architecture Entity。
6. Service 是否仍然表达业务用例，而不是退化成协议或 SQL 脚本。
7. 核心业务对象是否保持对 Spring、持久层框架、HTTP 和 Vendor SDK 的无感知。
8. 数据库状态、权限、多对象协作和事务规则是否仍由 Service / Manager 正确协调。
9. 是否为了形式新增 `Entity / UseCase / Repository / Command / Result` 而没有职责收益。
10. Request / Query / DTO / BO / DO / VO 转换是否都存在真实语义变化。
11. 提取核心规则后是否补充或调整了对应单元测试；应用流程变化是否覆盖相关用例测试。
12. 是否保持目标项目已有稳定结构，而不是借机全量迁移架构。

最终原则：

> 先识别最稳定的业务规则，再识别当前应用如何组织这些规则；保护业务语义免受 Web、数据库和框架细节污染，但只隔离真实变化，不机械增加架构层级。
