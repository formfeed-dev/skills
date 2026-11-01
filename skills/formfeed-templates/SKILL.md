---
name: formfeed-templates
description: Write, change, check, preview and render Formfeed document templates as files, or save drafts of a workspace's stored templates through the Formfeed MCP tools. Use when a project has a formfeed.json, when someone asks for a PDF, image, Word or PowerPoint template for Formfeed (invoices, offers, certificates, reports, labels, social images), when a template written in Jinja2, Liquid or Handlebars should print correctly as a PDF, or when templates should be moved to Formfeed from another provider.
license: MIT
---

# Formfeed templates

Formfeed renders documents from templates: HTML in Jinja2, Liquid or Handlebars that Chromium
prints as a PDF or an image, or a Word or PowerPoint file with tags in its text. In a project each
template is a folder of plain files, and the `formfeed` CLI (Node.js 22 or later, `npx formfeed`)
checks, previews and renders them with the same engine as the API.

Work in this order.

## 1. Find the project

The project is the folder with `formfeed.json`. Without one, run `formfeed init`, which needs no
account and writes the layout with a starter template (`--engine liquid` or `--engine handlebars`
for another language). One template is one folder below `templates/`, named after its slug. Which
file is what: [references/project-layout.md](references/project-layout.md).

## 2. Write in the template's language

The engine is the one the template's `template.json` names, else the project's. The three
languages never mix in one template, and they call the same helper three ways:
`{{ total | money("EUR") }}` in Jinja2, `{{ total | money: "EUR" }}` in Liquid,
`{{money total "EUR"}}` in Handlebars. Before you use a helper, read its row in
[references/helpers.md](references/helpers.md); the syntax of each language, and what differs from
the Jinja2, Liquid or Handlebars you know, is in [references/engines.md](references/engines.md).
Liquid does not escape: add `| escape` to every value that comes from people.

## 3. Keep the data in data sets

`data/default.json` and the other files in `data/` are sample data. They feed the preview, the
checks and the inferred schema, and never a customer's document: a render through the API always
uses the data of its request. Add a data set for each case worth seeing (one line, fifty lines, an
optional field missing) instead of changing the template to fit one example.

## 4. Lay the page out for print

A PDF is a web page Chromium prints at the paper's width. Paper, margins, header and footer are
settings in `settings.json`, not CSS; the header and footer get no stylesheet and load no images by
URL; a table row never splits across pages and its `<thead>` repeats; an element wider than the
page shrinks every page. Read [references/page-setup.md](references/page-setup.md) before you
touch layout, headers, footers, fonts or images.

## 5. Check, then render once

Run these from the project folder, and fix what they report before you go on:

```bash
formfeed validate invoice --json      # compiles and checks against the default data set; exit 1 on errors
formfeed test invoice                 # renders every data set and compares the approved snapshots
```

`validate` reports errors (syntax, unknown helpers, an include without a partial) and warnings
(a variable missing from the data set). Fix every error; read every warning, which is usually a
typo or a field the data set lacks. `formfeed test -u invoice` approves the current output as the
new snapshot: run it only for a change that was asked for, and say that you did.

Then render one document and look at it:

```bash
formfeed render invoice --data default --out invoice.pdf
```

Open the file and check the pages: the header and footer, where the pages break, nothing cut off
at the right edge. `formfeed dev invoice` serves a preview with hot reload on
`http://localhost:4400` for the person to watch.

## 6. Know what costs and what changes the workspace

- `init`, `validate`, `test`, `dev` and `einvoice` run on this machine and need no account. Work
  there until the template is right.
- Before the first command that calls the API, run `formfeed whoami --json`. Not signed in: ask the
  person to run `formfeed login` themselves, or to store a test key with `formfeed login ff_test_...`.
  Never ask for a key in the chat, and never write one into a file.
- `environment` in that answer says which key is in use. A `test` key renders free, with a
  watermark on the Free plan. A `live` key counts every render against the plan: tell the person,
  render once when `validate` and `test` pass, and never in a loop.
- Never run a command that changes the workspace the team shares unless this conversation asked
  for it: `templates push`, `channels set`, `channels promote`, `channels rollback`,
  `channels delete`, `partials push`, `files push`, `files delete`, `renders delete-outputs`,
  `webhooks resend`, `workspaces delete`. Each command and what it does is in
  [references/cli.md](references/cli.md).

## 7. Word and PowerPoint templates

A folder with `template.docx` or `template.pptx` instead of `template.html` is an office template.
Its document is edited in Word or PowerPoint, not here: work on its data sets, `settings.json`,
`schema.json` and snapshots, and tell the person which tags to type where.
[references/office.md](references/office.md) has the rules for tags in a document.

## 8. A document from a stored template, without editing anything

`formfeed templates list` names the workspace's templates, `formfeed templates pull invoice` writes
one with its schema and sample data into the project, and `formfeed render invoice --remote`
renders its published version. When this session has the Formfeed MCP tools, use those instead
(section 9): they need no project.

## 9. Stored templates through the Formfeed MCP tools

When this session has the Formfeed MCP tools, they work on the workspace's stored templates:

- **A document:** `list_templates` finds the slug, `get_template_schema` gives the shape of the
  data, `validate_template` checks the data for free, and `render` makes the document from the
  published version.
- **A change or a new template:** read the template with `get_template_source`, change only what
  the person asked for, and save it with `save_template`, passing the `checksum` you read as
  `base_checksum` (required for a template that exists) and a `change_note` that says what changed.
  Pass only the parts you changed; the tool keeps the others from the version you read. A new
  template is `new_template` together with `html`. When the save reports a conflict, someone saved
  a newer version since you read it: read it again and make the change on that version.
- **Always a draft:** `save_template` never publishes. Renders, `render` included, keep using the
  published version until a person publishes the draft in Formfeed. Fix every error the save's
  check reports and save again (`validate_template` with `version` checks a saved draft once more),
  then hand the person the `review_url`, the page where they review and publish it. Never say that
  the change is live.
- **Only with permission:** an API key needs the `template:write` scope. A signed-in agent saves
  drafts only when the person allowed it for this agent (on the consent page, or later under
  Account → Connected agents) and their role may edit templates. Otherwise the tool answers
  `403 forbidden` with the reason: tell the person, and try no other way around it.
- **Not Word or PowerPoint:** their document is changed in Word or PowerPoint and uploaded in
  Formfeed; the tools do not save it.
- Save only when the conversation asked for a change. A draft changes no document, but the team
  sees it in the version history, and a new template counts towards the plan's template limit.
  More: [template drafts](https://docs.formfeed.dev/integrations/mcp#drafts).

## 10. Templates from another provider

`formfeed import` turns templates of another provider into local folders, which you then check
like any other: [references/import.md](references/import.md).

Without a shell, as in a chat, write the files and give the person the commands to run.
