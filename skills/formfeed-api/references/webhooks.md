# Webhooks

Formfeed posts an event when a render or batch finishes: `render.completed`, `render.failed`,
`job.completed`, `job.failed`, and quota and storage events a workspace opts into. Endpoints are
registered in the app (**Webhooks**) or with `POST /v1/webhooks`, which answers with the endpoint's
secret (`whsec_...`) once. A single render can also carry its own `webhook_url` and
`webhook_secret`.

## Verify every delivery

The `Webhook-Signature` header is `t=<unix seconds>,v1=<hex>`: the HMAC-SHA256 of `t + "." + body`
with the secret as it was received (the whole `whsec_...` string) as the key.

1. Compute it over the raw request body, exactly as received. Parsing and serialising the JSON
   again changes the bytes.
2. Compare in constant time.
3. Reject a `t` more than five minutes away from the clock.

A library written for the Standard Webhooks specification rejects every Formfeed delivery: the
header names match, the signature does not. Use the SDKs:

```ts
import express from 'express';
import { parseWebhookEvent } from '@formfeed/sdk';

const app = express();

// express.text keeps the body as the exact string that was signed
app.post(
  '/hooks/formfeed',
  express.text({ type: 'application/json' }),
  async (req, res) => {
    let event;
    try {
      event = await parseWebhookEvent(
        process.env.FORMFEED_WEBHOOK_SECRET!,
        req.get('webhook-signature'),
        req.body,
      );
    } catch {
      return res.status(400).send('invalid signature');
    }
    if (event.type === 'render.completed') {
      // queue the work and answer at once
    }
    res.sendStatus(204);
  },
);
```

```python
import os
from formfeed import parse_webhook_event


def handle(raw_body: bytes, signature: str) -> int:
    try:
        event = parse_webhook_event(os.environ["FORMFEED_WEBHOOK_SECRET"], signature, raw_body)
    except ValueError:
        return 400
    if event.type == "render.completed":
        pass  # queue the work
    return 204
```

## A handler that holds

- Answer `2xx` within 10 seconds. Queue slow work (downloading the file, sending mail) and answer
  first.
- Deduplicate on the event's `id`: a delivery can arrive more than once.
- Expect any order. When order matters, read the render with `GET /v1/renders/{id}`.
- Find the record through `meta`, which the render carries in every event.
- A failed delivery is retried after 1 minute, 5 minutes, 30 minutes, 2 hours and 12 hours. After
  five events in a row whose retries all failed, the endpoint is switched off; the **Webhooks** page
  shows it.

## While developing

`formfeed listen --forward-to http://localhost:3000/hooks/formfeed` forwards the workspace's test
events to the local handler, signed with the session's secret; `formfeed listen --print-secret`
prints that secret for the handler's configuration, and the next session keeps it.
`formfeed trigger render.completed` makes a real render, so a real event arrives.
