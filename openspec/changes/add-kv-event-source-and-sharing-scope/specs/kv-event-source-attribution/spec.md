# Spec Delta

## Purpose

在部署者显式启用元数据后，为 KV event 订阅者提供原生 cache manager 和具体逻辑 connector 的来源身份；默认发布保持既有事件字段。身份覆盖配置拓扑、单次运行期稳定性和 replay，且与共享池身份、物理副本和跨重启连续性分别解释。

## ADDED Requirements

### Requirement: Explicit metadata publication opt-in

系统 MUST 通过 `--kv-events-config` 提供布尔配置 `enable_kv_event_metadata`，默认 `false`，统一控制本 change 的 `source` 和 `sharing_scope` 发布。仅当 KV events 和该开关均开启时，发布端 MUST 为原生 cache manager 与配置中的逻辑 connector 的三个事件类型提供 inline `source`。开关关闭或省略时，发布事件 MUST 完全省略这两个字段，而非发送 null；既有事件生成条件 MUST 保持不变。

#### Scenario: Existing configuration keeps the legacy event fields

- **WHEN** KV events 已启用，但配置未提供 `enable_kv_event_metadata`
- **THEN** 三个事件类型均按原有条件发布，省略 `source` 和 `sharing_scope`，保留其他字段的原有值

#### Scenario: Explicit disable suppresses producer metadata

- **WHEN** KV events 已启用且 `enable_kv_event_metadata=false`，即使 producer 或插件已为待发布事件提供 source 或 sharing scope
- **THEN** 发布事件仍完全省略这两个字段，不能通过 producer 自填值绕过开关

#### Scenario: Explicit opt-in emits attributed events

- **WHEN** KV events 已启用且 `enable_kv_event_metadata=true`，原生 cache manager 或某个逻辑 connector 产生待发布事件
- **THEN** 每条已发布事件包含对应生产者的 `source`，store/remove 同时按资源声明提供已知的 sharing scope

#### Scenario: Metadata does not enable a disabled producer

- **WHEN** `enable_kv_event_metadata=true`，但全局 KV events 被禁用，或某个 tier 的既有事件发布选项被禁用
- **THEN** 元数据开关不会使原先不产生的事件开始产生

### Requirement: Runtime metadata setting and publisher compatibility

运行时返回的 KV event 配置 MUST 准确反映 `enable_kv_event_metadata` 的实际配置值，不得在显式开启后报告默认关闭。新增开关 MUST NOT 要求既有自定义 publisher 增加构造参数；适用的框架发布路径 MUST 在交付事件给 publisher 时遵守元数据开关。

#### Scenario: Runtime configuration reflects the selected mode

- **WHEN** 已启用 KV events 的部署分别使用默认配置、显式 `false` 或显式 `true` 并查询运行时 publisher 配置
- **THEN** 返回的元数据设置分别为 `false`、`false`、`true`，且与该 publisher 接收的事件模式一致

#### Scenario: Existing custom publisher needs no new constructor parameter

- **WHEN** 部署使用按既有构造契约注册的自定义 publisher，且分别关闭或开启元数据
- **THEN** 两种配置均可完成 publisher 构造，框架交付的事件遵守对应的字段省略或标注规则

### Requirement: Source map format

启用元数据并提供 `source` 时，该 map MUST 包含非空字符串 `source_id`。配置的 connector MUST 同时提供配置中的 `connector_name`；原生 cache manager MUST 使用 `source_id="cache_manager"` 并省略 `connector_name`。本阶段 MUST NOT 增加 `kind`，且不得将 connector 类型名称解释为唯一身份或共享属性。

#### Scenario: Connector source uses its configured name

- **WHEN** 元数据已启用，一个配置名称为 `OffloadingConnector` 的逻辑 connector 发布事件
- **THEN** `source` 包含该实例的 `source_id` 和 `connector_name="OffloadingConnector"`，即使实际实现由插件或工厂映射解析

#### Scenario: Native cache manager source omits connector name

- **WHEN** 元数据已启用，原生 cache manager 发布事件
- **THEN** `source` 为 `{"source_id":"cache_manager"}`，不包含 `connector_name` 或 `kind`

### Requirement: Configuration occurrence identity

启用元数据时，每个配置中的逻辑 connector MUST 在同一 publisher、DP rank 和运行期内拥有不同的 `source_id`。单 connector MUST 使用 `connector/0`；MultiConnector 子项 MUST 使用从零开始的配置索引路径，嵌套时递归追加索引。消费者 MUST 将完整 ID 视为不透明标识。

#### Scenario: Standalone connector has a defined path

- **WHEN** 元数据已启用，publisher 配置了单独的 `MooncakeStoreConnector`
- **THEN** 该 connector 的事件使用 `source_id="connector/0"`

#### Scenario: Repeated connector types remain distinct

- **WHEN** 元数据已启用，MultiConnector 的第零项和第一项均为 `OffloadingConnector`
- **THEN** 两者分别使用 `connector/0` 与 `connector/1`，即使事件 payload 和 connector 类型名称完全相同

#### Scenario: Nested and temporarily empty children retain their positions

- **WHEN** 元数据已启用，第一层索引 1 的子项为 MultiConnector，且其索引 0 的子项本次没有事件、索引 1 的子项产生事件
- **THEN** 后者使用 `connector/1/1`，不因其他子项没有事件而重新编号

### Requirement: Identity lifetime and namespace

在启用元数据且配置拓扑固定的一次运行内，来源 ID MUST 在不同批次间保持稳定，同一逻辑来源的 worker 贡献 MUST 对应同一来源。ID MUST 结合 publisher 运行期和批次中的 DP rank 解释；它不提供全局唯一性，也不建立重启或拓扑变化前后的身份连续性。

#### Scenario: Repeated batches retain the same identity

- **WHEN** 元数据已启用，同一 publisher 运行期和 DP rank 内的同一配置生产者连续发布多个批次
- **THEN** 该生产者的 source 元数据保持一致，与 worker 身份及本次是否有事件无关

#### Scenario: Independent ranks can reuse a configuration path

- **WHEN** 元数据已启用，两个独立 publisher 或两个 DP ranks 均配置了单 connector
- **THEN** 两者均可报告 `connector/0`，完整来源上下文仍分别包含各自的 publisher 运行期与 DP rank

#### Scenario: Restart does not establish continuity

- **WHEN** publisher 在同一 endpoint 重启，或 connector 配置顺序发生变化
- **THEN** 相同路径字符串不足以证明旧来源状态继续有效，本阶段不新增用于自动检测该变化的 wire epoch

### Requirement: Producer isolation and payload preservation

无论元数据开关是否启用，发布端 MUST 保留每个逻辑生产者既有的 worker quorum、事件顺序与次数，且不得将不同来源的相同事件合并。除按开关提供或省略的新元数据外，hash、tokens、medium、group、locality、ownership 和 session 等既有内容 MUST 保持原值；生产者仍持有的原事件不得被原地改写。

#### Scenario: Equal payloads from different connectors survive publication

- **WHEN** 元数据已启用，两个逻辑 connector 各自完成既有 worker 聚合，并提供内容完全相同的 store
- **THEN** 订阅者收到两条具有不同 `source_id` 的事件，顺序与原汇合顺序一致

#### Scenario: Annotation leaves retained producer events unchanged

- **WHEN** 发布端为 producer 提供的事件增加框架来源元数据
- **THEN** 发布事件具有对应配置身份，producer 仍持有的事件及其 hash/token 列表内容保持原状

#### Scenario: Attribution does not relax worker quorum

- **WHEN** 某个逻辑 connector 的既有规则要求多个 worker 贡献，而本次贡献尚未满足，无论元数据是否启用
- **THEN** 开关不会导致提前发布；满足规则后按原有次数发布，仅在启用元数据时携带对应逻辑来源

#### Scenario: Suppression preserves events and producer state

- **WHEN** 元数据关闭，两个逻辑 connector 提供除来源或 scope 外内容相同的事件，且 producer 仍持有原事件
- **THEN** 订阅者仍按原顺序收到两条省略新字段的事件，producer 原事件及其 hash/token 列表内容保持不变

### Requirement: Clear attribution preserves the existing clear contract

启用元数据时，`AllBlocksCleared` MUST 只增加来源字段；关闭时 MUST 省略 source。两种模式下 MUST NOT 增加 `sharing_scope` 或改变现有清除范围。消费者不得把新增 source 自动解释为仅清除同 source 记录的过滤条件。

#### Scenario: Native clear identifies the emitter

- **WHEN** 元数据已启用，原生 cache manager 产生 `AllBlocksCleared`
- **THEN** 发布事件包含 `source={"source_id":"cache_manager"}`，省略 `sharing_scope`，其既有 clear 处理范围保持不变

#### Scenario: Disabled metadata preserves the legacy clear event

- **WHEN** 元数据关闭，原生 cache manager 产生 `AllBlocksCleared`
- **THEN** 发布事件省略 source 和 sharing scope，既有 clear 处理范围保持不变

### Requirement: Additive wire and replay behavior

事件 MUST 保持 tagged map 格式，现有 batch 和 ZMQ frames 的结构不因本阶段发生变化。元数据关闭时 MUST 保持新增字段之前的事件字段集合与原有值，启用时才扩展 map。新解码器 MUST 接受缺失或为 null 的 source 并解释为未知；宽松旧解码器 MUST 能读取新事件的原有字段。Replay MUST 保留原批次编码，不重新标注或删减字段。兼容性约束针对事件 schema 与内容，不承诺不同运行的完整消息 bytes 相同。

#### Scenario: Legacy event remains decodable

- **WHEN** 新解码器读取不含 source 的旧事件，或 source 为 null 的事件
- **THEN** source 被解释为未知，既有字段按原值解码

#### Scenario: Tolerant legacy decoder reads new events

- **WHEN** 忽略未知 map 字段的旧解码器读取带 source 和 sharing scope 的事件
- **THEN** 它仍能取得原有事件字段及批次信息

#### Scenario: Strict legacy decoder reads default and explicitly disabled events

- **WHEN** 符合新增元数据之前 schema 且拒绝未知字段的旧解码器读取元数据未配置或显式关闭时发布的三类事件
- **THEN** 它能读取这些事件，且不会因 source 或 sharing scope 新字段而失败

#### Scenario: Replay retains original attribution

- **WHEN** 元数据已启用，订阅者请求仍在 replay buffer 内的已发布批次
- **THEN** replay 返回该批次原始 payload bytes，包括原有 source 元数据和 DP rank，不重新分配身份

#### Scenario: Replay preserves omitted metadata

- **WHEN** 订阅者请求元数据关闭时发布且仍在 replay buffer 内的批次
- **THEN** replay 返回该批次原始 payload bytes，保持新字段省略及原有 DP rank，不补充元数据
