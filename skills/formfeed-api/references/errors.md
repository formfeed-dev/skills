# Problem codes

Generated from the error pages of the documentation. Every error is a problem (RFC 9457) with a
`code`, the HTTP `status`, a `detail` sentence and a `request_id`; branch on `code`, never on the
text. A render that failed after it was recorded also carries its `render_id`, and a template
error its `location` (line and column).

## 409 channel_in_use

The [release channel](https://docs.formfeed.dev/templates/release-channels) was not deleted, because renders of the last hour used it. Deleting it would make those callers fail with [`channel_not_found`](https://docs.formfeed.dev/errors/channel-not-found).

**Causes.**

- `DELETE /templates/{id}/channels/{name}` (or `formfeed channels delete <slug> <name>`) for a channel a staging environment, a test or an integration still renders.

**Fix.**

- Find out who renders the channel: `formfeed channels list <slug>` shows its renders of the last 24 hours, and the render log filters by template. Change those callers to another channel first.
- If the renders do not matter any more, delete with `?force=true` (`--force` in the CLI, `{ force: true }` / `force=True` in the SDKs).

**Retry.**

Once no render has used the channel for an hour, the same request succeeds. Otherwise change the callers or force the deletion.

## 422 channel_name_invalid

The name of a [release channel](https://docs.formfeed.dev/templates/release-channels) in the path of a channel endpoint is malformed or reserved. `channel` repeats the name. Nothing was changed.

**Causes.**

- `PUT`, `DELETE`, `promote` or `rollback` on `/templates/{id}/channels/{name}` with uppercase letters, spaces, underscores or a leading digit in the name.
- A name longer than 32 characters.
- The reserved names `latest` and `draft`, which already mean something as a render's `version`.

On renders and `GET /templates/{id}/versions/{name}`, a malformed name answers [`invalid_request`](https://docs.formfeed.dev/errors/invalid-request) instead.

**Fix.**

- Use 1 to 32 lowercase letters, digits and dashes, starting with a letter: `staging`, `beta`, `customer-a`.
- `published` is the built-in channel: move it with `PUT`, but it cannot be deleted (a `DELETE` answers [`invalid_request`](https://docs.formfeed.dev/errors/invalid-request)). See [Release channels](https://docs.formfeed.dev/templates/release-channels#names).

**Retry.**

No. The same name gets the same answer; correct it first.

## 404 channel_not_found

The request names a [release channel](https://docs.formfeed.dev/templates/release-channels) that the template does not have in the workspace of the key. On a render, `template` and `channel` repeat what the request named.

**Causes.**

- A render or batch item with `"version": "staging"` for a template that has no `staging` channel, for example because it was never created in this workspace or was deleted.
- `GET /templates/{id}/versions/{channel}` for such a channel, which is also what `formfeed types --channel` reads.
- `promote`, `rollback` or `DELETE` on a channel that does not exist.
- A misspelt channel name, such as `stagging`.

A template without the version a render names answers [`template_not_found`](https://docs.formfeed.dev/errors/template-not-found) instead. A name that cannot be a channel at all (uppercase letters, a leading digit, more than 32 characters) answers [`invalid_request`](https://docs.formfeed.dev/errors/invalid-request) on renders and version reads, and [`channel_name_invalid`](https://docs.formfeed.dev/errors/channel-name-invalid) on the channel endpoints. On the Free plan, a render or a `PUT` that names a channel other than `published` answers [`forbidden`](https://docs.formfeed.dev/errors/forbidden) (`feature: "channels"`) before the channel is looked up.

**Fix.**

- List the template's channels with `formfeed channels list <slug>` or `GET /templates/{id}/channels`.
- Create the channel (`formfeed channels set <slug> staging <version>`) or render another one. See [Release channels](https://docs.formfeed.dev/templates/release-channels).
- Check that the key belongs to the workspace you expect: channels belong to a template, and templates to one workspace.

**Retry.**

No. The same request gets the same answer. Fix what the problem names, then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 409 conflict

The request conflicts with what is stored now, so nothing was changed. `detail` says which conflict it is.

**Causes.**

- `POST /templates` with a slug that already exists in the workspace (_A template with this slug exists in the workspace_).
- `POST /templates/{id}/versions` with a `base_checksum` that no longer matches the latest version, because someone saved or published in between. `formfeed templates push` sends it for you, from the last pull.
- `PUT /partials/{name}` with a `base_version` that is no longer the shared partial's version. The problem then carries the stored partial as `current` (or `null` when it was deleted). See [Brand and partials](https://docs.formfeed.dev/api/brand-and-partials#conflicts).
- Redelivering copies to the workspace's own bucket while the [storage connection](https://docs.formfeed.dev/api/bring-your-own-storage) is paused; `paused_reason` says why (`plan` when the plan no longer includes it).
- A `formfeed login` device code that was already used.
- `POST /webhooks/{id}/test` for a disabled endpoint, which receives nothing (_The endpoint is disabled; set enabled to true before sending a test_).

**Fix.**

- For a slug: choose another one, or update the existing template.
- For a version: fetch the latest version, merge your change into it, and send its `checksum` as `base_checksum`. `formfeed templates push --force` overrides the check when you are sure.
- For a shared partial: read it again (`formfeed partials pull <name>`), merge, and send its new `version` as `base_version`, or push with `--force`.
- For a paused connection: fix the credentials and test the connection again in the app, or move to a plan with bring your own storage.
- For a device code: start `formfeed login` again.
- For a webhook test: switch the endpoint on (`PUT /webhooks/{id}` with `"enabled": true`, or its switch on the **Webhooks** page), then send the test again.

**Retry.**

No. The same request gets the same answer. Change it as above, then send it again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 503 conformance_unavailable

The service that turns a rendered PDF into an [e-invoice](https://docs.formfeed.dev/templates/e-invoices) and validates it, and that validates a file rendered with [`pdf.ua`](https://docs.formfeed.dev/templates/accessible-pdf), is busy or not reachable right now. Where a second attempt could help, Formfeed has already made it. A render that asked for an e-invoice never comes back as a plain PDF instead, and one that asked for PDF/UA-1 never comes back unchecked: you get the validated file or this problem. The answer carries `Retry-After` (5 seconds). Nothing was charged.

**Causes.**

- Many e-invoices or PDF/UA files at the same time in your region. Each file is converted or validated on its own, so bursts queue up (_The conformance service is busy; retry shortly_).
- The service restarting after an update (_The conformance service is not reachable; retry shortly_).
- A region in which e-invoices or the PDF/UA check are not available (_E-invoices are not available in this region_, _PDF/UA validation is not available in this region_).

A conversion or validation that takes longer than its 20 seconds answers [`render_timeout`](https://docs.formfeed.dev/errors/render-timeout) instead.

**Fix.**

- Retry after the number of seconds in the `Retry-After` header, with the same `Idempotency-Key`.
- Send large numbers of documents as [async renders or a batch](https://docs.formfeed.dev/api/async-and-batch): the queue retries this failure by itself, 5 and then 10 seconds later. When all three attempts fail, the render fails as [`worker_exhausted`](https://docs.formfeed.dev/errors/worker-exhausted), with this code's message in `detail`.
- If it keeps happening, send the `request_id` to support@formfeed.dev.

**Retry.**

Yes, after `Retry-After` seconds, with the same `Idempotency-Key`; answers of `503` are not stored under the key. The SDKs do this for you.

## 422 data_validation_error

The render asked for [`validate_data: true`](https://docs.formfeed.dev/api/rendering#validate-data), and its `data` does not match the [data schema](https://docs.formfeed.dev/templates/typed-data#contract) stored with the template version. Formfeed checks the data only when a render or batch item asks for it with `validate_data: true`; without it, the data is not checked against the schema at all.

The check runs before anything is rendered, recorded or counted, so the failed request costs nothing. `errors` lists every place where the data does not match, each with `path` and `message`; `detail` repeats the first and says how many more there are. In a batch, `detail` starts with the item (`Item 3: …`), `index` names it, `failing_items` says how many items of the same template version failed, and nothing is queued.

**Causes.**

- A required field is missing, or a field has the wrong type.
- A value is not one of the values an `enum` allows (`must be one of "draft", "sent"`).
- A field the schema does not allow, where it sets `additionalProperties: false` (`is not allowed by the schema`).
- The data was built for an older version of the template, and the published version's schema changed.

**Fix.**

- Fix the fields `errors` names. Paths start at `data`, so `data.invoice.lines[0].qty` is `qty` in the first line.
- Compare your data with `GET /v1/templates/{id}/schema`, or check it without rendering with [`POST /v1/templates/{id}/validate`](https://docs.formfeed.dev/api/templates#validate).
- Generate types from the schema with `formfeed types`, so your code cannot send data of the wrong shape. See [Typed template data](https://docs.formfeed.dev/templates/typed-data).

A request with `validate_data: true` for a template version without a stored schema, or for `html` or a `url`, is refused with [`invalid_request`](https://docs.formfeed.dev/errors/invalid-request) instead.

**Retry.**

No. The same request gets the same answer. Fix the data, then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 422 einvoice_data_invalid

The render makes an [e-invoice](https://docs.formfeed.dev/templates/e-invoices), and its invoice data cannot make a valid one. The answer comes from one of two places:

- **Before anything is rendered.** The `_invoice` block of `data` is missing, a field is missing, a code is not in its list, or a total does not add up. `errors` lists every finding with its `path`, its `message` and, where one applies, the `rule` of EN 16931 or of the French specifications it breaks; `detail` repeats the first. Nothing is rendered, counted or charged. In a batch, `index` names the first item that failed, and nothing is queued.
- **After rendering.** The block passed that check, and the validator still refused the finished file. The problem then carries `einvoice`, the [report](https://docs.formfeed.dev/templates/e-invoices#response) of the validator, and `render_id`. The render is `failed` and keeps the report. No file is stored, and no plain PDF is delivered instead. A failed render costs nothing.

**Causes.**

- The data has no `_invoice` block. A template whose settings declare an e-invoice needs one in every PDF render (`path` is `data._invoice`).
- A total that does not add up to the cent (`BR-CO-10` to `BR-CO-16`), or a VAT breakdown that does not match the lines. Formfeed computes and rounds nothing for you.
- An amount with more than two decimals, such as a line total that was never rounded.
- A code that is not in its list: a unit written as `hours` instead of `HUR` (`BR-CL-23`), a country, a currency.
- A field the standard asks for is missing: the due date or the payment terms of an invoice with an amount due (`BR-CO-25`), the IBAN of a credit transfer (`BR-61`), an identifier of the seller (`BR-CO-26`).
- An invoice between two French businesses without the [fields France asks for](https://docs.formfeed.dev/templates/e-invoices#french-invoices), or in the profile `basic`, which France's reform does not accept.
- The profile `basic` with a `totals.rounding` other than 0, which that profile cannot carry.

**Fix.**

- Fix the fields `errors` names. Paths start at `data._invoice`, and list items count from 0, so `data._invoice.lines[0].unit_code` is the unit of the first line. The [field reference](https://docs.formfeed.dev/templates/e-invoices#invoice-block) says what each field takes.
- Find the findings before you send the request: the **E-invoice** section of the editor's **Page setup** lists them for the data set on screen, and `formfeed validate` checks every data set of a template offline. Both run the check the API runs.
- When the problem carries `einvoice`, read `einvoice.validation.messages`: each finding has its `rule` and its `location`, an XPath into the XML. The failed render keeps the same report, in `GET /v1/renders/{id}` and in the render log. If the data looks right to you, send the `render_id` to support@formfeed.dev.
- Which VAT category and rate apply to a sale is a question for your tax adviser. Formfeed checks only that the block is consistent with what you state.
- If this request should produce a plain PDF from a template that declares an e-invoice, send `"post": { "einvoice": false }`.

**Retry.**

No. Retrying the same request gives the same answer; change the data first. Send the corrected request with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 422 einvoice_display_mismatch

The render asked for an [e-invoice](https://docs.formfeed.dev/templates/e-invoices) with `display_check: "strict"`, and the text of the finished PDF does not show a value its XML states: the invoice number (`BT-1`), the net total (`BT-109`), the VAT total (`BT-110`), the total (`BT-112`), the amount due (`BT-115`) where it differs from the total, and with more than one VAT rate the VAT amount of each (`BT-117`). Amounts of zero are not looked for. The [display check](https://docs.formfeed.dev/templates/e-invoices#display-check) explains what is looked for.

`detail` names the values that are missing, `einvoice.display.missing` their business terms. The file itself was valid, which `einvoice.validation` says. The render is `failed` and keeps the report; no file is stored, and a failed render costs nothing. With `display_check: "warn"`, the default, the same finding does not fail the render.

Where the render also set [`pdf.ua`](https://docs.formfeed.dev/templates/accessible-pdf#e-invoice), the problem carries `accessibility` as well: the file had been checked against PDF/UA-1 by then, and the block says what that check found.

**Causes.**

- The template adds up its own totals from other fields than `_invoice`, and they differ from the block, for example because it rounds line by line.
- The template does not print the value at all, such as an invoice without a line for the VAT total.
- The template prints a rounded amount: `1.142 €` does not show `1142.40`.
- The template prints another invoice number than `_invoice.number`.
- The value is part of a picture. The check reads the text of the PDF.

**Fix.**

- [Print the values of `_invoice`](https://docs.formfeed.dev/templates/e-invoices#print-the-values) in the template instead of values from other fields.
- In the editor, the **Flow** and **Paged** previews of an invoice template count the values the document shows, and the **Problems** list names each one that is missing, before anything is rendered.
- To get the file while you fix the template, set `display_check` back to `warn`. The render then succeeds and reports the same values in `einvoice.display.missing`. The XML is the invoice, and a PDF that shows other invoice details can count as a second invoice, so do not leave it at that.

**Retry.**

No. Retrying the same request gives the same answer; change the template or the data first. Send the corrected request with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 422 einvoice_profile_unsupported

The [e-invoice options](https://docs.formfeed.dev/templates/e-invoices#options) name a profile or a flavour that Formfeed does not create. Formfeed creates the profiles `en16931` and `basic`, under the names `factur-x` and `zugferd`. Nothing was rendered or charged.

`field` is `post.einvoice.profile` or `post.einvoice.flavour`, wherever the value was set: in `post.einvoice`, in the request's `settings.einvoice` or in the template's settings. `value` is the value that was refused, and `detail` says why. In a batch, `index` names the first item that failed, and nothing is queued.

**Causes.**

- `minimum` or `basic-wl`: profiles of the specification that carry no invoice lines and are not an invoice on their own.
- `extended`: a profile Formfeed does not offer.
- `xrechnung`: not available yet.
- A name Formfeed does not know, such as `comfort`, an older name of the EN 16931 profile, or a typing mistake.
- A `flavour` other than `factur-x` or `zugferd`.
- A template whose `settings.json` was pushed with such a value. `formfeed validate` reports it before the push.

**Fix.**

- Send `"profile": "en16931"`, which carries every field of the European standard and is the default, or `"profile": "basic"`, the smallest profile that is an invoice on its own.
- Send `"flavour": "factur-x"` or `"flavour": "zugferd"`. The two are one specification under two names, and the file is the same.
- If the value comes from the template, change it in the **E-invoice** section of the editor's **Page setup** or in the project's `settings.json`, and publish the version.

**Retry.**

No. Retrying the same request gives the same answer; change the options first. Send the corrected request with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 500 einvoice_validator_error

Every [e-invoice](https://docs.formfeed.dev/templates/e-invoices) is validated before it is delivered. Here the rules of the validator stopped before they finished, so they say nothing about the invoice either way, and Formfeed does not deliver a file that was not checked. The render is `failed`; no file is stored, no plain PDF is delivered instead, and a failed render costs nothing.

Nothing in your `_invoice` block is shown to be wrong. The validator failed, not the invoice, and that is ours to fix.

The problem carries `einvoice`, the report of the validator. Its `validation.messages` hold one error without a `rule`, which says that the invoice could not be validated. The failed render keeps the same report, with `xml_sha256`, the hash of the XML that was checked.

**Causes.**

- An expression of the rule set failed on the XML instead of reporting a finding. The rule set is the one the open-source validator Mustang carries; `einvoice.validation.versions` names its release.

**Fix.**

- Send the `render_id` to support@formfeed.dev. We see where the rules stopped, and fix it or report it to the makers of the validator.

**Retry.**

No. The same invoice stops the same rules. Async renders are not retried for this code either.

## 413 file_too_large

The uploaded file is larger than the limit for where it goes. `bytes` is the size of the file, `limit` the limit that applied, both in bytes. Nothing was stored or charged.

| Upload                                                                     | Limit  |
| -------------------------------------------------------------------------- | ------ |
| A file for the workspace's [file library](https://docs.formfeed.dev/api/files) (`POST /v1/files`) | 10 MB  |
| An office document sent to [`/pdf/convert`](https://docs.formfeed.dev/api/pdf-tools#convert)      | 20 MB  |
| The file of a [Word or PowerPoint template](https://docs.formfeed.dev/templates/office-templates) | 20 MB  |
| A logo of the brand kit, in the app                                        | 512 kB |

**Causes.**

- A photo straight from a camera, or a logo exported as a print-resolution PNG.
- A scanned PDF you meant to merge; scans of many pages grow quickly.
- An office document above 20 MB, usually because of large pictures in it.

**Fix.**

- Scale the image to the size the document shows it at. A logo 400 px wide is enough for print at the sizes templates use it.
- Compress: WebP or a JPEG at quality 80 is usually a fraction of the PNG.
- For a large PDF you only want to merge, split it and merge the parts, or keep it outside the library and render it in pieces.
- For an office document, compress its pictures before converting (in Word: File › Compress Pictures).

**Retry.**

No. The same file gives the same answer. Make the file smaller, then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 415 file_type_unsupported

The uploaded file is not one Formfeed takes here. The type is read from the file's bytes, never from its name, so renaming a file does not change the answer. `detail` says what the file is and what is accepted. Nothing was stored or charged.

| Where                                                                      | Accepted                                                     |
| -------------------------------------------------------------------------- | ------------------------------------------------------------ |
| [`/pdf/convert`](https://docs.formfeed.dev/api/pdf-tools#convert)                                 | DOCX, XLSX, PPTX, ODT, ODS, ODP, DOC, XLS, PPT, RTF and HTML |
| The file of a [Word or PowerPoint template](https://docs.formfeed.dev/templates/office-templates) | DOCX for a `docx` template, PPTX for a `pptx` template       |

Documents with macros are refused everywhere.

**Causes.**

- A PDF, image or plain text file sent to `/pdf/convert`. A `.doc` that is really HTML (Word's "save as web page", many report exports) is converted as HTML and not refused.
- A macro-enabled document (`.docm`, `.xlsm`, `.pptm`) or one that contains a VBA project or Basic macros.
- A template or slide show format (`.dotx`, `.xltx`, `.potx`, `.ppsx`) or a binary workbook (`.xlsb`).
- A PowerPoint file uploaded for a Word template, or the other way round (_A docx template takes a Word (DOCX) file; this one is pptx_).

**Fix.**

- Save the document as DOCX, XLSX or PPTX in Word, Excel or PowerPoint (File › Save As), which also removes macros.
- Send PDFs to the other PDF tools instead: `/pdf/merge`, `/pdf/protect`, `/pdf/watermark`.
- For a template, upload the format the template was created as, or create a new template of the other kind.

**Retry.**

No. The same file gives the same answer. Change the file, then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 403 forbidden

The key is valid, but this request is not allowed for it. There are three kinds of refusal, and the problem tells them apart:

- **A missing scope.** `detail` names it: _This API key lacks the "template:write" scope_.
- **A plan limit.** The problem carries `feature` (what the plan lacks), `plan` (the plan of the organisation) and, where the plan allows a number, `limit`.
- **A caller that may not do this**: a call from a browser, a connected AI agent, or a member without the role. `detail` says which.

**Causes.**

Missing scopes:

- Pushing or publishing templates with a key that has only the default scopes (`template:write` is needed).
- Managing webhooks without `webhook:manage`, for example when Zapier or n8n registers a trigger.
- Reading usage without `account:read`, deleting outputs without `file:delete`, reading a render's stored request without `render:input`, or deleting a workspace without `workspace:delete`.

Plan limits (`feature`):

| `feature`         | Refused                                                                                                                                                                                                                                                                                         | Allowed from   |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- |
| `post_processing` | `/pdf/merge`, `/pdf/protect`, `/pdf/watermark`, or a `post` block with `merge_after`, `watermark` or `password`. `/pdf/info` works on every plan.                                                                                                                                               | Starter        |
| `office`          | `/pdf/convert`, and live renders of [Word and PowerPoint templates](https://docs.formfeed.dev/templates/office-templates). On Free, test renders of these templates make the marked PDF.                                                                                                                               | Starter        |
| `channels`        | Rendering or setting a named [release channel](https://docs.formfeed.dev/templates/release-channels#plans) on Free; on Starter and Pro, a new channel beyond the plan's number per template (with `limit`: 2 on Starter, 10 on Pro and Business).                                                                      | Starter        |
| `canary`          | A channel with a canary.                                                                                                                                                                                                                                                                        | Pro            |
| `byo_storage`     | Connecting a bucket for [bring your own storage](https://docs.formfeed.dev/api/bring-your-own-storage).                                                                                                                                                                                                                | Pro            |
| `templates`       | A new template, a copy or a restored one when the organisation already has the plan's number (`limit`: 5 on Free). Archived templates do not count.                                                                                                                                             | Starter        |
| `webhooks`        | A new [webhook endpoint](https://docs.formfeed.dev/api/webhooks) when the organisation already has the plan's number (`limit`: 1 on Free, 5 on Starter), disabled ones included. `formfeed listen` sessions do not count.                                                                                              | Pro            |
| `einvoice`        | An [e-invoice](https://docs.formfeed.dev/templates/e-invoices#plans-and-units) from a live render on Free; or from a test render once the day's test e-invoices are used up (Free 20, Starter 100, Pro 500, Business 2,000). The second answer carries `limit`, `used` and `resets_at`, the start of the next UTC day. | Starter (live) |

Callers that may not do this:

- A request from a web page (`detail`: _This API key may not be used from a browser origin_): an API key only works server-side; see [From a browser](https://docs.formfeed.dev/api/overview#browsers). Microsoft's Power Automate, Power Apps and Copilot Studio portals are not affected; their connector calls go through Microsoft's servers.
- A connected AI agent (an MCP client signed in with OAuth) doing what only a person may do (`detail`: _A connected agent may not do this; sign in to Formfeed and do it there_), or an agent of an account with two-factor authentication that the account never confirmed.
- The routes under `/workspaces/{id}/storage` and the brand kit's logos: only owners and admins signed in to the app manage them, never an API key.

**Fix.**

- For a scope: `detail` names it. Create a key that has it; scopes cannot be added to an existing key. The [scope table](https://docs.formfeed.dev/api/authentication#scopes) says which endpoint needs which scope.
- For `post_processing`, `office`, `channels`, `canary`, `byo_storage`, `templates` or `webhooks`: upgrade the plan. Test keys on a paid plan can try a feature for free. For `channels`, you can also keep rendering and moving the `published` channel, which every plan has; for `templates`, archive a template you no longer use; for `webhooks`, delete an endpoint you no longer use (a no-code trigger removes its own when it is switched off).
- For `einvoice`: render with a test key to try it, wait until `resets_at` when the day is used up, or render live on Starter and above, which has no daily number. To get a plain PDF from a template that declares an e-invoice, send `"post": { "einvoice": false }`.
- For a browser call: call the API from your backend and hand the browser the result, for example the render's `download_url`.
- For an agent or the app-only routes: do it in the app, signed in as an owner or admin.

**Retry.**

No. The same request gets the same answer. Fix what the problem names, then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 400 invalid_request

The request could not be accepted as it is: the body is not JSON, a field has the wrong type or value, or fields are combined in a way the API does not allow. When the body or the query does not fit the contract, `issues` lists every problem, each with the field (`path`) and what is wrong (`message`). For a combination of fields, `detail` explains it and `field` names the field.

**Causes.**

- The body is not a JSON object, for example because it is empty or cut off. `issues` then has one entry with an empty `path`.
- A render names no source, or more than one: send exactly one of `template`, `html` or `url`.
- A value outside its list, such as `"mode": "later"` or `"output": "gif"`, or a field of the wrong type.
- An output the source cannot make: a PDF or image template, `html` and `url` make `pdf`, `png`, `jpg` or `webp`; a Word template `docx` or `pdf`; a PowerPoint template `pptx` or `pdf` (`field: "output"`, with the `allowed` values).
- `version` is not `published`, a channel name, `latest` or a version number, or it is `latest` with a live key. Only a test key renders the latest draft.
- A `url` or `webhook_url` that the static address rules refuse: not a valid URL, a scheme other than `https` (a `url` may also be `http`), a user name or password in the URL, or a host that is private, loopback, link-local, a cloud metadata address, `localhost`, a numeric shorthand such as `2130706433`, or a name under `.internal`, `.local`, `.localhost` or `.fly.dev`. The `message` starts with `URL rejected:`. A name that only resolves to a private address is refused later, with [`navigation_failed`](https://docs.formfeed.dev/errors/navigation-failed).
- `webhook_secret` without `webhook_url`, or shorter than 16 characters.
- `validate_data: true` for `html` or a `url`, or for a template version without a stored data schema (`field: "validate_data"`). See [Check the data first](https://docs.formfeed.dev/api/rendering#validate-data).
- `post` (merge, watermark, password) on an output other than PDF, a watermark with both `text` and `image` or neither, or `merge_after` sources stored in another region than the render.
- An [e-invoice](https://docs.formfeed.dev/templates/e-invoices) asked for on an image, or together with a password (`field: "post.password"`): PDF/A forbids encryption.
- [`pdf.ua`](https://docs.formfeed.dev/templates/accessible-pdf) with a value other than `true`, `false` or `{ "check": "strict" | "report" }`, on an image, on a Word or PowerPoint template, or together with `post.merge_after`.
- A query parameter out of range, such as `limit=500` on `GET /v1/renders` (1 to 100).
- A template whose [shared partials](https://docs.formfeed.dev/templates/shared-partials) add up to more than 2 MB in one render (`partials`, `bytes`, `limit`).
- A shared partial name that is not 1 to 64 lowercase letters, digits, dashes and underscores starting and ending with a letter or digit, or a new shared partial when the organisation already holds 200 (`feature: "partials"`).
- An upload that is empty, has no `file` part, or is not a type the file library takes.
- A watermark text with characters the watermark font cannot draw. The font covers Latin, Greek and Cyrillic.

Most of these are answered before anything is recorded. The last one, and a few others the render service finds itself, are found after the render was recorded: then the problem carries `render_id`, and the render log shows a failed render that cost nothing.

**Fix.**

- Read `detail` and the `issues` array: each issue names the field (`path`) and what is wrong with it. In a batch, `index` says which item.
- Compare the request with the [API reference](https://docs.formfeed.dev/api/reference/formfeed). The SDKs type every field, so most of these mistakes do not compile.

**Retry.**

No. The same request gets the same answer. Fix what the problem names, then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 409 last_workspace

The workspace was not deleted, because it is the last one of its organisation. An organisation always keeps at least one workspace. Nothing was changed.

**Causes.**

- [`DELETE /v1/workspaces/{id}`](https://docs.formfeed.dev/api/reference/delete-workspace) (or `formfeed workspaces delete <id> --yes`) for the only workspace an organisation has left, for example when a cleanup job deletes every preview workspace and the organisation has nothing else. The key needs the `workspace:delete` scope.

**Fix.**

- Keep one workspace, and let cleanup jobs skip it.
- To remove everything, delete the organisation in the app under **Settings**. The API does not delete organisations.

**Retry.**

Only after another workspace exists in the organisation.

## 410 listen_session_ended

The [`formfeed listen`](https://docs.formfeed.dev/api/develop-webhooks-locally) session is gone: the CLI was stopped or stayed disconnected until its lease of about two minutes ran out, a new session of the same key or member replaced it, or someone ended it in the app or with [`DELETE /webhooks/listen/{id}`](https://docs.formfeed.dev/api/reference/end-listen-session).

**Causes.**

- `formfeed webhooks resend` or [`POST /webhooks/events/{id}/resend`](https://docs.formfeed.dev/api/reference/resend-webhook-event) with the id of a session that has stopped.
- A second `formfeed listen` with the same key or login, which ends the first one.
- The session was ended on the **Webhooks** page of the app.

Deliveries Formfeed still had for the session end with this status too, and are not retried. While a session lives but no CLI is connected, its deliveries wait and are retried instead.

**Fix.**

- Start `formfeed listen` again. With the same API key the new session keeps the signing secret, so your local server needs no change.
- Send the event to the new session. See [Develop webhooks locally](https://docs.formfeed.dev/api/develop-webhooks-locally).

**Retry.**

No. The session does not come back; start a new one.

## 409 listen_session_limit

A new [`formfeed listen`](https://docs.formfeed.dev/api/develop-webhooks-locally) session was refused, because three sessions are already running in the workspace. The limit counts the sessions of all keys and members of the workspace together.

**Causes.**

- Three colleagues, or three machines with different keys, are listening in the same workspace.
- `POST /webhooks/listen` called by a script with different keys that never ends its sessions.

Your own earlier session does not count: a new session of the same key or signed-in member ends the previous one.

**Fix.**

- Stop a session you no longer need with Ctrl+C.
- Owners, admins and the member who started a session can end it on the **Webhooks** page of the app, or with [`DELETE /webhooks/listen/{id}`](https://docs.formfeed.dev/api/reference/end-listen-session).
- See [Develop webhooks locally](https://docs.formfeed.dev/api/develop-webhooks-locally#limits).

**Retry.**

Once a session has ended, the same request succeeds. If you call `POST /webhooks/listen` yourself with an `Idempotency-Key`, use a new one: a `4xx` answer is stored under its key for 24 hours.

## 422 navigation_failed

A `url` render could not load the page it names. The render is recorded as failed and costs nothing. `detail` carries the reason, often the browser's own message.

**Causes.**

- The page answered with an HTTP error, 400 or above (`URL returned HTTP 404`).
- The page did not finish loading within 30 seconds. By default Formfeed waits until the network is idle; a page that keeps polling, streams, or holds a connection open never gets there, and the problem says `Timeout 30000ms exceeded`.
- An `http:` address. The API accepts it, but the browser loads `https` only (`scheme http: is not allowed`).
- The host name does not resolve, or resolves to a private or internal address (`URL blocked: dns failure: …`, `URL blocked: resolves to private address …`). Addresses that are private on their face are refused earlier, with [`invalid_request`](https://docs.formfeed.dev/errors/invalid-request).
- The TLS certificate is invalid or expired, or the server refused the connection (`net::ERR_CERT_…`, `net::ERR_CONNECTION_REFUSED`).
- A redirect to an address Formfeed does not fetch.
- The page needs a login or blocks automated browsers, and answers with an error page or status. A `url` render sends no cookies or headers of yours.

**Fix.**

- Open the URL in a private browser window from outside your network.
- For pages that keep a connection open, set [`waitFor.networkIdle`](https://docs.formfeed.dev/templates/settings#waiting-for-content) to `false`: the page is printed right after its `load` event. Add `waitFor.selector` if its content arrives later.
- Render pages that need a login from HTML instead: fetch the content yourself and send it as `html`.

**Retry.**

Only when the page itself had a passing problem (a `5xx` answer, a timeout of a slow server). Otherwise the same request gets the same answer. Send the retry with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries). Async renders are not retried by the queue for this code.

## 404 not_found

The object the request names does not exist in the workspace of the key, or the path does not exist at all. Templates, template versions and channels have codes of their own: [`template_not_found`](https://docs.formfeed.dev/errors/template-not-found) and [`channel_not_found`](https://docs.formfeed.dev/errors/channel-not-found).

**Causes.**

- An id from another workspace: every key sees only its own workspace, and a workspace's renders, jobs and files are not visible from another one, not even in the same organisation.
- An id that is cut short or mistyped. Ids are a prefix and 32 hex digits (`rnd_7da09ab6cd9946b798c187c2a9350852`, `job_…`, `fil_…`); upper or lower case does not matter, and the UUID form is accepted as well.
- A path without `/v1` (`/renders` instead of `/v1/renders`), or a path that does not exist.
- The file of a Word or PowerPoint template version that is no longer stored, or the `/file` of an HTML template version, which has none.
- An event, webhook endpoint or `formfeed listen` session that does not exist in this workspace, or an endpoint that is disabled, for a resend.
- A shared partial name the organisation does not have.
- A `formfeed login` device code that is unknown or expired (_The code expired; start the login again_).

**Fix.**

- Check that the key belongs to the workspace that created the object: `GET /v1/account` or `formfeed whoami` shows it.
- Copy the id from the API's answer rather than typing it.
- For templates, see [`template_not_found`](https://docs.formfeed.dev/errors/template-not-found).

**Retry.**

No. Retrying the same request gives the same answer; correct the id, the path or the key first.

## 501 not_implemented

The code is reserved for an endpoint or option that is part of the API contract before it is available. No endpoint or option answers with it today: everything in the [API reference](https://docs.formfeed.dev/api/reference/formfeed) works.

**Causes.**

- None today. If you receive it, Formfeed made a mistake.

**Fix.**

- Send the `request_id` to support@formfeed.dev, and use the request without the option meanwhile.

**Retry.**

No. The same request gets the same answer until the feature is available.

## 422 office_conversion_failed

LibreOffice, which converts office documents to PDF, failed on this document. Formfeed already tried a second time where the 60 seconds of the conversion left room for it. `reason` is the first line of what the converter said. It happens on [`/pdf/convert`](https://docs.formfeed.dev/api/pdf-tools#convert) and when a [Word or PowerPoint template](https://docs.formfeed.dev/templates/office-templates) is rendered to PDF. Nothing was charged.

**Causes.**

- A document that Word or Excel can repair when opening it, but that is damaged underneath.
- Content LibreOffice does not support, such as some embedded objects from other applications.

A document LibreOffice refuses to read at all answers [`source_invalid`](https://docs.formfeed.dev/errors/source-invalid), one with a password [`source_encrypted`](https://docs.formfeed.dev/errors/source-encrypted), and a conversion that runs out of time [`render_timeout`](https://docs.formfeed.dev/errors/render-timeout).

**Fix.**

- Open the document in its application and save it again; Word and Excel repair many problems silently.
- Remove embedded objects (OLE objects, ActiveX controls) and convert again.
- If the document opens in LibreOffice on your computer and still fails here, contact support with the `request_id`.

**Retry.**

No. The conversion was already retried once; change the document first, then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 422 office_document_invalid

The file looks like an office document, but its container is damaged or fails one of the safety checks every document goes through before it is converted or filled: documents sent to [`/pdf/convert`](https://docs.formfeed.dev/api/pdf-tools#convert) and the files of [Word and PowerPoint templates](https://docs.formfeed.dev/templates/office-templates). `detail` says what is wrong; `entry`, where present, names the part of the file. Nothing was charged.

**Causes.**

- A file that was cut off during upload or download (_The file is not a valid ZIP container_, _The ZIP central directory is corrupt_, _Part … cannot be decompressed_).
- A document whose XML declares a `DOCTYPE` or an `ENTITY`, which office applications never write (_The document declares a DOCTYPE_).
- A container with duplicate or unsafe entry names (`../`, absolute paths), more than 2,000 entries, or an unsupported compression method, usually made by a tool rather than Word, Excel or PowerPoint.
- A damaged Word 97–2003, Excel or PowerPoint binary file (_The compound document is corrupt_).

**Fix.**

- Open the file in Word, Excel, PowerPoint or LibreOffice and save it again under a new name.
- Check that your upload sends the file's bytes unchanged (no text encoding, no base64 in a form field).

**Retry.**

No. The same file gives the same answer. Change the file, then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 422 office_document_too_large

The document is larger, once unpacked, than Formfeed processes. Office files are compressed archives: a 5 MB file can contain far more. The limit also applies to what a [Word or PowerPoint template](https://docs.formfeed.dev/templates/office-templates) becomes when it is filled with data. `limit` is the limit that applied, in bytes or, for pictures, as a count; `entry` names the part of the file where one is to blame. Nothing was charged.

| What                                             | Limit                  |
| ------------------------------------------------ | ---------------------- |
| All parts of the document together, unpacked     | 200 MB                 |
| One part, unpacked                               | 20 MB                  |
| A filled template                                | 50 MB                  |
| Pictures and codes placed in one filled template | 50, and 20 MB together |

**Causes.**

- A document with many large pictures, or a spreadsheet with very large sheets (_Part xl/worksheets/sheet1.xml is larger than 20971520 bytes_).
- A template filled with very large data, for example a table of thousands of rows (_The filled document is larger than 50 MB_).
- A template that places more than 50 pictures or codes (_A document can hold at most 50 images and codes_), or pictures of more than 20 MB together.

A container with more than 2,000 entries answers [`office_document_invalid`](https://docs.formfeed.dev/errors/office-document-invalid), and an upload over 20 MB [`file_too_large`](https://docs.formfeed.dev/errors/file-too-large).

**Fix.**

- Split the document, or compress its pictures (in Word: File › Compress Pictures).
- For spreadsheets, convert the sheets you need as separate files.
- For templates, send less data per document, or smaller pictures.

**Retry.**

No. The same request gets the same answer. Change the file or the data, then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 503 office_unavailable

The converter that turns Word, Excel and PowerPoint documents into PDFs is busy or not reachable right now. Formfeed already tried again within the 60 seconds a conversion has. The answer carries `Retry-After` (5 seconds). It happens on [`/pdf/convert`](https://docs.formfeed.dev/api/pdf-tools#convert) and when a [Word or PowerPoint template](https://docs.formfeed.dev/templates/office-templates) is rendered. Nothing was charged.

**Causes.**

- Many conversions at the same time in your region. Each document is converted on its own, so bursts queue up.
- The converter restarting after an update (_The office converter is not reachable; retry shortly_).
- A Word or PowerPoint template with WebP, GIF or AVIF pictures, which need the browser to be drawn, when no browser slot was free (_No browser slot was free to draw the document's pictures_).

**Fix.**

- Retry after the number of seconds in the `Retry-After` header, with the same `Idempotency-Key`.
- Spread large numbers of conversions over time, or send template renders [async or as a batch](https://docs.formfeed.dev/api/async-and-batch): the queue retries this failure by itself, 5 and then 10 seconds later. When all three attempts fail, the render fails as [`worker_exhausted`](https://docs.formfeed.dev/errors/worker-exhausted) with this message.

**Retry.**

Yes, after the seconds in `Retry-After`, with the same `Idempotency-Key`; answers of `503` are not stored under the key. The SDKs do this for you.

## 422 output_too_large

The render produced more than it may: the finished file is larger than the plan's limit for one output, or the HTML a template produced is larger than 20 MB. The render is recorded as failed, no file is stored, and it costs nothing. `detail` gives both sizes in bytes.

| Plan     | Largest output |
| -------- | -------------- |
| Free     | 10 MB          |
| Starter  | 50 MB          |
| Pro      | 100 MB         |
| Business | 250 MB         |

No output can be larger than 250 MB, whatever the plan.

**Causes.**

- Large images embedded at full resolution, especially as data URIs.
- A template that produces more than 20 MB of HTML, usually a very long table.
- A merge or `post.merge_after` whose documents add up to more than the limit.
- An e-invoice or a PDF/UA-1 file too large for the validator (_The PDF is too large to be made an e-invoice_).

A Word or PowerPoint template that grows too large while it is filled answers [`office_document_too_large`](https://docs.formfeed.dev/errors/office-document-too-large) instead, and `html` sent in a request is limited to 5 MB before anything is rendered ([`invalid_request`](https://docs.formfeed.dev/errors/invalid-request)).

**Fix.**

- Scale and compress images before you reference them; the `image` helper sets the display size, not the file size.
- Split very large documents into several renders.
- Upgrade the plan for a larger limit. See [Plan limits](https://docs.formfeed.dev/api/limits-and-usage#plan-limits).

**Retry.**

No. The same request gets the same answer. Change the data, the template or the plan first, then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 422 page_limit_exceeded

The PDF has more pages than the plan allows for one document. `page_count` is how many pages it would have had. The limit counts the finished file, so pages appended with `post.merge_after` count too, as do the pages of a merge or a conversion. The render is recorded as failed, no file is stored, and it costs nothing.

| Plan       | Pages per PDF |
| ---------- | ------------- |
| Free       | 20            |
| Starter    | 200           |
| Pro        | 500           |
| Business   | 1,000         |
| Enterprise | 5,000         |

Test keys have the same limit as live keys of the plan.

**Causes.**

- The data is much larger than expected, for example a report over a whole year instead of a month.
- A layout problem that repeats content, such as an element taller than a page or a page break inside a loop.
- `post.merge_after` or `/pdf/merge` appends documents that together pass the limit.
- An office document whose spreadsheet prints across many pages; `single_page_sheets` puts each sheet on one page.

**Fix.**

- Look at the page count of a preview with the same data; `page_count` in the problem tells you how many pages it would have been.
- Split the document into several renders, or upgrade the plan. See [Plan limits](https://docs.formfeed.dev/api/limits-and-usage#plan-limits).

**Retry.**

No. The same request gets the same answer. Change the data, the template or the plan first, then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 500 pdfa_conversion_failed

An [e-invoice](https://docs.formfeed.dev/templates/e-invoices) is a PDF/A-3b that carries the invoice as XML. The rendered PDF could not be made one, or the converted file did not pass the PDF/A check. The render is `failed`; no file is stored, no plain PDF is delivered instead, and a failed render costs nothing.

The problem carries one of two things:

- `einvoice`, the report of the validator, when the converted file was checked and refused. Its `validation.messages` with `"part": "pdf"` name the clause of PDF/A (`rule`) and the object of the PDF (`location`). The failed render keeps the same report.
- `reason`, the first line of what the converter said, when the PDF could not be converted at all.

**Causes.**

- A font that is not embedded (`6.2.11.4.1-1`). PDF/A needs every font inside the file. A PDF of the file library that is appended with `post.merge_after` can bring such a font.
- A font whose embedded program disagrees with the widths the PDF states (`6.2.11.5-1`). The office converter writes such a font for OpenType fonts with CFF outlines, which on our side is the font for Chinese, Japanese and Korean text in a Word or PowerPoint template.
- A failure of the converter itself, reported in `reason`.

**Fix.**

- Read `einvoice.validation.messages` to see which clause of PDF/A the file breaks, and where.
- For a font that is not embedded in an appended PDF: create that PDF again with its fonts embedded, or leave it out of the e-invoice.
- For anything else, send the `render_id` to support@formfeed.dev. A document that Formfeed rendered and cannot convert is ours to fix.

**Retry.**

No. The same document fails the same way. Async renders are not retried for this code either.

## 422 pdfua_validation_failed

The render set [`pdf.ua`](https://docs.formfeed.dev/templates/accessible-pdf) with the strict check, and the finished file fails at least one of the machine-checkable rules of PDF/UA-1. Nothing is delivered: a file that was asked for as PDF/UA-1 never comes back as a plain PDF instead.

`accessibility.failures` lists each rule that failed: its number in the validator's rule set, what to change in the template, and the start tags of the elements it is about. The render is `failed` and keeps the same block, so [`GET /renders/{id}`](https://docs.formfeed.dev/api/rendering) and the render log show it later. No file is stored, and a failed render costs nothing.

Where the render was also an [e-invoice](https://docs.formfeed.dev/templates/accessible-pdf#e-invoice), the problem carries `einvoice` beside `accessibility`: the invoice itself was valid, and the file it was converted to is the one that failed PDF/UA-1. No e-invoice is delivered either.

**Causes.**

- A heading level is skipped (`heading-order`): an `h3` straight after an `h1`, or a document that starts with an `h2`. Headings chosen for their size are the usual reason.
- An image has no `alt` attribute (`image-alt`), or a chart, a `canvas` or an `svg` with `role="img"` has no name (`figure-name`).
- The document has no title (`document-title`): no `pdf.metadata.title` in the settings and no `<title>`.
- The rows of a table span different numbers of columns (`table-rows`), often because of a `colspan` that does not add up.
- A data cell of a table is empty (`table-cell-empty`). The browser writes no cell for an empty `td`, so the row is shorter in the file than in the markup; a non-breaking space in the cell is enough.
- Text sits under a CSS `mask` (`masked-text`).

A failure without a `code` is a rule the template check has no finding for; its `message` is the validator's own description of the rule.

**Fix.**

- Change the template as `message` says. `elements` shows which tags are meant.
- In the editor, turn on **Accessible PDF (PDF/UA-1)** in the **Document** section of the page setup. The **Flow** and **Paged** previews then check the page and list each finding under **Problems**, at its line, before anything is rendered.
- `formfeed validate` reports what the source alone already shows: a missing title, an image without `alt`.
- To get the file while you fix the template, set the check to `report`: `"pdf": { "ua": { "check": "report" } }`. The render then succeeds, the file is delivered without the PDF/UA identifier, and `accessibility.conformant` is `false`.

[Accessible PDFs](https://docs.formfeed.dev/templates/accessible-pdf) explains each finding and what the check does and does not prove.

**Retry.**

No. Retrying the same request gives the same answer; change the template or the data first. Send the corrected request with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 402 quota_exceeded

A live render, batch or PDF tool was refused before anything was rendered, because the organisation has no [units](https://docs.formfeed.dev/api/limits-and-usage#units) left for it in this billing period. `period_end` is the day the next period starts. Test keys are never refused for this: test renders are not counted.

**Causes.**

- The Free plan has used its 100 units of the period. Free has no overage, so live renders stop until the next period.
- A paid plan reached its hard cap: the included units, the prepaid units of bought packs, and overage worth three times the plan's monthly price. Then `detail` reads _Overage for this period reached the hard cap of … units_, the problem carries `hard_cap_units`, and automatic top-up is off.
- A batch needs at least one unit per item, and fewer are left (_The batch needs at least 40 units; 12 remain in this period_). Nothing of the batch is queued.

A paid plan's period runs monthly from the day of the subscription, not by calendar month; see [Billing periods](https://docs.formfeed.dev/api/limits-and-usage#periods) and [When units run out](https://docs.formfeed.dev/api/limits-and-usage#when-units-run-out).

**Fix.**

- Wait for the next period (`period_end`), upgrade the plan, or buy a unit pack on the billing page.
- On a paid plan, buy a prepaid unit pack on the **Billing** page: its units raise the hard cap at once.
- Keep developing with a test key: test renders are not metered.
- Subscribe to the `quota.warning` [webhook](https://docs.formfeed.dev/api/webhooks) to hear about it when 80 % of the included units are used.

**Retry.**

Not until units are available again. Then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 429 rate_limited

The workspace sent more requests that start work than its plan allows per second. Nothing was done. `limit` is the size of the bucket that ran out and `window_seconds` the time it refills over (1 for the plan's requests per second, 60 for the minute of a Free test key), and the answer carries `Retry-After` (seconds to wait) and the `RateLimit-Limit`, `RateLimit-Remaining` and `RateLimit-Reset` headers.

The limit is a token bucket per workspace, shared by all its keys, live and test: its size and its refill per second are the plan's requests per second, so a short burst up to that size passes.

| Plan     | Requests per second |
| -------- | ------------------- |
| Free     | 1                   |
| Starter  | 5                   |
| Pro      | 15                  |
| Business | 40                  |

Test keys on the Free plan have a second bucket of 30 requests per minute.

**Causes.**

- A loop that renders documents one by one as fast as it can.
- Several services sharing one workspace.
- Free test keys sending more than 30 requests in a minute (then `limit` is 30).

Only requests that start work count: `POST` to `/v1/renders` and `/v1/renders/batch`, the PDF tools under `/v1/pdf/`, the [apitemplate.io-compatible endpoints](https://docs.formfeed.dev/api/compat-apitemplate), and saving or testing a storage connection. Reads such as `GET /v1/renders/{id}` are not limited, so polling never trips it.

**Fix.**

- Wait `Retry-After` seconds and retry with the same `Idempotency-Key`. The SDKs do this for you.
- Send many documents as one [batch](https://docs.formfeed.dev/api/async-and-batch#batch) instead of many single requests: up to 1,000 documents in one request.
- See [Limits and usage](https://docs.formfeed.dev/api/limits-and-usage#rate-limits) for the limits per plan.

**Retry.**

Yes, after the number of seconds in `Retry-After`, with the same `Idempotency-Key`. This answer is given before the key is looked at and is not stored, so the retry really runs.

## 503 region_unavailable

The render service of the region could not take the request. Formfeed renders in the EU (`eu`, the default); see [Data residency](https://docs.formfeed.dev/api/data-residency). The answer always carries `Retry-After`. Nothing was rendered or charged.

**Causes.**

- The render service of the region is busy or restarting (`detail`: _The render region is busy, retry shortly_; `Retry-After` comes from the service, usually 2). The PDF tools, conversions and checks such as `validate_data` try once more and then answer with this code. A synchronous render does not: it waits up to 30 seconds for a free render slot and then goes to the queue, answering `202` with the header `X-Render-Async-Fallback: true` ([the automatic fallback](https://docs.formfeed.dev/api/async-and-batch#the-automatic-fallback)).
- A workspace whose region has no render service (`detail`: _Region us is not available_, `Retry-After: 60`). The answer comes before anything is recorded, for synchronous and asynchronous renders and batches alike. Only `eu` is in service. A request that names a region other than its workspace's answers `400 invalid_request` instead.

**Fix.**

- For a busy region, retry after the seconds in `Retry-After`, with the same `Idempotency-Key`. The SDKs do this for you.
- Send large numbers of documents as [async renders or a batch](https://docs.formfeed.dev/api/async-and-batch): the queue waits for free slots instead of answering this code.
- Leave `region` out: a render runs in its workspace's region.

**Retry.**

Yes, after the seconds in `Retry-After`, with the same `Idempotency-Key`; answers of `503` are not stored under the key. A region without a render service answers the same way every time.

## 410 render_input_expired

The render exists, but its request is not kept: the workspace does not keep render requests, or the period it keeps them for has ended. `request_data_retention` in the problem is the workspace's setting: `off`, `24h` or `7d`.

**Causes.**

- `GET /renders/{id}/input` or `formfeed renders pull` in a workspace where **Keep render requests** is off, which is the default.
- The render completed more than 24 hours (or 7 days) ago.
- The render completed before the setting was turned on.
- The plan was downgraded, which shortened the retention at once.

**Fix.**

- Ask an owner or admin to turn on [**Keep render requests**](https://docs.formfeed.dev/templates/reproduce-a-failed-render#keep-render-requests) under **Settings** (Starter plan and above: up to 24 hours on Starter, up to 7 days from Pro on), then reproduce the problem with a new render.
- Pull renders soon after they fail. See [Reproduce a failed render](https://docs.formfeed.dev/templates/reproduce-a-failed-render).

**Retry.**

No. A request that was not kept does not come back.

## 504 render_timeout

A step of the render took longer than its limit. There is no limit on the render as a whole; each step has its own:

| Step                                                                                                                                             | Limit                                                 | `detail`                                                                                          |
| ------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| The template language (Jinja2, Handlebars) fills the template with the data                                                                      | 5 seconds                                             | _Template render took 6120 ms; the limit is 5000 ms_                                              |
| Chromium loads the document, until none of its requests is running ([`waitFor.networkIdle`](https://docs.formfeed.dev/templates/settings#waiting-for-content) says how) | 30 seconds                                            | _Render exceeded 30000 ms_                                                                        |
| Waiting until an element matching [`waitFor.selector`](https://docs.formfeed.dev/templates/settings#waiting-for-content) is visible                                     | `waitFor.timeoutMs`, 5 seconds by default, at most 30 | _Render exceeded 30000 ms_ (the message names 30 seconds in any case)                             |
| Taking the picture of an image output, or of the element of `image.clipSelector`                                                                 | 30 seconds                                            | _Render exceeded 30000 ms_                                                                        |
| Printing the PDF                                                                                                                                 | 120 seconds                                           | _Printing the PDF exceeded 120000 ms_                                                             |
| Filling a Word or PowerPoint template                                                                                                            | 10 seconds                                            | _Template render took … ms; the limit is 10000 ms_                                                |
| Converting a Word or PowerPoint template, or a document sent to `/pdf/convert`, to PDF                                                           | 60 seconds for the whole render or conversion         | _The conversion did not finish in time_                                                           |
| Making an [e-invoice](https://docs.formfeed.dev/templates/e-invoices), or validating a [PDF/UA-1 file](https://docs.formfeed.dev/templates/accessible-pdf)                                     | 20 seconds                                            | _The e-invoice conversion did not finish in time_, _The PDF/UA validation did not finish in time_ |

A document of several hundred pages prints in tens of seconds, well inside its limit. A `waitFor.delayMs` pause has no limit of its own and is capped at 10 seconds. A Liquid template that runs longer than 5 seconds answers [`template_runtime_error`](https://docs.formfeed.dev/errors/template-runtime) instead, and a `url` render whose page does not load in 30 seconds answers [`navigation_failed`](https://docs.formfeed.dev/errors/navigation-failed).

Handlebars partials nested deeper than 8 levels also answer this code (_Partials nest deeper than 8_), although no time ran out. Jinja2 and Liquid have no such depth limit.

The render is recorded as failed and costs nothing. [How long a render may take](https://docs.formfeed.dev/api/rendering#time-limits) lists the same limits from the request's side.

**Causes.**

- A template that loops over very large data, or nests loops deeply.
- Images, fonts or scripts from a slow server: the document counts as loaded only when none of its requests is running.
- A script that keeps the network busy, such as polling or an analytics beacon.
- A `waitFor.selector` that never becomes visible, because the script that adds the element failed, the selector does not match, or the element stays hidden.
- A Word or PowerPoint file with many large pictures or pages, which LibreOffice converts slowly.

**Fix.**

- Reduce the work in the template: compute totals and groupings in your application and send the results.
- Host images and fonts on a fast CDN, or in the [file library](https://docs.formfeed.dev/api/files), and check that `waitFor.selector` matches an element your script really adds.
- Set `waitFor.networkIdle` to `false` for documents that keep a connection open.
- For office documents, compress the pictures, or split the document.

**Retry.**

Only if the cause was passing, such as a slow server for images. A template that is too slow fails the same way again. An answer of `504` is not stored under the `Idempotency-Key`, so a retry with the same key really runs. Async renders are retried by the queue twice, and a render that times out every time ends as [`worker_exhausted`](https://docs.formfeed.dev/errors/worker-exhausted); a timeout of a Word or PowerPoint conversion is not retried.

## 409 schema_breaking_change

Publishing a version, moving a release channel, promoting its canary or rolling it back would put a version live whose stored data schema refuses data the current version accepts. The request changed nothing.

**Causes.**

- `POST /templates/{id}/versions/{number}/publish` for a version whose schema adds a required field, makes a field required, narrows a type or removes an enum value.
- `POST /templates/{id}/versions` with `"publish": true` and such a schema.
- `PUT /templates/{id}/channels/{name}` with a main or canary version like that, and `promote` or `rollback` to one.
- `formfeed templates push --publish` or `--channel`, and `formfeed channels set|promote|rollback`; the CLI prints the changes and exits with code 1.

The check only refuses when **both** versions store a data schema. With an inferred schema on either side, the changes are reported in `schema_check` as a warning. [What counts as breaking](https://docs.formfeed.dev/templates/release-channels#breaking-changes) lists every rule.

**Fix.**

- If callers would really break: change the schema so the new field is optional, or keep the old type or enum value, and create a new version.
- If every caller already sends what the new schema requires: repeat the request with `"allow_breaking": true` (`--allow-breaking` in the CLI, **Publish anyway** in the editor, `allowBreaking` / `allow_breaking` in the SDKs).
- Afterwards regenerate your types with `formfeed types`, so your code sees the new schema. See [Typed template data](https://docs.formfeed.dev/templates/typed-data).

**Retry.**

No. Retrying the same request gives the same answer; change the schema or send `allow_breaking` first. If you send an `Idempotency-Key` with the `POST`, use a new one: a `4xx` answer is stored under its key for 24 hours.

## 422 source_encrypted

A source is password-protected. An encrypted PDF cannot be merged, watermarked or protected again, and an encrypted Word, Excel or PowerPoint file cannot be converted: Formfeed never asks for document passwords. `detail` names the source. Nothing was charged.

**Causes.**

- Calling `/pdf/merge`, `/pdf/watermark` or `/pdf/protect` with the output of `/pdf/protect`, or with a render whose `post` set a password.
- A password-protected PDF from the file library in `sources` or `post.merge_after`.
- A password-protected Word, Excel or PowerPoint file sent to `/pdf/convert` (_The document is password-protected_), or uploaded as the file of a template.

**Fix.**

- Protect last: merge and watermark first, then call `/pdf/protect` on the result. A render's `post` block already applies its steps in that order (merge, watermark, password).
- For a library PDF, upload a copy without the password.
- For an office document, remove the password in its application (in Word: File › Info › Protect Document › Encrypt with Password, then clear the field), convert, and protect the PDF with `/pdf/protect` if it needs a password.

**Retry.**

No. The same request gets the same answer. Change the source, then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 422 source_invalid

A source exists, but it cannot be processed: the render has not succeeded, the file is not of the type the step needs, or it cannot be read. Merge, protect, watermark and `post.merge_after` need PDFs; an image watermark needs a PNG or JPEG; `/pdf/convert` with a render id needs a Word or PowerPoint output. `field`, `index` and `render_id` say which source, where the gateway found the problem. Nothing was charged.

**Causes.**

- The source render failed, or is still `queued` or `rendering`.
- The source render produced an image, not a PDF (_Render … produced png, not a PDF_), or a library file is not a PDF.
- A watermark `image` that is not a PNG or JPEG, or cannot be read.
- A PDF that cannot be read, for example a damaged file in the library (_terms.pdf is not a readable PDF_).
- `/pdf/convert` with a render whose output is not a Word or PowerPoint document; PDFs and images need no conversion.
- An office document that LibreOffice refuses to read (_The document cannot be converted_, with the converter's `reason`).

**Fix.**

- Wait until the source render has `status: "succeeded"` (poll it, or wait for its `render.completed` webhook), then call the tool.
- Use PDF outputs and PDF library files for merge, protect and watermark, and PNG or JPEG files for an image watermark.
- Open a PDF that cannot be read in a viewer and save or export it again, then upload it again.

**Retry.**

Only once the source has changed, for example when the source render has finished. Then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 422 source_not_found

A source the request names cannot be found. Sources are what the [PDF tools](https://docs.formfeed.dev/api/pdf-tools#sources) and a render's [`post`](https://docs.formfeed.dev/api/rendering#post-processing) read: earlier renders of the workspace, named by their `rnd_…` id, and files of the workspace's [file library](https://docs.formfeed.dev/api/files), named by their file name (`terms.pdf`). `field` names the request field, `index` the position in a list, and `render_id` the render, where the source is one. Nothing was rendered or charged.

**Causes.**

- The render id belongs to another workspace, or has a typo (_Render … does not exist in this workspace_).
- The source render's output was deleted or has expired (_The output of render … was deleted or has expired_). Outputs are kept as long as the plan's retention allows, at most 1 day on Free and on every test key, and `expires_in` or the workspace's own retention can make it shorter.
- A library file name that the workspace does not have (_No file terms.pdf in this workspace's library_), in `sources`, `post.merge_after` or a watermark's `image` (`field`: `watermark.image` or `post.watermark.image`).
- A library file named by its `fil_…` id. The id is not accepted as a source yet and answers this code; name the file by its name instead.
- An object that disappeared between the check and the render (_… is not in storage any more_).

**Fix.**

- Use render ids from the same workspace as the key, and process outputs before they expire. `expires_at` of the source render says until when.
- Name library files by their name, exactly as `GET /v1/files` lists it.
- The problem names the `field` and its `index`, so you know which source to fix.

**Retry.**

No. The same request gets the same answer. Fix the source, then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 409 storage_not_configured

The request needs the workspace's own bucket ([bring your own storage](https://docs.formfeed.dev/api/bring-your-own-storage)), but the workspace has no storage connection. On a render, `field` is `storage.key`. Nothing was rendered or charged.

**Causes.**

- A render, a batch item or a PDF tool request with `storage.key` in a workspace without a connection. For a batch, `detail` starts with the first such item (`Item 3: …`), and nothing is queued.
- A request built for one workspace, sent with the key of another, for example a test workspace that never got a connection.
- The connection was removed with **Disconnect** on the **Your own storage** card.
- Testing the connection or redelivering copies in the app in a workspace without a connection.

**Fix.**

- Connect a bucket: an owner or admin does that in the app under **Settings**, on the **Your own storage** card (Pro plan and above). See [Bring your own storage](https://docs.formfeed.dev/api/bring-your-own-storage#connect).
- Or leave `storage.key` out. Without a connection nothing is copied, and the render works as usual; `storage: false` is accepted with or without one.

**Retry.**

Only once the workspace has a connection, or without `storage.key`. Send that request with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 422 storage_test_failed

Before a storage connection is saved, and whenever it is tested again, Formfeed writes a small test object, `<prefix>.formfeed/connection-test.txt`, to the bucket. That write failed. `error` carries the provider's answer, reduced to its `code`, its `http_status` and the first line of its `message`; `key` is the object Formfeed tried to write.

When a connection was being saved, nothing was stored: a new connection was not created, and an existing one stays as it was. When a stored connection was tested again, the problem also carries the `connection`; a paused connection stays paused.

**Causes.**

| `error.code`                                        | Usually means                                                                                                                                                                                                                                                                                            |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AccessDenied`                                      | The key may not write to this bucket and prefix. On AWS, the policy must allow `s3:PutObject` on `arn:aws:s3:::<bucket>/<prefix>*`, plus `kms:GenerateDataKey` on the KMS key if the bucket encrypts with one. On Google Cloud Storage, a key that may not replace objects fails from the second test on |
| `InvalidAccessKeyId`                                | The access key ID is mistyped, deleted, or belongs to another account or provider                                                                                                                                                                                                                        |
| `SignatureDoesNotMatch`                             | The secret does not belong to the access key ID, or an S3-compatible service expects another signing region                                                                                                                                                                                              |
| `NoSuchBucket`                                      | The bucket name is wrong, or the bucket is in another account, or in the other R2 jurisdiction                                                                                                                                                                                                           |
| `PermanentRedirect`, `AuthorizationHeaderMalformed` | The region does not match the bucket's region                                                                                                                                                                                                                                                            |
| `EndpointBlocked`                                   | The endpoint is not `https`, carries credentials, or resolves to a private or internal address                                                                                                                                                                                                           |
| `TimeoutError`, `NetworkError`                      | The endpoint did not answer or does not exist; check the address                                                                                                                                                                                                                                         |
| `CredentialsUnavailable`                            | The stored credentials are gone, for example because the workspace was deleted                                                                                                                                                                                                                           |

**Fix.**

- Compare the key's permissions with the [minimal permissions](https://docs.formfeed.dev/api/bring-your-own-storage#permissions) for your provider.
- Check the bucket name, the region or R2 jurisdiction, and for S3-compatible services the endpoint and path-style addressing. See [Connect a bucket](https://docs.formfeed.dev/api/bring-your-own-storage#connect).
- Save again with the corrected values. A paused connection resumes as soon as a test passes.

**Retry.**

After fixing the key or the bucket. Repeating the same request gives the same answer, unless the provider was only briefly unreachable.

## 404 template_not_found

The workspace of the key has no template with this slug or id, or the template does not have the version the request names. A render names the version with `version` ([which version renders](https://docs.formfeed.dev/api/rendering#version)): `published` (the default), a channel name, `latest` or a version number. The Templates API answers this code too, for a template or version that does not exist.

**Causes.**

- The template exists but no version of it was ever published, and the request renders `published`. Saved versions can be rendered by their number, and the newest one as `latest` with a test key; unsaved edits in the editor are in no version.
- The key belongs to a different workspace than the template, for example a key from `staging` against a template in `production`.
- A typo in the slug, or an id of another template. A template is named by its slug, its `tpl_…` id or its UUID.
- The template was archived. Archived templates do not render; restore it in the app.
- A version number the template does not have.
- A Word or PowerPoint template version without its file (_Version 3 of template "contract" has no file_), for example after a failed upload.

A template that exists but has no channel of the name in `version` answers [`channel_not_found`](https://docs.formfeed.dev/errors/channel-not-found) instead.

**Fix.**

- Open the template in the app and check that a version is published, or publish one (`formfeed templates push --publish`, or [`POST /v1/templates/{id}/versions/{number}/publish`](https://docs.formfeed.dev/api/reference/publish-template-version)).
- Run `formfeed whoami` or call `GET /v1/account` to see which workspace the key belongs to.
- List what the key can see with `GET /v1/templates`.

**Retry.**

No. The same request gets the same answer. Fix what the problem names, then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 422 template_runtime_error

The template compiled, but failed while it ran with the data of this request. Nothing was printed; the render is recorded as failed and costs nothing.

`location` gives `line` and `column` where the template language knows them, in the body, the header or the footer (the problem does not say which), and is `null` where it does not. For a Word or PowerPoint template it names the `part` of the file.

**Causes.**

- An unknown helper or filter. Liquid runs with strict filters, so a typo stops the render instead of printing nothing.
- A calculation that came out as `NaN` reached `money`, `number` or `numToWords`: arithmetic on a value that is text (`"19 %"`) or missing. The helpers stop the render rather than print `NaN`; text that is not a number, such as `"on request"`, they print as it is.
- A method called on a missing value, for example `customer.name.upper()` when `customer` is not in the data.
- An include of a partial that was never pushed with the template, for example one added to `partials/` after the last `formfeed templates push` (see [Partials](https://docs.formfeed.dev/templates/writing-templates#partials)), or of a [shared partial](https://docs.formfeed.dev/templates/shared-partials) that was deleted, renamed or written for another template language.
- A partial that includes itself (in Handlebars: _Maximum call stack size exceeded_).
- A Liquid template that runs longer than the 5 seconds a template may take. Jinja2 and Handlebars report that as [`render_timeout`](https://docs.formfeed.dev/errors/render-timeout).

Rarely, the render service reports a failure of its own under this code, for example a browser that crashed. Then `detail` does not refer to your template at all. Retry once, and if it repeats, send the `request_id` to support@formfeed.dev.

**Fix.**

- Reproduce it with the same data: paste the request `data` as a data set in the editor, or save it as `data/failing.json` in the template folder and run `formfeed validate <slug> --data failing`. If the workspace keeps render requests, `formfeed renders pull <id>` fetches the data of the failed render; see [Reproduce a failed render](https://docs.formfeed.dev/templates/reproduce-a-failed-render).
- Guard optional values with `default` or an `if`.
- Check the name in the [helper reference](https://docs.formfeed.dev/templates/helpers).

**Retry.**

No, unless `detail` is not about your template. Fix the template or the data first, then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 422 template_syntax_error

The template could not be compiled, so nothing was rendered: a tag is not closed or misspelt, a bracket is missing, or the syntax belongs to another template language. The render is recorded as failed and costs nothing.

`location` says where: `line` and `column` in the part that failed. That part can be the template body, the header or the footer; the problem does not say which, so look in all three. For a Word or PowerPoint template, `location` names the `part` of the file instead, such as `word/document.xml`.

**Causes.**

- An unclosed or misspelt block: `{% for %}` without `{% endfor %}`, `{% endfo %}`, `{{#each}}` without `{{/each}}`.
- Syntax from another language, such as Liquid filters `| money: "EUR"` in a Jinja2 template (`| money("EUR")`).
- Stray `{{` or `{%` in text; escape them or wrap them in `{% raw %}…{% endraw %}` (Jinja2, Liquid).
- A `settings.pdf.metadata.title`, header or footer with template syntax that does not compile: they are compiled like the body.

**Fix.**

- Go to `location.line` and `location.column`. The editor's **Problems** list shows the same message and names the tab (template, header or footer); `formfeed validate` reports it in a project before you push.
- Compare with the [syntax table](https://docs.formfeed.dev/templates/languages) for the template language.

**Retry.**

No. The same request gets the same answer. Fix the template (and publish the fixed version, for a stored template), then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 403 test_output_unavailable

A test render asked a [Word or PowerPoint template](https://docs.formfeed.dev/templates/office-templates) for its own format (`docx` or `pptx`) on a plan whose test renders carry the test mark, which is the Free plan. Those test renders produce the marked PDF only: a mark inside an editable document could simply be deleted. `output` is what was asked for, `plan` the plan, `allowed` what a test render may ask for. Nothing was rendered or charged.

**Causes.**

- A test key (`ff_test_…`) on the Free plan rendering an office template without `output`, which defaults to the template's own format.
- An integration that always asks for `docx`.

**Fix.**

- Request `"output": "pdf"` to check the template with a test key. The PDF shows the same content, with the test mark.
- Upgrade to Starter or above to render Word and PowerPoint documents with a live key. There, test renders may ask for `docx` and `pptx` too.

**Retry.**

No. Change `output` or the plan, then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 500 tool_failed

A PDF step failed inside Formfeed, on a source that could be read: the password protection, another PDF tool, or the PDF/UA-1 validation of a render with [`pdf.ua`](https://docs.formfeed.dev/templates/accessible-pdf). This is Formfeed's failure, not your request's. Nothing was charged.

A source that cannot be read at all answers [`source_invalid`](https://docs.formfeed.dev/errors/source-invalid), and an encrypted one [`source_encrypted`](https://docs.formfeed.dev/errors/source-encrypted).

**Causes.**

- The program that encrypts PDFs (`/pdf/protect`, `post.password`) failed on the document.
- An unexpected failure while a PDF tool processed the document, for example a PDF structure the tools can open but not copy.
- The validator could not read the file of a render with `pdf.ua` (_The PDF could not be validated against PDF/UA-1_, with the validator's `reason`).
- A temporary failure while processing.

**Fix.**

- Retry once with the same `Idempotency-Key`. If it fails again, render the source again and use the new render id.
- If it keeps happening, send the `request_id` to support@formfeed.dev.

**Retry.**

Once. A temporary failure goes away on a retry with the same `Idempotency-Key` (answers of `500` are not stored under the key); a document that triggers it fails again. Async renders with a `post` step are retried by the queue, and end as [`worker_exhausted`](https://docs.formfeed.dev/errors/worker-exhausted) when every attempt fails.

## 401 unauthorized

The request carries no API key, or the key it carries is not one Formfeed accepts. `detail` says which: no key at all, a key in the wrong format, or `API key rejected:` followed by the reason (`unknown key`, `key revoked`, `key expired` or `organisation deleted`). An answer without any key also carries the header `WWW-Authenticate: Bearer realm="formfeed"`.

**Causes.**

- No `Authorization: Bearer …` or `X-API-Key` header, or a typo in the header name (`detail`: _Provide your API key as `Authorization: Bearer ff_live_…`_).
- Something other than a key in the header, such as quotes or a line break copied with it, a key cut short, or a placeholder left in (`detail`: _The API key format is not recognised_). A key is `ff_live_` or `ff_test_` followed by 20 to 64 letters and digits.
- The key was revoked, or it had an expiry date that has passed. A revoked key can still work for up to a minute, because key lookups are cached for 60 seconds.
- A key from another Formfeed environment, for example a staging key sent to production: there it is an `unknown key`.
- The organisation of the key was deleted.

**Fix.**

- Check the header: `Authorization: Bearer ff_live_…` (or `ff_test_…`), or `X-API-Key: ff_live_…`.
- Look the key up under **API keys** in the app: it shows whether the key is active and when it was last used.
- Create a new key if the old one was revoked or expired. See [Authentication](https://docs.formfeed.dev/api/authentication#errors).

**Retry.**

No. Retrying the same request gives the same answer; send a valid key first.

## 422 unsupported_in_office

A [Word or PowerPoint template](https://docs.formfeed.dev/templates/office-templates) uses something office templates do not support: an include of a partial, or a helper that returns HTML. Either would put HTML into the document. `location` names the `part` of the file (`word/document.xml`, a header, a slide) and, where known, the `paragraph`, counted from 1. The render is recorded as failed and costs nothing.

**Causes.**

- `{% include "footer" %}`, `{% render %}`, `{% extends %}`, `{% import %}`, `{% from %}` or a Handlebars partial `{{> footer }}` typed into the document.
- `chart` or `markdown`, which return HTML (_chart() is not supported in office templates; it returns HTML._).
- A tag copied from an HTML template.

**Fix.**

- Put the content into the document itself: Word's headers, footers and building blocks do what partials do in HTML templates.
- Brand kit values such as `{{ brand.legal_footer }}` work in office templates.
- The office template page of the app lists every tag of the file with its problems before anything is rendered.

**Retry.**

No. Change the template and publish the new version first, then send the request again with a new `Idempotency-Key` or without one: a `4xx` answer is stored under its key for 24 hours, so the old key returns this problem again. See [retrying](https://docs.formfeed.dev/api/errors#retries).

## 500 worker_exhausted

Something failed on Formfeed's side, and it is not a problem of your request. It comes in two ways:

- **An async render ran out of attempts.** The queue retries a render that fails on Formfeed's side (a timeout, a busy service, an unexpected failure) two more times, 5 and then 10 seconds later. When the third attempt fails too, the render fails with this code, and `detail` (`message` in the render's `error`) keeps what the last attempt said. The render's `error` also has `attempts`.
- **The API met an unexpected error** before a render existed, for example when the database did not answer or the brand kit could not be read. `detail` then reads _Unexpected error_.

A failed render costs nothing.

**Causes.**

- A render that times out every time, for example on a slow image server: the last message is _Render exceeded 30000 ms_. That is a problem of the template or its assets more than of Formfeed; see [`render_timeout`](https://docs.formfeed.dev/errors/render-timeout).
- The office converter or the conformance service was unavailable for all three attempts.
- An infrastructure problem: a crashed browser, a storage error.
- An unexpected error in the API.

**Fix.**

- Read `detail`: when it names a timeout, fix it as [`render_timeout`](https://docs.formfeed.dev/errors/render-timeout) explains; a retry will not help.
- Otherwise retry the request later.
- If it keeps happening, check [status.formfeed.dev](https://status.formfeed.dev) and send the `request_id` (and the `render_id`, if there is one) to support@formfeed.dev.

**Retry.**

Yes, with growing pauses (for example after 5, 30 and 120 seconds). For an answer of `500` to your request, use the same `Idempotency-Key`: such answers are not stored under the key, so each retry really runs. For an async render that failed later, send the request with a new `Idempotency-Key`: the first answer to the old key, the `202` with the failed render, is stored for 24 hours and would come back. This response has no `Retry-After` header, and the SDKs do not retry it by themselves.
