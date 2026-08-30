# Flow logic notes

## The Adaptive Card JSON-escaping fix

The first live run of the approval flow failed with:

```
InvalidJsonInBotAdaptiveCard: unexpected character ... Path 'body[8].text', position 1382
```

**Cause:** the Adaptive Card's message body is itself a JSON document. The four dynamic
fields (Author, Draft Text, Draft Body, Flagged Issues) were being substituted directly
into string literals inside that JSON, like:

```
"text": "[dynamic token]"
```

Real draft text contains quotation marks and, for longer fields, line breaks. Dropped
in raw, either one breaks the surrounding JSON — the card simply becomes malformed the
moment a draft contains a quote mark, which most realistic marketing copy does (client
quotes, "guaranteed," etc.).

**Fix:** replace the raw substitution with an expression that properly escapes the
value as a JSON string before it goes into the card:

```
slice(string(createArray(coalesce(triggerBody()?['field_name'],''))),1,-1)
```

Read from the inside out:

1. `coalesce(triggerBody()?['field_name'], '')` — get the field, default to empty
   string if missing.
2. `createArray(...)` — wrap it in a single-element array.
3. `string(...)` — serialize that array to a JSON string. This is the step that does
   the actual work: `string()` on an array/object properly escapes quotes, backslashes,
   and line breaks the way JSON requires.
4. `slice(..., 1, -1)` — the serialized array looks like `["the escaped value"]`;
   `slice` strips the outer `[` and `]`, leaving just `"the escaped value"` — a
   correctly escaped JSON string literal, ready to drop into the card body as-is.

Applied to all four dynamic fields (Author, Draft Text, Draft Body, Flagged Issues).
After the fix: flow checker reports 0 errors / 0 warnings, and the card renders
correctly regardless of what punctuation or line breaks the draft text contains.

## Why the flow is split into two

See the main README's "Two real bugs found by testing live" section for the full
story. In short: a single flow that writes the Dataverse row, waits on a human's
Approve/Reject click, and then responds to the calling agent will fail with an
`ActionResponseTimedOut` (HTTP 504) the moment a human takes more than a few minutes
to respond — which is the normal case, not an edge case. Splitting into two flows
(one synchronous, done in ~1 second; one asynchronous, waits however long it takes)
removes the timeout entirely without changing what either flow does.
