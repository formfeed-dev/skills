# Page setup, print and output

A PDF template is a web page that Chromium prints. Formfeed fills the template with the data, wraps
it in a document with a print reset, and prints it. What is specific to printing is below.

## Settings

`settings.json` in the template's folder. A render request can override any key; the three layers
(defaults, the template's settings, the request's) merge key by key.

```json settings
{
  "paper": { "format": "A4", "landscape": false },
  "margin": {
    "top": "25mm",
    "right": "18mm",
    "bottom": "22mm",
    "left": "18mm"
  },
  "header": { "height": "12mm" },
  "footer": { "height": "10mm" },
  "printBackground": true,
  "pdf": { "metadata": { "title": "Invoice {{ invoice.number }}" } },
  "locale": "de-DE",
  "timezone": "Europe/Berlin",
  "currency": "EUR"
}
```

- `paper.format`: `A0` to `A6`, `Letter`, `Legal`, `Tabloid`, `Ledger`. For anything else set
  `paper.width` and `paper.height` (`"100mm"` and `"150mm"` for a shipping label).
- Lengths take `mm`, `cm`, `in` or `px`, or a bare number of pixels. `pt`, `em` and `%` fail the
  render.
- Margins default to `20mm 15mm 20mm 15mm`. `"preferCssPageSize": true` lets `@page` rules in the
  CSS decide the page size instead.
- `pdf.metadata.title` is rendered with the data and becomes the document's title; `author`,
  `subject` and `keywords` are rendered the same way.
- `locale`, `timezone` and `currency` drive every date, number and amount the helpers format.
  Without them: `en-GB`, `Europe/Berlin`, `EUR`.
- `"tailwind": true` compiles the Tailwind utility classes the rendered document uses. Its base
  styles come with it: headings lose their sizes and lists their bullets.

## Page breaks

Chromium paginates by itself. The print reset already keeps a table row whole, repeats a `<thead>`
on every page, and offers the classes `page-break` (a break after the element) and `avoid-break`
(no break inside it); the `pageBreak` helper writes a break too.

```css
h2 {
  break-after: avoid;
} /* a heading stays with the text below it */
.signature {
  break-inside: avoid;
}
tbody tr:last-child {
  break-before: avoid;
} /* the last row never alone on a page */
```

- Build tables from `<table>`, `<thead>` and `<tbody>`, not divs, to get repeated headers.
- A `<tfoot>` repeats at the foot of every page. Put a grand total in a last `<tbody>` of its own
  to print it once.
- A table inside `display: inline-block` is not paginated: Chromium cuts it at the page edge. Use a
  block wrapper.
- An element wider than the page makes Chromium shrink every page to fit it, so everything prints
  smaller than designed. Give wide tables and images `max-width: 100%` and check the rendered PDF.

## Header, footer and page numbers

`header.html` and `footer.html` are rendered once with the same data and stamped onto every page.
Chromium fills the classes `pageNumber`, `totalPages`, `date` and `title` on each page.

```jinja2
<div style="display: flex; justify-content: space-between; width: 100%; font-size: 8pt;">
  <span>{{ company.name }} · Invoice {{ invoice.number }}</span>
  <span>Page <span class="pageNumber"></span> of <span class="totalPages"></span></span>
</div>
```

They are printed in a context of their own:

1. The stylesheet does not apply. Style every element inline, as above.
2. They sit inside the page margin. Make `margin.top` larger than the header and `margin.bottom`
   larger than the footer, or the body runs underneath. `header.height` sets the box, not the margin.
3. They start at 10px with 10mm side padding; `header.padding` and `footer.padding` replace it.
4. They load no images by URL, the file library included. Use a data URI, such as the `qrcode`
   helper's, or the brand kit's logos, which Formfeed inlines.
5. Scripts do not run there.

## Fonts

- Web fonts: put the `<link>` into `head.html`. The renderer waits for fonts before printing.
- Always name a font. The render servers' fallback fonts are not the ones of the person's laptop.
- Uploaded organisation fonts are used by naming their family in the CSS; the brand kit's are
  `var(--brand-font-heading)` and `var(--brand-font-body)`.

## Images

- A file of the library (`{{ asset('logo.png') }}`, or `<img src="logo.png">`), an `https://` URL
  or a data URI. `http://`, `localhost` and private addresses are refused, and the image then shows
  as broken without failing the render.
- `image` places an image at a fixed size with `fit` (`cover`, `contain`, `fill`) and an `alt`.
- `qrcode`, `barcode` and `epcQr` draw codes offline as SVG data URIs for an `<img src>`.

## Data values that are missing

A missing value prints as empty text in all three languages, also inside a missing object. Use
`default` for a deliberate fallback. `date` prints today's date for a missing value, so put an
optional date inside a condition:

```jinja2
{% if invoice.paid_at %}Paid on {{ invoice.paid_at | date("P") }}{% endif %}
```

```json data
{
  "company": { "name": "Fennlor Studio GmbH" },
  "invoice": { "number": "2026-0042", "paid_at": "2026-09-30" }
}
```

## Scripts and waiting

Scripts run as in a browser. Formfeed prints once the page has loaded, its requests are done and
its fonts are ready. Content a script builds later needs `waitFor` in the settings:

```json settings
{ "waitFor": { "selector": "#report-ready", "timeoutMs": 8000 } }
```

The script adds `#report-ready` when it is done. Without it by then, the render fails with
`504 render_timeout`.

## Image templates

A template of kind `image` renders a screenshot instead of a PDF: social cards, labels, badges.

```json settings
{
  "image": {
    "width": 1200,
    "height": 630,
    "deviceScaleFactor": 2,
    "format": "png"
  }
}
```

It has no header or footer. `image.transparent` gives PNG and WebP a transparent background,
`image.clipSelector` captures one element. A box marked `data-ff-fit="shrink"` with a fixed width
and height gets the largest font size at which its text fits.

## Limits

Filling the template may take 5 seconds and produce at most 20 MB of HTML; loading the page 30
seconds. A PDF may have 20 pages on the Free plan, 200 on Starter, 500 on Pro, 1,000 on Business
and 5,000 on Enterprise; more fails the render with `422 page_limit_exceeded`.
