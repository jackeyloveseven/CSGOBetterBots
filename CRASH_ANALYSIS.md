# CS:GO Legacy — Crash Investigation Report

> A reproducible investigation into repeatable crashes on a heavily-modded
> CS:GO Legacy listen server (Metamod 2.0.0.0 + SourceMod 1.13.0.7297, 9 bots,
> ~40 plugins). This document records **what was measured, how, and what it
> turned out to be** — including the dead ends, because they matter.

---

## TL;DR

Two unrelated problems were hiding behind one symptom ("the game crashes after
5–20 minutes"):

1. **An engine defect**: `CUtlMemory::Purge` in `server.dll` frees a buffer that
   points into `tier0.dll`'s `.data` section. Fixed with a 2-byte binary patch.
2. **32-bit address-space exhaustion**: the process reaches ~3.06 GB and the
   engine calls `__fastfail(FAST_FAIL_FATAL_APP_EXIT)` after a failed 128 KB
   allocation. Not fixable by patching — it is an architectural ceiling.

The single most useful diagnostic artifact was **the crash dump**, not the
event log, not plugin bisection.

---

## Symptoms

| Observation | Detail |
|---|---|
| Crash cadence | 6 / 12 / 18 / 19 / 25 / 27 minutes with all plugins; 43 min with bots disabled |
| Event log | `tier0.dll` faults, two different offsets |
| `.mdmp` files | All 0 bytes (`(failed)*.mdmp`) — engine's own dumper never worked |
| Perceived cause | "random", "input method related", "driver related" — all disproved |

---

## Investigation

### 1. Capture real dumps

The engine's minidump writer produced empty files. **Windows Error Reporting
LocalDumps** was used instead:

```bat
reg add "HKLM\SOFTWARE\Microsoft\Windows\Windows Error Reporting\LocalDumps\csgo.exe" ^
  /v DumpFolder /t REG_EXPAND_SZ /d "D:\dumps" /f
reg add "HKLM\SOFTWARE\Microsoft\Windows\Windows Error Reporting\LocalDumps\csgo.exe" ^
  /v DumpType /t REG_DWORD /d 1 /f
```

Result: 169–174 MB full dumps. **Without these, nothing below is possible.**

> Windows-specific: on 64-bit Windows a 32-bit process must be dumped by the
> **SysWOW64** PowerShell, otherwise thread stacks are truncated.

### 2. Do not trust naive stack scanning

A first pass that scanned the stack for "any address inside a known module"
produced a bogus call chain: it picked up **data values** from `.data` sections
(e.g. `server.dll+0xaaef64`, which is not code at all).

The reliable method on x86 is **walking the EBP chain**:

```
[ebp]     -> saved ebp
[ebp + 4] -> return address
```

Walking it across three separate dumps produced **byte-identical call chains**,
which is what finally proved determinism. (Script: `ebp_walk.py`, included in
this repo's companion tooling.)

### 3. Compare registers across crashes

```
dump A  tid=19544  ecx=0x63aaceb0  ebx=0x63aa14f0
dump B  tid=19544  ecx=0x63aaceb0  ebx=0x63aa14f0
dump C  tid=34144  ecx=0x63aaceb0  ebx=0x63aa14f0   <- thread id differs, addresses do not
```

`ecx` is the pointer being freed. It is **identical across three different
processes** — therefore it is a fixed global address, not heap noise.

### 4. Resolve the address

```
0x63aaceb0 -> tier0.dll+0x5ceb0   (.data section)
```

`free()` is being called on a **global variable inside tier0.dll**. That address
has no heap block header, so the allocator dereferences garbage and faults.

### 5. Disassemble the caller

`server.dll+0x1312b0` (reached from the EBP chain) is a `CUtlMemory<T>::Purge`
specialisation:

```asm
0x1312b0  push esi
0x1312b1  mov  esi, ecx              ; this
0x1312b3  mov  dword [esi+0xC], 0
0x1312ba  cmp  dword [esi+8], 0      ; m_nGrowSize
0x1312be  jl   0x1312e0              ; < 0 => external buffer, must NOT be freed
0x1312c0  mov  edx, [esi]            ; m_pMemory   <-- crashes here
0x1312c2  test edx, edx
0x1312c4  jz   0x1312d9
0x1312c6  mov  eax, ds:[0x107ea2c0]  ; g_pMemAlloc
0x1312cb  push edx
0x1312cc  mov  ecx, [eax]
0x1312ce  mov  eax, [ecx]
0x1312d0  call dword [eax+0x14]      ; IMemAlloc::Free(ptr)
```

Field layout matches `CUtlMemory<T>` exactly: `+0` = `m_pMemory`,
`+4` = `m_nAllocationCount`, `+8` = `m_nGrowSize`.

**Root cause hypothesis**: engine code attached a *static* buffer via
`SetExternalBuffer()` but left `m_nGrowSize` at `0` (≥ 0) instead of a negative
value, so `Purge()` treats the static buffer as if it owned it.

### 6. Memory sampling (the second, unrelated problem)

Polling `PrivateMemorySize64` every 20 s produced:

```
2317 MB at spawn  ->  3044 MB at crash  (77-118 MB/min)
OUT OF MEMORY! attempted allocation size: 131072
FAST_FAIL_FATAL_APP_EXIT (params=7)
```

So the *other* crash signature (`tier0+0x29314`, `0xc0000409`) is not a stack
corruption at all — it is the engine **deliberately terminating** after a
128 KB allocation failure at ~3.06 GB. 32-bit process, 4 GB ceiling.

---

## Findings

| # | Finding | Evidence | Status |
|---|---|---|---|
| F1 | `CUtlMemory::Purge` frees a static buffer in `tier0.dll` | identical `ecx` across 3 dumps + disassembly | **fixed** (binary patch) |
| F2 | Address-space exhaustion at ~3.06 GB | memory curve + `OUT OF MEMORY` string on stack | architectural limit |
| F3 | `metamod.2.csgo.dll+0x8a44` invalid vtable call | dump EBP chain | open |
| F4 | `panorama.dll` GS-cookie failure | dump EBP chain, all frames in panorama | open |
| F5 | `bot_inventory` accumulates `g_aMapWeapons` across rounds | source review | **fixed** (`Clear()` added) |

F3/F4 are plausibly *downstream symptoms* of F2: once allocations start failing,
object construction can silently fail and later code dereferences a null vtable.

---

## Fixes applied

### Binary patch (F1)

In `csgo\bin\server.dll`, at file offsets `0x1306c0` and `0x1306e0`:

```
8B 16   mov edx, [esi]     ->   33 D2   xor edx, edx
```

Both `Free()` calls are now skipped because `edx` is zero and the following
`test edx,edx / jz` branches over them. **Back up `server.dll` first.**

Verified: `tier0+0x1d889` never appears again in subsequent dumps.

Trade-off: the generic purge path no longer returns memory for this
specialisation. Acceptable here because the affected object holds a static
buffer; long-run leak behaviour was monitored via the memory sampler.

### Source fix (F5)

`bot_inventory.sp`:

```sourcepawn
public Action Event_OnRoundStart(...)
{
    g_aMapWeapons.Clear();   // was missing: array grew every round
    ...
}
```

---

## What did NOT work (kept for the record)

| Attempt | Result |
|---|---|
| Disabling `bot_stuff` | still crashed |
| Disabling all item/econ plugins | still crashed |
| Removing a Chinese IME (TSF-injected DLL) | still crashed (dumps from a clean process) |
| `DHook` detour on `CUtlMemory::Purge` | hook installed, **zero** interceptions — detour did not take effect |
| `sm prof` (SourceMod profiler) | reports **CPU time**, not memory — no help here |
| Reducing plugin count to 16 | stable for 46 min, which *pointed at* memory rather than a plugin bug |

---

## Reproduction

1. Install ~40 plugins + 9 bots on a 32-bit CS:GO Legacy listen server.
2. Enable WER LocalDumps (command above).
3. Play; expect `tier0+0x1d889` within minutes if `server.dll` is unpatched.
4. After patching, expect `tier0+0x29314` (`OUT OF MEMORY`) instead — this is F2.

---

## Tooling

| Script | Purpose |
|---|---|
| `ebp_walk.py` | x86 EBP-chain unwinder over a minidump (`MemoryListStream` based) |
| `mem_sample.ps1` | poll `PrivateMemorySize64` / handles / threads every 20 s |

Both are small, dependency-free (Python 3 / PowerShell 5), and use only
documented minidump streams.
