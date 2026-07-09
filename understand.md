# Understand Stage

This document is the entry point for the CCE instruction understand stage. The
stage identifies the target CCE instruction/signature, creates draft instruction
documents, asks the user to confirm them, and promotes confirmed drafts to
formal documents.

## Source Files

- `understand.md`: understand-stage workflow and handoff rules.
- `cce_docs/README.md`: metadata and `understanding.md` document formats.
- `database/README.md`: database case identity and JSON representation rules.
- `cce_reference/stub_fun.h`: source of raw CCE declarations.

## Workflow

1. Read `cce_docs/README.md`, `database/README.md`, and
   `cce_reference/stub_fun.h`.
2. Find all overloads for the requested instruction in `stub_fun.h`.
3. If there is one overload, create drafts for that signature. If there are
   multiple overloads, list candidates, summarize their differences, recommend a
   target overload, and ask the user to confirm which overloads to understand.
4. Check `cce_docs/<instruction>/metadata.json` and
   `cce_docs/<instruction>/understanding.md` if they already exist. Reuse
   existing confirmed content and only draft missing or changed signatures.
5. Write drafts only:
   - `cce_docs/<instruction>/metadata.draft.json`
   - `cce_docs/<instruction>/understanding.draft.md`
6. Stop and report the found signatures, selected/recommended overloads,
   uncertain parameter or extra-control semantics, and recommended maximum
   overload status.
7. After user confirmation, promote confirmed drafts to:
   - `cce_docs/<instruction>/metadata.json`
   - `cce_docs/<instruction>/understanding.md`

## Draft Rules

- Generate canonical signatures from `stub_fun.h`; do not invent or manually
  reformat signatures.
- `metadata.draft.json` must follow `cce_docs/README.md`.
- `understanding.draft.md` must follow `cce_docs/README.md`.
- DB representation tokens and JSON examples must follow `database/README.md`.
- Initial overload status should be `draft` unless the user explicitly confirms
  `approved_smoke` or `approved_sweep`.

## Existing Documents

When formal documents already exist, read them before drafting. If the requested
signature is already present, summarize the existing understanding and draft
only necessary corrections. If the instruction exists but the signature is
missing, draft an additive update for that signature.

## Confirmation

The Agent must not promote draft files to formal files without user
confirmation. Promotion means replacing or creating `metadata.json` and
`understanding.md` from the confirmed draft content.
