# Root cause

## Short version

`Path` data is parsed into two parallel structures: a list of operations (`ops`) and a
list of coordinates (`pts`). `Close` (`z`/`Z`) is the only operation that consumes no
coordinates.

The parser does not require that each loop iteration makes progress. If a `z` is
followed by a byte that is neither a command letter nor a separator, the cursor stays
on that byte. On the next iteration the parser finds no new command and falls back to
"repeat the previous command" — which is `Close` again. `Close` consumes nothing, so
the cursor still does not move, and the cycle repeats forever, appending one
`Path::Op::Close` per iteration.

The parser lives in `litehtml/src/el_svg.cpp` of the widget runtime, which Telegram
publishes as [`TelegramMessenger/twidget`](https://github.com/TelegramMessenger/twidget).
This is the code as of the affected pin (`1ade0d47`, the pin of Telegram Desktop
7.4.1 and 7.4.2):

```cpp
void parse(const char* d)
{
    char        cmd = 0;
    const char* s   = d;
    while(s && *s)
    {
        skip_separators(s);
        if(!*s)
        {
            break;
        }
        if(std::isalpha(static_cast<unsigned char>(*s)))
        {
            cmd = *s++;          // the cursor advances only here
        } else if(!cmd)
        {
            return;
        }
        bool  rel = std::islower(static_cast<unsigned char>(cmd)) != 0;
        ...
        switch(upper)
        {
        ...
        case 'Z':
            close();             // no operands, `s` is not touched
            break;
        default:
            return;
        }
        last = upper;
    }
}
```

With `d = "M0 0z4"` the loop reaches `z`, sets `cmd = 'Z'` and advances to `4`. `close()`
appends one op and leaves the cursor on `4`. The next iteration does not skip `4`
(it is not a separator), `isalpha('4')` is false, but `cmd` is still `'Z'`, so the
`else if(!cmd)` bail-out does not fire and `case 'Z'` runs `close()` again. Nothing in
the loop requires progress, so this repeats until memory runs out.

The painter already refuses to build enormous paths (`kMaxPathElements`,
`kMaxPathEdges`, `kMaxPathWork` in `tgw_painter.cpp`), but those guards run **after**
the parser has produced the vector.

Upstream fixed exactly this on 2026-11-11 in
[`3e25003`](https://github.com/TelegramMessenger/twidget/commit/3e250036cdd8d3fca0eecf84419810157f7403d1):
a command is now flagged `explicit_cmd`, and `case 'Z'` returns unless the command was
read from the input. See [`05-timeline.md`](05-timeline.md).

## Evidence from the dump

| Observation | Value |
|---|---|
| Spinning thread | UI thread, 26.34 s user + 5.52 s kernel; the next busiest thread has 1.31 s |
| Process totals | 31 s user / 7 s kernel — the spinning thread accounts for ~85% |
| Largest region | `0x21baaffd000 .. 0x21bf4126000`, 1226.0 MB, single allocation |
| Filled bytes | 1,169,248,812 B of repeating `04 00 00 00` |
| Elements | 292,312,203 × 4 B |
| Vector object | `std::vector<uint32_t>`, begin `0x21baaffd060`, end `0x21bf0b1228c`, capacity `0x21bf412500c` (306,487,275) |
| Vector contents | `[0] = Move`, `[1..36] = Line × 36`, `[37..] = Close × 292,312,166` |
| Sibling `pts` vector | unchanged at 296 B = 74 floats = 37 xy-pairs |

The sibling vector is the decisive detail: 36 `Line` ops produced exactly 37 points,
while 292 million `Close` ops produced none. The parser is not reading coordinates; it
is re-dispatching a zero-operand command.

At the moment of the dump the parser cursor pointed at a `4` that follows a `z` inside
a corrupted `d` attribute (`...7 0z4 d=...`). The registers and the attribute string
agree on that position.

## Where it comes from in the wild

The trigger was not constructed by a fuzzer. The original widget was a map (Da Nang
districts) copied through a terminal, and the copy lost a fragment `"/><path id=d`.
What remained is `...z4 d="M121...`: a `z` immediately followed by `4`. A corrupt copy
of a legitimate widget is enough.

## What is proven and what is not

Proven from the dump:

* the thread spins in the parser, not in a wait;
* the growth is a single 4-byte-element vector filled with the constant `4`;
* `Close` ops dominate it and the coordinate vector does not grow;
* the cursor stood on a non-command byte right after `z`.

Confirmed later against the published source:

* the reconstruction above was made from the binary alone. The published
  `el_svg.cpp` matches it line for line — the "missing advance" seen in the
  disassembly at `0x43fd517` is the `je` that skips `cmd = *s++` when the byte is not
  alphabetic, and `case 'Z'` really does call `close()` without touching the cursor.

Not proven:

* the exact iteration count before the process was killed;
* the iOS behaviour (no crash log was available).
