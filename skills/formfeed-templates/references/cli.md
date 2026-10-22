# The formfeed CLI

Generated from the CLI of this release. Run it as `formfeed` when it is installed, else as
`npx formfeed`; it needs Node.js 22 or later.

Every command is marked with what it does to the account:

- **local**: runs on this machine, needs no account and sends nothing to Formfeed.
- **reads the API**: calls the Formfeed API and changes nothing other people see.
- **renders: units with a live key**: renders through the API. Free with a test key; with a live key every render counts against the plan.
- **changes the workspace**: changes what the team shares. Run it only when the conversation asked for it.

## Global flags

| Flag | Meaning |
| --- | --- |
| `-V, --version` | output the version number |
| `--api-key <key>` | API key (overrides the stored login) |
| `--base-url <url>` | API base URL, used instead of the region’s address |
| `--workspace <org/workspace>` | workspace label shown by config list; the API key decides the workspace |
| `--region <region>` | API region (default eu) |
| `--json` | machine-readable output |
| `--ci` | do not open a browser for login; implied by CI=1 |

The key comes from `--api-key`, else `FORMFEED_API_KEY`, else the key `formfeed login` stored.

## Commands

### `formfeed login [key]`

Sign in: opens the browser for device authorisation, or stores a key with --api-key. **reads the API**: the person signs in; it stores a live key with the scopes of their role.

| Flag | Meaning |
| --- | --- |
| `--no-browser` | print the URL instead of opening a browser |

### `formfeed logout`

Remove the stored key. **local**.

### `formfeed whoami`

Show the workspace, plan and usage of the current key. **reads the API**: `--json` says whether the key is test or live.

### `formfeed init [dir]`

Create formfeed.json and the folder layout; --from-workspace pulls the existing templates. **local**: `--from-workspace` reads the workspace.

| Flag | Meaning |
| --- | --- |
| `--engine <id>` | default engine for new templates (jinja2, liquid, handlebars) |
| `--workspace <org/workspace>` | workspace label written to formfeed.json; the API key decides the workspace |
| `--from-workspace` | pull every template of the workspace (published version, else the latest) and its shared partials after creating the layout |

### `formfeed types [slugs]`

Generate TypeScript or Python types for the data of templates. **local**: `--remote` and `--channel` read the workspace.

| Flag | Meaning |
| --- | --- |
| `--lang <lang>` | ts or python (default: formfeed.json types.lang, else ts) |
| `--out <file>` | output file (default: formfeed.d.ts or formfeed_templates.py in the project root) |
| `--channel <name>` | read the schemas of this channel from the API (default: published) |
| `--remote` | read schemas from the API even inside a project |
| `--sdk-module <name>` | module the TypeScript file augments (default @formfeed/sdk) |
| `--check` | write nothing; exit 1 when the file differs from what would be generated |

### `formfeed validate [slug]`

Compile locally and print diagnostics; exit 1 on errors. **local**.

| Flag | Meaning |
| --- | --- |
| `--data <name>` | data set to check against (default: default) |

### `formfeed einvoice <slug>`

Write the e-invoice XML of a data set, offline: the XML a render embeds in the PDF. **local**.

| Flag | Meaning |
| --- | --- |
| `--data <name-or-file>` | data set name or a JSON file |
| `--out <file>` | output file (default: <slug>.xml) |
| `--profile <profile>` | basic or en16931 (default: the template's settings, else en16931) |

### `formfeed render <slug>`

True render through the API from the local files (test keys are free). **renders: units with a live key**.

| Flag | Meaning |
| --- | --- |
| `--data <name-or-file>` | data set name or a JSON file |
| `--out <file>` | output file (default: <slug>.<ext>) |
| `--output <format>` | pdf, png, jpg, webp, docx or pptx (default: the template's kind and settings) |
| `--remote` | render the published remote version instead of local files |

### `formfeed dev [slug]`

Local preview with hot reload (http://localhost:4400). **local**: its True render button renders through the API.

| Flag | Meaning |
| --- | --- |
| `--port <n>` | port (default `4400`) |
| `--host <host>` | bind address (default `127.0.0.1`) |
| `--data <name>` | initial data set |
| `--locale <tag>` | initial locale |
| `--open` | open the browser |

### `formfeed test [slug]`

Run every data set through compile and analysis and compare approved snapshots. **local**.

| Flag | Meaning |
| --- | --- |
| `--data <name>` | only this data set |
| `-u, --update-snapshots` | write the current output as the approved snapshot |

### `formfeed listen`

Forward the workspace's webhook events to a local server, signed with a session secret. **reads the API**: opens a session the Webhooks page lists; `--live` forwards live events too.

| Flag | Meaning |
| --- | --- |
| `--forward-to <url>` | local URL that receives the deliveries (default `http://localhost:3000/webhooks`) |
| `--events <types>` | comma-separated event types, e.g. render.completed,job.failed (default: all but the opt-in storage.delivered and storage.failed) |
| `--live` | include live-environment events (needs the webhook:manage scope) |
| `--print-secret` | start a session, print only its signing secret and end it again |
| `--skip-verify` | accept a self-signed certificate of an https:// forward URL |

### `formfeed trigger <event>`

Make a real async render so a genuine event reaches formfeed listen (render.completed). **renders: units with a live key**: makes a real asynchronous render.

| Flag | Meaning |
| --- | --- |
| `--template <slug>` | template to render |
| `--data <file>` | JSON data (default: the local default data set, else the sample data of the published version) |

### `formfeed workspaces delete <id>`

Delete a workspace (needs the workspace:delete scope); its content is removed after 30 days. **changes the workspace**: no undo.

| Flag | Meaning |
| --- | --- |
| `--yes` | confirm the deletion; required, because the API has no undo |

### `formfeed config list`

Show the effective settings and the config files they are read from. **local**.

### `formfeed config get <key>`

Read a user config value (apiKey, baseUrl, workspace, region). **local**.

### `formfeed config set <key> <value>`

Write a user config value. **local**.

### `formfeed templates list`

Remote templates with published version and status. **reads the API**.

### `formfeed templates pull [slug]`

Download published versions into files (--draft for the latest drafts). **reads the API**: overwrites the local folder.

| Flag | Meaning |
| --- | --- |
| `--draft` | pull the latest version even when it is a draft |

### `formfeed templates push [slug]`

Create draft versions from local files; --publish publishes them. **changes the workspace**: `--publish` and `--channel` change what renders.

| Flag | Meaning |
| --- | --- |
| `--publish` | publish the new version |
| `--message <note>` | change note |
| `--dry-run` | show what would change without pushing |
| `--force` | push even when the remote changed since the last pull |
| `--channel <name>` | point this release channel at the new version (e.g. staging) |
| `--allow-breaking` | publish or move the channel although the data schema breaks callers of the current version |

### `formfeed templates diff [slug]`

Show local changes against the latest remote version. **reads the API**.

### `formfeed channels list <slug>`

Channels of a template with their versions, canaries and renders of the last 24 hours. **reads the API**.

### `formfeed channels set <slug> <name> <version>`

Create a channel or point it at a version; on published this publishes. **changes the workspace**.

| Flag | Meaning |
| --- | --- |
| `--canary <version>` | a second version that receives --percent of the renders |
| `--percent <n>` | share of the canary, 1 to 50 |
| `--allow-breaking` | go ahead although the data schema breaks callers of the current version |

### `formfeed channels promote <slug> <name>`

Make the canary the main version of the channel. **changes the workspace**.

| Flag | Meaning |
| --- | --- |
| `--allow-breaking` | go ahead although the data schema breaks callers of the current version |

### `formfeed channels rollback <slug> <name>`

Return the channel to the version before its last move. **changes the workspace**.

| Flag | Meaning |
| --- | --- |
| `--allow-breaking` | go ahead although the data schema breaks callers of the current version |

### `formfeed channels delete <slug> <name>`

Delete a channel; --force when renders of the last hour still used it. **changes the workspace**.

| Flag | Meaning |
| --- | --- |
| `--force` | delete even when the channel was used in the last hour |

### `formfeed ci preview`

Render the templates changed since a base ref and write a summary for the pull request. **renders: units with a live key**.

| Flag | Meaning |
| --- | --- |
| `--base <ref>` | git ref to compare against (default `origin/main`) |
| `--out <dir>` | directory for the rendered files (default `formfeed-preview`) |
| `--summary <file>` | markdown summary file (default `formfeed-preview.md`) |
| `--all` | render every template instead of the changed ones |

### `formfeed brand pull`

Download the brand kit into .formfeed/brand.json; dev, render, test and ci preview use it as brand. **reads the API**.

### `formfeed partials list`

Remote shared partials with version and local sync status. **reads the API**.

### `formfeed partials pull [names]`

Download shared partials (all when no name is given) and record their versions in .formfeed/state.json. **reads the API**: overwrites the local files.

### `formfeed partials push [names]`

Create or update shared partials from the partials folder (all recorded ones when no name is given). **changes the workspace**: every template of the organisation uses the new version.

| Flag | Meaning |
| --- | --- |
| `--engine <id>` | engine of new partials (default: the recorded engine, else the project engine) |
| `--dry-run` | show what would be pushed |
| `--force` | push even when the remote partial changed since the last pull |

### `formfeed files list`

Files in the workspace library. **reads the API**.

| Flag | Meaning |
| --- | --- |
| `--prefix <prefix>` | only names starting with this, e.g. brand/ |

### `formfeed files push [names]`

Upload new and changed files from the files folder; the same name replaces the remote file. **changes the workspace**: the same name replaces the file for every template.

| Flag | Meaning |
| --- | --- |
| `--dry-run` | show what would be uploaded |

### `formfeed files pull [names]`

Download library files into the files folder. **reads the API**.

### `formfeed files delete <name>`

Remove a file from the workspace library (the local copy stays). **changes the workspace**.

### `formfeed pdf convert <source>`

Convert a Word, Excel, PowerPoint, OpenDocument, RTF or HTML file to PDF (Starter plan and above); a render id (rnd_…) of a Word or PowerPoint template converts its output. **renders: units with a live key**: Starter plan and above.

| Flag | Meaning |
| --- | --- |
| `--out <file>` | output file (default: the file name with .pdf, or <render id>.pdf) |
| `--page-ranges <ranges>` | pages to convert, e.g. 1-3,5 |
| `--landscape` | landscape for spreadsheets and documents without their own page setup |
| `--single-page-sheets` | each spreadsheet sheet on one page |
| `--no-download` | create the PDF render without downloading it |

### `formfeed renders get <id>`

Show one render. **reads the API**.

### `formfeed renders list`

Recent renders of the workspace, newest first. **reads the API**.

| Flag | Meaning |
| --- | --- |
| `--template <slug>` | only renders of this template |
| `--status <status>` | queued, rendering, succeeded or failed |
| `--environment <env>` | live or test |
| `--since <iso>` | only renders created at or after this time |
| `--until <iso>` | only renders created at or before this time |
| `--limit <n>` | page size (1-100) (default `20`) |
| `--all` | follow the cursor and list every match |

### `formfeed renders delete-outputs <id>`

Remove the stored files of a render before they expire. **changes the workspace**: the files are gone for good.

### `formfeed renders pull <id>`

Save a render's stored request data as a data set of its template; --run reproduces it locally. **reads the API**: writes a data set into the project.

| Flag | Meaning |
| --- | --- |
| `--as <name>` | data set name (default: render-<last 6 characters of the id>) |
| `--redact [paths...]` | replace strings with same-shape placeholders; only these data paths when given (customer.*, items[].name) |
| `--no-redact` | skip the redaction formfeed.json sets in pull.redact |
| `--run` | render the data set locally and print diagnostics with template line and column |
| `--snapshot` | write tests/__snapshots__/<name>.html from the local render (not for failed renders) |
| `--force` | overwrite an existing data set of that name |

### `formfeed webhooks resend <eventId>`

Send a stored event (evt_…) again, to the running formfeed listen session or to --to. **changes the workspace**: delivers the event again.

| Flag | Meaning |
| --- | --- |
| `--to <id>` | endpoint or listen session id (default: the session of a running formfeed listen with the same key) |

### `formfeed import apitemplate`

Create local templates from apitemplate.io: read with an API key, or from exported files. **local**: reads the other provider's API with its key.

| Flag | Meaning |
| --- | --- |
| `--key <key>` | apitemplate.io API key (or APITEMPLATE_API_KEY) |
| `--template <ids...>` | template ids to read with the key |
| `--all` | read every PDF template of the account with the key |
| `--group <name>` | read the PDF templates of this apitemplate.io group with the key |
| `--source-region <region>` | apitemplate.io region of the account: default, de, us, au, alt, alt-de, alt-us (or APITEMPLATE_REGION) |
| `--html <file>` | template body (HTML), instead of reading it with a key |
| `--css <file>` | stylesheet (with --html) |
| `--settings <file>` | settings JSON: paper, margins, header and footer (with --html) |
| `--sample <file>` | sample data JSON (with --html) |
| `--name <name>` | template name (with --html) |
| `--slug <slug>` | folder and slug (default: from the name; with --html) |

### `formfeed import pdfmonkey`

Create local templates from PDFMonkey code templates, read through their API. **local**: reads the other provider's API with its key.

| Flag | Meaning |
| --- | --- |
| `--key <key>` | PDFMonkey secret API key (or PDFMONKEY_API_KEY) |
| `--template <ids...>` | template ids to import |
| `--workspace-id <id>` | import every code template of this PDFMonkey workspace |
| `--snippet <name=file...>` | snippet code for templates that load_snippets (repeatable) |
| `--draft` | import the unpublished drafts |

### `formfeed import jsreport <file>`

Create local templates from a jsreport export (.jsrexport). **local**.

| Flag | Meaning |
| --- | --- |
| `--template <paths...>` | templates to import by folder path or name (default: every Handlebars chrome-pdf/chrome-image template) |
