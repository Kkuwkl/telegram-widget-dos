# 29 bytes freeze Telegram Desktop 7.4.1

A single widget message — 29 bytes of HTML — makes **Telegram Desktop 7.4.1 x64**
freeze permanently as soon as the chat is opened. No interaction is required.
The message is stored in the chat, so every subsequent open freezes the client again.

The cause is an infinite loop in the SVG `d`-attribute parser: a `z`/`Z` (Close)
followed by a byte that is neither a letter nor a separator leaves the input cursor
unmoved, and the "repeat previous command" fallback re-dispatches `Close`, which
consumes zero operands.

```
<svg><path d="M0 0z4"/></svg>
```

> ⚠️ This is a denial-of-service trigger. Do not send it to other people or to chats
> you do not own. Opening the chat will freeze the client.

## Impact

| | |
|---|---|
| Input | **29 bytes** (absolute minimum: 25 bytes) |
| Observed allocation | one **1226 MB** region |
| Path-op elements appended | **292,312,203** × 4 B = **1,169,248,812 B** |
| CPU before the dump | 26.34 s user + 5.52 s kernel in the UI thread |
| Client state | UI thread spins, window becomes "Not responding" |
| Recovery | kill the process; the message survives and re-triggers on the next open |

Amplification is roughly **40,000,000 : 1**. On memory-constrained devices the same
growth ends in an OOM kill instead of a freeze.

## Proof

`screenshots/1_widget.png` — the 29-byte payload delivered as a widget; the widget
area is empty.

`screenshots/3_hang_dialog.png` — the same chat a moment later: Windows reports
"Telegram Desktop is not responding".

## Affected versions

| Client | Version | Behaviour |
|---|---|---|
| Telegram Desktop, Windows x64 | **7.4.1** (build `157f5cfc85`, 2026-11-06) | empty widget → permanent freeze |
| Telegram Desktop, Windows x64 | **7.4.2** (2026-11-11) | still affected — same `twidget` pin |
| Telegram iOS | 13.2 | reported by the finder: sometimes the app is terminated, sometimes the client shows an "unsupported widget" fallback. Not confirmed with a crash log. |

The payload is accepted by the production `sendWidget` endpoint (HTTP 200) and by the
public widget sandbox, which does not hang — the failure is in the client-side parser.

**Fixed upstream** in the widget runtime on 2026-11-11 (`twidget` commit `3e25003`), but
not yet shipped in a desktop build: 7.4.2 and `tdesktop` `main` still pin the previous
commit. See [`docs/05-timeline.md`](05-timeline.md).
## Root cause in one paragraph

In the `twlite` SVG path parser, `Close` is a zero-operand command. When the parser
meets `z` followed by e.g. `4`, it does not advance the cursor past the command; the
next iteration sees no new command letter, reuses the previous one (`Close`) and
appends another `Path::Op::Close`. Nothing bounds the op vector in the parser: the
painter has `kMaxPathElements` / `kMaxPathEdges` / `kMaxPathWork`, but those limits are
applied after parsing, when a gigabyte-long vector already exists.

Details: [`docs/01-root-cause.md`](docs/01-root-cause.md).

## How it was found

The root cause was recovered from a full memory dump of the frozen process **without
any symbols** (Telegram does not publish PDBs; the binary is built with RTTI disabled).
Method: minidump parsing → `.pdata` function boundaries → stack filtering → disassembly
with Capstone → locating the growing `std::vector` on the thread stack.

Write-up: [`docs/02-detection.md`](docs/02-detection.md).

## Reproduction

[`docs/03-reproduction.md`](docs/03-reproduction.md)

## Suggested fix

[`docs/04-fix.md`](docs/04-fix.md) — require progress after every parsed command, and
cap the number of ops inside the parser, not only in the painter.

## Disclosure status

Discovered and reproduced on 2026-11-09. Fixed upstream in the widget runtime on
2026-11-11; not yet in a released desktop build. No vendor notification was sent by the
finder. Details: [`docs/05-timeline.md`](docs/05-timeline.md).

## Repository contents

```
README.md
docs/01-root-cause.md
docs/02-detection.md
docs/03-reproduction.md
docs/04-fix.md
docs/05-timeline.md
screenshots/1_widget.png
screenshots/3_hang_dialog.png
```

No bot tokens, API keys, chat identifiers, memory dumps or Telegram source code are
included in this repository.

## License

Text and screenshots: CC BY 4.0.
