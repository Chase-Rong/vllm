# Proposal

## Why

外部 KV indexer 收到的 CPU KV events 无法明确区分实例私有的 native offload DRAM 与 Mooncake Store 的共享 DRAM；scheduler 和 MultiConnector 汇合事件后，也缺少具体生产者的身份。第一阶段需要同时提供逻辑来源和资源共享范围，并通过默认关闭的独立开关让部署者显式启用，避免既有严格消费者在升级 producer 后被迫接收新增字段。

## What Changes

- 为 `BlockStored`、`BlockRemoved`、`AllBlocksCleared` 增加可选的 inline `source` map：包含必需的 `source_id` 和可选的 `connector_name`，不增加 `kind`。
- 覆盖原生 cache manager 与配置中的各个逻辑 connector；重复类型、嵌套 MultiConnector 子项具有不同身份。ID 在固定配置的单次运行内保持稳定。
- 为 `BlockStored`、`BlockRemoved` 增加可选的 `sharing_scope`，取值为 `instance` 或 `shared`；省略表示未知。该字段描述具体资源，不是整个 connector 的固定属性。
- 在 `KVEventsConfig` 增加 `enable_kv_event_metadata: bool = False`，通过现有 `--kv-events-config` 配置，统一控制本 change 的 `source` 和 `sharing_scope` 发布。全局 KV events 与该开关均开启时才输出元数据；省略或显式关闭时，所有发布事件完全省略这两个字段，即使 producer 已提供其值。
- 启用元数据后，原生 GPU cache 和内置 primary CPU offload cache 报告 `instance`，Mooncake Store pool 报告 `shared`。FS / OBJ 支持在 tier 配置中显式声明 `sharing_scope`，未声明时保持未知；该资源属性不绕过元数据发布开关。
- 既有事件生成条件、字段含义、传输 envelope 和 clear 语义保持不变。启用元数据时 `AllBlocksCleared` 仅增加 source，不增加 sharing scope；该开关也不替代既有 `self_describing_kv_events`。
- **兼容性边界**：默认发布事件保持新增字段之前的字段集合与原有内容；严格消费者可在未启用元数据时继续读取原事件格式，准备就绪后再显式启用。启用后的新增字段仍要求消费者接受该 schema；不承诺不同运行之间的完整消息 bytes 相同。新开关不成为自定义 publisher 的必需构造参数，运行时返回的 KV event 配置应反映实际设置。
- 第一阶段不改变 block/chunk 匹配、remove 对应关系或事件生命周期；不增加 pool / tenant / namespace 标识，不修改 Conductor。

## Capabilities

### New Capabilities

- `kv-event-source-attribution`: 元数据显式启用、发布事件的逻辑来源、配置位置身份、聚合隔离、运行期稳定性、replay 和默认兼容性。
- `kv-event-sharing-scope`: 同一元数据开关下的资源私有 / 共享声明、FS / OBJ 配置与传播、关闭时字段省略、未知值处理，以及 sharing scope 的解释边界。

### Modified Capabilities

无。当前 OpenSpec root 尚无已有 capability specs。

## Impact

- 代码区域：`vllm/config/kv_events.py`、`vllm/distributed/kv_events.py` 的 schema / factory / 运行时配置、scheduler 的事件汇合与发布边界、connector 基类和 MultiConnector，以及 native offload CPU / tier / event translator 和 Mooncake Store 事件生成路径。
- 用户可见内容：默认关闭的元数据配置、显式启用后的 KV event map 新字段、FS / OBJ tier 新配置；实现时必须更新可从 feature index 访问的 rendered docs，以及 native offload、Mooncake Store 使用文档。
- 验证区域：默认 / 显式关闭与开启的配置、严格及宽松旧解码器、event wire / ZMQ live 与 replay、connector / tier 和真实 `LLM` 入口；开关不得改变 worker quorum、事件顺序与次数，且须验证自定义 publisher 构造兼容性和运行时配置准确性。
- MultiConnector worker 事件转发在基线中尚不完整。已有 [PR #62637](https://github.com/vllm-project/vllm/pull/62637) 处理该通路，完整组合验收依赖该修复或等价的已协调实现，不在本 change 中重复实现。
- 此 change 是独立的本地规划。已有第三方 source attribution 工作与本阶段存在重叠；在任何上游 PR 提交前必须重新检查并协调范围，本规划不授权创建重复 PR。
