# Database Format

The database stores one current successful result for each unique benchmark
case.

## Measurements Table

```sql
CREATE TABLE measurements (
  id INTEGER PRIMARY KEY AUTOINCREMENT,

  arch_version TEXT NOT NULL,
  soc_version TEXT NOT NULL,
  cce_name TEXT NOT NULL,
  signature TEXT NOT NULL,

  cce_args_json TEXT NOT NULL,
  extra_params_json TEXT NOT NULL,

  template_id TEXT NOT NULL,
  time_ns REAL NOT NULL,
  updated_at TEXT NOT NULL,

  UNIQUE (
    arch_version,
    soc_version,
    cce_name,
    signature,
    cce_args_json,
    extra_params_json
  )
);
```

## Case Identity

The unique case identity is `arch_version`, `soc_version`, `cce_name`,
`signature`, `cce_args_json`, and `extra_params_json`. Re-testing the same case
overwrites `template_id`, `time_ns`, and `updated_at`.

`arch_version` and `soc_version` are plain text. `device_id` and `cann_version`
are not stored.

## Signature Fields

`cce_name` must exactly match `metadata.instruction`. `signature` must exactly
match one `metadata.overloads[].signature`; database writers must copy it
without reformatting.

## DB Representation Tokens

Allowed tokens are `{"alignment":<int>}`, `<int>`, `<float>`, `<boolean>`, `<string>`, `<enum-int>`, and `<bitfield-int>`. Direct CCE arguments must all appear in `cce_args_json`; address arguments use only `{"alignment":<int>}`; extra controls must not use the alignment object; `<derived>` and `not stored` are not valid DB representations.

## CCE Args JSON

`cce_args_json` stores all direct CCE function arguments as one compact JSON
list in complete signature order. Address arguments occupy their original
positions and store only alignment information; real addresses, address spaces,
offsets, and overlap information are not stored.

Example:

```json
[{"alignment":32},{"alignment":512},0,1,32,0,0]
```

For instructions without arguments:

```json
[]
```

## Extra Params JSON

`extra_params_json` stores performance-relevant controls that are not direct CCE
function parameters. Every key must be defined by the matching signature's
`Extra Controls` section in `understanding.md`.

Common first-version keys are `l2_mode` (`"use_l2"` or `"bypass_l2"`) and
`set_mask` (integer). They are not an exhaustive global whitelist; confirmed
instruction-specific keys are allowed.

Examples:

```json
{}
```

```json
{"l2_mode":"bypass_l2"}
```

```json
{"set_mask":128}
```

```json
{"l2_mode":"use_l2","set_mask":64}
```

## JSON Canonicalization

SQLite compares JSON fields as text for uniqueness. Store compact JSON without
extra whitespace, sort object keys lexicographically, keep lists in semantic
order, store empty objects as `{}` and empty lists as `[]`, and keep numeric
values as numbers.

## Result Fields

`template_id` stores the test code template identifier and does not participate
in case identity.

`time_ns` stores the average time for one CCE instruction execution, in
nanoseconds, rounded to one decimal place:

```text
time_ns = round((end_cycle - start_cycle) * 20.0 / iter, 1)
```

`iter` is used only for this calculation. Raw cycles, per-iteration cycles,
failed cases, and `iter` are not stored.
