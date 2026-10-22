# Formfeed for coding agents

Connects Claude Code and Cursor to [Formfeed](https://formfeed.dev/?utm_source=github-skills), the
API that renders PDFs and images from HTML, Word and PowerPoint templates. The plugin has two parts:

- **Two skills** that teach your agent how Formfeed works: `formfeed-templates` writes, checks,
  previews and renders templates as files with the `formfeed` command line, and `formfeed-api`
  writes the application code that creates documents, in TypeScript, Python or any language over
  HTTP. They tell the agent what it would otherwise guess: the folder of a template, how each
  template language calls a helper, which command checks what, and what costs units.
- **Formfeed's hosted MCP server**, so your agent can list your templates, read the data each one
  expects, check data, render documents and convert Office files, signed in with your account.

## Install

**Claude Code:**

```bash
claude plugin marketplace add formfeed-dev/skills
claude plugin install formfeed@formfeed-dev
```

Or, in a session: `/plugin install formfeed --marketplace formfeed-dev/skills`.

**Cursor:** add the server to `~/.cursor/mcp.json` for every project, or to `.cursor/mcp.json` for
one repository:

```json
{
  "mcpServers": {
    "formfeed": { "url": "https://mcp.formfeed.dev/mcp" }
  }
}
```

**The skills alone, in any agent that reads Agent Skills:**

```bash
npx skills add formfeed-dev/skills
```

## Signing in

The first time the agent connects to the server, Formfeed's sign-in opens in your browser (in
Claude Code: run `/mcp` and choose the Formfeed server). Sign in, choose the workspace the agent
works in, and allow it. The agent keeps a token and refreshes it itself. Every agent you allowed is
listed in the app under **Account → Connected agents**, where you change its workspace or revoke it.

The command line the skills use signs in on its own: run `npx formfeed login` yourself, or put a
test key into `FORMFEED_API_KEY`. The skills never ask for a key in the chat and never write one
into a file.

No Formfeed account yet? The free plan has 100 units a month and needs no credit card:
[formfeed.dev](https://formfeed.dev/?utm_source=github-skills).

## What the agent can do, and what it costs

A signed-in agent reads your templates and the schema of their data, checks data, renders documents
and converts Office files, in the workspace you chose. Through the server it cannot publish or
change templates, delete documents, or manage webhooks, keys, members or billing, whatever your
role.

The skills keep the agent to checks that cost nothing (`formfeed validate`, `formfeed test`, the
local preview) until a template is right, and then render once. A render with a live key counts
units of your plan, as any render does; with a test key it is free and carries a watermark on the
Free plan. The skills tell the agent to run a command that changes your workspace, such as
`formfeed templates push`, only when you asked for it.

## What it runs, sends and fetches

- **Runs:** the plugin itself runs nothing: it holds two skills, which are text, and the address of
  the server. The skills tell your agent to run Formfeed's command line, `formfeed` from npm
  (`npx formfeed`), in your project: it checks and previews templates on your machine, and calls
  Formfeed's API for the commands that render, pull or push.
- **Sends:** your agent connects to `https://mcp.formfeed.dev/mcp`, Formfeed's hosted MCP server,
  and sends it what the agent passes to a tool: template names, data, HTML, and documents to
  convert. The server passes each call on to Formfeed's API with the token of your sign-in and
  stores nothing itself. The command line sends your templates and their data to Formfeed's API,
  `https://api-eu.formfeed.dev`, when it renders, pulls or pushes, with the key you signed in with.
- **Fetches:** the answers of the API: your templates and their schemas, the results of a check,
  and the rendered files. `npx` fetches the command line from npm.

Rendering runs in Frankfurt, and rendered files are stored in the EU. How Formfeed handles your data:
[privacy notice](https://formfeed.dev/legal/privacy?utm_source=github-skills).

## More

- [Templates as code](https://docs.formfeed.dev/getting-started/templates-as-code)
- [The MCP server's tools](https://docs.formfeed.dev/integrations/mcp)
- [Formfeed's documentation](https://docs.formfeed.dev)

This repository is generated from Formfeed's main repository; see [CONTRIBUTING.md](CONTRIBUTING.md).
MIT licensed.
