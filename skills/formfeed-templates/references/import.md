# Importing templates from another provider

`formfeed import` reads templates of another provider and writes one local template folder per
template into the project. Nothing reaches the Formfeed workspace until someone runs
`formfeed templates push`. The import takes three sources:

| Source         | Command                                     | Reads                                                                                                                                                                                                    |
| -------------- | ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| apitemplate.io | `formfeed import apitemplate`               | their API with the account's key (`--key` or `APITEMPLATE_API_KEY`), templates by `--template <ids...>`, `--group <name>` or `--all`; or exported files with `--html`, `--css`, `--settings`, `--sample` |
| PDFMonkey      | `formfeed import pdfmonkey`                 | their API with the secret key (`--key` or `PDFMONKEY_API_KEY`), by `--template <ids...>` or `--workspace-id <id>`                                                                                        |
| jsreport       | `formfeed import jsreport export.jsrexport` | an export file, every Handlebars template in it or `--template <paths...>`                                                                                                                               |

The other provider's key belongs to the person. Ask them to set the environment variable, and do
not put it into a command line or a file.

An apitemplate.io template becomes a Jinja2 template, a PDFMonkey template a Liquid template, a
jsreport template a Handlebars template. The importer maps the settings (paper, margins, header and
footer), the foreign filters and the sample data, and reports what it could not map.

After an import:

1. Read the report it prints, and fix every error in the template it names.
2. `formfeed validate --json`, then `formfeed test -u` once to record the first snapshots, saying so.
3. `formfeed dev <slug>` for the person to compare the preview with the original.
4. `formfeed templates push` only when the person asks for it. It creates draft versions; nothing is
   published without `--publish`.

The migration guides list what each provider's syntax becomes:
[apitemplate.io](https://docs.formfeed.dev/migrate/apitemplate-io),
[PDFMonkey](https://docs.formfeed.dev/migrate/pdfmonkey),
[jsreport](https://docs.formfeed.dev/migrate/jsreport).
