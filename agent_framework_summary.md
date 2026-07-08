# CCE Instruction 自动性能采集框架计划

本文是当前确认后的第一版完整计划。目标是建立 Agent 可运行、可中断恢复、可持续覆盖 CCE instruction 参数空间的性能采集框架。第一版只做性能采集平台，不做微架构反向建模、不做自动重测检查、不做云端同步。

Agent 实际执行入口见：`Auto_test/docs/agent_workflow.md`。

## 1. 范围

第一版目标：

- 用户告诉 Agent 要测哪个 CCE instruction。
- Agent 先判断该 instruction 是否已经理解过。
- 未理解过则进入 understanding 阶段，生成文档和极简 metadata，人工确认后才能正式测试。
- 已理解过则校验 signature 是否 stale，然后生成或恢复 coverage plan。
- 持续小批生成 case、小批执行、写 SQLite。
- 支持 token 中断或执行中断后继续覆盖参数空间，不重复已完成 case。
- 单核测试定义为 launch 单 block / 单 AIV kernel instance。

第一版不做：

- 微架构反向探索和模型建立。
- 自动抽样重测 / regression 检查。
- 云端上传、多机 merge、自动 git push。
- 默认保存生成源码、binary、完整 stdout/stderr。
- smoke 结果入库。

## 2. 目录结构

第一版框架全部放在 `Auto_test/` 下，避免污染仓库根目录。

```text
Auto_test/
  docs/
    agent_workflow.md
    database_schema.md
    instruction_understanding.md
    request_format.md
    template_authoring.md
  schemas/
    instruction_metadata.schema.json
    request.schema.json
    coverage_plan.schema.json
  instructions/
    <instruction>/
      understanding.draft.md
      metadata.draft.json
      understanding.md
      metadata.json
  templates/
    <template_family>/
      ...
  requests/
    <reusable_request>.json
  scripts/
    init_db.py
    extract_signatures.py
    create_plan.py
    run_plan.py
    query_results.py
  migrations/
    001_initial.sql
  data/
    .gitkeep
    perf.sqlite3
    requests/
    tmp/
```

`Auto_test/data/` 默认进 `.gitignore`，只保留 `.gitkeep`。SQLite、运行中 plan、临时日志、临时 signature 抽取结果都不进 git。

`Auto_test/requests/` 存可复用 request 模板，可以进 git；运行展开后的 plan 快照放 `Auto_test/data/requests/`，不进 git。

## 3. 总流程

1. 用户指定 CCE instruction。
2. Agent 检查 `Auto_test/instructions/<instruction>/metadata.json` 是否存在 approved signature。
3. 若不存在，进入 understanding：
   - 从 `L2Bypass_test/stub_fun.h` 按需抽取该 instruction 的 signature / overload。
   - 查 CANN / AscendC header、已有测试和笔记，理解参数语义。
   - 生成 `understanding.draft.md` 和 `metadata.draft.json`。
   - 人工确认后提升为 `understanding.md` 和 `metadata.json`。
4. 若已存在 approved metadata：
   - 重新从 `stub_fun.h` 抽取目标 signature。
   - 对比 signature hash。
   - 硬 stale 则停止正式测试，要求重新理解。
5. Agent 阅读 request、`understanding.md`、`metadata.json`，生成结构化 coverage plan。
6. `create_plan.py` 将 coverage plan 写入 SQLite，并保存 cursor。
7. `run_plan.py` 小批生成 case、小批执行 batch、写成功 samples、更新 case_results。

## 4. Instruction Understanding

理解结果进入 git，运行数据不进入 git。

```text
Auto_test/instructions/<instruction>/
  understanding.draft.md
  metadata.draft.json
  understanding.md
  metadata.json
```

`stub_fun.h` 只作为 signature 来源，不负责解释参数单位、合法范围、cache 行为和架构差异。语义必须由 Agent 阅读 CANN/AscendC header、已有笔记、已有测试和必要 smoke 验证后写入 `.md`。

`understanding.md` 固定章节：

```markdown
# <instruction> Understanding

## Status
## Selected Signatures
## Parameter Semantics
## Baseline Case
## Coverage Space
## Template Guidance
## Measurement Notes
## Architecture Notes
## Risks And Unknowns
## Sources
```

`understanding.md` 放：

- 参数含义、单位、合法范围。
- baseline case。
- 推荐 smoke/sweep 参数空间。
- 参数空间限制。
- runtime / compile-time binding 判断。
- template family 建议。
- measurement 注意事项。
- L2 bypass 等架构差异。
- 风险、unknown、source、confidence。

`metadata.json` 极简，只做 signature registry 和 approval 状态。它不放参数范围、默认值、baseline、推荐参数空间、测试策略、template family、binding 判断。

metadata 最小字段：

```json
{
  "instruction": "copy_gm_to_ubuf",
  "metadata_version": 1,
  "signatures": [
    {
      "signature_id": "global.copy_gm_to_ubuf....",
      "status": "approved_sweep",
      "namespace": "global",
      "source_file": "L2Bypass_test/stub_fun.h",
      "source_line": 1482,
      "return_type": "void",
      "params": [
        {"name": "dst", "type": "__ubuf__ void*", "role": "address", "space": "UB"},
        {"name": "src", "type": "__gm__ void*", "role": "address", "space": "GM"},
        {"name": "sid", "type": "uint8_t", "role": "scalar"},
        {"name": "nBurst", "type": "uint16_t", "role": "scalar"},
        {"name": "lenBurst", "type": "uint16_t", "role": "scalar"},
        {"name": "srcStride", "type": "uint16_t", "role": "scalar"},
        {"name": "dstStride", "type": "uint16_t", "role": "scalar"}
      ],
      "signature_hash": "..."
    }
  ]
}
```

metadata status：

```text
draft
approved_smoke
approved_sweep
stale
```

`approved_smoke` 只允许临时 smoke，不允许正式入库和 sweep。`approved_sweep` 才允许创建正式 coverage plan。

approve 粒度是具体 signature / overload，不是 instruction name。

## 5. Signature ID 和 Stale

`signature_id` 基于：

```text
namespace + instruction name + return type + ordered param types
```

参数名不参与 identity，只记录。

硬 stale 条件：

- return type 变化。
- namespace / name 变化。
- 参数数量变化。
- 参数类型顺序变化。

软 warning：

- 参数名变化。
- source line 大幅变化。
- CANN version 变化。

每次正式测试前都要从 `stub_fun.h` 重新抽取目标 signature，并和 approved metadata 中的 signature hash 对比。

namespace 进入 signature identity。全局函数和 `bisheng::cce::...` 版本第一版视为不同 signature；alias 只作为 `.md` 说明。

## 6. Request 和 Coverage Plan

自然语言用于驱动 Agent，但持久入口是 request 文件。

request 可以模糊：

```json
{
  "request_id": "copy_gm_to_ubuf_sweep",
  "instruction": "copy_gm_to_ubuf",
  "strategy": "sweep",
  "budget": {"max_new_cases": 1000},
  "target": {"arch": "dav_c220", "soc": "Ascend910B3"},
  "device": 1
}
```

request 可以只写 instruction；如果有多个 approved_sweep signature，Agent 必须让用户或 request 选择。coverage plan 必须精确到单个 `signature_id`。

一个 coverage plan 只覆盖一个 signature。一个 request 可以被 Agent 展开成多个 plan。

coverage plan 是脚本执行输入，必须结构化并通过 schema 校验。脚本不从 `.md` 推导参数空间；Agent 阅读 `.md` 后生成 plan。

coverage plan 第一版只进入正式数据库，strategy 只保留：

```text
sweep
exhaustive
```

smoke 是临时验证命令，不写 coverage_plans。

参数空间用结构化 JSON，不设计新 DSL。可以支持少量约定：

```json
{"values": [1, 2, 4, 8]}
{"range": [1, 64, 1]}
{"powers_of_two": [1, 1024]}
```

coverage 顺序必须 deterministic：

1. baseline case。
2. 边界值和常用值。
3. 单变量 sweep。
4. `.md` 指定的重要双变量小网格。
5. 剩余组合按 deterministic shuffled order。

shuffle / random 必须记录 seed 和 cursor，使中断后可以恢复。

## 7. SQLite

SQLite 默认路径：

```text
Auto_test/data/perf.sqlite3
```

所有脚本支持 `--db` 覆盖。

第一版加入极简 migration：

```text
Auto_test/migrations/001_initial.sql
schema_migrations(version, applied_at, description)
```

核心长期表：

```text
intrinsic_signatures
coverage_plans
coverage_cases
cases
samples
case_results
schema_migrations
```

不建立长期 `runs` 表。batch/run 是执行过程，不作为历史实体长期保存。

可以有临时 lease / active batch 状态用于执行恢复，但不作为性能历史数据。

## 8. Cases

`cases` 表一行表示一个正式 sweep 参数点，`test_id` 唯一。

`test_id` 表示“测什么参数点”，保持精简。

参与 `test_id`：

```text
arch/soc + signature_id + addr_args + scalar_args + memory_modes
```

不参与 `test_id`：

```text
CANN version
device_id
hostname
coverage_plan_id
run_id / batch_id
samples
warmups
inner_iters
template_version
measurement_config
timestamp
```

`coverage_plan_id` 不参与 `test_id`。同一个 `test_id` 可由多个 plan 生成，用 `coverage_cases(plan_id, test_id)` 关联。

case 状态：

```text
pending
running
done
failed
invalid
```

含义：

- `pending`：已生成，等待正式 sweep。
- `running`：当前执行 lease 中。
- `done`：已有成功 sweep sample，并更新聚合结果。
- `failed`：运行环境、模板或执行失败，可能重试。
- `invalid`：参数组合本身不合法，不应自动重试。

失败不写 samples，只更新 `cases.status/error_type/error_summary/updated_at`。第一版只在 validator 或明确错误能证明参数非法时标 `invalid`，其他先标 `failed`。

## 9. 参数表示

case 不存真实地址。

落库拆分：

```text
addr_args_json
scalar_args_json
memory_modes_json
```

`signature_id` 用于恢复完整 CCE 调用顺序。

request / plan 可以用参数名表达 scalar 参数；入库前按 signature 参数顺序 canonicalize 成 `scalar_args_json` 列表。参数名统一使用 `stub_fun.h` 原始名字，不引入 canonical alias。

真实 GM/UB 地址不参与 `test_id`。地址 spec 只记录影响性能语义的逻辑属性，例如：

- address space。
- role。
- logical tile / offset。
- alignment / offset_mod。
- overlap。

涉及 GM 搬运指令时，上层统一使用：

```text
l2_mode: use_l2 | bypass_l2
```

`l2_mode` 参与 `test_id`，保存到 `memory_modes_json`。模板按 arch lowering：

- dav_c220 / 2201：通过 GM address alias 实现 L2 bypass。
- dav_c310 / 3510：通过 raw instruction 的 `l2_cache_ctl` operand 实现 L2 bypass。

不使用 `hotness` 字段。所有 case 正式计时前统一 warmup 一次。

## 10. Samples 和 Results

`samples` 只保存成功的正式 sweep 原始样本。smoke 不入库，失败不入 samples。

`samples` 一行代表一次成功 sample：

```text
sample_id
test_id
sample_index
measured_at
cycles_total
inner_iters
cycles_per_iter
arch
soc
device_id
cann_version
template_hash
generator_hash
instruction_hash
plan_hash
measurement_config_json
```

`instruction_hash` 建议覆盖 `metadata.json + understanding.md`。`plan_hash` 覆盖结构化 coverage plan。这样不保存源码也能追踪当时执行依据。

`case_results` 是物化聚合表，可重算：

```text
test_id
sample_count
median_cycles_per_iter
min_cycles_per_iter
mad_cycles_per_iter
quality
updated_at
```

原始事实源是 `samples`。如果以后统计口径变化，可通过脚本重算 `case_results`。

## 11. Coverage 执行

第一版默认不是全量笛卡尔积，而是预算驱动的小批持续覆盖。

生成方式：

- 小批生成 case，例如 100 到 1000 个。
- 小批执行 case，例如 10 到 100 个。
- pending 队列不足时，根据 `coverage_plans.cursor_json` 继续生成。

非法参数组合尽量在生成前过滤。runner 只兜底标 `invalid`。被过滤组合第一版不逐条入库，只在 plan counters 中记录：

```text
generated_count
inserted_count
filtered_invalid_count
duplicate_count
```

batch 可以混合同一个 signature 下的多个 case，不混 signature。

batch 分组 key：

```text
signature_id + target + template_variant_key
```

如果 compile-time variant 不同，不能放进同一个 batch。

`template_variant_key` 不进入 `test_id`，但执行产生的样本通过 `template_hash/generator_hash/measurement_config_json` 保留实现信息。如果某个 compile-time arg 是 CCE 入参的一部分，它仍必须出现在 `scalar_args_json` 中参与 `test_id`。

## 12. 模板

模板按指令族组织，不强行一套模板覆盖所有 CCE 指令。

示例：

```text
Auto_test/templates/
  copy_gm_ub/
  vector_unary/
  vector_binary/
  scalar_or_misc/
```

模板选择、binding 判断、参数限制写在 `understanding.md` 或 coverage plan 中，不放进 metadata。

默认方向：

- batch kernel 从 GM 读取参数数组。
- 每个参数点独立 warmup 一次。
- kernel 输出每个 case 的 cycles。
- 必要时将某些参数提升为 compile-time variant。

如果没有合适模板，不进入正式 sweep。

## 13. 测量规则

每个 case 的 kernel 内部逻辑：

```text
warmup once with same params
pipe_barrier(PIPE_ALL)
start = GetSystemCycle()
repeat instruction inner_iters times
pipe_barrier(PIPE_ALL)
end = GetSystemCycle()
store cycles
```

主指标：

```text
cycles_per_iter
```

同时保存：

- cycles_total。
- inner_iters。
- sample_index。
- cycle counter / conversion 信息放在 `measurement_config_json`。

正式 sweep 应保存多次成功 sample，并在 `case_results` 中聚合 median、min、MAD/CV 和 quality。

## 14. Smoke

smoke 是 understanding 阶段的临时验证：

- 可以验证 signature、模板、少量保守参数是否能编译运行。
- 不写 `cases`。
- 不写 `samples`。
- 不写 `case_results`。
- 不写 `coverage_plans`。

`approved_smoke` 表示允许临时 smoke，但不允许正式 sweep。只有 `approved_sweep` 可以创建正式 coverage plan 并写 SQLite。

## 15. Artifact 策略

默认不保存任何生成源码，包括失败。

默认不保存完整 stdout/stderr、binary。

默认长期保存：

- case 参数和 `test_id`。
- 成功 samples。
- case_results。
- cases 上的最后错误摘要。
- build/run command 可放在临时日志或 debug 输出，不作为长期 run 表。
- template/generator/instruction/plan hash。
- CANN version 等环境字段。

完整源码、完整日志、binary 只有显式 debug 开关才保存。

## 16. 脚本

第一版使用多个小脚本，不先做统一大 CLI：

```text
init_db.py
extract_signatures.py
create_plan.py
run_plan.py
query_results.py
```

`extract_signatures.py` 默认输出 JSON / stdout 给 Agent 使用，不污染 DB。显式 `--import-db` 时才写 SQLite。

`signatures.json` 不作为长期文件。approved 后以 `metadata.json` 为准。

三类 schema 都需要：

```text
instruction_metadata.schema.json
request.schema.json
coverage_plan.schema.json
```

metadata schema 最小校验 signature registry。request schema 宽松校验入口任务。coverage_plan schema 严格校验脚本执行输入。

## 17. 维护和同步

第一版不实现自动抽样重测或 regression 检查，但数据模型允许同一个 `test_id` 后续追加新的成功 samples。

第一版不实现云端上传或多机 merge，只预留导出能力。

理解文档、metadata、schema、模板、SOP、可复用 request 进 git。SQLite、运行中 plan、临时数据不进 git。

## 18. 第一版实施顺序

建议按以下顺序落地：

1. 建立 `Auto_test/` 目录结构和 `.gitignore`。
2. 写三类 schema：metadata、request、coverage_plan。
3. 写 `docs/` SOP：workflow、understanding、request、schema、template。
4. 写 `migrations/001_initial.sql` 和 `init_db.py`。
5. 写 `extract_signatures.py`，先支持按 instruction 从 `stub_fun.h` 抽取 overload。
6. 选一个现有指令做样例 understanding，例如 `copy_gm_to_ubuf`。
7. 写一个最小 `copy_gm_ub` template family。
8. 写 `create_plan.py`，支持从结构化 coverage plan 小批生成 case。
9. 写 `run_plan.py`，支持同 signature batch、warmup、GetSystemCycle、写 samples/case_results。
10. 写 `query_results.py`，输出某 instruction/signature 的 summary。
