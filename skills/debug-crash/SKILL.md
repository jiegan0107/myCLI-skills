---
name: debug-crash
description: Analyze a Linux kernel crash dump using the crash tool. Parses dump_info.txt, builds the crash command, runs analysis commands, and summarizes findings.
user-invocable: true
author: Jie Gan <jiegan@qti.qualcomm.com>
---

# Task: Analyze Linux Kernel Crash Dump with the `crash` Tool

## Overview

Parse a `dump_info.txt` file to extract all memory region BIN files and their
base addresses, construct the correct `crash` command, launch it, and perform
a standard kernel crash analysis.

---

## Inputs

You will be given:
- **Dump directory path**: the folder containing all `.BIN` files and `dump_info.txt`
- **`vmlinux` path**: path to the unstripped kernel image (optional — see resolution below)
- **crash tool parameters**: including `kimage_voffset`, `vabits_actual`, and `--kaslr`
- **`struct_type`** *(optional)*: name of a kernel struct to look up (e.g. `tmc_drvdata`)
- **`device_name`** *(optional)*: name of the device to find within the struct (matched against the `struct device` embedded in the struct, e.g. `tmc_etr0`); requires `struct_type` to be set

If dump directory, `kimage_voffset`, `vabits_actual`, or `kaslr` are not provided, ask the user before proceeding.

**vmlinux resolution order:**
1. Use the path explicitly provided by the user.
2. If not provided, search for a file named `vmlinux` in the dump directory.
3. If still not found, **stop immediately** and report:
   > ERROR: vmlinux not found. Please provide the path to the unstripped kernel image via the `vmlinux` parameter. Searched in: `<dump_dir>/vmlinux`

---

## Step 1: Parse `dump_info.txt`

The file is located at `<dump_dir>/dump_info.txt`. Its format is:

```
pref               base     length               region            file name
----------------------------------------------------------------------------
   1 0x0000000014680000           180224               OCIMEM           OCIMEM.BIN
   1 0x0000000880000000       2147483648   DDR CS1 part0 Memo         DDRCS1_0.BIN
   ...
   2 0x0000000081b476d0             5784           CMM Script             load.cmm
```

**Parsing rules:**
- Only include lines where the first field (pref) is `1`.
- Skip lines where pref is `2` or any non-BIN file (e.g., `load.cmm`).
- Skip lines that are headers, separators, or timestamps.
- Extract two fields per line: `base` (hex address) and `file name` (last whitespace-separated token).
- Only include BIN files that **actually exist** in the dump directory — skip missing ones.
- **For APPS subsystem analysis**: only load BIN files whose filename starts with `DDRCS` (e.g., `DDRCS0_0.BIN`, `DDRCS1_0.BIN`, `DDRCS2_3.BIN`). All other BIN files (e.g., `OCIMEM.BIN`, `CODERAM.BIN`) are from other subsystems and must be excluded.

---

## Step 2: Build the `crash` Command

Construct the command in this format:

```
./crash <vmlinux> FILE1.BIN@0xADDR1,FILE2.BIN@0xADDR2,... \
  -m kimage_voffset=<value> \
  -m vabits_actual=<value> \
  --kaslr=<value>
```

**Rules:**
- `vmlinux` (NAMELIST) comes first, before the memory images.
- Each BIN file entry is `<filename>@<base_address>` with no spaces.
- All entries are joined with commas (`,`), no spaces between them.
- File paths must be absolute (prepend the dump directory path).
- The `crash` binary should be invoked from its location or via full path if provided.

**Example output:**
```
./crash /path/to/vmlinux \
  /dumps/OCIMEM.BIN@0x14680000,/dumps/DDRCS0_0.BIN@0x80000000 \
  -m kimage_voffset=0xffffffe1a4000000 \
  -m vabits_actual=39 \
  --kaslr=0x24a0930000
```

---

## Step 3: Launch `crash` and Run Analysis Commands

Once the crash tool is running, **before executing any analysis commands**,
check the crash startup output for a vmlinux mismatch. The crash tool prints
a warning like:

```
WARNING: kernel version mismatch
  crash: 6.1.0 (vmlinux)
  dump:  6.6.0 (dump)
```

or similar messages containing phrases such as:
- `kernel version mismatch`
- `not a valid ELF file`
- `not a vmlinux`
- `invalid kernel virtual address`
- `cannot resolve`

**If any mismatch or incompatibility is detected, stop immediately** — do not
run any analysis commands. Report to the user:

> ERROR: vmlinux mismatch detected. The provided vmlinux does not match the
> crash dump. Please supply the correct vmlinux that was built for this kernel.
>
> crash startup output:
> `<paste the relevant warning lines>`

Do not attempt to continue analysis with a mismatched vmlinux, as all results
would be unreliable.

---

Once the crash tool starts cleanly (no mismatch warnings), execute the
following commands inside the interactive session and capture their output:

| Command       | Purpose                                      |
|---------------|----------------------------------------------|
| `log`         | Kernel message log (dmesg equivalent)        |
| `bt`          | Backtrace of the crashing task               |
| `bt -a`       | Backtraces of all tasks                      |
| `ps`          | Process list at time of crash                |
| `sys`         | System info (kernel version, uptime, etc.)   |
| `kmem -i`     | Memory usage summary                         |
| `files`       | Open files of the crashing task              |
| `foreach bt`  | Backtraces for every thread                  |

After running the above, type `quit` to exit the crash session.

---

## Step 3b: Struct Lookup by Device Name (optional)

If `struct_type` is provided, perform this step **inside the same crash session**, before quitting.

### If only `struct_type` is given (no `device_name`):

Use `crash` to print all instances of the struct:

```
crash> struct <struct_type>
```

### If both `struct_type` and `device_name` are given:

1. Find all instances of `struct_type` using `kmem` or `sym` to locate the struct addresses:

   ```
   crash> kmem -S <struct_type>
   ```

2. For each address returned, read the embedded `struct device` and check its `init_name` or `kobj.name` field for a match against `device_name`:

   ```
   crash> struct <struct_type>.dev.kobj.name <addr>
   ```

   Or if the device pointer is a `struct device *dev` field:

   ```
   crash> struct <struct_type>.dev <addr>
   crash> struct device.kobj.name <dev_addr>
   ```

3. Once the matching address is found, perform a **recursive struct expansion**:

   a. Dump the top-level struct:
      ```
      crash> struct <struct_type> <addr>
      ```

   b. For every field whose value is a **non-NULL pointer to another struct**
      (e.g. `byte_cntr = 0xffffff885cc94280`, `csdev = 0xffffff803b548400`,
      `usb_data = 0xffffff885a791a40`, `csr = 0xffffff80357ebaf8`, etc.),
      dereference and dump that struct too:
      ```
      crash> struct <pointed_type> <ptr_value>
      ```

   c. Repeat step (b) for any new non-NULL struct pointers found one level
      deeper, until all reachable structs are expanded. Skip pointers that
      are clearly generic kernel types unlikely to carry driver state
      (e.g. `struct device`, `struct kobject`, `struct list_head`,
      `spinlock_t`, `mutex`, `idr`, `xarray`).

   Collect all output to include in the report (Step 4).

---

## Step 4: Summarize Findings and Write Report

After collecting the output, write the full analysis to **`<dump_dir>/crash_analysis.txt`**
using the Write tool, then display the same content to the user.

The report must be structured as follows:

```markdown
# Kernel Crash Analysis

## !! ROOT CAUSE !!

<One or two sentences stating exactly what caused the crash — be specific.
Include the panic string, the triggering subsystem/driver, and the immediate
reason (e.g. "MSS firmware crash injected via Diag with remoteproc recovery
disabled caused qcom_q6v5_crash_handler_work() to call panic()").>

---

## System Info
...

## Crash Reason
...

## Crashing Task
...

## Call Trace
...

## Last Kernel Messages
...

## Suspicious Threads
...

## Memory Pressure
...

## Struct Dump: <struct_type> (<device_name>)   ← only present if struct_type was requested
...
```

**Formatting rules for the ROOT CAUSE section:**
- It must be the very first section, before all others.
- The heading must be `## !! ROOT CAUSE !!` (with the exclamation marks) so it
  stands out visually.
- State the root cause in plain language — no jargon without explanation.
- If the crash was intentional (e.g. crash injection, deliberate panic), say so
  explicitly.

**Section content:**

1. **ROOT CAUSE** — the single most important finding; see above
2. **System Info** — kernel version, hardware, uptime, load average from `sys`
3. **Crash Reason** — full panic message and taint flags from `log`
4. **Crashing Task** — process name, PID, CPU, workqueue (if applicable) from `bt`
5. **Call Trace** — full stack of the crashing thread
6. **Last Kernel Messages** — final ~20 lines of `log` before the panic
7. **Suspicious Threads** — any threads with notable backtraces from `bt -a` / `foreach bt`; write "None" if all threads look normal
8. **Memory Pressure** — OOM or low-memory indicators from `kmem -i`; write "None" if memory was healthy
9. **Struct Dump** *(only if `struct_type` was requested)* — address, how it was found, a key-fields summary table, and the full raw `crash` output in a code block.

After writing the file, tell the user:
> Report written to: `<dump_dir>/crash_analysis.txt`

---

## Notes

- If `crash` fails to launch, show the exact command that was attempted and the
  error output.
- If a BIN file listed in `dump_info.txt` is missing from the dump directory,
  warn the user but continue with the files that are present.
- Do not guess or fabricate any addresses or parameters — use only what is
  parsed from `dump_info.txt` and provided by the user.
