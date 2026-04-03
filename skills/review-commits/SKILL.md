---
name: review-commits
description: >
  Multi-phase Linux kernel code review for local commits (Mode A) or lore.kernel.org
  patch series fetched with b4 (Mode B). Runs checkpatch, W=1 build, dt_binding_check,
  and sparse; audits kernel coding style (indentation, naming, macros, error paths);
  maps per-function control-flow, data-flow, and state lifecycle; checks DT/DT-binding
  schemas, DTS nodes, and driver of_match consistency; enforces the single-responsibility
  rule, subject-line quality, and series bisectability. Findings tagged [BUG], [CONCERN],
  [MINOR], or [NIT]; saved as a structured review file with an Overall Summary block
  followed by per-commit detail sections. Use when asked to review recent commits,
  a patch series, a pull request diff, or a kernel patch Message-ID from lore.kernel.org.
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

If neither is clear, ask the user before proceeding.

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
mkdir -p ./tmp
b4 am <message-id> 2>&1 | tee ./tmp/b4_output.txt
```

If `b4 am` exits non-zero: print `./tmp/b4_output.txt` and **stop**.

Parse `./tmp/b4_output.txt` for:
- **Total patches** — integer after `Total patches:`
- **mbx filename** — filename on the `git am ./` line
- **Base commit** — hash on the `git checkout -b` line (use `HEAD` if absent)
- **Cover letter** — read `./<slug>.cover` if it exists (context only)

If no `Total patches:` line despite exit 0: report full output and **stop**.

Apply patches:
```bash
git checkout -b <branch> [<base-commit>]
git am ./<slug>.mbx
```

If `git am` fails: run `git am --abort`, report the conflict, and **stop**.

Confirm with `git log --oneline HEAD~<N>..HEAD`, then clean up:
```bash
rm -f ./<slug>.mbx ./<slug>.cover 2>/dev/null || true
```

## Step 2 — Gather Context

For every changed file:
- Read the full diff (`git show <hash>`)
- Read surrounding code for patterns, naming conventions, locking models
- Check related headers for struct definitions and macros
- Check `Kconfig` / `Makefile` if new files or config symbols are added
- Check `Documentation/ABI/` if sysfs/debugfs nodes are added

## Step 3 — Run Automated Tests

Run all tests **before** writing the review. Mark unavailable tools as `SKIP`.

### 3.1 checkpatch

```bash
mkdir -p ./tmp/review_patches
git format-patch HEAD~<N>..HEAD --output-directory ./tmp/review_patches/
for patch in ./tmp/review_patches/*.patch; do
    scripts/checkpatch.pl --no-tree "$patch"
done
find ./tmp/review_patches -name "*.patch" -delete
rmdir ./tmp/review_patches ./tmp 2>/dev/null || true
```

Record all `ERROR:` and `WARNING:` lines per patch verbatim.

### 3.2 Build (W=1)

Identify affected subsystem dirs from the diffs, then:
```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- W=1 <affected_dir>/
```

Skip if `.config` is absent. Record only `error:` / `warning:` lines in
files touched by the commits.

### 3.3 DT binding check

Only when `Documentation/devicetree/bindings/` `.yaml` files are changed:
```bash
make ARCH=arm64 DT_SCHEMA_FILES=<changed.yaml> dt_binding_check
```

### 3.4 Sparse (optional)

```bash
make ARCH=arm64 C=1 CF="-D__CHECK_ENDIAN__" <affected_dir>/
```

### 3.5 Test summary table

Output before the per-commit review:
```
| Test             | Result | Notes                              |
|------------------|--------|------------------------------------|
| checkpatch       | PASS   | 0 errors, 2 warnings (see below)   |
| Build (W=1)      | PASS   | No new errors or warnings          |
| dt_binding_check | SKIP   | No .yaml files changed             |
| sparse           | SKIP   | sparse not available               |
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
- Space after keywords: `if (`, `for (`, `while (`, `switch (`, `return (`.
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
- Combines a bug-fix with an unrelated clean-up or style change.
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
   Flag `[MINOR]` if `Cc: stable@vger.kernel.org` appears before `Fixes:`.
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

Evaluate every commit against these categories (cross-reference test results):

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

### Per-commit block

```
## Commit <short-hash>: <subject>

**Summary**: One sentence describing what the commit does.

### Code Logic Maps
<control-flow summary per changed function from Step 3c.1>
<data-flow notes from Step 3c.2 — highlight any unvalidated inputs>
<state/lifecycle notes from Step 3c.3 — if applicable>
<call-graph notes from Step 3c.4 — if applicable>
<before-vs-after delta from Step 3c.5>

### DT / DT-Binding Notes
<only present when the commit touches .yaml bindings, .dts/.dtsi, or of_match>
- Schema validation result (dt_binding_check / dtbs_check)
- compatible string correctness and vendor-prefix check
- Property definitions: types, constraints, descriptions, examples
- reg / interrupts / clocks / resets / gpio / pinctrl cross-check
- DTS node naming, unit-address, formatting
- Driver of_match consistency and deprecated API usage
- MAINTAINERS / vendor-prefixes.yaml / commit subject prefix

### Issues
- **[SEVERITY] Category**: Description.
  - File: `path/to/file.c`, line ~N
  - Suggestion: ...

### Minor / Style
- ...

### Positive notes (optional)
- ...
```

Severity levels: `[BUG]` · `[CONCERN]` · `[MINOR]` · `[NIT]`

### Overall summary block

Write this block first when composing the review file (it will be placed at
the top per Step 6).  Use the `================================================================` banner
lines to make it visually prominent.

**Full-detail rule**: every entry in `Key findings` must be written with
complete information — no one-liners that just name the problem.  Each
finding must include:
- The commit hash and subject it belongs to.
- A clear description of the problem (what is wrong and why it matters).
- The exact file path and approximate line number where applicable.
- A concrete suggestion or recommended fix.

Group findings by category using `────` sub-separators when there are
multiple distinct categories (e.g. "PATCH SCOPE VIOLATIONS", "CORRECTNESS
ISSUES", "STYLE / MINOR").  This makes the summary self-contained: a
reviewer reading only the summary must have enough detail to act on every
finding without needing to read the per-commit sections.

```
================================================================
## Overall Summary   <<<  VERDICT AT A GLANCE
================================================================

**Total commits reviewed**: N
**Bugs found**: N
**Concerns**: N
**Minor issues**: N

### Recommendation
READY TO APPLY | NEEDS FIXES | NEEDS DISCUSSION

### Key findings

────────────────────────────────────────────────────────────────
<CATEGORY LABEL — e.g. PATCH SCOPE VIOLATIONS>
────────────────────────────────────────────────────────────────

- [SEVERITY] <commit hash> — <short title>

  <Full description: what is wrong, why it matters, root cause.>

  <File: path/to/file.c, line ~N (if applicable)>

  Suggestion: <concrete fix or recommended action>

────────────────────────────────────────────────────────────────
<NEXT CATEGORY — e.g. CORRECTNESS ISSUES>
────────────────────────────────────────────────────────────────

- [SEVERITY] <commit hash> — <short title>

  <Full description.>

  <File: path/to/file.c, line ~N>

  Suggestion: <concrete fix>

================================================================
```

## Step 6 — Save the Review

**Filename**:
- Mode A: `review_<repo-basename>_last<N>_<YYYYMMDD>.txt`
- Mode B: `review_<message-id-slug>_<YYYYMMDD>.txt`

**Save location**: always `<project_path>` — the project directory supplied by
the user.  Never save to the current working directory, the home directory, or
any other location.

**File structure** (write in this order):

The **Overall Summary** must appear at the top of the file, immediately after
the header block, so the reader sees the verdict and key findings without
scrolling.  The detailed per-commit reviews and test results follow.

```markdown
# Review: <series subject or "Last N commits in <repo>">

**Date**: YYYY-MM-DD
**Reviewer**: AI agent (qgenie)
**Mode**: A — local commits  |  B — patch series
**Repository**: <project_path>
**Commits / Patches**: N
**Branch** (Mode B only): review/<slug>
**Message-ID** (Mode B only): <message-id>
**lore.kernel.org link** (Mode B only): https://lore.kernel.org/r/<message-id>

---
================================================================
## Overall Summary   <<<  VERDICT AT A GLANCE
================================================================

**Total commits reviewed**: N
**Bugs found**: N
**Concerns**: N
**Minor issues**: N

### Recommendation
READY TO APPLY | NEEDS FIXES | NEEDS DISCUSSION

### Key findings

────────────────────────────────────────────────────────────────
<CATEGORY LABEL — e.g. PATCH SCOPE VIOLATIONS>
────────────────────────────────────────────────────────────────

- [SEVERITY] <commit hash> — <short title>

  <Full description: what is wrong, why it matters, root cause.>

  <File: path/to/file.c, line ~N (if applicable)>

  Suggestion: <concrete fix or recommended action>

────────────────────────────────────────────────────────────────
<NEXT CATEGORY — e.g. CORRECTNESS ISSUES>
────────────────────────────────────────────────────────────────

- [SEVERITY] <commit hash> — <short title>

  <Full description.>

  <File: path/to/file.c, line ~N>

  Suggestion: <concrete fix>

================================================================

---
## Test Results
## Per-Commit Reviews
```

**Writing strategy — chunked `apply_patch` (mandatory)**

Large review files MUST be written in multiple chunks using `apply_patch` to
avoid shell heredoc timeouts.  Never use a single `cat > file << 'REVIEW'`
heredoc for the full review — it stalls when the content exceeds ~200 lines.

**Procedure**:

1. **Create** the file with the header + overall-summary block using
   `apply_patch` `*** Add File:`:

   ```
   *** Begin Patch
   *** Add File: <project_path>/<filename>
   +<header line 1>
   +<header line 2>
   +...  (overall summary section only, ~60-80 lines max per chunk)
   *** End Patch
   ```

2. **Append** the Test Results section using a second `apply_patch`
   `*** Update File:` hunk that adds lines at the end of the file:

   ```
   *** Begin Patch
   *** Update File: <project_path>/<filename>
   @@ (append test results)
    <last 2 lines already in file>
   +<test results content>
   *** End Patch
   ```

3. **Append** each per-commit review block one at a time (one `apply_patch`
   call per commit) using the same `*** Update File:` append pattern.

3b. **Summarize & compress after each commit (mandatory — context-window hygiene)**

   After appending a commit's review block to the file (step 3), immediately
   discard the full diff and all intermediate analysis for that commit from
   working memory.  Retain only a one-sentence summary of the commit and a
   compact bullet list of its findings (severity tag + one-line description).
   This compressed record is sufficient to write the Overall Summary later.

   **Why**: kernel patch diffs, checkpatch output, and code-logic maps are
   large.  Keeping them in context across multiple commits exhausts the model's
   context window.  Writing to disk and compressing in-memory state after each
   commit keeps the working context small regardless of series length.

   **Compressed record format** (keep in memory, never write to file):
   ```
   <hash> "<subject>"
   - [SEV] <one-line finding>
   - [SEV] <one-line finding>
   ```

   Discard everything else for that commit before moving to the next one.

4. **Confirm** after all chunks are written:
   ```bash
   echo "Review saved to: $(realpath <filename>)"
   wc -l <filename>
   ```

**Key rules**:
- Each `apply_patch` chunk should be ≤ 150 lines of new content.
- Always use 2 lines of existing context before the `+` lines so the patch
   applies cleanly.
- Do NOT use shell heredocs (`cat << 'EOF'`) for the review file — they time
   out on large content.
- Do NOT use `printf` or `echo` loops — they are slow and error-prone.
- `apply_patch` is the only approved method for writing the review file.
- After writing each commit's block, compress its in-memory representation
  to the one-sentence + bullet format from step 3b.  Never carry full diffs,
  checkpatch output, or code-logic maps across commit boundaries.

## Notes

- Never guess or fabricate file contents — read them from the repository.
- If a referenced file does not exist in the repo, say so.
- If `b4` or `git am` fails, report the exact error and stop.
- Prioritise bugs and safety over style.
- Always include file path and approximate line number when referencing code.
- Print the saved review file path as the last line of output.
