# Timeline

| Date (UTC) | Event |
|---|---|
| 2026-11-09 | A legitimate map widget was copied through a terminal; the copy lost the fragment `"/><path id=d`, leaving `...z4 d="M121...` in the SVG path data. |
| 2026-11-09 | Opening the chat in Telegram Desktop 7.4.1 x64 froze the client. Restarting did not help: the stored message froze it again. |
| 2026-11-09 | Reproduced through the production Bot API (`sendWidget`, HTTP 200) in a private test chat. The message was deleted afterwards and the deletion was verified. |
| 2026-11-09 | Full memory dump of the frozen process captured; the spinning thread, the 1226 MB op vector and the parser loop were identified. |
| 2026-11-09 | Minimal PoC reduced to 29 bytes (`<svg><path d="M0 0z4"/></svg>`); absolute minimum 25 bytes. |
| — | Vendor notification: **pending**. |
| — | Vendor response / fix: **unknown**. |

This file will be updated when Telegram responds or ships a fix.

## Disclosure policy

The finder's rule is not to publish before a fix or explicit permission. If you are
reading this and the table above still says "pending", the details were published ahead
of vendor confirmation — treat the repository as a technical write-up, not as a
coordinated disclosure.
