# How the bug was found

No symbols, no source of the widget runtime, no debugger installed. The whole analysis
was done on a full memory dump of the frozen process.

## 1. Reproduce and capture

1. Deliver the payload as a widget message to a chat you control.
2. Open the chat. The client freezes.
3. Task Manager → Details → `Telegram.exe` → **Create dump file**.

That produces a full-memory minidump: `ThreadList`, `ModuleList`, `Memory64List`,
`ThreadInfoList`, `ThreadNames`, `MemoryInfoList`.

Note: in a full dump `MINIDUMP_THREAD.Stack.MemoryLocation.Rva` is often `0`; the stack
must be read through `Memory64ListStream` (virtual address → file offset mapping), not
through the thread descriptor.

## 2. Find the busy thread

Parse `ThreadList` and read `Rip`/`Rsp` from each `CONTEXT` (x64 offsets `0xF8` and
`0x98`). Then parse `ThreadInfoList` for per-thread kernel/user time.

All threads but one were parked in `ntdll.dll`/`win32u.dll`. One thread had 26.34 s of
user time against 1.31 s for the next one, and its `Rip` was inside the executable —
not a wait, a spin.

## 3. Get function boundaries without symbols

The binary has no PDB and no RTTI, but it does have `.pdata`: 413,035
`RUNTIME_FUNCTION` entries with `BeginAddress`/`EndAddress`. Sorting them gives exact
function ranges, which turns raw addresses into `function+offset` and lets the stack be
filtered reliably.

## 4. Filter the stack

A naive "any qword that points into `.text`" scan produces hundreds of false positives.
Keeping only qwords whose preceding bytes form a `call` (`E8 rel32`, or `FF /2` with an
optional REX prefix) reduced the chain to a few dozen real return addresses.

## 5. Disassemble the hot function

Capstone, 171 bytes. The function has no backward jump — it is a leaf helper
(`isalpha`-style table lookup). That ruled out a local `while(true)` and pointed at the
caller: the hot function was being called in a loop.

Disassembling the caller showed the parser loop and the `Close` handler that appends a
constant `4` to a vector without advancing the cursor.

## 6. Find the growing object

Scanning committed memory for a `(begin, end, capacity)` triple where `begin` falls
inside the largest region identified the vector owner:

```
begin    0x21baaffd060
end      0x21bf0b1228c
capacity 0x21bf412500c
```

`(end - begin) / 4 = 292,312,203` elements. The vector was on the spinning thread's own
stack, reachable from the parser state.

## 7. Confirm the contents

The region is filled with the repeating dword `04 00 00 00` from offset 0 up to
~1.17 GB, then zeros (committed but untouched pages). The first elements decode as
`Move`, then 36 × `Line`, then `Close` for the remaining 292,312,166 entries.

The parallel coordinate vector stayed at 296 bytes — independent proof that `Close`
consumes no operands.

## Why this is a good method

It recovers an exact root cause from a stripped, RTTI-less 257 MB binary using only:

* a minidump parser (Python + `mmap`),
* the PE `.pdata` table,
* a disassembler,
* one memory scan for the vector header.

No debugger, no symbols, no vendor cooperation.
