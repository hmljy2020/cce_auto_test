# Database Format

The first database version is intentionally minimal. It keeps one current result
for each unique benchmark case.

## Core Table

Use one core table:

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

## Unique Case Identity

One row represents one unique benchmark case and its current result.

The unique case identity is:

```text
arch_version
soc_version
cce_name
signature
cce_args_json
extra_params_json
```

The following fields do not participate in case identity:

```text
template_id
time_ns
updated_at
```

If the same case is tested again, overwrite the existing result.

## Environment Fields

`arch_version` and `soc_version` are plain text fields. The first version does
not enforce enum values or naming rules.

`device_id` is not stored.

`cann_version` is not stored.

## Signature

`cce_name` must exactly match the `instruction` field in metadata.

`signature` stores the complete CCE function signature string from metadata. It
must exactly match one `overloads[].signature` value and is used directly to
distinguish overloads.

Database writers must copy the canonical metadata signature exactly. They should
not reformat or re-canonicalize it during insertion.

Example:

```text
void copy_gm_to_ubuf(__ubuf__ void* dst, __gm__ void* src, uint8_t sid, uint16_t nBurst, uint16_t lenBurst, uint16_t srcStride, uint16_t dstStride);
```

## CCE Arguments

`cce_args_json` stores all direct CCE function arguments as one list in complete
signature declaration order.

Address arguments occupy their original argument positions. For address
arguments, store only alignment information. Do not store real addresses,
address spaces, offsets, or overlap information.

Address argument objects must use exactly this shape:

```json
{"alignment": 32}
```

The only supported key is `alignment`, and its value must be an integer.

For this declaration:

```cpp
void copy_gm_to_ubuf(__ubuf__ void* dst, __gm__ void* src, uint8_t sid, uint16_t nBurst, uint16_t lenBurst, uint16_t srcStride, uint16_t dstStride);
```

`cce_args_json` stores:

```json
[
  {"alignment": 32},
  {"alignment": 512},
  0,
  1,
  32,
  0,
  0
]
```

The list corresponds to:

```text
dst, src, sid, nBurst, lenBurst, srcStride, dstStride
```

For instructions without arguments:

```json
[]
```

## Extra Parameters

`extra_params_json` stores performance-relevant controls that are not direct CCE
function parameters. The keys may differ between instructions.

For the first version, supported common keys are:

- `l2_mode`: use `"use_l2"` or `"bypass_l2"`.
- `set_mask`: use an integer value.

If an instruction does not involve a given extra control, omit that key.

Examples:

```json
{}
```

```json
{
  "l2_mode": "bypass_l2"
}
```

```json
{
  "set_mask": 128
}
```

```json
{
  "l2_mode": "use_l2",
  "set_mask": 64
}
```

## JSON Canonicalization

All JSON fields must be canonicalized before writing to SQLite. SQLite compares
the JSON fields as text for uniqueness.

Rules:

- Use compact JSON without extra whitespace.
- Sort object keys lexicographically.
- Keep lists in their semantic order.
- Store empty objects as `{}`.
- Store empty lists as `[]`.
- Keep numeric values as numbers, not strings.

For `cce_args_json`, the list order must strictly follow the CCE function
argument order in `signature`.

## Result Fields

`template_id` stores the test code template identifier used for the measurement.
It should be a hash-like text value. It does not participate in case identity.

`time_ns` stores the average time for one CCE instruction execution, in
nanoseconds, rounded to one decimal place.

When using `GetSystemCycle`, compute:

```text
time_ns = round((end_cycle - start_cycle) * 20.0 / iter, 1)
```

`iter` is used to compute `time_ns`, but it is not stored.

Do not store raw start cycles, end cycles, total cycles, per-iteration cycles,
or `iter`.

## Update Policy

The database keeps only one result per unique case. Re-testing the same case
should overwrite:

```text
template_id
time_ns
updated_at
```

Only successful measurements are stored. Failed cases are not inserted into the
database.

## Excluded Features

The first database version does not include:

- `device_id`.
- `cann_version`.
- `iter`.
- failed case records.
- raw start cycles, end cycles, total cycles, or per-iteration cycles.
- schema migration or schema version tables.
- extra query indexes beyond SQLite's automatically created unique index.
