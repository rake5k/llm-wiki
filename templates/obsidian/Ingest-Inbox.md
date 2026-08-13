---
ingest-inbox: true
type: reference
updated: "{{DATE}}"
---

## About

Capture queue for `/wiki ingest inbox`. A session appends one line per durable learning instead of
running a full ingest mid-task — capture is cheap, page writes are batched.

Line format: `<ISO date> -- <Wiki/Namespace target or ?> -- <one-sentence fact> -- src: <repo / MR / URL / session>`

`/wiki ingest inbox` removes a line only after its target page is written and committed. Draining is
deliberate, never automatic.

Exempt from orphan / stale / demote lint rules, like [[Wiki/Reference/Access-Log]].

NEVER capture credentials — the wiki is git-tracked. Quick rules and gotchas belong in L1 memory,
not here; see the L1/L2 split in [[Wiki/Schema]].

## Pending (append-only, newest at bottom)
