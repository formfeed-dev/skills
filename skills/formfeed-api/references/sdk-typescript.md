# @formfeed/sdk

```bash
npm install @formfeed/sdk
```

No dependencies; it uses `fetch` and Web Crypto and runs in Node 22 or later, Deno, Bun, Cloudflare
Workers and browsers (keep the key on the server).

## Client

```ts
import { Formfeed } from '@formfeed/sdk';

const client = new Formfeed({ apiKey: process.env.FORMFEED_API_KEY! });
```

Options: `region` (`'eu'`, the default), `baseUrl`, `maxRetries` (3; retries `429`, `503` and
network errors), `timeoutMs` (120000 per attempt), `fetch`, `headers`.

## Render and download

```ts
import { Formfeed } from '@formfeed/sdk';

const client = new Formfeed({ apiKey: process.env.FORMFEED_API_KEY! });

const render = await client.renders.create(
  {
    template: 'invoice',
    data: { invoice: { number: '2026-0042' } },
    filename: 'invoice-2026-0042.pdf',
  },
  { idempotencyKey: 'order-8812-invoice' },
);
const done =
  render.status === 'succeeded'
    ? render
    : await client.renders.waitFor(render.id);
const bytes = await client.renders.download(done);
console.log(done.page_count, done.units, bytes.byteLength);
```

- `client.renders.create` answers a finished render, or one that is still `queued` or `rendering`
  after the synchronous wait: `waitFor` then waits for it (`{ timeoutMs, intervalMs }`).
- `mode: 'async'` answers at once with a `queued` render.
- HTML instead of a template: `{ html: '<h1>Hello</h1>', output: 'png' }`, with `engine: 'jinja2'`
  to fill it with `data`.

## Batches

```ts
import { Formfeed } from '@formfeed/sdk';

const client = new Formfeed({ apiKey: process.env.FORMFEED_API_KEY! });
const orders = [{ number: '2026-0042' }, { number: '2026-0043' }];

const job = await client.renders.batch({
  template: 'invoice',
  items: orders.map((order) => ({
    data: { invoice: order },
    meta: { number: order.number },
  })),
  zip: true,
});
const finished = await client.jobs.waitFor(job.id);
console.log(finished.succeeded, finished.failed, finished.zip_url);
```

`finished.items` lists the renders in the order of `items`.

## Errors

```ts
import { Formfeed, FormfeedError } from '@formfeed/sdk';

const client = new Formfeed({ apiKey: process.env.FORMFEED_API_KEY! });

try {
  await client.renders.create({ template: 'invoice', data: {} });
} catch (e) {
  if (e instanceof FormfeedError) {
    console.error(e.code, e.status, e.requestId, e.problem?.location);
  }
  throw e;
}
```

`e.code` is the problem's code; failures on the client's side have status `0` and the codes
`network_error`, `timeout` (a `waitFor` ran out of time; the render goes on), `not_ready` and
`download_failed`.

## Typed data

`formfeed types` in a template project writes `formfeed.d.ts`, which teaches `renders.create` and
`renders.batch` the data each template expects; a wrong field is then a type error.
