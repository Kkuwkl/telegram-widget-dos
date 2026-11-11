# Timeline

All dates are UTC.

| Date | Event |
|---|---|
| 2026-11-09 | A legitimate map widget was copied through a terminal; the copy lost the fragment `"/><path id=d`, leaving `...z4 d="M121...` in the SVG path data. |
| 2026-11-09 | Opening the chat in Telegram Desktop 7.4.1 x64 froze the client. Restarting did not help: the stored message froze it again. |
| 2026-11-09 | Reproduced through the production Bot API (`sendWidget`, HTTP 200) in a private test chat. The message was deleted afterwards and the deletion was verified. |
| 2026-11-09 | Full memory dump of the frozen process captured; the spinning thread, the 1226 MB op vector and the parser loop were identified. |
| 2026-11-09 | Minimal PoC reduced to 29 bytes (`<svg><path d="M0 0z4"/></svg>`); absolute minimum 25 bytes. |
| 2026-11-09 | Technical write-up published: `github.com/Kkuwkl/telegram-widget-dos`. |
| 2026-11-10 16:06 | Last widget-runtime commit that Telegram Desktop pins (`twidget` `1ade0d47`). It still contains the bug. |
| 2026-11-11 02:23 | **Fix upstream** in `twidget` `main`, commit [`3e25003`](https://github.com/TelegramMessenger/twidget/commit/3e250036cdd8d3fca0eecf84419810157f7403d1) — "SVG path data after a closepath that is no command ends the path". A command is flagged `explicit_cmd`; `case 'Z'` returns unless the command was read from the input. The commit cites the same minimal input and reports 17 GB in 20 s on macOS. |
| 2026-11-11 02:26 | Regression corpus entry added in `widget-sandbox`: `corpus/attacks/render-svg-path-closepath.html`. |
| 2026-11-11 04:37 | Telegram Desktop **7.4.2** released. Its `twidget` pin is still `1ade0d47`, so **7.4.2 is affected**. |
| — | Vendor notification: **not sent**. The fix was published upstream before any report from the finder. |
| — | A Telegram Desktop build with the fixed `twidget` pin: **not released yet**. |

## Status

The bug is fixed in the widget runtime, not yet in the desktop client.

* `twidget` `main` = `3e25003` — fixed.
* `twidget` `fix-tdesktop` = `1ade0d47` — the branch Telegram Desktop pins, **not** fixed;
  the fix is a separate commit on `main` that the branch has not merged.
* `tdesktop` `v7.4.2` and `tdesktop` `main` — both pin `1ade0d47`, so both are vulnerable
  until the pin is bumped.

Reproduce on any build pinned to `1ade0d47` or earlier. A build that contains
`3e25003` ends the path at the error instead of looping; the expected result is an
empty or truncated path and a responsive client.

## Disclosure policy

The finder's rule was not to publish before a fix or explicit permission. The
repository was published on 2026-11-09, before any vendor notification; the upstream
fix landed two days later. Whether it was prompted by this write-up or found separately
is not known — the commit's own measurement (17 GB in 20 s on a Mac) differs from the
numbers here (1.17 GB in 26 s on Windows).
