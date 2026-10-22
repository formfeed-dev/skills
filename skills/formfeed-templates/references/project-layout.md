# Project layout

A Formfeed project is a folder with a `formfeed.json`. Each template is a folder below
`templates/`, named after its slug (lowercase letters, digits and dashes).

```text
formfeed.json                  workspace binding and options
templates/
  invoice/
    template.json              name, kind, engine, description, tags
    template.html              the body, in the template's language
    style.css                  optional stylesheet
    head.html                  optional extra <head> content: font links, scripts
    header.html, footer.html   optional running header and footer of a PDF
    settings.json              paper, margins, header and footer size, image size, locale, PDF options
    schema.json                optional JSON Schema of the data
    i18n.json                  optional translations: locale -> key -> text
    data/
      default.json             the sample data set validate and ci preview use
      many-lines.json          more data sets, one file each
    tests/
      __snapshots__/           approved snapshots of formfeed test, one per data set
partials/
  footer.html                  fragments templates include by name
files/
  logo.png                     the workspace file library: asset('logo.png')
.formfeed/
  state.json                   what was last pulled or pushed (written by the CLI)
  brand.json                   the organisation's brand kit (formfeed brand pull)
```

## The files of a template

- `template.html` is required, or instead `template.docx` or `template.pptx` for an office template.
  A folder holds exactly one of the three.
- `template.json` is optional. Its fields: `name`, `kind` (`pdf`, `image`, `docx`, `pptx`),
  `engine` (`jinja2`, `liquid`, `handlebars`), `description`, `tags`. Missing fields default to a
  title made from the slug, `pdf` (or the kind of the document in the folder) and the engine of
  `formfeed.json`. `templates push` uses them when it creates a template; for an existing template it
  changes none of them. To move a template to another language, create a new one.
- `header.html` and `footer.html` are rendered with the same language and data as the body and
  printed on every page of a PDF. They travel as `settings.header.html` and `settings.footer.html`.
- `settings.json` holds the settings except the header and footer markup; see
  [page-setup.md](page-setup.md).
- `data/default.json` is the version's sample data. Without it, the first file in alphabetical
  order takes its place. `formfeed test` renders every file in `data/`.
- `schema.json` is the contract of the data. Without it, `formfeed types` and the publish check
  infer one from the data sets.
- `design.json` belongs to an image template made in the app's visual editor. If it exists, do not
  edit `template.html` and `style.css` by hand: the editor writes them from the design, and
  `formfeed validate` warns (`design-stale`) when they no longer match it.

```json template.json
{
  "name": "Invoice",
  "kind": "pdf",
  "engine": "jinja2",
  "description": "Invoice with line items over several pages",
  "tags": ["billing"]
}
```

## Partials

`{% include "footer" %}` (Jinja2 and Liquid) or `{{> footer}}` (Handlebars) reads `partials/footer`
or `partials/footer.html`. `templates push` sends every partial a template includes with its
version. A template whose include has no file fails `validate`, and `push` refuses it.

Shared partials of the organisation (a letterhead every document uses) are pulled with
`formfeed partials pull` and are not part of any template version: every render uses their current
version.

## The file library

`files/` mirrors the workspace's file library. A file's path below the folder is its library name:
`{{ asset('brand/logo.svg') }}`, or a relative path such as `<img src="logo.png">`. `formfeed dev`
serves the folder, so images work offline. The library accepts PNG, JPEG, WebP, GIF, SVG, AVIF and
PDF files up to 10 MB, with names of letters, digits, `.`, `-`, `_` and `/`. A file is shared by
every template of the workspace, and the same name replaces it for all of them.

## Sync state

`.formfeed/state.json` records the version and checksum of the last pull or push. `templates push`
skips folders that did not change, and refuses with `409 conflict` when someone saved a newer
version in between: pull, merge, push again. `formfeed templates push --dry-run` shows what would
change without touching the workspace.
