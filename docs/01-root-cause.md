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

Illustratively:

```
while (cursor < end) {
    cmd = readCommand(cursor);        // may fall back to the previous command
    switch (cmd) {
    case 'z': ops.push_back(Close);   // 0 operands, cursor not advanced
              break;
    ...
    }
    // no check that the iteration consumed anything
}
```

The painter already refuses to build enormous paths (`kMaxPathElements`,
`kMaxPathEdges`, `kMaxPathWork` in `tgw_painter.cpp`), but those guards run **after**
the parser has produced the vector.

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

Not proven:

* the exact source of `twlite` (it is not part of the public Telegram Desktop
  repository, so the code above is reconstructed from the binary);
* the exact iteration count before the process was killed;
* the iOS behaviour (no crash log was available).
