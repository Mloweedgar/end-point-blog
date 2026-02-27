---
author: Edgar Mlowe
title: "Why Your AI Extractor Fails on .msg Emails (and How to Fix Decoding)"
description: "A practical guide to fixing .msg HTML decoding issues that silently corrupt text before AI extraction."
date: 2026-02-27
tags:
- artificial-intelligence
- email
- unicode
- data-processing
- troubleshooting
---

I want to share one debugging lesson that saved me from tuning the wrong layer in an AI extraction pipeline.

I started with a familiar symptom: extraction output looked inconsistent. Some rows were fine, some had weird characters, especially accents. My first instinct was the same one most of us have: maybe the model needs prompt tuning.

It turned out not to be a model problem.

The root cause was upstream data integrity: decoding `.msg` email HTML with the wrong charset.

### The pattern that gives it away

If you see this mix, think decoding first:

- output is mostly correct, but certain names and addresses look garbled
- problems appear only for some senders or date ranges
- `.eml` looks stable, but `.msg` is inconsistent

A classic sign looks like this:

- expected: `Müller`
- corrupted: `MÃ¼ller`

By the time your extractor sees that text, meaning is already damaged.

### Why `.msg` bites harder than `.eml`

Quick definitions:

- `.eml` is the standard MIME email format and usually includes charset metadata per part.
- `.msg` is an Outlook container format (MAPI), where body bytes and encoding hints can be stored separately.

That difference matters.

If your code assumes UTF-8 for `.msg` HTML bytes, non-UTF messages can decode into garbage. Then downstream steps (HTML-to-PDF, OCR, LLM extraction, post-processing) just preserve and propagate bad text.

### The hidden failure point in the pipeline

The break is usually here:

1. Read `.msg` HTML body bytes.
2. Decode with wrong charset or permissive replacement.
3. Pass corrupted text to extraction.
4. Blame extraction quality.

If step two is wrong, steps three and four never had a chance.

### The fix: strict, explicit, controlled

You do not need a big rewrite. A small decode policy change can remove a whole class of silent failures.

For `.msg` HTML bytes:

1. Read the encoding hint from message metadata.
2. Map that hint to a decoder codec.
3. Decode in strict mode.
4. If needed, use one controlled strict fallback.
5. If decode still fails, fail loud.

Minimal example:

```python
from extract_msg.encoding import lookupCodePage

PR_INTERNET_CODEPAGE = "3FDE0003"

def decode_msg_html_bytes(html_bytes: bytes, message) -> str:
    codepage_id = message.getPropertyVal(PR_INTERNET_CODEPAGE)
    codec = (
        lookupCodePage(codepage_id)
        if isinstance(codepage_id, int) and codepage_id > 0
        else "utf-8"
    )
    try:
        return html_bytes.decode(codec, errors="strict")
    except (LookupError, UnicodeDecodeError):
        return html_bytes.decode("utf-8", errors="strict")
```

### Why I prefer fail-loud here

`errors="replace"` keeps jobs moving, but it can hide real data corruption.

For low-stakes preview features, that may be acceptable.
For transactional extraction (orders, invoices, legal, shipping), silent corruption is usually worse than an explicit failure.

Use this decision rule:

| Use case | Policy |
|---|---|
| Preview/search UX | Best-effort can be acceptable with clear flags |
| Transactional extraction | Strict decode + fail loud |
| Mixed systems | Strict on extraction path, best-effort on preview path |

### How to roll this out safely

Keep blast radius low:

1. Change only the failing decode path first.
2. Validate on a representative dataset, not one sample file.
3. Leave unrelated paths untouched until evidence says otherwise.
4. Expand strict policy incrementally.

This gives reliability without destabilizing the rest of the ingestion stack.

### Observability that makes this easier next time

Log these fields per message:

- source file
- content source used (HTML or plain text)
- encoding hint found
- codec selected
- fallback used or not
- decode result (success, fallback, manual review, fail)

With this, “random extraction quality” turns into a clear ingestion signal.

### Final takeaway

If your `.msg` extraction quality looks inconsistent, check input integrity before touching prompts or model settings.

Upstream data integrity is often the hidden cause of downstream AI errors.

In my case, the highest-leverage fix was simple:

- decode with the right charset hint
- decode strictly
- avoid silent replacement on critical paths

Small change, big reliability gain.
