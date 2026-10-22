---
name: formfeed-api
description: Write application code that creates documents with the Formfeed API. Use when code should render a PDF, image, Word or PowerPoint file from a Formfeed template or from HTML, wait for an asynchronous render, send a batch, receive and verify Formfeed webhooks, or handle Formfeed's errors, in TypeScript or JavaScript with @formfeed/sdk, in Python with the formfeed package, or over plain HTTP from any other language.
license: MIT
---

# The Formfeed API from application code

The application sends a template's slug and the data; Formfeed answers with the rendered document.
Use the SDK where there is one, plain HTTP elsewhere:

- TypeScript and JavaScript: `@formfeed/sdk` (Node 22 or later, Deno, Bun, Workers), see
  [references/sdk-typescript.md](references/sdk-typescript.md).
- Python: `formfeed` from PyPI, sync and asyncio, see
  [references/sdk-python.md](references/sdk-python.md).
- Any other language: `POST https://api-eu.formfeed.dev/v1/renders` with
  `Authorization: Bearer <key>` and a JSON body; the same request in eight languages is in
  [references/requests.md](references/requests.md).

## The key

- Read it from the environment, `FORMFEED_API_KEY`. Never write a key into code, a file or a
  commit, and never ask for one in the chat: the person puts it into the environment or the secret
  store of the deployment.
- A test key (`ff_test_...`) renders free, with a watermark on the Free plan, and is what code runs
  with while it is being written and tested. A live key (`ff_live_...`) counts every render against
  the plan and belongs in production.
- Keep the key on the server. A browser must not hold it.

## One render

```ts
import { Formfeed } from '@formfeed/sdk';

const client = new Formfeed({ apiKey: process.env.FORMFEED_API_KEY! });

const render = await client.renders.create({
  template: 'invoice',
  data: { invoice: { number: '2026-0042' } },
  filename: 'invoice-2026-0042.pdf',
  meta: { order_id: '8812' },
});
const done =
  render.status === 'succeeded'
    ? render
    : await client.renders.waitFor(render.id);
const pdf = await client.renders.download(done);
```

What to get right:

- **`template`** is the slug (or the `tpl_...` id). The data must have the shape the template
  reads; `GET /v1/templates/{id}/schema` returns it, and `formfeed types` generates types from it.
- **A synchronous render can come back unfinished.** One that waits for a free slot or runs past a
  minute answers `202` with the render `queued` and finishes in the background. Handle it as above:
  `waitFor` (`wait_for` in Python), or follow the `Location` header over plain HTTP.
- **Long documents and anything a queue feeds** use `mode: 'async'`, then a webhook or `waitFor`.
- **`filename`** is used as sent: no template expressions, so build the name in code.
- **`meta`** is echoed in the render and its webhooks: put the application's own ids there.
- **`download_url`** is signed and expires. Store the render's id, and ask `GET /v1/renders/{id}`
  for a fresh link when the file is needed again.
- **Many documents** go in one batch, up to 1,000 items, with an optional zip:
  `client.renders.batch(...)`, then `client.jobs.waitFor(job.id)`. Its items come back in the order
  they were sent.

## Retries, idempotency and errors

- The SDKs retry `429` and `503` after the server's `Retry-After`, and send an `Idempotency-Key`
  with every render, so a retried request never renders twice. Over plain HTTP, do the same: send a
  key tied to the record (`order-8812-invoice`) and repeat it when retrying.
- The first answer to a key is kept for 24 hours, errors below 500 included. After fixing a
  request that answered `422`, send it with a new key.
- Errors are problems with a `code` (`404 template_not_found`, `422 template_runtime_error`,
  `402 quota_exceeded`, `429 rate_limited`). Branch on `code`, never on the message. A
  template error carries `location` with line and column. Every code, with what to do, is in
  [references/errors.md](references/errors.md).

## Webhooks

Asynchronous renders and batches report through webhooks. The handler must verify the signature
over the raw request body before it trusts an event, answer `2xx` within 10 seconds, and
deduplicate on the event's `id`: [references/webhooks.md](references/webhooks.md).
`formfeed listen` forwards a workspace's events to a local server while the handler is written.

## Before it ships

- The code reads the key from the environment and runs against a test key.
- A `202` and every problem `code` it can meet are handled.
- The data matches the template: with a stored schema, `validate_data: true` checks it before the
  render and answers `422 data_validation_error` instead of a wrong document.

The site has complete programs per language and task:
[guides](https://formfeed.dev/guides?utm_source=github-skills).
