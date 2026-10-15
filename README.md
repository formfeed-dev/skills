# Formfeed for coding agents

Connects Cursor and Claude Code to [Formfeed](https://formfeed.dev/?utm_source=github-skills), the
API that renders PDFs and images from HTML, Word and PowerPoint templates. Your agent can list your
templates, read the data each one expects, check data against a template, render documents, check
e-invoice data and convert Office files to PDF, while it works on the code that sends the data.

The plugin connects your agent to Formfeed's hosted MCP server. It signs in with your Formfeed
account, so no key is written into any file.

## Install

**Cursor:** add the server to `~/.cursor/mcp.json` for every project, or to `.cursor/mcp.json` for
one repository:

```json
{
  "mcpServers": {
    "formfeed": { "url": "https://mcp.formfeed.dev/mcp" }
  }
}
```

**Claude Code:**

```bash
claude plugin marketplace add formfeed-dev/skills
claude plugin install formfeed@formfeed-dev
```

Or, in a session: `/plugin install formfeed --marketplace formfeed-dev/skills`.

## Signing in

The first time the agent connects, Formfeed's sign-in opens in your browser (in Claude Code: run
`/mcp` and choose the Formfeed server). Sign in, choose the workspace the agent works in, and allow
it. The agent keeps a token and refreshes it itself. Every agent you allowed is listed in the app
under **Account → Connected agents**, where you change its workspace or revoke it.

No Formfeed account yet? The free plan has 100 units a month and needs no credit card:
[formfeed.dev](https://formfeed.dev/?utm_source=github-skills).

## What the agent can do, and what it costs

A signed-in agent reads your templates and the schema of their data, checks data, renders documents
and converts Office files, in the workspace you chose. It cannot publish or change templates,
delete documents, or manage webhooks, keys, members or billing, whatever your role.

It renders live: every render counts units of your plan, as any render does, and converting an
Office file counts as a PDF and needs the Starter plan or above. Reading templates and checking
data cost nothing. To render free with a watermark while you build, connect with a test
key instead, as the [MCP server's documentation](https://docs.formfeed.dev/integrations/mcp#hosted)
shows.

## What it runs, sends and fetches

- **Runs:** nothing on your machine. The plugin holds no commands, hooks or scripts, only the
  address of the server.
- **Sends:** your agent connects to `https://mcp.formfeed.dev/mcp`, Formfeed's hosted MCP server,
  and sends it what the agent passes to a tool: template names, data, HTML, and documents to
  convert. The server passes each call on to Formfeed's API with the token of your sign-in and
  stores nothing itself.
- **Fetches:** the answers of the API: your templates and their schemas, the results of a check,
  and the download links of rendered files.

Rendering runs in Frankfurt, and rendered files are stored in the EU. How Formfeed handles your data:
[privacy notice](https://formfeed.dev/legal/privacy?utm_source=github-skills).

## More

- [The MCP server's tools](https://docs.formfeed.dev/integrations/mcp)
- [Formfeed's documentation](https://docs.formfeed.dev)

This repository is generated from Formfeed's main repository; see [CONTRIBUTING.md](CONTRIBUTING.md).
MIT licensed.
