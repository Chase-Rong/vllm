# Spec Delta

## Purpose

在显式启用元数据后，让 KV event 消费者识别事件所描述资源是实例私有还是可由兼容实例共享，避免从 CPU / STORAGE medium、位置或 connector 名称猜测。该能力遵循与 source 相同的默认关闭开关，定义资源声明、可配置 tier 和未知值边界，不代替存储域关联与实际访问确认。

## ADDED Requirements

### Requirement: Sharing scope follows the metadata publication switch

`sharing_scope` MUST 与 source 共用默认 `false` 的 `enable_kv_event_metadata` 配置，不增加独立的 scope 发布开关。只有全局 KV events 与元数据开关均开启时，发布事件才能携带已知 scope。关闭或省略元数据开关时，store/remove MUST 完全省略 scope，而非发送 null，即使内置 producer、tier 配置或插件已提供其值。该开关 MUST NOT 改变既有事件生成条件或资源属性。

#### Scenario: Built-in declarations stay off the wire by default

- **WHEN** KV events 已启用，但元数据开关未配置，GPU / primary CPU / Mooncake Store 按既有条件产生事件
- **THEN** 发布消息省略 sharing scope 和 source，保留事件的原有数量、顺序及其他字段

#### Scenario: Tier or plugin declaration does not override explicit disable

- **WHEN** 元数据显式关闭，FS / OBJ 配置了有效的 sharing scope，或插件已提供 scope
- **THEN** 资源声明仍有效，但发布事件省略 sharing scope 和 source

#### Scenario: Opt-in publishes known scope with source

- **WHEN** KV events 与元数据开关均开启，资源已声明 scope，且其既有事件选项允许发布
- **THEN** 事件包含该资源的 scope 和对应逻辑来源；未声明 scope 的资源仅提供可确定的 source

### Requirement: Sharing scope wire format

`BlockStored` 和 `BlockRemoved` MUST 支持可选的字符串字段 `sharing_scope`，取值仅为 `instance` 或 `shared`。字段省略或为 null MUST 表示未知，不得默认解释为 `instance`。`AllBlocksCleared` MUST 不因本能力增加 sharing scope。

#### Scenario: Known scope is explicitly serialized

- **WHEN** KV events 与元数据开关均开启，producer 已知当前资源为实例私有或共享资源并产生 store/remove
- **THEN** store/remove 的 `sharing_scope` 分别序列化为 `instance` 或 `shared`

#### Scenario: Missing scope stays unknown

- **WHEN** 消费者读取未提供 sharing scope 的事件，包括旧事件、元数据关闭时的事件，或该字段为 null 的事件
- **THEN** 该资源的共享范围保持未知，不能仅凭 medium、locality 或 source 将其视为实例私有

### Requirement: Scope belongs to the reported resource

启用元数据并发布 scope 时，生产者 MUST 按事件所描述的具体资源填写 scope，而不是为整个 connector 统一赋值。同一来源中不同 tier 的 scope MUST 能分别表达。`instance` 表示发布该事件的逻辑 engine / DP 副本私有的缓存；`shared` 表示相应存储域内兼容 reader 可以共享的资源。

#### Scenario: One connector reports private and shared tiers

- **WHEN** 元数据已启用，同一 native offloading connector 同时报告 primary CPU cache 和显式配置为 shared 的 FS tier
- **THEN** 两类事件使用相同 connector source，CPU 事件的 scope 为 `instance`，FS 事件的 scope 为 `shared`

#### Scenario: Local storage can still be shared

- **WHEN** 元数据已启用，某个 tier 的 `locality` 为 `LOCAL`，且其 sharing scope 显式配置为 `shared`
- **THEN** 发布事件保留两个属性，不将 LOCAL 自动改写或推断为 instance

### Requirement: Built-in resource scope declarations

启用元数据时，原生 cache manager 的 GPU prefix cache 和内置 native offloading primary CPU cache MUST 报告 `instance`，Mooncake Store pool 的 connector 事件 MUST 报告 `shared`。这些声明 MUST 来自实际 producer / 资源契约，不能通过通用的 medium 到 scope 映射覆盖其他资源；关闭开关时声明不进入发布消息。

#### Scenario: Native GPU event reports instance scope

- **WHEN** 元数据已启用，原生 cache manager 发布 medium 为 `GPU` 的 store 或 remove
- **THEN** source 为 `{"source_id":"cache_manager"}`，sharing scope 为 `instance`

#### Scenario: Native CPU offload event reports instance scope

- **WHEN** 元数据已启用，内置 native offloading primary CPU cache 发布 medium 为 `CPU` 的 store 或 remove
- **THEN** 事件的 connector source 保持对应配置身份，sharing scope 为 `instance`

#### Scenario: Mooncake pool store reports shared scope

- **WHEN** 元数据已启用，Mooncake Store connector 成功写入并按既有条件发布 store
- **THEN** 事件保持当前 `medium="cpu"` 和原有 payload，并携带对应 connector source 及 `sharing_scope="shared"`

#### Scenario: Other CPU resources are not classified by medium alone

- **WHEN** 元数据已启用，其他 tier 或插件报告 CPU medium，但未声明该资源的共享范围
- **THEN** 框架仍提供可确定的 connector source，且不根据 CPU 标签擅自填入 sharing scope

### Requirement: Filesystem and object tier configuration

FS / OBJ tier MUST 支持可选资源属性配置 `sharing_scope`。`instance` 和 `shared` MUST 原样成为该 tier 的声明；未配置或 null MUST 保持未知，不能根据路径、bucket 或 locality 自动推断。其他取值 MUST 在初始化阶段被拒绝，无论元数据开关是否开启。有效声明的发布 MUST 同时受既有事件选项与元数据开关控制。

#### Scenario: Configured filesystem scope reaches its events

- **WHEN** 元数据已启用，FS tier 配置 `sharing_scope="shared"`，且既有全局与 tier 事件选项允许发布
- **THEN** 该 tier 发布的 store 携带 `sharing_scope="shared"`

#### Scenario: Configured object tier can be instance scoped

- **WHEN** 元数据已启用，OBJ tier 配置 `sharing_scope="instance"`，且既有全局与 tier 事件选项允许发布
- **THEN** 该 tier 发布的 store 携带 `sharing_scope="instance"`，不因对象存储类型而自动改成 shared

#### Scenario: Unspecified tier scope preserves existing configurations

- **WHEN** FS / OBJ tier 未配置 sharing scope，或配置值为 null
- **THEN** 初始化接受该配置，事件省略 sharing scope，既有启用条件和 I/O 行为保持不变

#### Scenario: Invalid tier scope fails initialization

- **WHEN** FS / OBJ tier 的 sharing scope 配置为 `local`、空字符串、布尔值或其他非允许值，无论元数据开关是否开启
- **THEN** 初始化报告清晰的配置错误，不静默转换为未知或实例私有

### Requirement: Consistent scope propagation

启用元数据时，producer 已声明的 scope MUST 经过事件转换、worker 聚合、汇合发布与 replay 后保持不变，适用于完整和 hash-only payload；已有 remove MUST 携带对应资源的相同 source 和 scope。关闭时，所有 payload 形式的发布事件 MUST 省略新字段，replay MUST 保留原发布模式。两种模式均 MUST 保留既有聚合、顺序、次数及 producer 原事件，且不得制造原先没有的 remove。

#### Scenario: Native event translation preserves scope for both payload forms

- **WHEN** 元数据已启用，已声明 scope 的 offload 资源根据既有条件产生完整 store 或 hash-only store
- **THEN** 两种 payload 都携带相同资源 scope，且既有 hash、token、medium、group、locality 和 ownership 内容保持不变

#### Scenario: Existing removal identifies the same resource declaration

- **WHEN** 元数据已启用，某个 producer 按既有规则撤回之前报告的资源
- **THEN** remove 携带该逻辑来源的 source 和该资源 scope，删除 hash 的含义及数量保持不变

#### Scenario: Disabled metadata covers full stores placeholders and removals

- **WHEN** 元数据关闭，已声明 scope 的资源按既有条件发布完整 store、hash-only store 或 remove
- **THEN** live 与 replay 中的这些事件均省略 source 和 sharing scope，replay 保留原始 payload bytes，原 payload 内容、事件次数和 producer 原事件保持不变

#### Scenario: Store-only producers retain their lifecycle behavior

- **WHEN** 某个 producer 仅发布 store，且新增 scope 已成功传播
- **THEN** 本阶段不会据此增加 remove 或宣称该流能够完整表达 backend 驻留状态

### Requirement: Shared scope requires deployment domain association

`shared` MUST 仅表示资源共享属性，不表示所有订阅者均可读取，也不标识物理副本、pool、tenant 或 namespace。第一阶段 MUST 通过部署配置关联存储域；相同 source 类型、scope 或内容 hash 不得单独作为跨域访问、合并副本或删除其他来源记录的依据。

#### Scenario: Independent pools do not become one domain

- **WHEN** 两个连接不同 pool 的实例发布相同内容 hash、相同 connector 名称和 shared scope
- **THEN** 这些字段不足以建立同一存储域或可互相读取的结论，域关联仍由部署配置提供

#### Scenario: Scope metadata is not a residency guarantee

- **WHEN** 消费者收到 shared store，但该来源不提供完整失效事件或读取前发生淘汰
- **THEN** sharing scope 本身不保证数据仍然驻留，消费者的生命周期与真实 lookup 策略保持独立
