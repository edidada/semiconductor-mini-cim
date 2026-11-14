# **Java 晶圆厂 Mini-CIM 系统：Maven 多 Module 架构规划**

建议采用 **Java 21 + Spring Boot 3.x + Maven Multi-Module + DDD + 事件驱动架构**，按照半导体制造领域划分模块，而不是简单按照 Controller、Service、Mapper 分模块。

目标是建设一个可以实际运行、压测、故障注入、逐步演进到分布式架构的 Mini-CIM 系统，覆盖晶圆制造中的 Lot/Wafer、Route、Track-In/Out、Hold/Release、EAP、Recipe、生产履历等关键能力。

## **一、整体架构**

**客户端 / 外部系统**

Web UI · REST API · ERP · APS · QMS

**API Gateway / IAM**

认证 · 鉴权 · 路由 · 审计 · 限流

**MES 核心领域服务**

Lot Service

Wafer Service

Route Service

Track Service

Hold Service

History Service

**EAP 设备自动化**

SECS/GEM · HSMS · Equipment

**Recipe 管理**

版本 · 审批 · 校验 · 下发

**基础设施**

PostgreSQL · Redis · Kafka · MinIO · OpenTelemetry

这里的 Service 表示逻辑服务边界，不要求第一天就部署成独立微服务。

**建议第一阶段采用模块化单体（Modular Monolith）**：所有核心 MES 领域模块运行在一个 Spring Boot 进程中，保证 Lot 状态变更、Track-In/Out 和履历写入的本地事务一致性。EAP 通信可以单独进程部署。

## **二、推荐 Maven Multi-Module 目录**

仓库名称建议：`semiconductor-mini-cim`

```
semiconductor-mini-cim/
│
├── pom.xml                         # Maven 根聚合 POM
├── README.md
├── .mvn/
├── .github/workflows/
│
├── cim-bom/                         # 依赖版本管理
├── cim-common/                      # 公共基础模块
│   ├── common-core/
│   ├── common-web/
│   ├── common-security/
│   ├── common-persistence/
│   ├── common-messaging/
│   ├── common-observability/
│   └── common-test/
│
├── cim-contracts/                   # 跨模块公开契约
│   ├── mes-contract/
│   ├── equipment-contract/
│   └── integration-events/
│
├── cim-mes/                         # MES 核心
│   ├── mes-lot/
│   ├── mes-wafer/
│   ├── mes-route/
│   ├── mes-track/
│   ├── mes-hold/
│   ├── mes-history/
│   └── mes-integration/
│
├── cim-equipment/                   # 设备管理
│   ├── equipment-domain/
│   ├── equipment-application/
│   └── equipment-infrastructure/
│
├── cim-eap/                         # Equipment Automation Program
│   ├── eap-core/
│   ├── eap-secs-gem/
│   ├── eap-hsms/
│   ├── eap-simulator/
│   └── eap-adapter/
│
├── cim-recipe/                      # Recipe Management
│   ├── recipe-domain/
│   ├── recipe-application/
│   └── recipe-infrastructure/
│
├── cim-integration/                 # 外部系统集成
│   ├── integration-erp/
│   ├── integration-aps/
│   └── integration-qms/
│
├── cim-apps/                        # 可执行应用
│   ├── mes-server/
│   ├── eap-server/
│   └── simulator-server/
│
├── cim-benchmark/                   # 性能测试
│   ├── benchmark-jmh/
│   └── benchmark-load/
│
├── cim-tests/                       # 集成/系统测试
│   ├── integration-tests/
│   ├── contract-tests/
│   ├── e2e-tests/
│   └── chaos-tests/
│
├── deploy/
│   ├── docker/
│   ├── docker-compose/
│   ├── kubernetes/
│   └── helm/
│
└── docs/
    ├── architecture/
    ├── adr/
    ├── domain-model/
    ├── api/
    ├── database/
    ├── equipment/
    ├── consistency/
    ├── disaster-recovery/
    └── performance/
```

以上是目标结构，约 30 多个 Maven 子模块。初期不建议全部创建空模块，可以按业务阶段逐步增加。

## **三、核心模块职责与依赖**

| Maven Module         | 核心职责                    | 关键实体                    |
| -------------------- | ----------------------- | ----------------------- |
| `mes-lot`            | Lot 创建、拆批、合批、状态管理       | Lot, LotGenealogy       |
| `mes-wafer`          | Wafer ID、Slot、所属 Lot、状态 | Wafer, WaferPosition    |
| `mes-route`          | 工艺路线、步骤、版本、条件           | Route, RouteStep        |
| `mes-track`          | Track-In、Track-Out、工序执行 | TrackSession, Operation |
| `mes-hold`           | Hold、Release、原因码、审批     | HoldRecord              |
| `mes-history`        | 不可变生产履历、事件查询            | HistoryEvent            |
| `equipment-domain`   | 设备、腔体、端口、状态             | Equipment, Chamber      |
| `eap-core`           | 设备命令、会话、消息分发            | EquipmentCommand        |
| `eap-hsms`           | HSMS 连接与协议帧             | HsmsSession             |
| `eap-secs-gem`       | SECS-II/GEM 消息、事件       | SecsMessage             |
| `recipe-domain`      | Recipe 元数据、版本、审批        | Recipe, RecipeVersion   |
| `recipe-application` | Recipe 校验、下载、激活         | RecipeDeployment        |

### **关键业务规则**

- Lot 可以包含多个 Wafer，Wafer 不能同时归属多个活动 Lot。
- Route 发布后不可直接修改，应通过版本演进。
- Track-In 必须验证 Lot 状态、当前 RouteStep、设备能力、Recipe 适配及 Hold 状态。
- Track-Out 必须匹配有效 TrackSession，验证完成条件并推进工艺步骤。
- Hold 需要记录作用范围（Lot/Wafer/Equipment）、原因和操作人；Release 必须检查全部有效阻断条件。
- 所有关键状态变更都应生成不可变履历事件。

其中，**Track-In/Out 应由&#x20;**`mes-track`**&#x20;统一编排**，不应允许多个服务绕过业务规则直接更新 Lot 的当前工序。

## **四、每个领域模块内部的 DDD 结构**

不建议在所有模块里再拆出 `domain`、`application`、`infrastructure` 三个独立 Maven 工程，否则会产生大量构建与依赖维护成本。

例如 `mes-lot`：

```
mes-lot/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/com/example/cim/mes/lot/
    │   │   ├── api/
    │   │   │   ├── LotFacade.java
    │   │   │   └── LotView.java
    │   │   ├── application/
    │   │   │   ├── LotApplicationService.java
    │   │   │   └── command/
    │   │   │       ├── CreateLotCommand.java
    │   │   │       └── SplitLotCommand.java
    │   │   ├── domain/
    │   │   │   ├── model/
    │   │   │   │   ├── Lot.java
    │   │   │   │   ├── LotId.java
    │   │   │   │   └── LotStatus.java
    │   │   │   ├── repository/
    │   │   │   │   └── LotRepository.java
    │   │   │   ├── service/
    │   │   │   └── event/
    │   │   ├── infrastructure/
    │   │   │   └── persistence/
    │   │   └── interfaces/
    │   │       └── rest/
    │   │           └── LotController.java
    │   └── resources/
    │       └── db/migration/
    └── test/
        └── java/
```

`api` 是对其他业务模块公开的 Java 接口，`interfaces/rest` 是对外 HTTP 接口。两个边界应分开。

## **五、Maven 依赖关系设计**

**mes-server**

Spring Boot 启动与装配

mes-lot

mes-route

mes-track

mes-hold

mes-wafer

mes-history

**cim-contracts / cim-common**

公开接口、事件契约、基础类型

建议制定以下依赖规则：

1. `cim-common` 不依赖任何业务模块。
2. `cim-contracts` 不依赖业务实现。
3. 领域模块只依赖必要的公共契约和明确公开的领域 API。
4. `mes-track` 可以依赖 `mes-lot`、`mes-route`、`mes-hold` 的公开接口，但不能访问它们的 Repository。
5. `mes-server` 作为 Composition Root，负责 Spring Bean 装配。
6. EAP 与 MES 之间通过命令/事件契约交互，不直接共享数据库表。

注意：`cim-bom` 是依赖版本管理模块，不属于业务运行时依赖链。

## **六、Maven POM 示例**

### **根&#x20;**`pom.xml`

```
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example.cim</groupId>
    <artifactId>semiconductor-mini-cim</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <properties>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <spring-boot.version>3.5.0</spring-boot.version>
    </properties>

    <modules>
        <module>cim-bom</module>
        <module>cim-common</module>
        <module>cim-contracts</module>
        <module>cim-mes</module>
        <module>cim-equipment</module>
        <module>cim-eap</module>
        <module>cim-recipe</module>
        <module>cim-integration</module>
        <module>cim-apps</module>
        <module>cim-benchmark</module>
        <module>cim-tests</module>
    </modules>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.14.0</version>
                    <configuration>
                        <release>21</release>
                    </configuration>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
```

这里的 Spring Boot 版本是示例固定版本，并非声明其为当前最新版。完整工程还应在父 POM 中集中锁定 Surefire、Failsafe、Enforcer 等插件版本。各目录级聚合模块也需要自己的 `packaging=pom` 和 `<modules>`。

### `mes-track/pom.xml`

```
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.example.cim</groupId>
        <artifactId>semiconductor-mini-cim</artifactId>
        <version>1.0.0-SNAPSHOT</version>
        <relativePath>../../../pom.xml</relativePath>
    </parent>

    <artifactId>mes-track</artifactId>

    <dependencies>
        <dependency>
            <groupId>com.example.cim</groupId>
            <artifactId>mes-contract</artifactId>
            <version>${project.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
    </dependencies>
</project>
```

上面采用跨模块契约方式，具体 Lot、Route、Hold 的访问接口应放在 `mes-contract` 中，避免 `mes-track` 直接依赖其他模块的内部实现。

## **七、Track-In / Track-Out 核心事务设计**

Track-In 执行流程

1. 接收 TrackInCommand，验证身份、权限、请求幂等键
2. 锁定 Lot 聚合，检查状态及当前 RouteStep
3. 检查 Hold、设备状态、设备能力及 Recipe 适配
4. 创建 TrackSession，更新 Lot 为 Processing
5. 同一数据库事务写入 History 和 Outbox 事件
6. 事务提交后，由 EAP 工作流异步执行设备侧命令
7. 根据设备确认或超时结果，推进、重试或进入待人工处理状态

关键点是不能把数据库事务一直保持到设备通信完成。真实设备执行可能持续数分钟甚至数小时，应通过状态机和持久化工作流管理。

**特别注意：MES 的 Track-In 成功不等于设备已经开始加工。** 需要区分 MES 受理、设备命令已发送、设备确认、实际加工开始等状态。

## **八、数据库设计建议**

第一阶段建议使用 PostgreSQL，按业务 Schema 隔离：

| Schema        | 主要表                                       |
| ------------- | ----------------------------------------- |
| `mes_lot`     | lot, lot_genealogy                        |
| `mes_wafer`   | wafer, wafer_position                     |
| `mes_route`   | route, route_version, route_step          |
| `mes_track`   | track_session, track_operation            |
| `mes_hold`    | hold_record, release_approval             |
| `mes_history` | history_event, history_projection         |
| `equipment`   | equipment, chamber, equipment_state       |
| `recipe`      | recipe, recipe_version, recipe_deployment |
| `integration` | outbox_event, inbox_message               |

重点数据库机制包括：

- Lot 乐观锁 `version`，避免并发 Track-In。
- TrackSession 唯一约束，避免重复活动加工会话。
- Idempotency Key，保证请求重试不会重复执行。
- Outbox 保证业务变更与待发布事件原子提交。
- History 使用 append-only 模型，保留操作者、设备、工艺、Recipe、时间和关联 ID。
- 对关键业务操作记录 `occurred_at` 与 `recorded_at`，区分事件实际发生时间和系统记录时间。

## **九、分阶段实施计划**

**阶段 1**

MVP

**Lot + Wafer + Route + Track + Hold + History**

完成核心领域模型、数据库迁移、REST API、单元测试和端到端生产流程。目标是能够模拟一个 Lot 通过多道工序。

**阶段 2**

设备

**Equipment + EAP + Recipe**

构建设备模拟器、HSMS/SECS-GEM 协议适配、Recipe 校验与下发流程。增加断线重连、命令超时及状态恢复测试。

**阶段 3**

可靠性

**Kafka + Outbox + Inbox + 审计**

实现跨服务可靠事件、消费去重、失败重试、死信处理、消息积压监控和一致性恢复。

**阶段 4**

工业化

**Kubernetes + 性能压测 + 混沌测试**

验证并发 Track-In、设备断连、数据库故障、Kafka 不可用、重复消息及恢复后的履历一致性。

## **十、建议定义的工程验收指标**

以下是 **Mini-CIM 实验项目的目标值**，不是对长鑫、长江存储或其他晶圆厂生产系统真实指标的声明。

| 指标                | 初期建议目标                          |
| ----------------- | ------------------------------- |
| Track-In API P99  | ≤ 200 ms（不包含设备加工）               |
| Track-Out API P99 | ≤ 200 ms                        |
| Track-In 重复提交     | 不产生重复有效会话                       |
| 生产履历完整性           | 关键已提交状态变更 100% 有履历              |
| Lot 状态一致性         | 不出现非法状态转移                       |
| 设备断连恢复            | 可恢复或明确标记待人工处理                   |
| 消息重复投递            | 不重复产生业务效果                       |
| 故障恢复测试            | 覆盖进程崩溃、网络分区、数据库切换               |
| 压测规模              | 从 1,000 Lot、25,000 Wafer 开始逐步扩展 |

其中 P99 必须明确硬件规格、并发数、数据规模、数据库配置和压测持续时间，否则无法形成可比较的性能报告。

## **十一、我建议的第一版实际开发范围**

为了避免模块过度拆分，第一版可以只创建以下 15 个可维护的 Maven 子模块：

```
semiconductor-mini-cim/
├── pom.xml
├── cim-common/
├── cim-contracts/
├── mes-lot/
├── mes-wafer/
├── mes-route/
├── mes-track/
├── mes-hold/
├── mes-history/
├── equipment-core/
├── eap-core/
├── eap-simulator/
├── recipe-core/
├── mes-server/
├── eap-server/
└── integration-tests/
```

第一版先将这些模块直接放在根目录；等模块规模增长后，再迁移到 `cim-mes/`、`cim-eap/` 等二级聚合目录，避免一开始出现过深的 POM 层级。

**最终架构原则：领域边界先于微服务边界，业务正确性先于吞吐量，设备协议与生产执行解耦，生产履历必须可审计、可追溯。**

这套规划既可以作为 GitHub 上的半导体 CIM 开源学习项目，也适合作为逐步扩展到工业级架构设计、详细设计、数据库设计、接口规范、压测报告和故障恢复实验的工程基础。