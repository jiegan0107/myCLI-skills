---
name: review-commits
description: >
  Multi-phase Linux kernel code review for local commits (Mode A), lore.kernel.org
  patch series fetched with b4 (Mode B), or a single source file (Mode C). Runs
  checkpatch, W=1 build, dt_binding_check, and sparse; audits kernel coding style
  (indentation, naming, macros, error paths); maps per-function control-flow,
  data-flow, and state lifecycle; checks DT/DT-binding schemas, DTS nodes, and
  driver of_match consistency; enforces the single-responsibility rule, subject-line
  quality, and series bisectability. Findings tagged [BUG], [CONCERN], [MINOR], or
  [NIT]; saved as a structured review file with an Overall Summary block followed by
  per-file or per-commit detail sections. Use when asked to review recent commits, a
  patch series, a pull request diff, a kernel patch Message-ID from lore.kernel.org,
  or a specific source file in a project.
author: "Jie Gan <jiegan@qti.qualcomm.com>"
---

# Review Commits

## Overview

Automated code review for Linux kernel commits and patch series. Runs standard
kernel test tools, then produces a structured per-commit review with severity-
tagged findings and an overall recommendation.

## Mode Selection

| Inputs provided | Mode |
|---|---|
| `Project path` + `Number of commits` | **A** — review last N local commits |
| `Project path` + `Message-ID` | **B** — fetch, apply, and review a lore.kernel.org patch series |
| `Project path` + `File path` | **C** — review a single source file as-is in the working tree |

If neither is clear, ask the user before proceeding.

## Step 0 — Sync Repository and Create Review Branch

Before obtaining any commits, ensure the local repository is up to date.

```bash
cd <project_path>

# Pull latest HEAD from the tracked remote branch
git pull
```

- If `git pull` reports conflicts or a non-fast-forward situation, stop and
  report the error to the user before proceeding.
- The review branch is created later: Mode B creates it during `git am` in
  Step 1; Modes A and C do not require a dedicated branch.

## Step 1 — Obtain the Commits

### Mode A

```bash
cd <project_path>
git log --oneline -<N> HEAD
git show <hash>          # repeat for each commit
```

### Mode B

```bash
cd <project_path>
b4 --version   # confirm b4 is available; print version for the record
               # if this fails, report the error and stop
mkdir -p <project_path>/tmp
b4 am <message-id> 2>&1 | tee <project_path>/tmp/b4_output.txt
```

If `b4 am` exits non-zero: print `<project_path>/tmp/b4_output.txt` and **stop**.

Parse `<project_path>/tmp/b4_output.txt` for:
- **Total patches** — integer after `Total patches:`
- **mbx filename** — filename on the `git am ./` line
- **Base commit** — hash on the `git checkout -b` line (use `HEAD` if absent)
- **Cover letter** — read `./<slug>.cover` if it exists (context only)

If no `Total patches:` line despite exit 0: report full output and **stop**.

**Cover letter dependency check (mandatory — do this before applying patches):**

Read `./<slug>.cover` immediately after `b4 am` succeeds.  Scan the cover
letter for any stated external dependencies — these appear as any of:

- `Depends-on:` pseudo-tag followed by a Message-ID or URL
- Prose such as "this series depends on", "requires", "based on", "on top of",
  "prerequisite", or "applies after" followed by a series title, Message-ID,
  or lore.kernel.org URL
- A `Link:` or `https://lore.kernel.org/` reference in the cover letter body
  that is described as a prerequisite (not merely a related discussion)

For each dependency found:

1. **Extract the identifier** — Message-ID, commit hash, or lore URL.
2. **Check whether it is present in the base commit:**
   ```bash
   git log --oneline <base-commit> | head -20
   # or, if a commit hash is given:
   git merge-base --is-ancestor <dep-hash> <base-commit> && echo "present" || echo "MISSING"
   ```
3. **If the dependency is present**: note it as satisfied in the review header
   and proceed normally.
4. **If the dependency is missing or cannot be verified**:
   - Record it as `DEPENDENCY MISSING` in the review header card (add a
     `Dependencies` row to the header table).
   - Add a `[CONCERN] Missing prerequisite` finding to the verdict banner
     with the dependency identifier and a note that findings in this review
     may be incorrect or incomplete until the prerequisite is applied.
   - Do **not** stop the review — continue and flag individual findings that
     appear to be caused by the missing dependency with the note
     `"May be resolved by prerequisite: <identifier>"`.
   - Do **not** dismiss real bugs just because a dependency is missing —
     only flag findings where the missing code is the direct cause.

If no cover letter exists or no dependencies are mentioned, record
`Dependencies: none stated` in the header and proceed.

Apply patches:
```bash
git checkout -b <branch> [<base-commit>]
git am ./<slug>.mbx
```

If the `git checkout -b` line is absent from `b4 am` output, use the branch name
`review/<slug>` where `<slug>` is derived from the Message-ID (the same slug used
for the filename in Step 6).

If `git am` fails: run `git am --abort`, report the conflict, and **stop**.
Do NOT fall back to reading patches from the mbx file directly and continuing
the review.  Reading multiple patches in one batch collapses the per-patch
before/after boundary, making inter-patch bisectability violations invisible:
a write removed in patch N and restored in patch N+1 looks correct when both
are read together, but the tree is broken between them.  The only safe review
path requires each patch applied as a discrete commit.

Confirm with `git log --oneline HEAD~<N>..HEAD`, then proceed to Step 2.
**Patch count check (mandatory):** count the commits in that log output and
compare against `Total patches:` from `b4 am`.  If the counts differ, `b4 am`
silently dropped patches (e.g. threaded replies misidentified as non-patches).
Stop, report the discrepancy with the exact counts, and ask the user how to
proceed — do not review a partial series as if it were complete.
Do **not** delete `.mbx` / `.cover` yet — they may be needed for context during
the review.  Cleanup happens after the review file is written (see Step 5).

### Mode C

```bash
cd <project_path>
cat <file_path>          # read the full file
wc -l <file_path>        # note total line count
```

- `<file_path>` may be absolute or relative to `<project_path>`.
- If the file does not exist, report the error and **stop**.
- Read the file in full before proceeding to Step 2.
- Also read related headers, Kconfig, and Makefile entries that reference
  the file (same rules as Step 2 for Mode A/B).
- There are no commits to review; skip all commit-message and patch-scope
  checks (Steps 3e, 4 Patch Scope column).  Apply all other review steps
  (coding style, logic mapping, DT/DT-binding if applicable, build, sparse).
- The review is structured as a single **per-file block** (not per-commit).

## Step 2 — Gather Context

**Tree state discipline (mandatory):** The working tree after all patches are
applied reflects the *final* state, not the state after any individual patch.
A later patch may modify, extend, or fix something introduced by an earlier
patch; reviewing from the final tree makes those inter-patch deltas invisible
and produces incorrect findings for the earlier patch.  For every patch N in a
series of total T patches, the diff AND all surrounding context must be read
from the tree state immediately after patch N is applied — never from the final
tree.

**Per-patch review procedure (one iteration per patch, in order 1 → T):**

**BATCHING PATCHES IS FORBIDDEN.**  Every patch must be reviewed individually
at its own tree state.  Grouping multiple patches into a single commit block
(e.g. "Patches 1–6") is a violation of this rule regardless of how similar or
preparatory the patches appear.  A 39-patch series requires 39 separate commit
blocks, each produced after the corresponding checkout below.

For each patch N from 1 to T (inclusive), execute this sequence in full before
moving to patch N+1:

```bash
# Step A — move to the correct tree state
git checkout HEAD~$((T - N))

# Step B — mandatory self-audit: confirm HEAD is now patch N
git log --oneline -1
# Record the hash and subject printed here.
# If the hash does not match the expected patch N hash from the series log,
# STOP and report the mismatch — do not proceed with a wrong tree state.

# Step C — obtain patch N's own diff
git show HEAD
# This is the authoritative source for what patch N changed.
# Do NOT use git show <hash> without the checkout above — surrounding
# files would still reflect the wrong tree state.

# Step D — read surrounding context files for patch N
# (headers, Kconfig, Makefile, Documentation/ABI/ as applicable)
```

After writing the commit block for patch N, advance to patch N+1:
```bash
git checkout HEAD~$((T - N - 1))
# Then repeat Step B–D for patch N+1.
```

**Self-audit rule (mandatory):** The hash printed by `git log --oneline -1`
in Step B must be recorded in the commit block header.  If at any point the
recorded hash does not match the patch being reviewed, the review of that patch
is invalid and must be redone from Step A.  This makes tree-state errors
self-evident rather than silently producing contaminated findings.

**Why `git show HEAD` must be run after the checkout, not before:**
Running `git show HEAD` before the checkout gives the diff of whichever commit
HEAD currently points to, which may be a later patch.  The checkout moves HEAD
to the correct commit; only then does `git show HEAD` return patch N's diff.
Never use `git show <hash>` as a shortcut without also checking out that commit
first — the surrounding files would still reflect the wrong tree state.

**Inter-patch contamination check:** Before writing any finding for patch N,
explicitly verify that the code under review was introduced by patch N and not
by a later patch.  If a function or variable visible in `git show HEAD` for
patch N is also modified by patches N+1..T, note the dependency in the per-commit
Code Logic Maps section so the reviewer of the later patch can cross-reference.

When the series has only one patch, or when reviewing Mode A local commits,
the same rule applies: check out the commit being reviewed before reading
context files or obtaining its diff.

For every changed file (at the correct tree state per above):
- Read the full diff (`git show HEAD`)
- Read surrounding code for patterns, naming conventions, locking models
- Check related headers for struct definitions and macros
- Check `Kconfig` / `Makefile` if new files or config symbols are added
- Check `Documentation/ABI/` if sysfs/debugfs nodes are added

## Step 3 — Run Automated Tests

Run all tests **before** writing the review. Mark unavailable tools as `SKIP`.

### 3.1 checkpatch

```bash
mkdir -p <project_path>/tmp/review_patches
git format-patch HEAD~<N>..HEAD --output-directory <project_path>/tmp/review_patches/
for patch in <project_path>/tmp/review_patches/*.patch; do
    scripts/checkpatch.pl --strict "$patch"
done
find <project_path>/tmp/review_patches -name "*.patch" -delete
rmdir <project_path>/tmp/review_patches <project_path>/tmp 2>/dev/null || true
```

Record all `ERROR:` and `WARNING:` lines per patch verbatim.

### 3.2 Build (W=1)

Identify affected subsystem dirs from the diffs, then:
```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- W=1 <affected_dir>/
```

For patches touching arch-neutral subsystems (e.g. `drivers/`, `fs/`, `mm/`),
also run without `ARCH=arm64` to catch x86_64-specific warnings:
```bash
make W=1 <affected_dir>/
```

Skip if `.config` is absent. Record only `error:` / `warning:` lines in
files touched by the commits.

### 3.3 DT binding check

Only when `Documentation/devicetree/bindings/` `.yaml` files are changed:
```bash
make ARCH=arm64 DT_SCHEMA_FILES=<changed.yaml> dt_binding_check
```

### 3.4 Sparse

```bash
make ARCH=arm64 C=1 CF="-D__CHECK_ENDIAN__" <affected_dir>/
```

Mark result as `SKIP` in the test table only when the `sparse` tool is not installed
(`which sparse` returns non-zero); never skip by design.

### 3.5 Maintainer check

```bash
for patch in <project_path>/tmp/review_patches/*.patch; do
    scripts/get_maintainer.pl "$patch"
done
```

Record the output in the test summary table with result `INFO`.  This is
informational (not pass/fail) but reminds patch authors of the correct
`To:` / `Cc:` recipients before sending.

### 3.6 Test summary table

Output before the per-commit review:
```
| Test             | Result | Notes                              |
|------------------|--------|------------------------------------|
| checkpatch       | PASS   | 0 errors, 2 warnings (see below)   |
| Build (W=1)      | PASS   | No new errors or warnings          |
| dt_binding_check | SKIP   | No .yaml files changed             |
| sparse           | SKIP   | sparse not available               |
| get_maintainer   | INFO   | To/Cc list (see below)             |
```

List checkpatch and build findings in full beneath the table.

## Step 3b — Kernel Coding Style Checklist

Apply **every rule below** to all changed `.c` / `.h` files before writing the
review.  These rules come directly from `Documentation/process/coding-style.rst`
(the authoritative upstream reference).  Flag any violation with the appropriate
severity tag.

### Indentation & Whitespace
- Tabs (hard, 8-column) for indentation — never spaces.
- No trailing whitespace on any line.
- Lines ≤ 80 columns; tolerate up to 100 only when breaking would hurt
  readability (e.g. long string literals, function signatures).
- One blank line between logical blocks inside a function; two blank lines
  between top-level definitions.
- No blank line immediately after an opening brace or before a closing brace.

### Braces & Spacing
- Opening brace at end of line for all compound statements
  (`if`, `for`, `while`, `switch`, function bodies).
- Closing brace on its own line, except `} else {` and `} while (…);`.
- Single-statement `if`/`for`/`while` bodies: **no braces** unless the
  companion branch uses braces.
- Space after keywords: `if (`, `for (`, `while (`, `switch (`. Use `return expr;`
  without parentheses unless the expression wraps across lines; do not flag
  `return x;` as a style violation.
- No space between function name and `(`: `foo(args)` not `foo (args)`.
- Space around binary operators; no space after unary operators.
- No space inside parentheses: `(x + y)` not `( x + y )`.

### Naming
- `lower_case_with_underscores` for variables, functions, and file-scope
  symbols — never camelCase.
- `UPPER_CASE` for macros and `#define` constants.
- Descriptive names; avoid single-letter names except loop counters (`i`, `j`).
- No Hungarian notation or type-encoding prefixes.
- Typedefs only for opaque types and function-pointer types; never typedef a
  plain struct just to avoid writing `struct`.

### Functions
- Functions should do one thing and fit on one or two screens (~50–70 lines).
- Maximum ~5–7 local variables; more is a sign the function should be split.
- Exit via a single `return` at the bottom when using `goto`-based cleanup.
- `goto` labels flush-left, named after what they undo (e.g. `err_free_buf:`).

### Comments
- Block comments: `/* … */` style; `//` line comments are acceptable in C99
  kernel code but block style is preferred for multi-line.
- Comment *why*, not *what* — the code already says what.
- No commented-out code in submitted patches.
- Function kerneldoc (`/**`) required for exported (`EXPORT_SYMBOL`) functions.

### Macros & Preprocessor
- Macros with multiple statements wrapped in `do { … } while (0)`.
- Macro arguments parenthesised: `#define SQ(x) ((x) * (x))`.
- Prefer `static inline` functions over function-like macros where possible.
- `#if`/`#ifdef` blocks indented with a space after `#`:
  `#  ifdef CONFIG_FOO` inside nested blocks.

### Data Structures & Types
- Use kernel types (`u8`, `u16`, `u32`, `u64`, `s32`, etc.) for fixed-width
  fields; use `int`/`long` for general-purpose values.
- Avoid `bool` in structures shared with userspace or hardware.
- Bit-fields only for hardware register maps; never rely on their layout.
- `sizeof(var)` preferred over `sizeof(type)` to stay type-safe.

### Error Paths & Resource Management
- Check every allocation / API return value.
- Use `devm_*` helpers where the driver model supports it.
- Unwind in reverse-init order using `goto` labels.
- Return negative `errno` values (`-ENOMEM`, `-EINVAL`, …); never positive
  error codes from kernel functions.

### Commit Message Style
- Subject line: imperative mood, ≤ 72 characters, no trailing period.
- Blank line between subject and body.
- Body wraps at 72 characters; explains *why* the change is needed.
- Correct tag order: `Fixes:`, `Link:`, `Cc:`, `Reported-by:`, `Tested-by:`,
  `Reviewed-by:`, `Signed-off-by:`.
- `Fixes:` tag format: `Fixes: <12-char-hash> ("subject")`.
- **Single responsibility**: one patch must address exactly one issue or add
  exactly one feature.  A patch that fixes multiple independent bugs, or that
  mixes a bug-fix with a clean-up or a new feature, violates upstream kernel
  policy and must be flagged as `[CONCERN]`.  Apply Step 3e for the full
  patch-scope checklist.

## Step 3c — Code Logic Mapping

Before writing any review findings, build a complete picture of the logic
introduced or changed by the patch.  Do this for **every function** that is
added or modified.  The goal is to understand *what the code does* before
judging *whether it does it correctly*.

### 3c.1 Control-Flow Picture

For each changed function, trace every execution path through the diff:
- Identify all entry conditions and guard clauses (`if`, early `return`,
  `goto`).
- Map the happy path from entry to successful return.
- Map every error / exceptional path: what triggers it, what cleanup it
  performs, and where it exits.
- Note any loops: loop variable, termination condition, loop body side-effects,
  and whether the loop can run zero times.
- Note any `fallthrough` in `switch` statements and confirm it is intentional.
- **Diagnostic coverage check**: enumerate every `dev_err` / `dev_warn` /
  `pr_err` call site and the exact set of conditions that reaches it.  Confirm
  no two call sites fire for the same condition (i.e. a branch that already
  emits its own message must not fall through into a second generic message for
  the same event).  Flag redundant diagnostics as `[MINOR]` — duplicate messages
  in `dmesg` are a real usability problem and harder to bisect; `[NIT]` understates
  the impact.

Produce a plain-text control-flow summary per function, e.g.:
```
foo_init():
  1. Validate args → return -EINVAL if NULL
  2. Allocate buf (kzalloc) → goto err_alloc on failure
  3. Register device → goto err_reg on failure
  4. Return 0
  err_reg:  free buf
  err_alloc: return -ENOMEM
```

### 3c.2 Data-Flow Picture

For each changed function, trace how data moves:
- Identify all inputs (parameters, global/static state, hardware registers,
  user-space buffers).
- Follow each input through transformations (arithmetic, bitwise ops, casts,
  struct field assignments).
- Identify all outputs (return value, pointer writes, global/static state
  mutations, hardware register writes, callbacks invoked).
- Flag any input that reaches a sensitive sink (memory allocation size,
  array index, copy_to/from_user length, hardware register) without
  validation — these are prime correctness and security targets.

#### Register read data-flow checklist

Apply whenever a function reads a hardware register (`readl_relaxed`,
`readl`, `readq`, `ioread32`, etc.):

- **Width match**: is the local variable type wide enough for the register?
  Use `u32` for 32-bit registers, `u64` for 64-bit.  A narrower type
  silently truncates bits; a signed type (`int`) is a style violation and
  can cause subtle bugs if the value is later used in signed arithmetic or
  widened to a 64-bit signed type.
- **Sign match**: trace every use of the register value after the read.
  If the value is stored in a signed type, check whether it is ever:
  (a) used in signed arithmetic or a signed comparison, or
  (b) widened to a larger signed type (e.g. assigned to `long` or `s64`).
  If neither applies, the mismatch is a style issue (`[NIT]`), not a bug —
  C99 §6.3.1.3 guarantees that converting a negative `int` back to `u32`
  preserves the bit pattern.  Only file `[BUG]` if a signed intermediate
  value reaches signed arithmetic, a signed comparison, or sign-extension
  across a wider type.
- **Field extraction**: are `FIELD_GET()` / `BMVAL()` / shift+mask used
  correctly?  Confirm the mask covers exactly the documented bit range and
  the shift matches.
- **Write-back (read-modify-write)**: confirm the read and write use the
  same register offset, and that reserved bits are masked off before OR-ing
  new values to avoid corrupting them.
- **Endianness**: confirm `readl_relaxed` / `writel_relaxed` (little-endian
  MMIO) is appropriate for the bus.  Flag if `__raw_readl` or `ioread32be`
  would be needed instead.
- **Barrier sufficiency**: confirm `readl_relaxed` (no ordering guarantee)
  is appropriate at this call site, or whether `readl` (implicit barrier)
  is required.

### 3c.3 State-Machine / Lifecycle Picture

When the patch touches objects with a lifecycle (devices, buffers, locks,
reference counts, state flags):
- List all states the object can be in and the transitions the patch adds or
  modifies.
- Confirm every transition is guarded by the correct lock.
- Confirm reference counts are incremented before use and decremented on every
  exit path.
- Confirm no state transition is reachable from an uninitialised or
  already-freed object.
- **For every boolean/enum flag that gates an error return or a branch**:
  trace all writers of that flag across all functions and files, and confirm
  whether the flag can actually be in the assumed state at the call site in
  question.  Paired prepare/unprepare functions often enforce invariants
  implicitly (e.g. a store function returning `-EBUSY` while a flag is set),
  making certain error conditions unreachable.  Document the invariant
  explicitly rather than assuming the worst.  Then apply the two-part gate
  from Step 4: (1) is the condition reachable? (2) if reachable, is the
  resulting behavior actually harmful?  A no-op write or intentional omission
  managed by a per-instance field elsewhere is not a bug.
  For pointer-dereference and missing-guard findings specifically: trace the
  full call chain from the public entry point (file operation, sysfs callback,
  interrupt handler) to the dereference site and confirm whether a prior gate
  in the chain (e.g. an open/prepare function that returns an error) makes the
  condition unreachable before filing.

### 3c.4 Interaction Picture

When the patch spans multiple functions or files:
- Draw the call graph for new/changed call sites (caller → callee chain).
- Identify shared data structures accessed by more than one function and
  confirm consistent locking.
- Note any callbacks, notifiers, or interrupt handlers introduced and confirm
  their context (process / softirq / hardirq) is compatible with the locks and
  APIs used inside them.

### 3c.5 Before-vs-After Delta

For each changed function, explicitly state:
- **What the old code did** (one or two sentences, from context lines).
- **What the new code does** (one or two sentences, from `+` lines).
- **Why the change is needed** (from the commit message or inferred from the
  diff).

This delta is the foundation for every correctness finding in Step 4.


## Step 3d — Device Tree & DT-Binding Review Checklist

Apply this checklist whenever the patch touches any of:
- `Documentation/devicetree/bindings/**/*.yaml` (binding schema)
- `arch/*/boot/dts/**/*.dts` / `*.dtsi` (device tree source)
- Driver `of_match_table`, `of_device_id`, `of_*` API call sites

Rules are derived from the authoritative upstream references:
`Documentation/devicetree/bindings/writing-schema.rst`,
`Documentation/devicetree/bindings/writing-bindings.rst`, and
`Documentation/process/submitting-patches.rst`.

### 3d.1 DT-Binding Schema (`.yaml`) Rules

**File placement & naming**
- Schema file lives under `Documentation/devicetree/bindings/<subsystem>/`.
- Filename matches the `compatible` string vendor prefix and device name:
  `<vendor>,<device>.yaml`.
- One schema file per binding; do not describe multiple unrelated devices in
  one file.  Use `allOf: [$ref: ...]` to share common properties.

**Required top-level fields**
- `$id`: must be
  `http://devicetree.org/schemas/<subsystem>/<vendor>,<device>.yaml#`
  and match the file path exactly.
- `$schema`: must be
  `http://devicetree.org/meta-schemas/core.yaml#`.
- `title`: one-line human-readable description.
- `maintainers`: at least one valid `Name <email>` entry.
- `description`: explains what the hardware is and what the binding covers.
- `properties`: declares every property the binding uses.
- `required`: lists the minimum set of mandatory properties.
- `additionalProperties: false` (or `unevaluatedProperties: false` when using
  `allOf`) — prevents undeclared properties from silently passing validation.

**`compatible` property**
- Must use the form `"<vendor>,<device>"` with a vendor prefix from
  `Documentation/devicetree/bindings/vendor-prefixes.yaml`.
- New SoC-specific compatibles must be listed most-specific first, with a
  fallback generic compatible last.
- Never use a generic string (e.g. `"gpio-leds"`) as the sole compatible for
  new hardware — always pair with a vendor-specific string.
- If the binding is a new version of an existing one, the old compatible must
  still be listed and handled.

**Property definitions**
- Every property must have a `description` field explaining its meaning and
  units.
- Use standard schema types: `uint32`, `uint64`, `string`, `phandle`,
  `phandle-array`, `bool`, etc.
- Numeric properties must specify `minimum` / `maximum` or an `enum` where
  the value space is bounded.
- Array properties must specify `minItems` / `maxItems`.
- Reuse existing common properties via `$ref`:
  - `reg`, `interrupts`, `clocks`, `resets`, `power-domains`,
    `iommus`, `dmas`, `pinctrl-*`, `#address-cells`, `#size-cells` etc.
    must reference the standard schema in
    `Documentation/devicetree/bindings/` rather than being redefined.
- Deprecated properties must be marked with `deprecated: true`.
- Do not invent new properties that duplicate existing standard ones.

**`reg` and `interrupts`**
- `reg` entries must be described with `reg-names` when there are multiple
  ranges; each name must be documented.
- `interrupts` entries must be described with `interrupt-names` when there
  are multiple; each name must be documented.
- `reg` / `interrupts` counts must match `minItems` / `maxItems` in the
  schema.

**Clock, reset, power-domain, pinctrl**
- `clocks` must be paired with `clock-names`; each clock name documented.
- `resets` must be paired with `reset-names`; each reset name documented.
- `power-domains` must reference `power-domain-names` when multiple domains
  are used.
- `pinctrl-0` / `pinctrl-names` must be present when the device uses pin
  multiplexing.

**Examples**
- Every binding schema must contain at least one `examples:` block.
- The example must be a minimal but complete DTS node that passes
  `dt_binding_check` without errors.
- Examples must not reference non-existent phandles; use placeholder labels
  (`&clk0`, `&rst0`) that are defined within the example block.

**Validation**
- Run `make dt_binding_check DT_SCHEMA_FILES=<changed.yaml>` and confirm
  zero errors and zero warnings.
- Run `make dtbs_check` on any in-tree DTS that uses the binding and confirm
  zero new warnings.

### 3d.2 Device Tree Source (`.dts` / `.dtsi`) Rules

**Node naming**
- Node names follow the form `<name>@<unit-address>` where `<name>` is the
  generic device class (e.g. `i2c`, `spi`, `gpio`, `ethernet`) — not the
  vendor chip name.
- Unit address must match the first `reg` value (hex, no leading zeros beyond
  one digit, no `0x` prefix in the node name).
- Label names use `lower_case_with_underscores`; avoid camelCase labels.

**`compatible` in DTS**
- Must exactly match a string declared in the corresponding binding schema.
- Most-specific compatible listed first; generic fallback last.
- No trailing whitespace or extra quotes.

**`reg` values**
- Must match the hardware address map; cross-check against the SoC TRM or
  existing nodes for the same SoC.
- Address and size cells must be consistent with the parent bus node
  (`#address-cells`, `#size-cells`).

**Interrupt specifiers**
- Interrupt number and flags must match the hardware; cross-check against
  existing nodes or the SoC datasheet.
- Use symbolic IRQ flag constants (`IRQ_TYPE_LEVEL_HIGH`, etc.) via the
  appropriate `#include <dt-bindings/interrupt-controller/...>` header.

**Clock, reset, GPIO, pinctrl references**
- All phandle references (`clocks`, `resets`, `gpios`, `pinctrl-0`) must
  point to nodes that exist in the same DTS/DTSI tree.
- Clock and reset indices must match the provider's `#clock-cells` /
  `#reset-cells` and the binding's `clock-names` / `reset-names`.
- GPIO flags must use constants from
  `<dt-bindings/gpio/gpio.h>` (`GPIO_ACTIVE_HIGH`, `GPIO_ACTIVE_LOW`).

**`status` property**
- Disabled-by-default peripherals use `status = "disabled"` in the DTSI and
  are enabled in board-specific DTS with `status = "okay"`.
- Never set `status = "okay"` in a shared DTSI unless the peripheral is
  always present on every board using that DTSI.

**`#include` / `#define` headers**
- Use `<dt-bindings/...>` headers for constants (IRQ types, GPIO flags, clock
  IDs, reset IDs) — never hard-code raw integers for these.
- Do not include C headers directly in DTS files.

**Formatting**
- Indentation: one tab per nesting level.
- Opening brace on the same line as the node name.
- One property per line; no trailing whitespace.
- Hex values lower-case: `0xdeadbeef` not `0xDEADBEEF`.
- Multi-value arrays aligned with angle brackets on the same line:
  `reg = <0x1000 0x100>;`

**New board DTS files**
- Must `#include` the SoC-level DTSI.
- Must define `/` root node with `compatible` (board-specific string first,
  SoC-generic string second) and `model` property.
- `model` string format: `"<Vendor> <Board Name>"`.
- Must be added to the correct `arch/*/boot/dts/*/Makefile` with
  `dtb-$(CONFIG_<SOC>) += <board>.dtb`.

### 3d.3 Driver `of_match` & `of_*` API Consistency

- Every `compatible` string in `of_device_id[]` must have a corresponding
  binding schema in `Documentation/devicetree/bindings/`.
- `of_device_id` table must be terminated with `{}` sentinel.
- `MODULE_DEVICE_TABLE(of, ...)` must be present for loadable modules.
- `of_match_ptr()` must wrap the `of_device_id` table in the
  `platform_driver` / `i2c_driver` etc. struct when `CONFIG_OF` may be
  disabled.
- `of_property_read_*` return values must be checked; missing optional
  properties handled with a documented default.
- `of_get_named_gpio()` is deprecated — use `devm_gpiod_get*()` instead.
- `of_clk_get_by_name()` is deprecated — use `devm_clk_get()` instead.
- `of_node_put()` must be called on every `of_find_*` / `of_get_*` result
  that is no longer needed (or use `of_node_get/put` scope helpers).

### 3d.4 MAINTAINERS & Documentation

- New binding files must add or update a `MAINTAINERS` entry with an `F:`
  line covering `Documentation/devicetree/bindings/<subsystem>/`.
- If the binding introduces a new vendor prefix, the prefix must be added to
  `Documentation/devicetree/bindings/vendor-prefixes.yaml` in the same or a
  preceding patch.
- Binding patches and the driver patch that consumes them should be in the
  same series; the binding patch must come first.
- Commit subject prefix for binding patches:
  `dt-bindings: <subsystem>: <description>`.
- Commit subject prefix for DTS patches:
  `arm64: dts: <soc>: <description>` (adjust arch as appropriate).

## Step 3e — Commit Message & Patch Scope Review

Apply this checklist to **every commit** in the series.  The upstream kernel
community enforces a strict one-patch-one-purpose rule; violations are a
common reason for patch rejection on the mailing list.

### 3e.1 Single-Responsibility Rule  ← most important

**Upstream policy: one patch = one purpose.**

Every patch submitted to the Linux kernel must have a single, clearly
stated purpose.  This is not a style preference — it is a hard upstream
requirement enforced by maintainers and documented in
`Documentation/process/submitting-patches.rst`:

> "Separate each **logical change** into a separate patch."
> "Each patch should do one thing and do it well."

A patch has exactly one purpose when it does **one** of the following and
nothing else:
- Fixes one specific, identified bug (one `Fixes:` tag, one root cause).
- Adds one new feature or driver.
- Performs one clean-up or refactor.
- Updates one piece of documentation or one binding.

**How to verify**: after reading the diff, write a single imperative
sentence that fully describes every change in the patch.  If you cannot
do so without using "and" to join two independent actions, the patch
violates this rule.

**Raise `[CONCERN] Patch Scope`** whenever a patch:
- Fixes two or more independent bugs in a single commit (each bug should be
  a separate patch, each with its own `Fixes:` tag).
- Combines a bug-fix with a clean-up or style change that touches **different files
  or subsystems**, or where the clean-up is large enough to obscure the fix.  A
  minor comment or whitespace correction in the same function being fixed is
  acceptable and must **not** be flagged.
- Adds a new feature and simultaneously fixes a pre-existing bug.
- Touches unrelated subsystems or files that have no logical dependency.
- Has a subject line that contains "and", "also", "plus", or lists multiple
  actions — a strong signal that the patch is doing more than one thing.
- Has a body that describes two or more distinct problems being solved.
- Has a subject line that **understates** the actual diff scope — i.e. the
  subject describes only one of the things the patch does while the diff
  contains additional independent changes (e.g. subject says "Add X" but
  the diff also corrects pre-existing bug values unrelated to X).

**Acceptable combinations** (do NOT flag these):
- A bug-fix that must also update the corresponding documentation or binding
  in the same patch (tightly coupled change).
- A new driver split across multiple patches in a series where each patch
  adds one logical layer (e.g. patch 1: dt-binding, patch 2: core driver,
  patch 3: platform data).
- A preparatory refactor in patch N that is a prerequisite for the fix in
  patch N+1, as long as each patch is independently coherent.

### 3e.2 Subject Line Quality

- Imperative mood: "Add support for X", not "Adding" or "Added".
- ≤ 72 characters; no trailing period.
- Subsystem prefix matches the files changed:
  `subsystem: component: description`
  e.g. `dt-bindings: media: qcom,sm8550-iris: Add X1P42100 compatible`
- Does not contain "and" joining two independent actions.
- Does not use vague verbs: "fix issue", "update code", "misc changes".
- **Accurately reflects all changes in the diff** — cross-check the subject
  against the actual diff: if the diff contains changes not described by
  the subject (e.g. bug-fix value corrections hidden inside a feature patch),
  flag it as `[MINOR]` subject understatement and as a secondary signal of
  a single-responsibility violation.

### 3e.3 Commit Body Quality

- Explains **why** the change is needed, not just what was changed.
- If fixing a bug: describes the root cause, the symptom, and how the patch
  resolves it.
- If adding a feature: describes the hardware/software requirement and how
  the implementation satisfies it.
- Does not describe two separate problems — if it does, the patch should be
  split.
- `Fixes:` tag present when the patch corrects a kernel regression or bug
  (with correct 12-char hash and quoted subject).
- `Cc: stable@vger.kernel.org` present when the fix should be backported.
- No `Depends-on:` pseudo-tag in the body — use a cover-letter note or
  ensure prerequisites are in the same series.

**Mandatory per-commit body checks** — apply every item below to every commit:

1. **Two-problems test**: read the body and count how many distinct problems
   or goals are described. If more than one, flag `[CONCERN] Patch Scope`
   and cross-reference the diff to confirm.
2. **Subject vs. diff cross-check**: compare the subject line against the
   actual diff hunks. If the diff contains changes not covered by the
   subject, flag `[MINOR]` subject understatement.
3. **Fixes: symptom check**: if a `Fixes:` tag is present, verify the body
   describes (a) the observable failure/symptom, (b) the root cause, and
   (c) how the patch resolves it. Flag `[MINOR]` if any of the three is
   absent.
4. **Grammar and spelling**: flag `[NIT]` for any grammar errors or typos
   in the subject or body.
5. **Tag ordering**: correct order is `Fixes:`, `Link:`, `Cc:`,
   `Reported-by:`, `Tested-by:`, `Reviewed-by:`, `Signed-off-by:`.
   Flag `[MINOR]` if `Cc: stable@vger.kernel.org` is present but `Fixes:` is
   absent, or if `Fixes:` appears **after** `Cc: stable@vger.kernel.org`
   (correct order: `Fixes:` first, then `Cc: stable@vger.kernel.org`).
6. **Fixes: format**: hash must be exactly 12 characters; subject must be
   quoted. Flag `[MINOR]` if malformed.

### 3e.4 Series-Level Scope Check

When reviewing a multi-patch series:
- Each patch in the series must be independently bisectable (the tree must
  build and boot after applying each patch individually).
- Patches that are tightly coupled (e.g. binding + driver) are acceptable
  in the same series but must be in dependency order (binding first).
- A series must not bundle unrelated changes just for convenience — each
  logical group of changes should be a separate series.
- If the series mixes `Fixes:` patches (bug-fixes) with new-feature patches,
  flag it: bug-fixes should be sent separately so they can be picked up for
  stable backports without carrying new features.

## Step 4 — Review Each Commit

Evaluate every commit against these categories (cross-reference test results).

**Mandatory pre-filing gate — two-part reachability + harm check**: Before
writing up any `[BUG]` or `[CONCERN]` finding on an error path, missing state
update, or branch condition, pass **both** gates below.  Failing either gate
means the finding must be dismissed.

*Gate 1 — Reachability*: Answer: *"What is the exact call sequence that puts
the system into the bad state?"*  Trace all writers of every flag or variable
involved, across all functions and files, and confirm the triggering condition
is actually reachable at that call site given the locks held and state
invariants enforced by the surrounding code (e.g. a store function returning
`-EBUSY` while a flag is set makes the corresponding error path unreachable).
If you cannot construct a concrete triggering scenario, dismiss the finding and
document the invariant that prevents it as a positive note instead.

*Gate 2 — Harm*: Even when the condition is reachable, ask: *"Does the
resulting behavior actually cause incorrect or harmful outcomes?"*  A no-op
write to an already-`false` flag, a safe cleanup on an error path, or an
omitted state update that is intentionally managed by a different per-instance
field are not bugs.  Consider whether the omission is deliberate: does the
subsystem manage equivalent state elsewhere (e.g. a per-instance flag vs. a
shared flag)?  Only file the finding if you can show the fallthrough or missing
update leads to genuinely incorrect behavior — not merely to code that looks
asymmetric or incomplete.

**Exception — always `[BUG]`, two-part gate does not apply**: the following
classes of defect are correctness violations regardless of whether the resulting
behavior "looks" harmless in a specific code path:
- Reachable resource leaks: memory (`kzalloc` without matching `kfree` on all
  error paths), OF node references (`of_find_*` / `of_get_*` without
  `of_node_put()`), file descriptors, IRQ lines.
- Sleeping in atomic/interrupt context: `mutex_lock()`, `msleep()`,
  `schedule()`, or any function that may sleep, called while a spinlock is
  held or from a softirq/hardirq handler.
- `copy_to_user()` / `copy_from_user()` called without a prior `access_ok()`
  check, or called while holding a spinlock.
File these as `[BUG]` directly; do not apply the two-part gate.

**Signed/unsigned type mismatches on register reads** — apply the register
read data-flow checklist from Step 3c.2 before filing any finding.  A
`u32` value stored in an `int` local and immediately returned as `u32` is
safe (C99 §6.3.1.3 preserves the bit pattern); file at most `[NIT]` for
the style violation.  Only escalate to `[BUG]` if the signed intermediate
value is demonstrably used in signed arithmetic, a signed comparison, or
widened to a larger signed type in a way that produces a wrong result.

| Category | Key questions |
|---|---|
| **Correctness** | Logic errors, off-by-one, wrong conditions, NULL checks, integer overflow, use-after-free, uninitialised variables |
| **Locking & Concurrency** | Correct lock type, initialisation, consistent protection, lock ordering, IRQ-safe variants, sleeping in spinlocks |
| **Memory Management** | Allocation failure checks, `devm_` usage, leaks on error paths, correct `sizeof` |
| **Error Handling** | All paths handled, meaningful error codes, appropriate `WARN_ON`/`BUG_ON` |
| **Kernel API Usage** | Correct API use, deprecated APIs avoided, correct memory barriers |
| **Style & Maintainability** | checkpatch-clean, clear names, accurate comments, no magic numbers, well-formed commit message (≤72 chars, imperative mood) |
| **Documentation & ABI** | New sysfs/debugfs nodes in `Documentation/ABI/`, new DT bindings in `.yaml`, `MAINTAINERS` updated |
| **Kconfig / Build** | Correct placement, minimal `depends on`, correct `Makefile` entry, clean W=1 build |
| **Kernel Coding Style** | Apply all rules from Step 3b: indentation (tabs/8-col), brace placement, naming conventions, function length, comment style, macro hygiene, type usage, error-path unwinding, commit message format |
| **Code Logic** | Cross-reference Step 3c maps: verify control-flow correctness, data-flow validation, state-machine integrity, call-graph interactions, and before-vs-after delta |
| **DT / DT-Binding** | Apply Step 3d checklist: schema structure, compatible strings, property definitions, reg/interrupts/clocks/resets, DTS node naming, driver of_match consistency, MAINTAINERS, commit subject prefix |
| **Patch Scope** | Apply Step 3e: one patch = one purpose; flag any patch that fixes multiple independent bugs, mixes bug-fix with unrelated clean-up, or has a subject line with multiple actions; check series-level bisectability and stable-backport hygiene |
## Step 5 — Output Format

All output is written as **HTML** using the structure defined in Step 6.
The templates below describe the *logical content* of each block; map each
element to the corresponding HTML structure from Step 6.

### Per-commit block

Each commit gets one `<div class="commit-block">` containing:
- A `.commit-header` with the short hash and subject.
- A `.commit-body` with:
  - A `.commit-summary` paragraph (one sentence).
  - An `<h3>Code Logic Maps</h3>` section with a `<pre>` block for
    control-flow, data-flow, state/lifecycle, call-graph, and before-vs-after
    delta (from Steps 3c.1–3c.5).
  - An `<h3>DT / DT-Binding Notes</h3>` section (only when the commit touches
    `.yaml` bindings, `.dts`/`.dtsi`, or `of_match`).
  - An `<h3>Issues</h3>` section with one `.finding-card` per finding.
  - An `<h3>Minor / Style</h3>` section with `.finding-card` elements.
  - An `<h3>Positive Notes</h3>` section with `.positive-note` elements
    (optional).

### Per-file block (Mode C only)

Same structure as the per-commit block but without a commit hash in the
header.  The header shows the relative file path instead.  Omit patch-scope
and commit-message finding categories.

### Overall Summary / Verdict Banner

Rendered as `<div class="verdict-banner [class]">` at the top of the page
(immediately after the header card).  Contains:
- The verdict pill (`READY TO APPLY` / `NEEDS FIXES` / `NEEDS DISCUSSION`).
- Stats row chips: commits reviewed, bugs, concerns, minor issues.
- Key findings grouped by category using `.findings-category` dividers and
  `.finding-card` elements.

**Full-detail rule**: every `.finding-card` in the verdict banner must include
the patch subject line (not the commit hash), a full description, the file path + line number, and a
concrete suggestion.  No one-liners.

Severity levels: `[BUG]` · `[CONCERN]` · `[MINOR]` · `[NIT]`

## Step 6 — Save the Review

**MANDATORY**: Writing the review file is not optional.  Every review run
MUST produce a saved file.  Do not output the review only to the terminal
and skip the file.  The file is the primary deliverable — the terminal
output is secondary.  If the file is not written, the review is incomplete.

**Filename**:
- Mode A: `review_<repo-basename>_last<N>_<YYYYMMDD>.html`
- Mode B: `review_<message-id-slug>_<YYYYMMDD>.html`
- Mode C: `review_<repo-basename>_<filename-no-ext>_<YYYYMMDD>.html`
  where `<filename-no-ext>` is the basename of the reviewed file with its
  extension stripped (e.g. reviewing `iris_vpu3x.c` → `review_linux-next_iris_vpu3x_20260403.html`).

**Save location**: always `<project_path>` — the project directory supplied by
the user.  Never save to the current working directory, the home directory, or
any other location.

**File structure** (write in this order):

The **Overall Summary** must appear at the top of the file, immediately after
the header block, so the reader sees the verdict and key findings without
scrolling.  The detailed per-commit reviews and test results follow.

The output file is a **fully structured HTML document**.  Use semantic HTML5
with embedded CSS for readability.  The structure below is mandatory.

### HTML skeleton

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Review: <series subject></title>
  <style>
    /* ── Reset & base ── */
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
                   "Helvetica Neue", Arial, sans-serif;
      font-size: 14px; line-height: 1.6;
      background: #f5f5f5; color: #222;
    }
    a { color: #0366d6; text-decoration: none; }
    a:hover { text-decoration: underline; }

    /* ── Layout ── */
    .page-wrap { max-width: 1100px; margin: 0 auto; padding: 24px 16px; }

    /* ── Header card ── */
    .review-header {
      background: #fff; border: 1px solid #d0d7de;
      border-radius: 8px; padding: 20px 24px; margin-bottom: 24px;
    }
    .review-header h1 { font-size: 1.4em; margin-bottom: 12px; }
    .review-header table { border-collapse: collapse; width: 100%; }
    .review-header td { padding: 3px 8px; vertical-align: top; }
    .review-header td:first-child { font-weight: 600; white-space: nowrap;
                                    color: #555; width: 180px; }

    /* ── Verdict banner ── */
    .verdict-banner {
      border-radius: 8px; padding: 20px 24px; margin-bottom: 24px;
      border: 2px solid;
    }
    .verdict-banner.needs-fixes  { background: #fff8f0; border-color: #e36209; }
    .verdict-banner.needs-discussion { background: #fffbdd; border-color: #b08800; }
    .verdict-banner.ready        { background: #f0fff4; border-color: #2da44e; }
    .verdict-banner h2 { font-size: 1.2em; margin-bottom: 12px; }
    .verdict-pill {
      display: inline-block; padding: 4px 14px; border-radius: 20px;
      font-weight: 700; font-size: 1em; margin-bottom: 16px;
    }
    .verdict-pill.needs-fixes  { background: #e36209; color: #fff; }
    .verdict-pill.needs-discussion { background: #b08800; color: #fff; }
    .verdict-pill.ready        { background: #2da44e; color: #fff; }
    .stats-row { display: flex; gap: 16px; flex-wrap: wrap; margin-bottom: 16px; }
    .stat-chip {
      padding: 4px 12px; border-radius: 6px; font-size: 0.9em; font-weight: 600;
    }
    .stat-chip.bugs     { background: #ffd7d7; color: #9e1c1c; }
    .stat-chip.concerns { background: #fff0b3; color: #7d5a00; }
    .stat-chip.minors   { background: #ddf4ff; color: #0550ae; }
    .stat-chip.commits  { background: #e8f5e9; color: #1b5e20; }

    /* ── Key findings ── */
    .findings-section { margin-top: 16px; }
    .findings-category {
      font-size: 0.75em; font-weight: 700; letter-spacing: 0.08em;
      text-transform: uppercase; color: #555;
      border-bottom: 2px solid #d0d7de; padding-bottom: 4px;
      margin: 20px 0 10px;
    }
    .finding-card {
      border-left: 4px solid; border-radius: 0 6px 6px 0;
      padding: 12px 16px; margin-bottom: 12px; background: #fff;
    }
    .finding-card.bug     { border-color: #cf222e; }
    .finding-card.concern { border-color: #e36209; }
    .finding-card.minor   { border-color: #0550ae; }
    .finding-card.nit     { border-color: #6e7781; }
    .finding-card .badge {
      display: inline-block; padding: 1px 8px; border-radius: 4px;
      font-size: 0.78em; font-weight: 700; margin-right: 6px;
    }
    .badge.bug     { background: #cf222e; color: #fff; }
    .badge.concern { background: #e36209; color: #fff; }
    .badge.minor   { background: #0550ae; color: #fff; }
    .badge.nit     { background: #6e7781; color: #fff; }
    .finding-card .patch-subject {
      font-size: 0.8em; color: #555; font-style: italic; margin-bottom: 4px;
    }
    .finding-card .title { font-weight: 600; }
    .finding-card .body  { margin-top: 8px; color: #333; }
    .finding-card .file-ref {
      font-family: "SFMono-Regular", Consolas, "Liberation Mono", Menlo, monospace;
      font-size: 0.85em; color: #555; margin-top: 6px;
    }
    .finding-card .suggestion {
      margin-top: 8px; padding: 8px 12px;
      background: #f6f8fa; border-radius: 4px;
      font-size: 0.9em; color: #333;
    }
    .finding-card .suggestion::before {
      content: "Suggestion: "; font-weight: 600;
    }

    /* ── Section cards ── */
    .section-card {
      background: #fff; border: 1px solid #d0d7de;
      border-radius: 8px; padding: 20px 24px; margin-bottom: 24px;
    }
    .section-card h2 {
      font-size: 1.1em; margin-bottom: 16px;
      padding-bottom: 8px; border-bottom: 1px solid #d0d7de;
    }
    .section-card h3 { font-size: 1em; margin: 16px 0 8px; color: #333; }
    .section-card h4 { font-size: 0.95em; margin: 12px 0 6px; color: #444; }

    /* ── Test results table ── */
    .test-table { width: 100%; border-collapse: collapse; margin-bottom: 16px; }
    .test-table th, .test-table td {
      padding: 8px 12px; text-align: left;
      border: 1px solid #d0d7de;
    }
    .test-table th { background: #f6f8fa; font-weight: 600; }
    .test-table tr:nth-child(even) { background: #fafafa; }
    .result-pass { color: #2da44e; font-weight: 700; }
    .result-fail { color: #cf222e; font-weight: 700; }
    .result-warn { color: #e36209; font-weight: 700; }
    .result-skip { color: #6e7781; font-weight: 700; }
    .result-info { color: #0550ae; font-weight: 700; }

    /* ── Commit blocks ── */
    .commit-block {
      background: #fff; border: 1px solid #d0d7de;
      border-radius: 8px; margin-bottom: 24px; overflow: hidden;
    }
    .commit-header {
      background: #f6f8fa; padding: 12px 20px;
      border-bottom: 1px solid #d0d7de;
      display: flex; align-items: baseline; gap: 10px;
    }
    .commit-hash {
      font-family: "SFMono-Regular", Consolas, "Liberation Mono", Menlo, monospace;
      font-size: 0.85em; color: #0550ae; font-weight: 600;
    }
    .commit-subject { font-weight: 600; font-size: 1em; }
    .commit-body { padding: 16px 20px; }
    .commit-summary { color: #555; margin-bottom: 16px; font-style: italic; }

    /* ── Code / pre ── */
    pre, code {
      font-family: "SFMono-Regular", Consolas, "Liberation Mono", Menlo, monospace;
      font-size: 0.85em;
    }
    pre {
      background: #f6f8fa; border: 1px solid #d0d7de;
      border-radius: 6px; padding: 12px 16px;
      overflow-x: auto; white-space: pre-wrap; word-break: break-word;
      margin: 8px 0;
    }
    code { background: #f0f0f0; padding: 1px 5px; border-radius: 3px; }

    /* ── Lists ── */
    ul, ol { padding-left: 20px; margin: 8px 0; }
    li { margin-bottom: 4px; }

    /* ── Positive notes ── */
    .positive-note {
      background: #f0fff4; border-left: 4px solid #2da44e;
      border-radius: 0 6px 6px 0; padding: 10px 14px; margin-bottom: 8px;
      font-size: 0.9em;
    }

    /* ── Footer ── */
    .page-footer {
      text-align: center; color: #888; font-size: 0.8em;
      margin-top: 32px; padding-top: 16px;
      border-top: 1px solid #d0d7de;
    }
  </style>
</head>
<body>
<div class="page-wrap">

  <!-- ═══ HEADER CARD ═══ -->
  <div class="review-header">
    <h1>Review: <series subject or "Last N commits in &lt;repo&gt;" or "File &lt;path&gt; in &lt;repo&gt;"></h1>
    <table>
      <tr><td>Date</td><td>YYYY-MM-DD</td></tr>
      <tr><td>Reviewer</td><td>AI agent (qgenie)</td></tr>
      <tr><td>Mode</td><td>A — local commits | B — patch series | C — single file</td></tr>
      <tr><td>Repository</td><td><code>&lt;project_path&gt;</code></td></tr>
      <tr><td>Commits / Patches</td><td>N  <!-- omit row for Mode C --></td></tr>
      <!-- Mode B only: -->
      <tr><td>Branch</td><td><code>review/&lt;slug&gt;</code></td></tr>
      <tr><td>Message-ID</td><td><code>&lt;message-id&gt;</code></td></tr>
      <tr><td>lore.kernel.org</td>
          <td><a href="https://lore.kernel.org/r/<message-id>">https://lore.kernel.org/r/&lt;message-id&gt;</a></td></tr>
      <!-- Mode B only — always present, even when no dependencies: -->
      <tr><td>Dependencies</td><td>
        <!-- If none stated: -->
        None stated
        <!-- If present and satisfied: -->
        <!-- <code>&lt;identifier&gt;</code> — present in base commit -->
        <!-- If missing: -->
        <!-- <strong style="color:#cf222e">MISSING: &lt;identifier&gt;</strong>
             — findings may be affected; see verdict banner -->
      </td></tr>
      <!-- Mode C only: -->
      <tr><td>File</td><td><code>&lt;relative/path/to/file&gt;</code></td></tr>
    </table>
  </div>

  <!-- ═══ VERDICT BANNER ═══ -->
  <!-- Use class "needs-fixes", "needs-discussion", or "ready" on both
       .verdict-banner and .verdict-pill to match the recommendation. -->
  <div class="verdict-banner needs-fixes">
    <h2>Overall Summary — Verdict at a Glance</h2>
    <div class="verdict-pill needs-fixes">NEEDS FIXES</div>
    <div class="stats-row">
      <span class="stat-chip commits">N commits reviewed</span>
      <span class="stat-chip bugs">N bugs</span>
      <span class="stat-chip concerns">N concerns</span>
      <span class="stat-chip minors">N minor issues</span>
    </div>

    <!-- Key findings grouped by category -->
    <div class="findings-section">

      <div class="findings-category">PATCH SCOPE VIOLATIONS</div>

      <div class="finding-card bug">
        <span class="badge bug">[BUG]</span>
        <span class="title">&lt;patch subject&gt; — &lt;short title&gt;</span>
        <div class="patch-subject">Patch: &lt;NN/TT&gt; &lt;full commit subject line&gt;</div>
        <div class="body">Full description: what is wrong, why it matters, root cause.</div>
        <div class="file-ref">File: path/to/file.c, line ~N</div>
        <div class="suggestion">Concrete fix or recommended action.</div>
      </div>

      <div class="findings-category">CORRECTNESS ISSUES</div>

      <div class="finding-card bug">
        <span class="badge bug">[BUG]</span>
        <span class="title">&lt;patch subject&gt; — &lt;short title&gt;</span>
        <div class="body">Full description.</div>
        <div class="file-ref">File: path/to/file.c, line ~N</div>
        <div class="suggestion">Concrete fix.</div>
      </div>

      <div class="findings-category">STYLE / MINOR</div>

      <div class="finding-card minor">
        <span class="badge minor">[MINOR]</span>
        <span class="title">&lt;patch subject&gt; — &lt;short title&gt;</span>
        <div class="body">Full description.</div>
        <div class="suggestion">Concrete fix.</div>
      </div>

    </div><!-- /findings-section -->
  </div><!-- /verdict-banner -->

  <!-- ═══ TEST RESULTS ═══ -->
  <div class="section-card">
    <h2>Test Results</h2>
    <table class="test-table">
      <thead>
        <tr><th>Test</th><th>Result</th><th>Notes</th></tr>
      </thead>
      <tbody>
        <tr>
          <td>checkpatch</td>
          <td><span class="result-pass">PASS</span></td>
          <td>0 errors, 2 warnings (see below)</td>
        </tr>
        <tr>
          <td>Build (W=1)</td>
          <td><span class="result-pass">PASS</span></td>
          <td>No new errors or warnings</td>
        </tr>
        <tr>
          <td>dt_binding_check</td>
          <td><span class="result-skip">SKIP</span></td>
          <td>No .yaml files changed</td>
        </tr>
        <tr>
          <td>sparse</td>
          <td><span class="result-skip">SKIP</span></td>
          <td>sparse not available</td>
        </tr>
        <tr>
          <td>get_maintainer</td>
          <td><span class="result-info">INFO</span></td>
          <td>To/Cc list (see below)</td>
        </tr>
      </tbody>
    </table>
    <!-- Verbatim checkpatch / build output goes here in a <pre> block -->
    <h3>checkpatch findings</h3>
    <pre>...verbatim output...</pre>
  </div>

  <!-- ═══ PER-COMMIT REVIEWS ═══ -->
  <!-- Repeat one .commit-block per patch/commit -->
  <div class="commit-block">
    <div class="commit-header">
      <span class="commit-hash">&lt;short-hash&gt;</span>
      <span class="commit-subject">&lt;subject line&gt;</span>
    </div>
    <div class="commit-body">
      <p class="commit-summary">One sentence describing what the commit does.</p>

      <h3>Code Logic Maps</h3>
      <pre>control-flow summary per changed function
data-flow notes
state/lifecycle notes (if applicable)
call-graph notes (if applicable)
before-vs-after delta</pre>

      <!-- DT / DT-Binding Notes — only when applicable -->
      <h3>DT / DT-Binding Notes</h3>
      <ul>
        <li>Schema validation result</li>
        <li>compatible string correctness</li>
      </ul>

      <h3>Issues</h3>
      <div class="finding-card bug">
        <span class="badge bug">[BUG]</span>
        <span class="title">Category: short summary of the bug.</span>
        <div class="patch-subject">Patch: &lt;NN/TT&gt; &lt;full commit subject line&gt;</div>
        <div class="body">Detailed analysis: root cause, how it manifests, why it is harmful.</div>
        <div class="file-ref">File: path/to/file.c, line ~N</div>
        <div class="suggestion">Concrete fix.</div>
      </div>

      <h3>Minor / Style</h3>
      <div class="finding-card minor">
        <span class="badge minor">[MINOR]</span>
        <span class="title">Category: short summary.</span>
        <div class="patch-subject">Patch: &lt;NN/TT&gt; &lt;full commit subject line&gt;</div>
        <div class="body">Detailed analysis.</div>
        <div class="suggestion">Concrete fix.</div>
      </div>

      <h3>Positive Notes</h3>
      <div class="positive-note">Positive observation about the patch.</div>

    </div><!-- /commit-body -->
  </div><!-- /commit-block -->

  <!-- ═══ FOOTER ═══ -->
  <div class="page-footer">
    Generated by AI agent (qgenie) &mdash; <em>YYYY-MM-DD</em>
  </div>

</div><!-- /page-wrap -->
</body>
</html>
```

### Severity-to-CSS-class mapping

| Severity tag | CSS class on `.finding-card` and `.badge` |
|---|---|
| `[BUG]`     | `bug`     |
| `[CONCERN]` | `concern` |
| `[MINOR]`   | `minor`   |
| `[NIT]`     | `nit`     |

### Verdict-to-CSS-class mapping

| Recommendation      | CSS class on `.verdict-banner` and `.verdict-pill` |
|---|---|
| READY TO APPLY      | `ready`            |
| NEEDS FIXES         | `needs-fixes`      |
| NEEDS DISCUSSION    | `needs-discussion` |

### Result-to-CSS-class mapping (test table)

| Result | CSS class on `<span>` |
|---|---|
| PASS   | `result-pass` |
| FAIL   | `result-fail` |
| WARN   | `result-warn` |
| SKIP   | `result-skip` |
| INFO   | `result-info` |

**Writing strategy — chunked Write + Edit (mandatory)**

Large review files MUST be written in multiple chunks to keep each tool call
manageable.  Never write the entire review in a single tool call.

**Procedure**:

1. **Create** the file with the `<head>`, embedded CSS, and the header card +
   verdict banner (Overall Summary) using the `Write` tool (≤ 120 lines):

   - Write from `<!DOCTYPE html>` through the closing `</div>` of the
     verdict banner.
   - Do NOT include the Test Results section or per-commit blocks yet.
   - Leave the file open (do not write `</body></html>` yet).
   - **The verdict banner written here is a DRAFT** based on the automated
     test results and initial code reading.  It MUST be reconciled with the
     final per-commit findings before the file is closed (see step 4b).

2. **Append** the Test Results `<div class="section-card">` block using the
   `Edit` tool.  Before constructing the `old_string` anchor, run
   `tail -8 <project_path>/<filename>` and copy the exact output verbatim.

3. **Append** each per-commit `<div class="commit-block">` one at a time using
   the `Edit` tool (one call per commit).  Before each call, run
   `tail -8 <project_path>/<filename>` and copy the exact output verbatim as
   the `old_string` anchor.

3b. **Summarize & compress after each commit (mandatory — context-window hygiene)**

   > **INTERNAL REVIEWER NOTE — never emit this section or the compressed record
   > format below into the HTML output.  This is working-memory guidance only.**

   After appending a commit's block to the file, immediately discard the full
   diff and all intermediate analysis for that commit from working memory.
   Retain only a one-sentence summary and a compact bullet list of findings.

   **Compressed record format** (keep in memory, never write to file):
   ```
   <hash> "<subject>"
   - [SEV] <one-line finding>
   - [SEV] <one-line finding>
   ```

   Discard everything else for that commit before moving to the next one.

4. **Close** the HTML document by appending the footer and closing tags:
   ```html
   <div class="page-footer">
     Generated by AI agent (qgenie) &mdash; <em>YYYY-MM-DD</em>
   </div>
   </div><!-- /page-wrap -->
   </body>
   </html>
   ```

4b. **Reconcile the verdict banner (mandatory)**

   The verdict banner was written in step 1 before the per-commit deep-dive.
   The per-commit analysis may have promoted, demoted, dismissed, or reworded
   findings.  Before confirming the file, diff the banner findings against the
   compressed per-commit records from step 3b and fix every discrepancy:

   - Every `[BUG]` / `[CONCERN]` card in the banner must correspond exactly
     to a finding of the same severity in a per-commit block.
   - Remove any banner finding that was dismissed or downgraded during the
     per-commit review.
   - Add any finding that was discovered during the per-commit review but is
     missing from the banner.
   - Update the stats-row chip counts (`N bugs`, `N concerns`, `N minor
     issues`) to match the reconciled totals.
   - Update the verdict pill and banner CSS class if the overall verdict
     changed (e.g. a dismissed `[BUG]` drops the verdict from NEEDS FIXES to
     NEEDS DISCUSSION).

   Use the `Edit` tool to patch the banner in-place.  Do not rewrite the
   entire file.

5. **Confirm** after all chunks are written:
   ```bash
   wc -l <project_path>/<filename>
   ```

6. **Print the Overall Summary to the terminal** after confirming the file.
   Output the verdict, counts, and key findings as plain text so the user can
   see the result without opening the file.  Follow it with the saved file
   path on its own line.

7. **Mode B only — clean up patch files** after the review file is confirmed
   written and the terminal summary is printed:
   ```bash
   rm -f ./<slug>.mbx ./<slug>.cover 2>/dev/null || true
   ```

**Key rules**:
- **One commit block per patch — no batching.**  A series of T patches requires
  exactly T `.commit-block` elements.  Grouping multiple patches into one block
  is forbidden regardless of patch count or similarity.
- Each `Write` or `Edit` chunk should add ≤ 120 lines of new content.
- Use the `Write` tool only for the initial file creation (step 1).
- Use the `Edit` tool for all subsequent appends (steps 2–4).
- **Before every `Edit` append, run `tail -8 <file>` and use the exact output
  verbatim as `old_string`.  Never construct `old_string` from memory, from
  text drafted in a previous turn, or by adding HTML comments that were not
  literally written to the file.  An invented or stale anchor causes a silent
  match failure that blocks all subsequent work.**
- Do NOT use shell heredocs, `printf`, or `echo` loops to write the file.
- Do NOT use the Bash tool to write the review file.
- Escape all user-supplied strings for HTML: `<` → `&lt;`, `>` → `&gt;`,
  `&` → `&amp;`, `"` → `&quot;` when placing them inside HTML attributes or
  text nodes.
- After writing each commit's block, compress its in-memory representation
  to the one-sentence + bullet format from step 3b.  Never carry full diffs,
  checkpatch output, or code-logic maps across commit boundaries.

**HTML conformance rules — MUST follow exactly, no custom CSS or structure**:

1. **Use the exact CSS class names from the template.**  Do NOT invent
   alternative class names (e.g. `.finding` instead of `.finding-card`,
   `.badge-bug` instead of `.badge.bug`, `.section` instead of
   `.section-card`, `.content` instead of `.page-wrap`).  The template CSS
   is the only CSS; do not add extra rules or override it.

2. **Verdict banner must contain all three sub-elements.**  The
   `.verdict-banner` div MUST contain: (a) a `.verdict-pill` element,
   (b) a `.stats-row` with `.stat-chip` elements (commits, bugs, concerns,
   minors), and (c) a `.findings-section` with grouped `.finding-card`
   elements for every `[BUG]` and `[CONCERN]`.  A single `<div>` with
   inline text is not acceptable.

3. **Every `.finding-card` must use the five sub-elements in order.**  Each card
   MUST contain, in this order: `.badge`, `.title` (short summary), `.patch-subject`
   (full commit subject line prefixed with "Patch: NN/TT"), `.body` (detailed
   analysis), `.file-ref`, and `.suggestion`.  Do not flatten these into inline
   `<br>`-separated text.  The `.patch-subject` element is mandatory so the reader
   always knows which patch a finding belongs to without having to scroll to the
   commit header.

4. **Every `.commit-block` must use `.commit-header` / `.commit-body`.**
   The header contains `.commit-hash` and `.commit-subject` spans.  The
   body contains the summary paragraph, Code Logic Maps `<pre>`, optional
   DT notes, Issues section, Minor/Style section, and Positive Notes.
   **The `.commit-hash` span must contain the exact 12-character hash
   recorded from `git log --oneline -1` in Step B of the per-patch procedure
   — never a hash range, never a fabricated value.**  One block per patch;
   a T-patch series produces exactly T `.commit-block` elements.

5. **Code Logic Maps section is mandatory in every commit block.**  Every
   `<div class="commit-body">` MUST contain an `<h3>Code Logic Maps</h3>`
   followed by a `<pre>` block with the control-flow summary, data-flow
   notes, and before-vs-after delta from Steps 3c.1–3c.5.  Omitting this
   section is not acceptable even for trivial one-liner patches.

6. **DT / DT-Binding Notes section is mandatory for any commit that touches
   `.yaml` binding files, `.dts`/`.dtsi` files, or `of_match_table`.**
   This includes binding-only patches (patches 1–5 in a typical panel
   series).  The section must apply the Step 3d checklist items relevant
   to that commit.

7. **Document order is fixed.**  The file must be written in this order:
   `<head>` + CSS → header card → verdict banner → test results
   `<section-card>` → per-commit blocks → footer.  Test results must
   appear BEFORE the per-commit blocks, not after them.

8. **No orphan closing tags.**  Every `</div>` must close a `<div>` opened
   in the same chunk.  Do not emit extra `</div>` pairs to "close" sections
   that were already closed.  Count open/close tags before each Edit call.

9. **Header card uses white background and table layout.**  Use the
   `.review-header` class with the `<table>` metadata layout from the
   template.  Do not use a dark-background custom header.

10. **Footer text is fixed.**  The footer must read exactly:
    `Generated by AI agent (qgenie) &mdash; <em>YYYY-MM-DD</em>`
    Do not substitute "Claude Code (Anthropic)" or any other string.

## Notes

- Never guess or fabricate file contents — read them from the repository.
- If a referenced file does not exist in the repo, say so.
- If `b4` or `git am` fails, report the exact error and stop.
- Prioritise bugs and safety over style.
- Always include file path and approximate line number when referencing code.
- Print the saved review file path as the last line of output.
