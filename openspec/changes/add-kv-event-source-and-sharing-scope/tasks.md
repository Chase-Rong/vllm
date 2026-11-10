# Tasks

以下为本 change 的实施与验收任务。需求以两个 capability specs 为准，架构见 [design.md](design.md)，本次源码、环境、测试结果与未完成验收见 [implementation-validation.md](implementation-validation.md)。实施复用已有 source / scope 代码，补齐默认关闭开关与相应回归；仅在指定行为和验证均已完成后勾选，组件结果不替代真实入口或外部集成验收。

## 1. 基线与依赖确认

- [x] 1.1 记录实际分支、commit、tracked / untracked 状态及 Python / backend 环境，遵循仓库指南使用 uv 和 `.venv/bin/python`；对照修订时的 `9c239ee65ec805e00c7ebc501a3aa49dfccb9e69` 确认已有实现与待补差异，以可复现的基线记录和隔离工作区确认没有覆盖用户现有状态。
- [x] 1.2 核对 #62637 或等价已协调 transport 修复的状态及最终接口，记录所用 commit 和可用的逐 connector worker 聚合 / 路由通路；不存在依赖时明确保留 6.3 的未完成状态，不在本 change 中重复实现修复。

## 2. 配置、wire schema 与兼容契约

- [x] 2.1 复核可选 source descriptor、三个事件的 source、store/remove 的 sharing scope 和 equality / hash；扩展 `tests/distributed/test_kv_cache_events.py`，保留省略 / null、不同来源不合并及宽松旧解码器回归，增加对齐新字段之前 schema 的严格旧解码器，验证不含新字段的三类事件可读、带新字段时拒绝，并记录当前自动输出与目标默认格式的差异。
- [x] 2.2 在 `KVEventsConfig` 增加默认 false 的 `enable_kv_event_metadata` 并沿现有 `--kv-events-config` 解析；factory 提取该字段而不向 publisher constructor 透传，运行时配置保留实际值及解析后的 endpoints。复用现有配置和 `test_events.py` 的 factory / runtime-config 用例，验证省略 / false / true、全局关闭时不启用发布、两种模式的既有自定义 publisher 构造和准确配置返回。
- [x] 2.3 创建 `docs/features/kv_event_metadata.md` 并从 `docs/features/README.md` 链接，记录默认关闭、显式启用示例、字段省略契约、严格消费者迁移与运行时配置；核对三类事件、无新增 kind / batch 字段，并说明此开关仅控制 source / scope，与 `self_describing_kv_events` 及 tier 资源声明分别定义。

## 3. 框架来源标注与原生 cache manager

- [x] 3.1 将 connector 包装入口与 scheduler 的来源标注条件改为全局事件和元数据开关同时开启；关闭时使用既有 drain，并在 scheduler 汇合后、交给 publisher 前统一省略 source 和 store/remove 的 scope。复用 `test_multi_connector.py` 的真实构造及 scheduler 回归，覆盖未配置 / false / true、producer 或插件自填元数据、单项 / 重复 / 嵌套 / 空贡献、配置名称、原对象不变，以及关闭后内容相同的两条事件不被合并；记录修订前后结果。
- [x] 3.2 复核原生 GPU store/remove 的 instance 声明与 clear 事件，沿 3.1 的发布边界落实开关；扩展现有 prefix-cache 和 scheduler 测试，验证开启时三类事件的 cache_manager source、store/remove 的 instance scope，关闭时完全省略新字段，clear 无 scope 且范围未缩小，共用 pool 的其他资源未被按 medium 误分类。
- [ ] 3.3 在 `tests/entrypoints/llm/test_generate.py` 复用 `vllm_runner`、已有模型及真实生成 / ZMQ live/replay 用例，参数化未配置、显式 false 和 true；通过 `.venv/bin/python -m pytest tests/entrypoints/llm/test_generate.py -k kv_event_metadata -v` 验证默认及关闭时字段省略且严格旧解码器可读、开启时 native source / scope、两种模式的 replay 原 bytes 与 DP 信息，记录 backend、模型和修订前后结果，使用条件轮询而非固定 sleep。
- [x] 3.4 将来源路径、单次运行身份边界、opaque composite 和 clear 语义写入 metadata 文档；复核例子与 3.1 / 3.2 的两种发布模式一致，且不承诺跨重启检测或按 source 清除。

## 4. Native CPU 与 offload 事件转换

- [x] 4.1 为 raw `OffloadingEvent` 增加默认未知的 scope，在内置 CPU manager 的既有公开 store/remove 路径声明 instance；扩展 CPU manager 现有 suite，验证实际 store / eviction 事件及 metadata-only 清理仍保持原有生命周期，未声明的其他资源不会被统一分类。
- [x] 4.2 复核 offload event translator 对完整 store、placeholder store 和既有 remove 的 scope 传播；扩展 `tests/v1/kv_connector/unit/offloading_connector/test_events.py` 并复用发布边界用例，验证内部资源声明保留、开启时 scope 发布、关闭时所有 payload 形式省略新字段，以及 group / locality / ownership、hash 数量和含义不变，保存基线与实现结果。
- [x] 4.3 更新 `docs/features/kv_offloading_usage.md` 的原生 CPU 声明和完整 / hash-only 事件说明，给出元数据显式启用配置；核对关闭时省略 source / scope 不改变 `self_describing_kv_events` 的 payload 选择，文档不把元数据描述为 chunk 匹配或 removal 契约的修复。

## 5. FS / OBJ 的配置与资源传播

- [x] 5.1 复核 FS / OBJ tier 可选 sharing scope 配置及初始化校验，保持其他 tier constructor 和原位置参数含义；扩展 `test_fs_tier.py`、`test_obj_tier.py` 及相关 factory 用例，验证 instance / shared、缺失 / null、非法字符串 / 非字符串和旧配置正常构造，并确认元数据关闭不跳过资源属性校验。
- [x] 5.2 复核各 FS / OBJ tier 的 scope 经 raw events、tiering 汇合和 translator 保留；复用 private CPU + shared FS / OBJ 组合及发布边界用例，验证开启时两种 scope、完整 / placeholder、LOCAL + shared 共存及未知值省略，关闭时有效 tier 声明也不进入 wire，既有全局 / tier 事件开关行为不变。
- [x] 5.3 在 native offload 使用文档中增加 FS / OBJ 两种 scope 示例、未知与非法值规则，以及全局元数据开关配置；核对真实 factory 配置与两种发布模式一致，明确资源声明不绕过开关，保留 locality、ownership、scope 各自含义和不推断路径 / bucket 的边界。

## 6. Mooncake Store 与依赖集成

- [x] 6.1 复核 Mooncake Store worker 成功 store 的 shared 声明；扩展现有 `test_mooncake_store_worker.py`、`test_mooncake_store_connector.py` 并复用发布边界回归，验证既有 shard 成功条件、跨 poll / TP replication quorum、小写 cpu、原 payload 及次数均保持，开启时 source / shared 正确、关闭时 producer 已有元数据仍被省略。
- [x] 6.2 更新 `docs/features/mooncake_store_connector_usage.md` 并链接 metadata 文档，给出显式启用配置和默认省略行为；验证示例说明 shared 仍需要部署域关联，connector store 与 backend 驻留 / remove 的职责保持独立。
- [ ] 6.3 在 1.2 的 transport 依赖满足后，使用真实 Mooncake Store 与适用 backend 验证 standalone 及包含 worker-originated connector 的 MultiConnector 从生成、worker 聚合到实际发布 / replay 的两种模式：开启时 source / scope 正确，关闭时字段省略且原事件次数与 payload 保留；记录依赖 commit、配置及结果，缺少依赖或环境时保留本项未完成，不以 mock 或组件结果替代组合验收。

## 7. 发布通路与最终验证

- [x] 7.1 扩展 `tests/distributed/test_events.py` 的真实 ZMQ 用例，验证开启 / 关闭两种模式的 live 与 replay 原 bytes、顺序 / 次数及 DP 信息；相同 payload、不同来源的两条事件在两种模式下都保留，关闭时严格旧解码器可读且 replay 不补充字段，开启时 replay 不重建或删减元数据。
- [x] 7.2 在相关 Buildkite test areas 显式覆盖新增 / 扩展 suites 和实际修改文件的触发条件，尤其确认 `test_kv_cache_events.py` 被选中、KV transfer 的既有排除不会漏掉相关回归；以实际 CI 选择结果和符合仓库格式的 commands 作为完成证据。
- [ ] 7.3 汇总各组已运行命令与原始结果，并执行必要的跨组回归、改动文件的 pre-commit 和 `git diff --check`；区分 wire / component、真实 LLM 入口和 Mooncake 依赖集成的证据，所有强制验收完成后才标记实现完成，不新增无依据的模型准确率或性能结论。

## Workflow follow-up

- 由用户审阅全部实现与测试结果后决定是否推进上游贡献；本 change 不自动 commit、push 或发布评论 / PR。
- 上游提交前按仓库要求刷新 RFC #65548 的 issue comments、关联 PR 和领域关键词检查，协调第三方来源标注等重叠工作，再运行 `pr-checklist`；若仍重复则不提交竞争 PR。
- 只有在任务实际完成且用户要求归档时，才进入 OpenSpec archive workflow。
