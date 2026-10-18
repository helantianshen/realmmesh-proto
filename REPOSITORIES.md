# 仓库组织与手动版本管理

状态：组织方案已确认。日期：2026-10-18。

## 1. 四个独立仓库

| 仓库 | 技术栈 | 职责 |
| --- | --- | --- |
| [realmmesh-server](https://github.com/helantianshen/realmmesh-server) | Go | 中心服务、HTTP / RPC、身份认证、资产管理与数据库 |
| [realmmesh-mod](https://github.com/helantianshen/realmmesh-mod) | Java | Minecraft 客户端与游戏服务端 Mod、游戏事件、物品操作和 RPC 客户端 |
| [realmmesh-web](https://github.com/helantianshen/realmmesh-web) | TypeScript | 玩家资产管理、管理后台及 HTTP API 调用 |
| [realmmesh-proto](https://github.com/helantianshen/realmmesh-proto) | Protobuf | RPC 接口与数据结构的唯一源定义 |

本地四仓库并列位于 `/home/helan/mcmod/RealmMesh`。父目录只是工作区容器，不是第五个仓库。跨仓库需求与计划目前集中在协议仓库，具体实现文档以后按归属放入各仓库。

当前远程地址使用 `helantianshen` 所有者。统一放入 GitHub Organization 是可选管理方式，不是技术依赖；本次不创建 Organization 或迁移仓库。

### Mod 仓库内部组织（已确认）

`realmmesh-mod` 采用 [MultiLoader-Template 的 1.21.1 分支](https://github.com/jaredlll08/MultiLoader-Template/tree/1.21.1) 所示的 Gradle 多子项目组织，不额外拆分加载器仓库，也不以 Architectury API 为基础依赖。

| 子项目 | 职责与首期范围 |
| --- | --- |
| `common` | 共享业务逻辑、协议调用和可复用的 Minecraft 代码；不直接引用加载器专属 API |
| `neoforge` | NeoForge 入口与平台适配；Minecraft 1.21.1 为首期开发和测试目标 |
| `fabric` | 预留后续 Fabric 入口与平台适配；首期不开发和验收 |
| `forge` | 预留后续 Forge 入口与平台适配；首期不开发和验收 |

当前已创建并启用 `common` 与 `neoforge`，完成 NeoForge 1.21.1 构建、客户端主菜单及独立服务端加载验证。`fabric` 与 `forge` 仅包含预留说明，未加入 Gradle。两端预留不代表已经完成构建、运行或兼容性验证。首期验收限定 NeoForge 1.21.1；后续每接入一个加载器，分别验证构建、游戏运行、依赖打包及物品数据兼容性。共享源码不意味着三个加载器使用同一个 JAR，也不保证物品数据能够跨加载器直接恢复。

## 2. 协议共享

- `realmmesh-proto` 是唯一协议源，不向 Go、Java 仓库手工复制 `.proto`。
- 后续 `realmmesh-server/proto/` 与 `realmmesh-mod/proto/` 分别作为 Git Submodule 引用该仓库。
- Submodule 固定具体 Git 提交；协议仓库的新提交不会让消费者自动升级。Tag 用于标识人工选择的版本，消费者实际记录的是提交引用。
- Go、Java 仓库使用各自构建工具生成本语言代码，不在协议仓库集中维护两份生成代码，也不跨仓库复制生成结果。
- `realmmesh-web` 使用 HTTP/JSON，无需引用协议仓库。中心使用 Kratos，HTTP 路由和请求结构手写，HTTP 接口文档由中心服务维护，具体描述格式留待开发阶段确定。
- 生成代码纳入各消费者仓库的 Git。Go 生成工具固定为 `protoc 36.1`、`protoc-gen-go v1.36.11` 和 `protoc-gen-go-grpc v1.6.2`，使用 `paths=source_relative`；生成命令由服务仓库维护。Java 生成工具版本和命令在接入 Mod 时确定。

### RPC 契约组织方向（已确认）

按业务能力组织，只在出现真实复用时提取公共类型，不镜像数据库实体。当前职责为 `auth`（注册登录）、`audit`（游戏侧事件）、`asset`（资产能力）、`transfer`（物品转移）；不建立原先候选的 `session` 授权体系。具体职责见 [认证流程与协议职责](AUTHENTICATION.md)。这些是文档中的组织方向，尚未创建协议目录或文件。

协议命名空间使用 `v1` 等标识一代契约，与仓库 Git Tag 独立演进；兼容字段扩展不自动要求新命名空间。最小通信验证协议为 `probe/v1/probe.proto`，命名空间为 `realmmesh.probe.v1`；业务协议包名和生成选项在定义对应协议时确定。

## 3. 手动升级流程

以下是后续开发流程说明，本次未执行其中任何提交、Tag 或远程写入操作。

1. 在协议仓库修改定义并验证，形成可供消费者使用的协议提交。
2. 按独立协议版本发布 Git Tag，例如 `v0.2.0`；示例不代表已有版本。
3. 在 Go 仓库手动将 Submodule 更新到选定 Tag 对应的提交，生成 Go 代码，完成构建及相关测试，并记录协议引用变更。
4. 在 Java 仓库执行同样的手动选择、生成与验证，单独记录协议引用变更。
5. 验证预期部署组合的互通性，再分别发布消费者版本；不能仅因双方使用同一协议版本就认为已完成运行时验证。

消费者无需同时升级或使用相同应用版本。例如中心服务 `v0.3.0`、Mod `v0.1.5` 均可依赖协议 `v0.2.0`；这些均为说明性示例。实际可互通范围需要测试确认。

克隆消费者仓库时需同时获取其固定的 Submodule 提交。具体命令将在真正接入 Submodule 时记录，当前不创建占位目录。

## 4. 范围边界

- GitHub Multi-repo，独立构建与发布，手动管理版本。
- 不引入 Buf Schema Registry 等额外协议托管平台，不自行搭建制品仓库。
- 当前生成流程按 Protobuf 编译器及语言插件规划，不把 Buf 工具作为必需依赖。
- 不额外拆分 SDK、公共库或基础设施仓库。
- GitHub Actions、自动发布、自动协议兼容检查在出现实际需要时再讨论，不作为初始开发的前置条件。
- 手动流程仍需基本兼容性验证；版本命名与发布工具不替代真实测试。
