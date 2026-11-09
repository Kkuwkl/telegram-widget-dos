# Suggested fix

The bug is a missing progress guarantee, not a missing size limit. Both should be
added, in this order.

## 1. Require progress (the actual fix)

After every parsed command, verify that the parser consumed input or advanced the
cursor. If nothing was consumed, stop parsing the attribute.

```cpp
const auto before = cursor;
parseCommand();
if (cursor == before) {
    return {};   // no progress: reject the rest of the attribute
}
```

This turns an infinite loop into a rejected `d` attribute, which is exactly what a
malformed path should produce.

## 2. Bound the op count in the parser

The painter already has:

```
kMaxPathElements = 1 << 18
kMaxPathEdges    = 1 << 20
kMaxPathWork     = 1 << 27
```

but they are checked after the parser has built the vector, so they cannot prevent the
allocation. The parser should enforce its own cap, for example refuse to append beyond
`kMaxPathElements` and mark the path invalid.

## 3. Cover the whole family

`Close` is the only zero-operand command today, but the same pattern can appear in any
parser loop that may fail to advance:

* number parsing (a failed conversion that does not move the cursor),
* `transform`, `points`, `stroke-dasharray` and other list attributes,
* CSS/HTML tokenizers.

A single unit test is enough to catch this class:

```cpp
// must return without hanging and without a huge allocation
parsePath("M0 0z4");
parsePath("z4");
parsePath("z.");
```

## 4. Server side (defence in depth)

`sendWidget` accepted a 29-byte payload that deterministically freezes a current
client. Server-side validation cannot know every client parser, but a cheap sanity
check on `d` attributes (reject a `z`/`Z` not followed by end-of-string, whitespace or
a command letter) would stop the simplest form of the payload at delivery time.
