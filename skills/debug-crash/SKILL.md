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

Once the crash tool is running, execute the following commands inside the
interactive session and capture their output:

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

## Step 4: Summarize Findings

After collecting the output, provide a structured summary:

1. **Crash reason** — panic message, BUG, oops, or watchdog trigger from `log`
2. **Crashing task** — process name, PID, CPU from `bt`
3. **Call trace** — the full stack of the crashing thread
4. **Suspicious threads** — any other threads with notable backtraces from `bt -a`
5. **Memory pressure** — any OOM or low-memory indicators from `kmem -i`
6. **Last kernel messages** — final lines of `log` before the crash

---

## Notes

- If `crash` fails to launch, show the exact command that was attempted and the
  error output.
- If a BIN file listed in `dump_info.txt` is missing from the dump directory,
  warn the user but continue with the files that are present.
- Do not guess or fabricate any addresses or parameters — use only what is
  parsed from `dump_info.txt` and provided by the user.
