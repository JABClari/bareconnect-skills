---
name: zend-email
description: Send transactional email with Zend (tryzend.com) — endpoint, auth, request/response shape, the real-world gotchas, domain verification, and a provider-agnostic adapter. Use when wiring email delivery into any app or an AI-built storefront.
---

# Zend Email — integration skill

[Zend](https://www.tryzend.com) ([docs](https://www.tryzend.com/docs)) is a
transactional email API — HTML + plain text, delivered on trusted IPs. This skill
is everything learned wiring it into production, so the next integration is
copy-paste, not trial-and-error.

> 🙌 **Credit to the [Try Zend](https://www.tryzend.com) team** — clean API, quick to
> integrate. This skill documents their service for AI builders.

---

## Quick start

```
POST https://api.tryzend.com/email/send
Header: x-api-key: <YOUR_API_KEY>
Content-Type: application/json

{
  "from": "Your Brand <hello@yourdomain.com>",
  "to": "recipient@example.com",
  "subject": "Subject line",
  "html": "<p>HTML body</p>",
  "text": "Plain-text alternative"
}
```

Success (HTTP 2xx): `{ "_id": "...", "status": "pending", "cost": <number> }`.
The send is **queued asynchronously** — `status: "pending"` means *accepted*, not
delivered.

---

## Gotchas (learned the hard way)

1. **`to` must be a PLAIN email address.** Sending `"Name <email>"` fails with
   `"to must be an email"`. Put the display name only in `from`, never `to`.
2. **`from` must be on a domain you've VERIFIED in Zend.** Otherwise:
   `"You do not own the sending domain for this from address"`. There is **no
   sandbox/default sender** (unlike Resend's `onboarding@resend.dev`) — you must
   verify a domain before *any* send works.
3. **No idempotency header.** Zend won't dedupe for you — dedupe on your side
   (e.g. a unique `(kind, entity_id)` log row, insert-before-send). See below.
4. **Async status.** `pending` is normal and successful. Track delivery via `_id`
   and (if available) webhooks — don't treat `pending` as a failure.
5. **Auth is `x-api-key`,** not `Authorization: Bearer`.

## Verifying a sending domain
Zend dashboard → **Email → Domains** → add your domain → publish the **DKIM + SPF**
DNS records it shows → wait for verification. Then set `from` to an address on that
domain. This is the one manual step; do it before integration testing.

---

## Provider-agnostic adapter (recommended pattern)

Never call the API from feature code. Put it behind a small interface with swappable
adapters (Zend / Resend / a keyless dev "console" adapter), selected by env. This is
the house pattern and makes Zend a one-line swap.

```ts
// EmailProvider interface: send({ to:{email,name?}, subject, html, text }) -> { ok, messageId?, error? }
export function createZendProvider(config: { apiKey: string; from: string }) {
  return {
    name: "zend",
    async send(input) {
      try {
        const res = await fetch("https://api.tryzend.com/email/send", {
          method: "POST",
          headers: { "x-api-key": config.apiKey, "Content-Type": "application/json" },
          body: JSON.stringify({
            from: config.from,
            to: input.to.email,          // PLAIN email — not "Name <email>"
            subject: input.subject,
            html: input.html,
            text: input.text,
          }),
          cache: "no-store",
        });
        const body = await res.json().catch(() => ({}));
        if (!res.ok) return { ok: false, error: body?.message ?? `Zend ${res.status}` };
        return { ok: true, messageId: body?._id ?? null };  // async: status is "pending"
      } catch (err) {
        return { ok: false, error: err instanceof Error ? err.message : "Zend request failed" };
      }
    },
  };
}
```

**Send-once (dedupe, since Zend has no idempotency):** insert a log row on a unique
`(kind, entity_id)` with `onConflictDoNothing` *before* sending — only the caller
that wins the insert sends. Safe under retries/webhooks. Keep sends **best-effort**:
an email must never break the action that triggered it.

## Integration checklist
- [ ] Verify a sending domain in Zend (DKIM + SPF). No send works without it.
- [ ] `ZEND_API_KEY` + `EMAIL_FROM` (verified sender) in server env — never client-side.
- [ ] Adapter behind a provider interface; keyless "console" adapter for dev.
- [ ] `to` is a plain email; display name only in `from`.
- [ ] Send-once dedupe on your side; sends are best-effort.
- [ ] Treat `status: "pending"` as success.

---

*Documented by Sirius Arc. Service by the [Try Zend](https://www.tryzend.com) team — thank you.*
