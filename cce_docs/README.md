# CCE Instruction Documents

This directory stores per-instruction documents produced by the understand
stage.

## Directory Layout

```text
cce_docs/
  <instruction>/
    metadata.draft.json
    understanding.draft.md
    metadata.json
    understanding.md
```

Draft files are used while the Agent is organizing content or waiting for user
confirmation. Formal benchmarking reads only `metadata.json` and
`understanding.md`.

Draft filenames and overload status are separate concepts. Smoke validation may
read draft files, but the selected overload must still have
`status=approved_smoke` or `status=approved_sweep`. An overload with
`status=draft` must not be used for smoke or formal benchmarking.

## Metadata

Metadata is intentionally minimal:

```json
{
  "instruction": "copy_gm_to_ubuf",
  "metadata_version": 1,
  "overloads": [
    {
      "status": "approved_sweep",
      "signature": "void copy_gm_to_ubuf(__ubuf__ void* dst, __gm__ void* src, uint8_t sid, uint16_t nBurst, uint16_t lenBurst, uint16_t srcStride, uint16_t dstStride);"
    }
  ]
}
```

Allowed `status` values:

- `draft`: recorded but not confirmed; not allowed for smoke or formal sweep.
- `approved_smoke`: allowed for temporary smoke validation only.
- `approved_sweep`: allowed for formal micro-benchmarking and database writes.

`signature` must be the canonical one-line declaration generated from
`cce_reference/stub_fun.h`, terminated by `;`. Database writers must copy
`metadata.overloads[].signature` exactly.

Do not hand-write derived metadata fields such as `signature_id`, parameter
lists, return type, or signature hash. Scripts should derive them from
`signature`.

## Understanding

`understanding.md` records human-reviewed argument semantics and extra controls
for one instruction.

Top-level structure:

```markdown
# <instruction> Understanding

## Signature: `<canonical signature from metadata>`

### Signature Overview
### Parameters
### Extra Controls
### Sources
```

If an instruction has multiple overloads, repeat the full `## Signature:` block
for each overload. The document top level should contain only the title.

## Signature Block

The signature heading must copy `metadata.overloads[].signature` exactly, on one
line, with the trailing `;`, wrapped in Markdown inline code.

### Signature Overview

Write one or two sentences describing what this signature does.

### Parameters

```markdown
| Order | Name | Type | Role | Unit | DB Representation | Notes |
|---:|---|---|---|---|---|---|
| 0 | dst | `__ubuf__ void*` | address:dst:UB | bytes | `{"alignment":<int>}` | Real address is allocated by the template. |
```

The table must contain exactly one row for every direct CCE function argument in
signature order. `Name` and `Type` must match the canonical signature. `DB
Representation` tokens are defined by `database/README.md`.

Allowed `Role` values: `address:src:GM`, `address:dst:GM`, `address:src:UB`,
`address:dst:UB`, `address:src:L1`, `address:dst:L1`, `scalar`, `length`,
`stride`, `repeat`, `config`, `enum`, `immediate`, `derived`, `unknown`.

Allowed `Unit` values: `none`, `bytes`, `elements`, `blocks`, `bits`,
`boolean`, `enum`, `unknown`, `instruction-specific:<name>`.

`Unit` describes the value unit, not the parameter role. Notes may describe
parameter semantics and validity constraints needed to generate correct template
code. If required semantics are unknown, write `unknown` or `TBD` and keep the
overload status as `draft` until the user confirms otherwise.

### Extra Controls

Extra controls are non-CCE arguments that affect benchmark case identity and are
stored in `extra_params_json`.

```markdown
| Name | Values | DB Key | DB Representation | Notes |
|---|---|---|---|---|
| l2_mode | `use_l2`, `bypass_l2` | `l2_mode` | `<string>` | Controls L2 behavior for GM-related access when applicable. |
```

If there are no extra controls, write `None.`

`DB Key` must exactly equal the key written to `extra_params_json`. `DB
Representation` tokens are defined by `database/README.md`. If a control cannot
be represented in `extra_params_json`, or if `Values` is `TBD`, keep the
overload status as `draft` until the user confirms the representation.

### Sources

Use a short list. Each signature section must have at least one source.

```markdown
- `cce_reference/stub_fun.h`: signature only.
- user confirmation `2026-07-09`: confirms parameter roles and extra controls.
```
