# 项目与业务模块目录规范

本文档定义 Java 后端项目的物理目录、业务模块组织和模块内部 Package 结构。

本文负责回答：

> 一个新模块应该放在哪里、业务代码按什么维度组织、`module` / `common` 等目录应该承载什么。

类的逻辑职责、依赖方向、模型分类和具体 Package 语义统一读取：

- [应用分层与模型边界](layering.md)

Java 类本身怎么写读取：

- [Java 编码](../coding/java.md)

核心原则：

> 先按业务能力组织代码，再在业务模块内部按职责分层；目录结构服务于定位、边界和协作，不为了形式创建空 Package，也不无授权迁移已有稳定项目结构。

---

## 1. 目标项目结构优先

本规范提供的是新项目或新增模块缺少明确约定时的默认组织方式。

如果目标项目已经形成稳定结构，例如：

```text
feature/<module>
modules/<module>
business/<module>
<module>/controller
```

应优先延续当前项目，不为了套用 `module` 目录批量搬迁历史代码。

只有以下情况才评估结构调整：

* 当前任务明确要求模块化重构；
* 新模块尚未落地且项目没有统一约定；
* 现有目录已经造成明确职责混乱或依赖问题；
* 调整范围和兼容影响能够被控制。

原则：

> “推荐目录”是缺省方案，不是迁移命令。

---

## 2. 新项目优先按业务模块组织

在没有目标项目既有约定时，推荐使用业务优先的模块化组织。

示意：

```text
src/main/java/com/example/app/
├── common/                 真正跨业务复用的公共能力
├── config/                 应用级框架配置
├── module/                 业务模块
│   ├── place/
│   ├── casecenter/
│   └── equipment/
└── Application.java
```

资源仍遵循 Maven / Gradle 与项目现有结构，例如：

```text
src/main/resources/
src/test/java/
src/test/resources/
```

`module` 表达的是应用内部的业务组织边界，不意味着未来一定拆成微服务，也不应为了“以后可能拆服务”提前制造远程调用抽象。

如果项目已经使用其他清晰的业务目录名称，可以继续使用，不强制改名为 `module`。

---

## 3. 业务优先于技术优先

新项目或新业务区域缺少既有约定时，优先：

```text
module.place.controller
module.place.service
module.place.mapper

module.casecenter.controller
module.casecenter.service
module.casecenter.mapper
```

而不是先建立全局技术目录，再把所有业务散进去：

```text
controller.place
controller.casecenter
service.place
service.casecenter
mapper.place
mapper.casecenter
```

业务优先组织有利于：

* 快速定位一个业务的完整实现；
* 看清模块边界和跨模块依赖；
* 控制公共代码膨胀；
* 让相关 Controller、Service、Mapper、模型和测试保持邻近。

但目标项目已经稳定采用技术优先结构时，不为此无授权迁移。

---

## 4. `module` 内部按真实职责组织

一个业务模块可以按实际需要组织为：

```text
module/place/
├── controller/             HTTP 入站适配
├── service/                业务用例和流程
├── manager/                可选：可复用应用能力 / 原子组合
├── mapper/                 数据库访问
├── client/                 可选：外部技术调用
├── adapter/                可选：外部协议适配
├── request/                接口输入
├── query/                  查询条件
├── dto/                    可选：内部数据传输
├── bo/                     可选：业务处理中间对象
├── domain/                 持久化 DO
└── vo/                     具体业务输出
```

这只是职责地图，不要求每个模块都创建全部目录。

简单模块完全可以只有：

```text
place/
├── controller/
├── service/
├── mapper/
├── request/
├── domain/
└── vo/
```

只有出现真实职责时才增加：

```text
manager
dto
bo
client
adapter
query
```

禁止为了“目录完整”预先创建大量空 Package 或无职责的占位类。

模型职责的唯一详细事实来源是 `layering.md`，不得把 Request、DTO、BO、DO、VO 全部机械塞入一个泛化 `domain` 或 `dto` 目录。

---

## 5. `common` 不是公共垃圾桶

`common` 只存放真正满足以下条件的能力：

* 与具体业务模块无关；
* 至少存在明确跨模块复用价值；
* 语义稳定；
* 不依赖某个业务模块的内部实现。

适合的例子可能包括：

```text
common.mybatis.handler
common.mybatis.interceptor
common.json
common.validation
common.web
```

具体名称继续遵循目标项目。

不应仅因为“多个地方都能调用”就把业务代码移入：

```text
common
util
shared
```

例如场所审核规则即使被多个入口使用，也仍属于场所业务能力，不应变成 `common.util.PlaceAuditUtils`。

原则：

> 公共代码按稳定跨模块职责提取，不按“看起来能复用”提取。

---

## 6. 不机械建立全局 `constant` / `util` / `handler` / `third`

项目根目录不强制存在：

```text
constant
util
handler
interceptor
listener
third
```

这类目录必须由真实职责决定。

例如：

```text
MyBatis TypeHandler
→ common.mybatis.handler

Web Interceptor
→ common.web.interceptor 或项目既有 Web 基础设施 Package

Place 业务常量
→ place 模块内部拥有该业务语义的位置
```

避免形成无限增长的：

```text
GlobalConstants
CommonUtils
ThirdUtils
```

目录名称不能代替职责设计。

---

## 7. 第三方集成按边界归属，不统一塞入 `third`

模块专属的外部能力优先跟随业务模块，例如：

```text
module.place.client.FaceRecognitionClient
module.place.adapter.FaceRecognitionAdapter
```

真正跨模块共享的外部基础设施，可以按照项目约定进入稳定公共边界，例如：

```text
common.storage
common.integration
common.client
```

但不要因为依赖来自第三方，就把所有 SDK、HTTP Client、Redis、OSS、消息代码机械放进一个巨大 `third` Package。

技术适配和 Manager 的职责区别读取：

- [layering.md](layering.md#5-manager-层)

原则：

> 外部依赖按“谁拥有这项技术能力、是否跨模块复用”归属，不按“是不是第三方”统一归档。

---

## 8. 模块内 MVC / 应用分层保持单向

业务模块目录最终仍服从逻辑分层：

```text
Controller / 其他入站适配器
        ↓
      Service
        ↓
    Manager（按需）
      ↙       ↘
   Mapper    Client / Adapter
```

目录结构不能成为绕过依赖规则的理由。

例如即使都位于：

```text
module.place
```

也禁止：

```text
Controller → Mapper
Mapper → Service
Client → Service
```

详细职责读取：

- [layering.md](layering.md)

---

## 9. 跨模块调用不穿透数据访问层

业务模块之间优先通过对方稳定的 Service / Facade 能力协作。

推荐：

```text
module.casecenter.CaseService
        ↓
module.place.PlaceService / PlaceFacade
```

避免：

```text
CaseService
    ↓
PlaceMapper
```

`module` 目录的意义之一就是让跨模块依赖更容易识别，而不是把所有代码放到同一 JVM 后任意穿透调用。

---

## 10. 新建模块 / Package 前的判断流程

新增目录或类前按顺序判断：

```text
当前项目是否已有同类结构？
        ↓
属于哪个业务模块？
        ↓
当前类是什么职责？
        ↓
该职责是否已经存在实现？
        ↓
应该进入哪个 Package？
        ↓
是否真的需要新建目录 / 类？
```

不要反过来：

```text
先创建 controller/service/manager/mapper 全套目录
        ↓
再想办法往里面填类
```

---

## 11. Codex 目录结构检查

涉及新模块、新 Package 或大范围移动代码时检查：

1. 是否先查看目标项目已有目录和类似业务模块。
2. 是否无授权把已有项目强制迁移到 `module` 结构。
3. 新业务是否优先保持业务内聚，而不是散落到多个全局技术目录。
4. `module` 内是否只创建真实需要的职责 Package。
5. 是否把所有模型机械放入 `domain` / `dto`。
6. `common` 是否出现具体业务语义或成为公共垃圾桶。
7. 是否无依据创建巨大 `util`、`constant`、`third` 等兜底目录。
8. 第三方集成是否放在正确的 Client / Adapter 或公共技术边界。
9. 跨模块调用是否穿透到其他模块 Mapper。
10. Package 是否由类的真实职责决定，而不是由当前文件位置或调用方便决定。

最终原则：

> 业务模块负责内聚业务，职责 Package 负责表达边界，`common` 只承载真实公共能力；结构清晰比目录数量多更重要。
