# formfeed (Python)

```bash
pip install formfeed
```

Sync and asyncio, built on httpx, with retries on `429` and `503` after the server's
`Retry-After` and an `Idempotency-Key` on every render.

## Client

```python
import os
from formfeed import Formfeed

client = Formfeed(os.environ["FORMFEED_API_KEY"])
```

The key is the first argument; `region`, `base_url`, `max_retries` (3) and `timeout` (120 seconds)
are keyword arguments. Use the client in a `with` block or call `client.close()`.

## Render and download

```python
import os
from formfeed import Formfeed

with Formfeed(os.environ["FORMFEED_API_KEY"]) as client:
    render = client.renders.create(
        template="invoice",
        data={"invoice": {"number": "2026-0042"}},
        filename="invoice-2026-0042.pdf",
        idempotency_key="order-8812-invoice",
    )
    if render.status != "succeeded":
        render = client.renders.wait_for(render.id)
    with open("invoice-2026-0042.pdf", "wb") as f:
        f.write(client.renders.download(render))
```

`renders.create` answers a finished render, or one still `queued` or `rendering` after the
synchronous wait: `wait_for` waits for it. `mode="async"` answers at once.

## Batches

```python
import os
from formfeed import Formfeed

orders = [{"number": "2026-0042"}, {"number": "2026-0043"}]
with Formfeed(os.environ["FORMFEED_API_KEY"]) as client:
    job = client.renders.batch(
        [{"data": {"invoice": order}, "meta": {"number": order["number"]}} for order in orders],
        template="invoice",
        zip=True,
    )
    job = client.jobs.wait_for(job.id)
    print(job.succeeded, job.failed, job.zip_url)
```

## asyncio

```python
from formfeed import AsyncFormfeed


async def thumbnail(key: str) -> bytes:
    async with AsyncFormfeed(key) as client:
        render = await client.renders.create(html="<h1>Hello</h1>", output="png")
        return await client.renders.download(render)
```

## Errors

```python
import os
from formfeed import Formfeed, FormfeedError

client = Formfeed(os.environ["FORMFEED_API_KEY"])
try:
    client.renders.create(template="invoice", data={})
except FormfeedError as e:
    print(e.code, e.status, e.request_id, e.problem)
    raise
```

`parse_webhook_event` raises `ValueError` for a bad signature, not `FormfeedError`.

## Typed data

`formfeed types --lang python` writes `formfeed_templates.py`: a TypedDict per template and
`typed(client).invoice(data)`, which a type checker holds to the template's schema.
