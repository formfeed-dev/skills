# Word and PowerPoint templates

An office template is a Word document (`template.docx`, kind `docx`) or a PowerPoint presentation
(`template.pptx`, kind `pptx`) with tags typed into its text. A render fills the tags and keeps
everything else of the file: styles, headers and footers, page setup, tables, pictures, slide
design. The output is the filled document or a PDF of it.

The document is edited in Word or PowerPoint by a person. An agent cannot type into it. Work on the
other files of the folder and tell the person which tag to type where.

```text
templates/
  offer/
    template.json        kind docx, engine, name
    template.docx        the document with its tags
    settings.json        default output, locale, time zone, currency
    schema.json, i18n.json
    data/default.json
    tests/__snapshots__/default.xml
```

There is no `style.css`, `head.html`, `header.html`, `footer.html` or partial: the document brings
its own. `settings.json` holds `office.output` (`pdf`, or the document's own format, which is the
default), and `locale`, `timezone` and `currency` for the helpers:

```json settings
{
  "office": { "output": "pdf" },
  "locale": "de-DE",
  "timezone": "Europe/Berlin",
  "currency": "EUR"
}
```

## Tags

Tags are written in the template's language, as plain text in the document: variables, helpers,
`if`, `for`, `set`, comments.

- A block tag alone in its paragraph stands for the whole paragraph. A loop then repeats the
  paragraphs between its tags, and a condition shows or hides them:

  ```text
  {% for line in offer.lines %}
  • {{ line.description }}: {{ line.total | money }}
  {% endfor %}
  ```

- If that paragraph is also alone in its table row (every other cell empty), the tag stands for the
  row: a row with `{% for line in offer.lines %}`, a data row and a row with `{% endfor %}` repeat
  the data row once per line.
- A block tag inside running text stays inline: `{% if paid %}PAID{% endif %}`.
- The opening and the closing tag must be in the same place: both in the body, or both in the same
  table cell, header, footer or text box.
- A tag must be typed in one go. A tag that Word split across a link, a field, a tracked change or
  two paragraphs is refused (`office-tag-spans-elements`); retype it.
- Includes and partials do not work in a document (`unsupported-in-office`).

Tags are filled in Word's body, text boxes, headers, footers, footnotes and endnotes, and in
PowerPoint's slides, tables and speaker notes. Not in comments, charts, SmartArt, document
properties, slide masters or layouts.

In PowerPoint a code or picture helper becomes a picture shape placed in the text box that holds it
alone, and whole slides cannot be repeated.

## Checking and rendering

- `formfeed validate offer` lists the findings of the tags by part and paragraph
  (`word/document.xml`, paragraph 3).
- `formfeed test offer` fills the document with every data set and compares the XML of the filled
  parts with `tests/__snapshots__/<data set>.xml`.
- `formfeed render offer --output docx --out offer.docx` fills the document on this machine.
- `formfeed render offer --output pdf --out offer.pdf` fills it here and converts it through the
  API, which needs the Starter plan or above.
- `formfeed templates push` uploads the document only when its content changed since the last sync.
