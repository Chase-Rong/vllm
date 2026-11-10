# Design

## Context

动机见 [proposal.md](proposal.md)。原设计依据 vLLM `dc74b8cbbbe3dbb9399fe369757d63defd614cc2`。本次修订核对的 checkout 为 detached HEAD `9c239ee65ec805e00c7ebc501a3aa49dfccb9e69`，已有 source / scope 实现，但仍随 KV events 自动发布，尚无独立元数据开关。下文描述修订后的目标；进入 apply 前须核对实际基线、已有实现与依赖状态，不能将代码存在视为验收完成。

| 已观察的实现 | 对设计的约束 |
| --- | --- |
| `vllm/distributed/kv_events.py` 的事件是 tagged maps，batch 是 positional array；现有可选字段默认省略 | 只扩展事件 map，不改变 batch / transport frames |
| scheduler 将 cache manager 与 connector 事件合并；MultiConnector 的 `take_events()` 展开各子项 | source 必须在丢失逻辑来源边界之前附加 |
| 现有事件聚合通过事件计数和 worker 数量判定 quorum；Mooncake Store 还处理 TP replication factor | 不将每个物理 worker 分配成不同逻辑 source，不改 quorum |
| `OffloadingEvent` 已携带 medium、locality、ownership；tracker 转换为公开 store/remove | scope 可沿同一 raw-event 路径传播，不改 hash / remove 含义 |
| CPU manager 明确管理私有 primary cache；FS / OBJ 具有可选 locality 和 tier 事件开关 | 私有 CPU 由 producer 声明；FS / OBJ scope 单独配置 |
| Mooncake Store worker 生成 `medium="cpu"` 的成功 store，scheduler 端完成逻辑聚合 | worker 填 scope，发布边界填 source，不制造 remove |
| 当前 GPU / CPU / Mooncake producer 已填写 scope，scheduler 随 KV events 开启 source 标注 | 关闭元数据时须在汇合后的发布边界省略两个字段，仅跳过 source 标注不够 |
| publisher factory 将配置展开为构造参数，ZMQ publisher 重新构造运行时配置 | 新开关须由框架处理，避免新增自定义 publisher 参数或错误报告默认值 |
| MultiConnector 尚无通用 worker 事件转发，已有 PR #62637 修复该通路 | 组合验证必须区分 transport 依赖与来源标注能力 |

当前 OpenSpec root 没有既有 specs。此 change 新建两个 capability，不修改 Mooncake 仓库。讨论中已确认的字段和启用规则是本设计的输入，不依赖第三方 #65814 的实现。

## Goals / Non-Goals

**Goals:**

- 同一逻辑生产者可被稳定识别，不同 connector 的同内容事件保持独立。
- scope 随实际资源传播；source 由框架赋值，scope 由资源契约或 tier 配置提供。
- 以少量生产者元数据和浅复制完成标注，不创建新的长期驻留索引。
- 使用现有发布和 replay 通路；元数据默认关闭，保留既有事件字段，部署者显式启用后才发布来源和 scope。

**Non-Goals:**

- 不改变检索粒度、chunk boundary、删除 hash 展开、引用计数或失效完整性。
- 不定义共享 pool / tenant / namespace ID、访问授权、物理对象去重或跨重启 epoch。
- 不实现 Conductor 适配、snapshot recorder 或 MultiConnector worker 传输修复。
- 不改变 `AllBlocksCleared` 的既有清除范围，不根据其 source 引入局部清除行为。

## Decisions

### 1. Inline source 与 sharing scope 分别表达

在事件 schema 中新增一个不可变、可复用的 source descriptor，字段为 `source_id: str`、`connector_name: str | None = None`。三个事件类型的 `source` 均默认 `None`；仅 store/remove 新增默认 `None` 的 `sharing_scope`。未知值省略，旧事件和显式 null 解码为未知。

source 表示具体报告方，sharing scope 表示被报告资源。两者在同一个第一阶段 change 交付，并由同一默认关闭的发布开关控制；不将 scope 放进 source descriptor，避免把一个 connector 的所有 tier 错误绑定到同一属性。保留可选 schema 以接受两种发布模式，具体启用规则见决策 6。

以下示例以 KV events 和 `enable_kv_event_metadata` 均开启为前提，只展示本 change 涉及的字段；完整事件仍包含既有必需字段：

```json
[
  {
    "type": "BlockStored",
    "medium": "CPU",
    "source": {
      "source_id": "connector/0",
      "connector_name": "OffloadingConnector"
    },
    "sharing_scope": "instance"
  },
  {
    "type": "BlockStored",
    "medium": "cpu",
    "source": {
      "source_id": "connector/0",
      "connector_name": "MooncakeStoreConnector"
    },
    "sharing_scope": "shared"
  },
  {
    "type": "BlockStored",
    "medium": "GPU",
    "source": {"source_id": "cache_manager"},
    "sharing_scope": "instance"
  },
  {
    "type": "AllBlocksCleared",
    "source": {"source_id": "cache_manager"}
  }
]
```

两个 `connector/0` 示例属于独立 publisher 上下文；同一运行期内两个 connector 使用不同配置位置。

替代方案：只有 connector 名称无法区分重复类型；只有 scope 无法表达具体生产者；增加 `kind` 没有当前消费需求；首次实现 discovery / descriptor registry 会引入晚订阅和 descriptor 变化处理，因此本阶段选择 inline。

### 2. 按配置位置分配来源，运行期稳定

- 原生 cache manager 使用 `cache_manager`，省略 connector 名称。
- 单 connector 使用 `connector/0`。
- 顶层 MultiConnector 按子项配置索引形成 `connector/0`、`connector/1`；嵌套 MultiConnector 再追加索引，例如 `connector/1/0`。
- 子项本轮无事件时保留位置；重复类型仍按配置位置区分。
- `connector_name` 取配置中的名称，不从解析后的 Python 类名推导。配置名提供诊断信息，不能作为共享属性或长期检索契约。

完整来源上下文是 `(publisher runtime, DP rank, source_id)`。ID 不是 pool ID、物理副本 ID 或事件唯一键，消费者不能仅按 source 去重事件。固定配置下的 descriptor 在初始化时建立并复用。

不复用 `KVTransferConfig.engine_id`，因为 MultiConnector 子项默认可继承同一 engine ID；不为每个 worker 或事件生成随机 ID。框架只识别其可见的配置实例：自定义 opaque composite 在没有显式暴露子生产者的情况下是一个 source，不根据任意插件内部结构猜测身份。

重启可能复用相同 endpoint 和路径，本阶段不自动检测这种变化。部署生命周期负责让消费者撤销旧来源状态；重新排序或替换 connector 需要开始新的消费者来源上下文。

### 3. 启用元数据时在 producer 聚合之后、混合之前附加 source

```text
CPU / FS / OBJ raw events (scope)
    --> offload event translation
    --> logical connector drain --------+
                                       |
Mooncake worker stores (scope)          v
    --> existing per-connector quorum --> attach connector source
                                       |
Native cache-manager events             |
    --> attach cache-manager source ----+--> merge --> batch --> publisher
                                                               |
                                                               v
                                                          replay bytes
```

在 connector 公共接口提供默认的 metadata-aware drain 包装入口，内部调用既有 `take_events()`，为公开事件创建带框架 source 的浅复制。原有 connector 无需为了来源标注修改自己的 drain 实现。MultiConnector 在该入口按固定子配置递归构造来源后调用各子项，而不是先展开所有事件再统一赋值。

scheduler 在全局事件和元数据开关均开启时标注原生 cache manager 事件，并调用 connector 的包装入口，再按既有顺序汇合。包装入口也使用同一启用条件。框架覆盖该边界的 producer 自填 source，避免错误或重复 ID；producer 负责的 scope 保持不变。

元数据关闭时走既有 drain 路径，不做框架来源标注。在 scheduler 完成汇合、构造发布 batch 之前，统一将待发布副本的 source 和 store/remove 的 sharing scope 置为默认值，使编码完全省略这些字段。该边界同样覆盖内置 producer、MultiConnector 子项和插件自填元数据，不能仅依赖 producer 自觉停止填写。已经没有元数据的事件可直接复用；需要省略字段时仅浅复制，不修改 producer 持有的对象。处理发生在既有 worker quorum 完成之后，省略后不得重新计数或去重。

不原地修改 producer 仍持有或已作为计数 key 使用的事件；不复制大型 hash/token 列表，不改变 list 内容。公开事件的 equality / hash 需要一致考虑新字段，避免来源或 scope 不同的记录被后续使用方意外合并。worker 侧在来源附加前维持原有表示和 quorum；scope 在同一逻辑资源的 workers 间必须一致。

替代方案：worker 随机生成来源会破坏逻辑聚合；仅在最外层 scheduler 加一个 connector 名称会丢失 MultiConnector 子项；直接修改 drain 返回对象会影响 producer 保留状态或计数容器。

### 4. Scope 由 producer / tier 声明，发布受统一开关控制

| Producer / resource | 决策与修改位置 |
| --- | --- |
| 原生 GPU prefix cache | 在 `vllm/v1/core/block_pool.py` 对应资源的 store/remove 路径声明 `instance`；共用 pool 代码不能据 medium 全局猜测其他资源的范围 |
| 内置 primary CPU cache | 在 `vllm/v1/kv_offload/cpu/manager.py` 的既有公开 store/remove 对应 raw events 声明 `instance` |
| FS / OBJ | 各自可选配置解析为 `instance`、`shared` 或未知，并写入该 tier 的 raw events |
| Mooncake Store | 在 `mooncake/store/worker.py` 的成功 store 构造处声明 `shared`，保留现有 TP 聚合 |
| 其他 tier / 插件 | 保留 producer 已声明值；无法确定则未知，不根据 CPU / STORAGE 标签统一填充 |

给 `vllm/v1/kv_offload/base.py` 的 raw `OffloadingEvent` 增加默认未知的 scope，再经 `offloading/events.py` 传递到完整 store、placeholder store 和既有 remove。tiering manager 的合并不改写 scope。原有 medium 大小写、locality、ownership 和 removal hash 表示全部保持。

上述声明和内部传播不因发布开关而改变资源属性；只有启用元数据时才将 scope 送入发布 batch。关闭时在决策 3 的边界省略两个新字段，覆盖完整 store、placeholder 和既有 remove，replay 保存对应模式下的原始 bytes。

尤其不能在 scheduler 中将所有 `OffloadingConnector` CPU 事件统一改成 instance：该 connector 还可能报告其他资源，例如 KVCR inventory。只有资源契约明确时才填写。

### 5. FS / OBJ 增加独立的 tier 属性配置

```json
{
  "type": "fs",
  "root_dir": "/shared/kv-cache",
  "enable_kv_events": true,
  "locality": "LOCAL",
  "sharing_scope": "shared"
}
```

配置值接受 `instance`、`shared`、省略或 null；非法字符串、空字符串和非字符串值在初始化时报错，无论元数据开关是否开启，不做大小写转换或真假值转换。省略和 null 对应事件字段省略。校验规则是实现默认，不推断目录或 bucket 的共享关系。

FS / OBJ 构造及配置透传适配新可选参数，保留既有位置参数含义；其他插件 tier 不被强制增加新的 constructor 参数。已有全局与 tier 的 `enable_kv_events` 条件继续控制事件是否产生。`sharing_scope` 是资源属性，不是元数据启用开关；上述 tier 示例只有同时配置全局 `enable_kv_event_metadata=true` 才会对外发布 scope，关闭时仍发布原事件但省略新字段。

不将 `ownership` 重新定义为 source 或 pool ID；不把 LOCAL 等同于 instance、REMOTE 等同于 shared。显式声明只描述部署属性，不改变文件 / 对象读写、淘汰或权限。

### 6. 默认关闭的元数据开关与兼容性边界

在 `KVEventsConfig` 增加 `enable_kv_event_metadata: bool = False`，使用现有 JSON 配置入口，不新增顶层 CLI 参数：

```bash
--kv-events-config '{"enable_kv_cache_events":true,"enable_kv_event_metadata":true}'
```

| KV events | `enable_kv_event_metadata` | 发布行为 |
| --- | --- | --- |
| 关闭 | 任意 | 元数据配置不使原先禁用的事件开始产生 |
| 开启 | 省略或 `false` | 按原条件发布事件，完全省略 source 和 sharing scope |
| 开启 | `true` | 三类事件提供 source；store/remove 提供已知 scope，未知则省略；clear 不增加 scope |

此开关仅控制本 change 的两个新字段，不删除 locality、ownership、session 等既有信息，也不改变 offload 的 `self_describing_kv_events` 所控制的完整 / placeholder payload 选择。FS / OBJ 的 scope 配置不能绕过该开关，元数据开关也不能绕过全局或 tier 的事件生成条件。

开关在初始化时确定，本阶段不新增动态切换接口。标注与字段省略在 scheduler / connector 的事件收集和发布边界完成，统一适用于框架注册的 publishers。factory 在展开 constructor 参数之前提取该开关，不将其作为新参数传给 ZMQ 或既有自定义 publisher。框架在返回运行时 KV event 配置的路径上保留实际配置值，同时保留 publisher 解析后的 endpoint 等值，避免 ZMQ 重新构造配置时将显式开启误报为默认关闭；沿用既有配置查询通路，不新增 discovery endpoint。

保留 `source=None` 与 `sharing_scope=None` 默认以接受旧事件，并利用默认值省略编码恢复默认发布的旧字段集合。元数据关闭时，同一既有事件的字段和值保持不变，严格旧解码器不应因为这两个新增字段而失败；发送显式 null 不满足此契约。启用时新增 map 字段会改变 bytes，宽松旧解码器仍须通过测试，严格消费者须先适配再开启。这里约束的是事件 schema 与内容，不承诺跨运行的 timestamp、序号或完整消息 bytes 相同，也不将事件兼容性承诺扩大到新增配置字段。

Replay 保存经过发布模式处理后的编码 bytes，直接复用原机制；两种模式下均不得在 replay 时重新标注、补字段或删字段。不把这一结论延伸为尚未集成的 snapshot recorder 兼容性。未来若接入 snapshot，其 scope key 和合成事件也必须处理两种模式，但不在本阶段实现。

替代方案：默认开启只能让部署者在发现问题后退出，仍会改变既有配置的事件格式；拆成 source 和 scope 两个开关会增加组合却没有当前需求；仅依赖宽松解码器不能覆盖严格消费者。因此采用单一、默认关闭的显式启用开关。

### 7. MultiConnector transport 是明确依赖

基线 `multi_connector.py` 仍保留 `get_kv_connector_kv_cache_events()` 的 TODO，worker 端继承基类返回 None。因此 scheduler drain 标注不能修复 Mooncake / LMCache 子项未转发的事件。

[PR #62637](https://github.com/vllm-project/vllm/pull/62637) 是已有 transport 工作，探索时仍为 open。实现本 change 前核对其最终 API；本 change 只接入已完成传输和逐 connector 聚合的边界，不重做该修复。

原生 cache manager、standalone connector 和已有 scheduler-originated MultiConnector 事件可分别验证。涉及 worker-originated MultiConnector 的完整验收必须在该依赖满足后执行并单独记录，不能用 mock drain 的通过替代组合验收；依赖未满足时不得声称第一阶段完整验收完成。

### 8. 验证采用最轻的真实路径并保留证据边界

测试设计：模块负责发布缓存报告；输入是 producer 事件、元数据开关与 tier 配置；输出是保留原 payload、按开关提供或省略新字段的 live/replay 事件；主要防止默认格式变化、字段遗漏省略、跨来源合并、错误 scope、quorum 变化和旧事件解码失败。

| 层级 | 复用现有测试与验收目标 |
| --- | --- |
| Configuration / publisher | 复用配置及 `tests/distributed/test_events.py` 的 factory / runtime-config 用例：省略、显式 false / true、全局关闭时无效、既有自定义 constructor、运行时准确报告实际开关和解析后的 endpoints |
| Wire | `tests/distributed/test_kv_cache_events.py`：省略 / null、三类事件形状、关闭时严格旧解码器、开启时宽松旧解码器、equality / hash 的来源与 scope 区分；严格 schema 对齐新增两个字段之前的版本 |
| Publication / replay | `tests/distributed/test_events.py`：真实 ZMQ，两种模式的原编码 replay、一致顺序 / 次数、关闭时字段完全省略，bounded polling，不使用固定 sleep |
| Native / tiers | `offloading_connector/test_events.py`、CPU manager、`test_fs_tier.py`、`test_obj_tier.py`：完整及 placeholder、既有 remove、两种配置与未知 / 非法值、不同 tier 独立声明 |
| Connector / aggregation | `test_multi_connector.py`、`test_mooncake_store_connector.py`、`test_mooncake_store_worker.py`：开启时重复及嵌套来源、关闭时 producer / 插件元数据省略、两种模式的原对象与 TP replica quorum 不变 |
| Real entry point | 扩展 `tests/entrypoints/llm/test_generate.py`：复用 `vllm_runner` 与已有模型 fixture，真实生成并订阅 live/replay，覆盖未配置、显式 false 和 true；验证默认字段省略、开启时 native source / scope 和严格旧解码器的默认路径，按可用 backend 运行 |
| Dependency integration | transport 依赖满足后，在真实 Mooncake Store 环境验证 standalone 与 MultiConnector worker 事件达到发布端，关闭 / 开启分别验证字段省略与元数据，记录所用源码、backend 和结果 |

新测试使用真实构造和公开路径；仅 mock 外部服务。在当前自动发布实现上先复现默认消息包含新字段、严格旧解码器不接受该消息或新开关尚不受支持，再验证修订结果；保留已有显式元数据的来源与 scope 回归，并记录每次基线。现有 block/chunk 匹配不在这里声称被修复。

CI 要覆盖这些 suites 与新的 source-file dependencies。基线 `misc.yaml` 已选取 connector / offload suites，`distributed.yaml` 选取 ZMQ event suite，`entrypoints.yaml` 选取 `test_generate.py`；`test_kv_cache_events.py` 需要显式确认并补齐选择。部分步骤排除了 `kv_transfer`，必须为实际受影响的 connector 文件补齐触发条件。

仅增加事件元数据与属性配置，不改变模型计算，第一阶段无需声称新的 accuracy / performance 改善。真实入口、backend 集成和吞吐测量是不同证据；没有执行的层级如实标记。

## Risks / Trade-offs

- [严格消费者拒绝新增字段] -> 默认完全省略新字段，部署者确认消费兼容后显式开启；以严格旧 schema 验证关闭模式。
- [只关闭 source 标注而泄漏 scope] -> 汇合后统一落实两个字段的发布规则，覆盖内置 producer、tier 与插件，省略操作不改变原对象或聚合结果。
- [配置透传破坏 publisher 或运行时误报] -> factory 不传新 constructor 参数，运行时配置保留实际开关，分别验证自定义 publisher 与 ZMQ 配置查询。
- [source 被误当作共享池或物理副本] -> 文档明确 producer、sharing scope 和 storage domain 的职责，Conductor 仍通过部署配置关联域。
- [路径在重启后复用] -> 依靠部署生命周期重建消费者状态，不声称 ID 自带 restart detection。
- [每事件复制与编码增量] -> 复用不可变 descriptor，仅浅复制事件；不创建驻留表，保留原 token/hash 列表。
- [错误 scope 由配置传入] -> 严格校验允许值；部署者负责资源属性的真实性，系统不从 locality、路径或 bucket 猜测。
- [MultiConnector transport 尚不可用] -> 按明确依赖完成组合验收，不将事件丢失误报成归属正确或测试成功。
- [只发 store 的来源仍可能漂移] -> 本阶段只声明共享属性，不增加虚假删除或驻留准确性保证。
- [已有第三方方案重叠] -> 保持独立本地规划；任何上游提交前刷新 issue / PR 检查并由用户协调贡献范围，不自动发布评论或竞争 PR。

## Migration Plan

1. Apply 前确认当前源码、环境和 transport 依赖；按测试指南建立所需的 uv 环境及可用 backend。
2. 对照已有 schema、来源包装和资源 scope 传播补齐默认关闭开关、发布字段省略、factory 与运行时配置处理；已有元数据回归显式开启，旧配置保留既有事件格式。
3. 实现时新增 `docs/features/kv_event_metadata.md` 并从 feature index 链接，更新 native offload 与 Mooncake Store 使用文档；记录默认 / 显式启用配置、三类事件、两个 store/remove 字段的省略规则、clear 边界、FS / OBJ 配置及严格消费者迁移方式。
4. 运行组件、真实入口和依赖集成验证，区分所有未执行项；由用户审阅每行实现和测试结果后再决定上游贡献。
5. 升级 producer 时保持元数据默认关闭；待相关消费者接受新 schema 后，在启动配置中显式开启。需要回退事件格式时设为 false 并重启，无需仅为关闭元数据而降级 producer；消费者继续将缺失信息视为未知，并按部署生命周期清理旧上下文。已有自动输出版本的部署若需要保留元数据，升级时须显式设置 true。
