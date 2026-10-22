# Helpers

Generated from the helpers of this release. Every helper exists in all three languages with the
same name; each example below is rendered by the engine in a test, so it runs as written.
Dates, numbers and money follow the template's `locale`, `timezone` and `currency` unless an
argument says otherwise.

## Formatting

### `money`

Also called `currency`.

`money(value, currency?, locale?)`

Formats a number as currency with Intl.NumberFormat. Defaults come from the template settings. Text that is not a number is printed as it is; a calculation that came out as NaN stops the render.

- Jinja2: `{{ line.total \| money('EUR') }} → 1.234,50 €`
- Liquid: `{{ line.total \| money: 'EUR' }} → 1.234,50 €`
- Handlebars: `{{money line.total 'EUR'}} → 1.234,50 €`

### `number`

`number(value, decimals?, locale?)`

Formats a number with grouping. With decimals, exactly that many (rounded half away from zero); without, the decimals the number has, at most three, so 1234.5 → 1.234,5 and 2.34567 → 2,346. Use number(0) for whole numbers.

- Jinja2: `{{ qty \| number(2) }} → 1.234,00`
- Liquid: `{{ qty \| number: 2 }} → 1.234,00`
- Handlebars: `{{number qty 2}} → 1.234,00`

### `date`

`date(value, format?, tz?, locale?)`

Formats a date with date-fns tokens (e.g. 'dd.MM.yyyy', 'PPP'). Accepts ISO strings, epoch numbers and 'now'.

- Jinja2: `{{ invoice.date \| date('dd.MM.yyyy') }}`
- Liquid: `{{ invoice.date \| date: 'dd.MM.yyyy' }}`
- Handlebars: `{{date invoice.date 'dd.MM.yyyy'}}`

### `dateAdd`

Also called `date_add`.

`dateAdd(value, amount, unit)`

Adds days, weeks, months or years to a date; returns an ISO string for further formatting.

- Jinja2: `{{ invoice.date \| dateAdd(14, 'days') \| date('PP') }}`
- Liquid: `{{ invoice.date \| dateAdd: 14, 'days' \| date: 'PP' }}`
- Handlebars: `{{date (dateAdd invoice.date 14 'days') 'PP'}}`

### `add`

Also called `plus`.

`add(a, b)`

a + b (Handlebars has no arithmetic of its own).

- Jinja2: `{{ line.qty \| add(1) }}`
- Liquid: `{{ line.qty \| add: 1 }}`
- Handlebars: `{{#each invoice.lines}}{{add @index 1}}{{/each}}`

### `subtract`

Also called `minus`, `sub`.

`subtract(a, b)`

a minus b.

- Jinja2: `{{ total \| subtract(discount) }}`
- Liquid: `{{ total \| subtract: discount }}`
- Handlebars: `{{subtract total discount}}`

### `multiply`

Also called `times`, `mul`.

`multiply(a, b)`

a times b.

- Jinja2: `{{ line.qty \| multiply(line.price) \| money }}`
- Liquid: `{{ line.qty \| multiply: line.price \| money }}`
- Handlebars: `{{money (multiply line.qty line.price)}}`

### `divide`

Also called `divided_by`, `div`.

`divide(a, b)`

a divided by b; empty when b is 0.

- Jinja2: `{{ total \| divide(count) }}`
- Liquid: `{{ total \| divide: count }}`
- Handlebars: `{{divide total count}}`

### `round`

`round(value, decimals = 0, method = 'common')`

Rounds to the given decimals, as Jinja2's round does: 'common' rounds half away from zero, 'floor' always down and 'ceil' always up.

- Jinja2: `{{ 2.345 \| round(2) }} gives 2.35, {{ 2.349 \| round(2, 'floor') }} gives 2.34`
- Liquid: `{{ 2.345 \| round: 2 }} gives 2.35, {{ 2.349 \| round: 2, 'floor' }} gives 2.34`
- Handlebars: `{{round 2.345 2}} gives 2.35, {{round 2.349 2 'floor'}} gives 2.34`

### `numToWords`

Also called `num_to_words`, `spellout`.

`numToWords(value, locale?)`

Spells a number out in words (de, en, fr, es, it, nl).

- Jinja2: `{{ total \| numToWords('de') }} → zweihundertdreiundvierzig`
- Liquid: `{{ total \| numToWords: 'de' }} → zweihundertdreiundvierzig`
- Handlebars: `{{numToWords total 'de'}} → zweihundertdreiundvierzig`

## Text

### `upper`

Also called `upcase`.

`upper(value)`

Upper-cases a string.

- Jinja2: `{{ name \| upper }}`
- Liquid: `{{ name \| upper }}`
- Handlebars: `{{upper name}}`

### `lower`

Also called `downcase`.

`lower(value)`

Lower-cases a string.

- Jinja2: `{{ name \| lower }}`
- Liquid: `{{ name \| lower }}`
- Handlebars: `{{lower name}}`

### `title`

Also called `capitalize_words`.

`title(value)`

Capitalises every word.

- Jinja2: `{{ 'hello world' \| title }} → Hello World`
- Liquid: `{{ 'hello world' \| title }} → Hello World`
- Handlebars: `{{title 'hello world'}} → Hello World`

### `truncate`

`truncate(value, length = 255, suffix = '…')`

Cuts a string to `length` characters and appends the suffix when cut.

- Jinja2: `{{ description \| truncate(40) }}`
- Liquid: `{{ description \| truncate: 40 }}`
- Handlebars: `{{truncate description 40}}`

### `replace`

`replace(value, search, replacement)`

Replaces every occurrence of `search`.

- Jinja2: `{{ iban \| replace(' ', '') }}`
- Liquid: `{{ iban \| replace: ' ', '' }}`
- Handlebars: `{{replace iban ' ' ''}}`

### `split`

`split(value, separator = ',')`

Splits a string into a list.

- Jinja2: `{% for tag in keywords \| split(',') %}{{ tag }}{% endfor %}`
- Liquid: `{% assign tags = keywords \| split: ',' %}{% for tag in tags %}{{ tag }}{% endfor %}`
- Handlebars: `{{#each (split keywords ',')}}{{this}}{{/each}}`

### `default`

Also called `d`.

`default(value, fallback)`

Returns the fallback when the value is empty, null or undefined.

- Jinja2: `{{ customer.vat \| default('—') }}`
- Liquid: `{{ customer.vat \| default: '—' }}`
- Handlebars: `{{default customer.vat '—'}}`

### `nl2br`

`nl2br(value)`

Escapes the text and turns line breaks into <br>.

- Jinja2: `{{ address \| nl2br }}`
- Liquid: `{{ address \| nl2br }}`
- Handlebars: `{{nl2br address}}`

### `markdown`

`markdown(value)`

Renders Markdown to HTML (no sanitiser: template data is trusted by the author).

- Jinja2: `{{ notes \| markdown }}`
- Liquid: `{{ notes \| markdown }}`
- Handlebars: `{{markdown notes}}`

### `safe`

Also called `raw`.

`safe(value)`

Marks a string as HTML so the engine does not escape it.

- Jinja2: `{{ html_snippet \| safe }}`
- Liquid: `{{ html_snippet \| safe }}`
- Handlebars: `{{safe html_snippet}}`

### `t`

Also called `translate`.

`t(key, params?)`

Looks a key up in the template's i18n dictionary for the request locale; `{name}` placeholders come from params.

- Jinja2: `{{ t('invoice.title', { number: invoice.number }) }}`
- Liquid: `{{ 'invoice.title' \| t: number: invoice.number }}`
- Handlebars: `{{t 'invoice.title' number=invoice.number}}`

## Collections

### `join`

`join(list, separator = ', ')`

Joins a list into a string.

- Jinja2: `{{ tags \| join(' · ') }}`
- Liquid: `{{ tags \| join: ' · ' }}`
- Handlebars: `{{join tags ' · '}}`

### `sum`

`sum(list, path?)`

Sum of the items, or of `path` inside each item.

- Jinja2: `{{ invoice.lines \| sum('total') \| money }}`
- Liquid: `{{ invoice.lines \| sum: 'total' \| money }}`
- Handlebars: `{{money (sum invoice.lines 'total')}}`

### `avg`

`avg(list, path?)`

Average of the items or of `path` inside each item.

- Jinja2: `{{ scores \| avg \| number(1) }}`
- Liquid: `{{ scores \| avg \| number: 1 }}`
- Handlebars: `{{number (avg scores) 1}}`

### `min`

`min(list, path?)`

Smallest number.

- Jinja2: `{{ prices \| min }}`
- Liquid: `{{ prices \| min }}`
- Handlebars: `{{min prices}}`

### `max`

`max(list, path?)`

Largest number.

- Jinja2: `{{ prices \| max }}`
- Liquid: `{{ prices \| max }}`
- Handlebars: `{{max prices}}`

### `groupBy`

Also called `group_by`, `groupby`.

`groupBy(list, path)`

Groups items by a field; returns a list of `{ key, items }`.

- Jinja2: `{% for group in lines \| groupBy('category') %}{{ group.key }}{% endfor %}`
- Liquid: `{% assign groups = lines \| groupBy: 'category' %}{% for group in groups %}{{ group.key }}{% endfor %}`
- Handlebars: `{{#each (groupBy lines 'category')}}{{key}}{{/each}}`

### `sortBy`

Also called `sort_by`, `sort`.

`sortBy(list, path?, direction = 'asc')`

Sorts items by a field (numbers numerically, strings by locale).

- Jinja2: `{% for l in lines \| sortBy('position') %}{{ l.sku }}{% endfor %}`
- Liquid: `{% assign sorted = lines \| sortBy: 'position' %}{% for l in sorted %}{{ l.sku }}{% endfor %}`
- Handlebars: `{{#each (sortBy lines 'position')}}{{sku}}{{/each}}`

### `where`

Also called `filter_by`.

`where(list, path, value?)`

Keeps items whose field equals `value` (or is truthy when omitted).

- Jinja2: `{% for l in lines \| where('taxable', true) %}{{ l.sku }}{% endfor %}`
- Liquid: `{% assign taxable = lines \| where: 'taxable', true %}{% for l in taxable %}{{ l.sku }}{% endfor %}`
- Handlebars: `{{#each (where lines 'taxable' true)}}{{sku}}{{/each}}`

### `pluck`

Also called `map`.

`pluck(list, path)`

Extracts one field from every item.

- Jinja2: `{{ lines \| pluck('sku') \| join(', ') }}`
- Liquid: `{{ lines \| pluck: 'sku' \| join: ', ' }}`
- Handlebars: `{{join (pluck lines 'sku') ', '}}`

### `chunk`

Also called `batch`.

`chunk(list, size)`

Splits a list into lists of `size` items (rows of labels, columns).

- Jinja2: `{% for row in labels \| chunk(3) %}{{ row \| join(', ') }}{% endfor %}`
- Liquid: `{% assign rows = labels \| chunk: 3 %}{% for row in rows %}{{ row \| join: ', ' }}{% endfor %}`
- Handlebars: `{{#each (chunk labels 3)}}{{join this ', '}}{{/each}}`

### `range`

`range(start, end?, step = 1)`

List of numbers from start to end (exclusive).

- Jinja2: `{% for i in range(1, 4) %}{{ i }}{% endfor %}`
- Liquid: `{% assign nums = 1 \| range: 4 %}{% for i in nums %}{{ i }}{% endfor %}`
- Handlebars: `{{#each (range 1 4)}}{{this}}{{/each}}`

### `first`

`first(list)`

First item.

- Jinja2: `{{ tags \| first }}`
- Liquid: `{{ tags \| first }}`
- Handlebars: `{{first tags}}`

### `last`

`last(list)`

Last item.

- Jinja2: `{{ tags \| last }}`
- Liquid: `{{ tags \| last }}`
- Handlebars: `{{last tags}}`

### `length`

Also called `size`, `count`.

`length(value)`

Number of items in a list, keys in an object or characters in a string.

- Jinja2: `{{ lines \| length }}`
- Liquid: `{{ lines \| length }}`
- Handlebars: `{{length lines}}`

## Logic

### `eq`

`eq(a, b)`

True when both values are equal; `1` equals `"1"`.

- Jinja2: `{% if eq(invoice.total, 1000) %}…{% endif %}`
- Liquid: `{% assign hit = invoice.total \| eq: 1000 %}{% if hit %}…{% endif %}`
- Handlebars: `{{#if (eq invoice.total 1000)}}…{{/if}}`

### `ne`

`ne(a, b)`

True when the values differ.

- Jinja2: `{% if ne(invoice.total, 1000) %}…{% endif %}`
- Liquid: `{% assign hit = invoice.total \| ne: 1000 %}{% if hit %}…{% endif %}`
- Handlebars: `{{#if (ne invoice.total 1000)}}…{{/if}}`

### `gt`

`gt(a, b)`

Greater than; numbers as numbers, anything else as text.

- Jinja2: `{% if gt(invoice.total, 1000) %}…{% endif %}`
- Liquid: `{% assign hit = invoice.total \| gt: 1000 %}{% if hit %}…{% endif %}`
- Handlebars: `{{#if (gt invoice.total 1000)}}…{{/if}}`

### `gte`

`gte(a, b)`

Greater than or equal.

- Jinja2: `{% if gte(invoice.total, 1000) %}…{% endif %}`
- Liquid: `{% assign hit = invoice.total \| gte: 1000 %}{% if hit %}…{% endif %}`
- Handlebars: `{{#if (gte invoice.total 1000)}}…{{/if}}`

### `lt`

`lt(a, b)`

Less than.

- Jinja2: `{% if lt(invoice.total, 1000) %}…{% endif %}`
- Liquid: `{% assign hit = invoice.total \| lt: 1000 %}{% if hit %}…{% endif %}`
- Handlebars: `{{#if (lt invoice.total 1000)}}…{{/if}}`

### `lte`

`lte(a, b)`

Less than or equal.

- Jinja2: `{% if lte(invoice.total, 1000) %}…{% endif %}`
- Liquid: `{% assign hit = invoice.total \| lte: 1000 %}{% if hit %}…{% endif %}`
- Handlebars: `{{#if (lte invoice.total 1000)}}…{{/if}}`

### `and`

`and(a, b, …)`

True when every value is true; an empty list counts as false.

- Jinja2: `{% if and(customer.vatId, eq(country, 'DE')) %}…{% endif %}`
- Liquid: `{% assign german = country \| eq: 'DE' %}{% assign both = customer.vatId \| and: german %}{% if both %}…{% endif %}`
- Handlebars: `{{#if (and customer.vatId (eq country 'DE'))}}…{{/if}}`

### `or`

`or(a, b, …)`

True when any value is true.

- Jinja2: `{% if or(invoice.paid, invoice.credited) %}…{% endif %}`
- Liquid: `{% assign any = invoice.paid \| or: invoice.credited %}{% if any %}…{% endif %}`
- Handlebars: `{{#if (or invoice.paid invoice.credited)}}…{{/if}}`

### `not`

`not(value)`

The opposite truth value.

- Jinja2: `{% if invoice.paid \| not %}…{% endif %}`
- Liquid: `{% assign unpaid = invoice.paid \| not %}{% if unpaid %}…{% endif %}`
- Handlebars: `{{#if (not invoice.paid)}}…{{/if}}`

## Codes

### `qrcode`

Also called `qr`.

`qrcode(value, { size = 160, ecc = 'M', margin = 1, color, background })`

QR code as an SVG data URI for an <img> src. Give the image an `alt` that says where the code leads: a screen reader cannot scan it.

- Jinja2: `<img src="{{ qrcode(order.url, { size: 120 }) }}" alt="QR code: {{ order.url }}">`
- Liquid: `<img src="{{ order.url \| qrcode: size: 120 }}" alt="QR code: {{ order.url }}">`
- Handlebars: `<img src="{{qrcode order.url size=120}}" alt="QR code: {{order.url}}">`

### `barcode`

`barcode(value, { type = 'code128', height = 12, width, text = true, scale = 2 })`

Barcode (code128, ean13, ean8, upc, itf14, code39, datamatrix, pdf417) as an SVG data URI. Give the image an `alt` with the value it encodes.

- Jinja2: `<img src="{{ barcode(item.sku, { type: 'code128' }) }}" alt="Barcode {{ item.sku }}">`
- Liquid: `<img src="{{ item.sku \| barcode: type: 'code128' }}" alt="Barcode {{ item.sku }}">`
- Handlebars: `<img src="{{barcode item.sku type='code128'}}" alt="Barcode {{item.sku}}">`

### `epcQr`

Also called `epc_qr`, `girocode`.

`epcQr({ name, iban, bic?, amount?, reference?, text? }, { size })`

SEPA credit transfer QR code (EPC069-12 / GiroCode) that banking apps scan to prefill a payment. Put an invoice number into `text` (the remittance line); `reference` is only for a structured creditor reference (RF…). Give the image an `alt`, and print the bank details as text as well: the code is a picture.

- Jinja2: `<img src="{{ epcQr({ name: company.name, iban: company.iban, amount: invoice.total, text: invoice.number }) }}" alt="QR code to pay invoice {{ invoice.number }}">`
- Liquid: `<img src="{{ company.iban \| epcQr: name: company.name, amount: invoice.total, text: invoice.number }}" alt="QR code to pay invoice {{ invoice.number }}">`
- Handlebars: `<img src="{{epcQr name=company.name iban=company.iban amount=invoice.total text=invoice.number}}" alt="QR code to pay invoice {{invoice.number}}">`

## Document

### `asset`

`asset(name)`

URL of a file in the workspace library (logo, background) on the CDN. Upload it on the Files page or with `POST /files` and use the name it has there; relative image paths resolve against the same library.

- Jinja2: `<img src="{{ asset('logo.png') }}">`
- Liquid: `<img src="{{ 'logo.png' \| asset }}">`
- Handlebars: `<img src="{{asset 'logo.png'}}">`

### `pageBreak`

Also called `page_break`.

`pageBreak()`

Starts a new page in PDF output.

- Jinja2: `{{ pageBreak() }}`
- Liquid: `{{ '' \| pageBreak }}`
- Handlebars: `{{pageBreak}}`

### `chart`

`chart({ type, labels, values, label, color } | { type, labels, series } | { type, data, options }, { width, height, alt })`

Draws a Chart.js chart (bar, line, pie, doughnut, radar, …) from plain data, `labels` with `values` or several `series` of `{ label, values, color }`, or from a full Chart.js `data` and `options`; animations are disabled so PDF and preview match. `alt` says in words what the chart shows: a screen reader reads it in place of the picture, and a tagged PDF needs it.

- Jinja2: `{{ chart({ type: 'line', labels: sales.labels, values: sales.quarters, width: 480, alt: 'Sales by quarter' }) }}`
- Liquid: `{{ '' \| chart: type: 'line', labels: sales.labels, values: sales.quarters, width: 480, alt: 'Sales by quarter' }}`
- Handlebars: `{{chart type='line' labels=sales.labels values=sales.quarters width=480 alt='Sales by quarter'}}`

### `image`

`image(url, { width, height, fit, alt })`

Places an image with fixed size and crop mode (`cover`, `contain`, `fill`); the render-worker fetches it through its cache and blocks private hosts. `alt` says what it shows; without one the image counts as decoration and a screen reader passes over it.

- Jinja2: `{{ image(product.photo, { width: 120, height: 80, fit: 'cover', alt: product.name }) }}`
- Liquid: `{{ product.photo \| image: width: 120, height: 80, fit: 'cover', alt: product.name }}`
- Handlebars: `{{image product.photo width=120 height=80 fit='cover' alt=product.name}}`

## Debugging

### `json`

Also called `tojson`, `dump`.

`json(value, indent = 2)`

Serialises a value as JSON; handy while building a template.

- Jinja2: `<pre>{{ invoice \| json }}</pre>`
- Liquid: `<pre>{{ invoice \| json }}</pre>`
- Handlebars: `<pre>{{json invoice}}</pre>`
