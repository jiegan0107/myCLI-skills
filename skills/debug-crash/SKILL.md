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

After collecting the output, write the full analysis to **`<dump_dir>/crash_analysis.html`**
as a self-contained HTML file using the Write tool, then tell the user the path.

### HTML structure

Use the following template. Fill in all `<!-- ... -->` placeholders with real data.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Kernel Crash Analysis</title>
<style>
  body { font-family: monospace; background: #1e1e1e; color: #d4d4d4; margin: 2em; line-height: 1.5; }
  h1   { color: #ffffff; border-bottom: 2px solid #555; padding-bottom: 0.3em; }
  h2   { color: #9cdcfe; margin-top: 2em; border-left: 4px solid #9cdcfe; padding-left: 0.5em; }
  h3   { color: #ce9178; margin-top: 1.5em; }
  .root-cause { background: #3a1a1a; border: 2px solid #f44747; border-radius: 6px;
                padding: 1em 1.5em; margin: 1em 0; color: #f44747; font-size: 1.05em; }
  .root-cause h2 { color: #f44747; border-left-color: #f44747; }
  table  { border-collapse: collapse; width: 100%; margin: 1em 0; }
  th     { background: #2d2d2d; color: #9cdcfe; text-align: left; padding: 6px 10px;
           border: 1px solid #444; }
  td     { padding: 6px 10px; border: 1px solid #444; vertical-align: top; }
  tr:nth-child(even) { background: #252525; }
  code   { background: #2d2d2d; padding: 1px 4px; border-radius: 3px; color: #ce9178; }
  pre    { background: #2d2d2d; padding: 1em; border-radius: 6px; overflow-x: auto;
           color: #d4d4d4; border: 1px solid #444; white-space: pre; }
  .badge { display: inline-block; padding: 2px 8px; border-radius: 4px; font-size: 0.85em; }
  .badge-warn { background: #5a4a00; color: #ffd700; }
  .badge-oot  { background: #1a3a1a; color: #6dbf67; }
  hr { border: none; border-top: 1px solid #444; margin: 2em 0; }
</style>
</head>
<body>

<h1>Kernel Crash Analysis</h1>

<div class="root-cause">
  <h2>!! ROOT CAUSE !!</h2>
  <p><!-- One or two sentences: panic string, triggering driver, immediate reason.
         If intentional (crash injection, deliberate panic), say so explicitly. --></p>
</div>

<hr>

<h2>System Info</h2>
<table>
  <tr><th>Field</th><th>Value</th></tr>
  <tr><td>Kernel</td><td><code><!-- release --></code></td></tr>
  <tr><td>Hardware</td><td><!-- hardware name --></td></tr>
  <tr><td>Uptime</td><td><!-- uptime --></td></tr>
  <tr><td>Load Average</td><td><!-- load avg --></td></tr>
  <tr><td>Tasks</td><td><!-- task count --></td></tr>
  <tr><td>Memory</td><td><!-- total memory --></td></tr>
  <tr><td>Taint</td><td><!-- taint badges using span.badge --></td></tr>
</table>

<hr>

<h2>Crash Reason</h2>
<pre><!-- full panic string --></pre>

<hr>

<h2>Crashing Task</h2>
<table>
  <tr><th>Field</th><th>Value</th></tr>
  <tr><td>Process</td><td><code><!-- comm --></code></td></tr>
  <tr><td>PID</td><td><!-- pid --></td></tr>
  <tr><td>CPU</td><td><!-- cpu --></td></tr>
  <tr><td>State</td><td><!-- state --></td></tr>
  <tr><td>Workqueue</td><td><!-- workqueue if applicable, else — --></td></tr>
</table>

<hr>

<h2>Call Trace</h2>
<pre><!-- full call trace --></pre>

<hr>

<h2>Last Kernel Messages</h2>
<pre><!-- final ~20 log lines before panic --></pre>

<hr>

<h2>Suspicious Threads</h2>
<p><!-- "None." or description of notable threads --></p>

<hr>

<h2>Memory Pressure</h2>
<pre><!-- kmem -i output --></pre>
<p><!-- interpretation --></p>

<!-- ===== STRUCT DUMP SECTION — only include if struct_type was requested ===== -->
<hr>

<h2>Struct Dump: <!-- struct_type --> (<!-- device_name -->)</h2>

<h3>How Found</h3>
<p><!-- address and discovery method --></p>

<h3>Key Fields</h3>
<table>
  <tr><th>Field</th><th>Value</th><th>Notes</th></tr>
  <!-- one <tr> per notable field -->
</table>
<p><!-- notable observations --></p>

<h3><!-- struct_type --> @ <!-- addr --></h3>
<pre><!-- raw crash output --></pre>

<!-- repeat <h3> + <pre> for each recursively expanded sub-struct -->
<!-- ===== END STRUCT DUMP SECTION ===== -->

</body>
</html>
```

### Formatting rules

- The ROOT CAUSE box uses the `.root-cause` div — red border, red text, always first.
- Taint flags render as colored `<span class="badge badge-warn">` / `<span class="badge badge-oot">` etc.
- All `crash` command output (log, bt, struct dumps) goes in `<pre>` blocks.
- Tables are used for System Info, Crashing Task, and the key-fields summary.
- Each recursively expanded sub-struct gets its own `<h3>` heading and `<pre>` block,
  labelled with the struct type, address, and the field path that led to it
  (e.g. `byte_cntr @ 0x... (tmc_drvdata.byte_cntr)`).
- The file must be self-contained (no external CSS or JS).

After writing the file, tell the user:
> Report written to: `<dump_dir>/crash_analysis.html`

---

## Notes

- If `crash` fails to launch, show the exact command that was attempted and the
  error output.
- If a BIN file listed in `dump_info.txt` is missing from the dump directory,
  warn the user but continue with the files that are present.
- Do not guess or fabricate any addresses or parameters — use only what is
  parsed from `dump_info.txt` and provided by the user.
