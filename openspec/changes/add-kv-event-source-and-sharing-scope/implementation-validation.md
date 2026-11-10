# Implementation validation

## Baseline and workspace

- Date: 2026-11-10.
- Source: `9c239ee65ec805e00c7ebc501a3aa49dfccb9e69`, detached HEAD.
- Original checkout: `/Users/charles/Code/vllm`, initially no tracked modifications; existing OpenSpec and skill directories are untracked.
- Managed implementation worktree: `/Users/charles/.codex/worktrees/11d1/vllm`, created at the same source commit. The separately existing `45b1` worktree has user documentation edits and is not modified by this work.
- Host: Darwin arm64; uv `0.13.7` at `/Users/charles/.local/bin/uv`; isolated `.venv` uses CPython `3.12.15`.
- Runtime/source declarations already exist at the baseline. This implementation adds the default-off publication switch and its compatibility coverage.
- Raw logs and review reports are retained under `.superpowers/sdd/tasks/` in the implementation worktree.

## Transport dependency

Command: `gh pr view 62637 --repo vllm-project/vllm --json state,headRefOid,mergeCommit,url,title`.

Observed state: OPEN; head `b0eced4a4c241d35f234420704b793f0a425c6f2`; no merge commit. The checked-out `multi_connector.py` still has the TODO for `get_kv_connector_kv_cache_events` forwarding. No equivalent transport implementation was found at this baseline.

Task 1.2 is complete as a dependency audit. Task 6.3 remains incomplete: worker-originated MultiConnector plus real Mooncake Store acceptance requires the transport dependency and an applicable backend. This change does not duplicate PR #62637.

## Environment setup

- Installed `requirements/lint.txt` with uv and installed pre-commit hooks.
- Required first attempt: `VLLM_USE_PRECOMPILED=1 /Users/charles/.local/bin/uv pip install --python .venv/bin/python -e . --torch-backend=auto`.
- Result: failed because no `cu130` / `arm64` precompiled wheel was published within the searched commits. See `install-precompiled.log`; this is an environment/setup result, not a regression result.
- CPU source installation succeeded: `VLLM_TARGET_DEVICE=cpu MAX_JOBS=4 /Users/charles/.local/bin/uv pip install --python .venv/bin/python -e . --torch-backend=auto`. Native C++ and Rust artifacts were built locally; see `install-cpu.log`.
- Verified `.venv/bin/python`, torch `2.14.0`, `CpuPlatform`, `vllm.__file__`, and `vllm._C.__file__` all refer to this implementation worktree/environment; see `environment.log`.
- Installing the complete `requirements/test/cuda.in` fails on macOS because `runai-model-streamer[s3]==0.17.1` only has Linux wheels. See `install-tests.log`. The targeted suites use the installed CPU runtime plus `pytest`, `pytest-asyncio`, `pytest-timeout`, and `tblib`, installed with uv; see `install-targeted-tests.log`. The full CUDA test environment is not claimed available.

## Before-change evidence

- `.venv/bin/python -m pytest tests/distributed/test_events.py -k 'metadata_opt_in_factory or metadata_preserves_custom' -v`: 9 failed, 19 deselected. Failures demonstrate the missing configuration flag and missing runtime configuration; see `baseline-factory.log`.
- `.venv/bin/python -m pytest tests/v1/kv_connector/unit/test_multi_connector.py -k 'scheduler_metadata or connector_metadata_disabled' -v`: 15 failed, 52 deselected. Default scheduler cases fail because `KVEventSource(source_id='cache_manager', connector_name=None) != None`; explicit flag cases reject the unknown configuration keyword. See `baseline-scheduler.log`.
- `.venv/bin/python -m pytest tests/v1/kv_offload/cpu/test_manager.py tests/v1/kv_connector/unit/offloading_connector/test_events.py tests/v1/kv_offload/tiering/test_fs_tier.py tests/v1/kv_offload/tiering/test_obj_tier.py -v`: 277 passed in 46.68 s before production edits. Existing CPU scope, successful store, eviction ordering and metadata-only cleanup assertions passed; task 4.1 is revalidated. See `baseline-resource-suites.log`.
- Initial real `LLM` attempt stalled before assertions; a native sample showed parent waiting in ZMQ poll. A second run with `VLLM_ENABLE_V1_MULTIPROCESSING=0` reached GPT2 CPU model initialization but did not complete weight loading. Both owned test processes were terminated and logs preserved (`baseline-llm.log`, `baseline-llm-inline.log`). These are not event-regression results or serving acceptance.

## Implemented behavior

- `enable_kv_event_metadata` defaults to false and uses the existing `--kv-events-config` JSON argument.
- Source attribution requires both event and metadata switches. Default/false publication omits source and sharing scope after existing quorum and merge, including producer-provided values, without recounting or deduplicating events.
- Existing custom publisher constructors receive no new keyword. Framework runtime configuration reflects the actual metadata setting, including when a plugin returns fresh config copies.
- Internal scope declarations, clear semantics, existing offload payload selection, event ordering and original producer objects are preserved.
- Rendered docs are linked from the feature index and offload/Mooncake guides. A fresh local documentation review found the default, enabling instructions, compatibility behavior and payload distinction; JSON examples and file links passed. Opaque-ID and migration/restart clarifications were incorporated. No published docs preview or web/GitHub discoverability is claimed.

## After-change component verification

Commands below run in the managed implementation worktree with `.venv/bin/python`.

| Arguments | Result | Raw log under `.superpowers/sdd/tasks/` |
| --- | --- | --- |
| `-m pytest tests/distributed/test_kv_cache_events.py tests/distributed/test_events.py tests/engine/test_arg_utils.py -v` | 189 passed, 1 unrelated CPU platform failure | `runtime-final-wire-cli-02.log` |
| `-m pytest tests/engine/test_arg_utils.py -k kv_event_metadata_json_cli -v` | 3 passed | `runtime-final-cli-targeted.log` |
| `-m pytest tests/v1/kv_connector/unit/test_multi_connector.py -k 'connector_metadata or scheduler_metadata' -v` | 21 passed | `runtime-final-scheduler-02.log` |
| `-m pytest tests/v1/kv_connector/unit/offloading_connector/test_events.py tests/v1/kv_offload/tiering/test_fs_tier.py tests/v1/kv_offload/tiering/test_obj_tier.py -k scope -v` | 75 passed | `runtime-final-offload-tiers-02.log` |
| `-m pytest tests/v1/kv_connector/unit/test_mooncake_store_connector.py tests/v1/kv_connector/unit/test_mooncake_store_worker.py -k 'scope or quorum' -v` | 14 passed | `runtime-final-mooncake-02.log` |
| `-m pytest tests/v1/core/test_prefix_caching.py -k 'kv_cache_events or null_parent_block_hash' -v` | 7 passed | `runtime-final-prefix.log` |

The one broad-suite failure is `test_multi_node_world_size_includes_pcp`: CPU without Triton does not support prefill context parallelism. The identical failure was reproduced on the untouched original checkout using `PYTHONPATH=/Users/charles/Code/vllm /Users/charles/.codex/worktrees/11d1/vllm/.venv/bin/python -m pytest tests/engine/test_arg_utils.py::test_multi_node_world_size_includes_pcp -v`, after verifying imports resolve to `/Users/charles/Code/vllm/vllm/__init__.py`. See `baseline-unrelated-cpu-cli.log`; no skip, threshold change or unrelated fix was added.

`.venv/bin/pre-commit run --files <changed files> docs/features/kv_event_metadata.md` passed, including ruff, formatting, Markdown, mypy, configuration validation and test tethering. See `runtime-precommit-02.log`. `git diff --check` passed. Local CI selection uses the repository test-tethering parser and confirms affected suites and marker selection; see `runtime-ci-selection.log`. These are not remote Buildkite execution results.

Earlier failed test-authoring and implementation runs are retained alongside corrected outputs; the implementer report lists each cause and correction. They have not been removed or reported as successes.

## Outstanding acceptance

- Task 3.3: real `LLM` regression is implemented but has not generated tokens. Online model acquisition stalled; a bounded direct download retrieved 5,614,766 of 352,824,413 bytes before its 120-second timeout (`model-download-curl.log`). The final offline command `HF_HUB_OFFLINE=1 VLLM_ENABLE_V1_MULTIPROCESSING=0 VLLM_CPU_KVCACHE_SPACE=1 OMP_NUM_THREADS=4 .venv/bin/python -m pytest tests/entrypoints/llm/test_generate.py -k kv_event_metadata -v -x --timeout=60 --timeout-method=thread` failed before generation with `Cannot find any model weights` for the distilgpt2 snapshot (`final-llm-offline.log`). Model used: `distilbert/distilgpt2`, snapshot `2290a62682d06624634c1f46a6ad5be0f47f38aa`; backend: macOS ARM CPU.
- Task 6.3: actual Mooncake Store and MultiConnector worker transport acceptance remains unavailable as described above. Component tests with mocked external Store services do not satisfy this gate.
- Task 7.3: overall acceptance remains incomplete until 3.3 and 6.3 pass. No model accuracy, performance, GPU/NPU serving or complete phase acceptance is claimed.

Next validation: with model weights available on an applicable backend, run `.venv/bin/python -m pytest tests/entrypoints/llm/test_generate.py -k kv_event_metadata -v`. For 6.3, first integrate the coordinated worker transport dependency and use a real Mooncake Store environment, then record source, backend, configuration and both publication modes.

## Independent review

`final-review.md` found no blocking implementation findings and accepted spec compliance/code quality for this delta, while retaining incomplete tasks 3.3, 6.3 and 7.3. `docs-review.md` verified local discoverability and examples. Reports are retained in the raw-evidence directory above.

The scheduler test helper decodes replay bytes before returning events; its strict checks therefore cover reconstructed encodings. The separate ZMQ suite verifies raw field absence and exact live/replay bytes. Neither substitutes for the uncompleted real-LLM or external-Mooncake gates.
