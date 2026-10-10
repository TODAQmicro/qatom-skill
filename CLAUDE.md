# qatom-skill

The public Qatom agent skill and plugin package. It tells AI agents how to connect to the hosted Qatom MCP server (`https://mcp.m.todaq.net`, OAuth sign-in at `https://pay.m.todaq.net`) and how to use its tools to search the catalog, buy items, manage wallets and manage a business account. README covers Codex, the ChatGPT desktop app, Claude, Gemini CLI, Hermes and OpenClaw. The repo holds Markdown and JSON plus one legacy Python script, with no server code. Codex and Hermes install straight from `main` (README's Claude Code steps do too, but need the missing `.claude-plugin/`; see Architecture), so the files here are the product and merging to `main` is the release. The Claude chat app and Gemini CLI connect to the server only, and OpenClaw installs the copy published to ClawHub.

## Commands

| Command | What it does |
|---|---|
| (none) | There is no install, build, dev server or package script. `package.json` has no scripts |
| No unit tests, lint or typecheck | Use the checks below, plus the CI scanner (see Deploy & release) |
| `git grep -n 'todaq\.net'` | Lists every hard-coded host and address, including the support email and company URL. An endpoint move changes every match for that host |
| `python3 toda_mcp.py --help` | Usage for the legacy CLI. Needs Python 3.10 or later and `httpx`, which no file declares. `auth` signs in to production; the other commands call the production server |

Run this gate before pushing:

```sh
for f in .mcp.json .codex-plugin/plugin.json; do python3 -m json.tool "$f" >/dev/null || echo "invalid JSON: $f"; done
grep -oE '`[a-z]+(_[a-z]+)+`' skills/qatom/SKILL.md | sort -u   # also prints field names; every tool name here must be in the live tool list
grep -nE '"url"|clientId|scopes' .mcp.json; grep -nE 'url:|clientId:|scopes:' skills/qatom/SKILL.md   # the two must match
```

## Setup

- Editing needs only Git, plus `python3` for the gate's JSON check. There is no Node or Python project to set up (so no `.nvmrc`), no GitHub Packages dependency and no `.env`; nothing loads one.
- To try a change you need an MCP client with OAuth support (Codex, Claude Code, Claude desktop, claude.ai or Gemini CLI) and a Qatom account. Sign-in takes the account's email and a one-time code emailed to it, or a passkey. README's "2FA code from the Qatom app" wording predates this; correct it when you edit that section.
- There is no local server. The skill always points at the hosted production server, so every live test uses a real account. Testing says which tools an agent may call.
- Legacy CLI only: Python 3.10 or later and `pip install httpx`.

## Configuration

Nothing in this repo reads environment variables. Connection settings are literal values in files, which each client copies into its own config when the plugin is installed. A change therefore reaches existing users only when they reinstall or update. Because clients read these files directly, the documented public endpoints are hard-coded here by design; this repo is the exception to the shared rule against hard-coding hosts.

| Setting | Where | Notes |
|---|---|---|
| MCP server URL | `.mcp.json` (`mcp_servers.qatom.url`), the SKILL.md frontmatter (`mcp.qatom.url`), the SKILL.md body, README | The server's canonical URL is its origin, `https://mcp.m.todaq.net`. The server keeps `/mcp` as an alias for connectors saved with it. `.mcp.json`, the frontmatter and README use `/mcp`; the SKILL.md body uses the origin |
| OAuth `clientId` and `scopes` | `.mcp.json`, SKILL.md frontmatter | Identical in both files. Scopes: `openid profile email twin` |
| Sign-in host | SKILL.md, README | `https://pay.m.todaq.net` |
| Plugin metadata | `.codex-plugin/plugin.json` | `name`, `version`, `description`, `skills: ./skills/`, `mcpServers: ./.mcp.json` |
| Legacy | `config.json`, `toda_mcp.py` constants | Nothing in this repo reads `config.json`. The CLI hard-codes its hosts and callback port `3333`, keeps tokens in `~/.toda-mcp-token.json` and its client registration in `~/.toda-mcp-client.json` |

## Architecture

| Path | Role | Read by |
|---|---|---|
| `skills/qatom/SKILL.md` | The skill: when to use it, how to connect, the tools and flows | Codex plugin (via `skills/`), OpenClaw/ClawHub, Hermes, any agent that loads it |
| `.mcp.json` | Hosted server connection and OAuth | Codex plugin (`mcpServers`) |
| `.codex-plugin/plugin.json` | Plugin manifest | Codex, ChatGPT desktop |
| `.codexignore` | Keeps the legacy files out of the plugin package | Codex packaging |
| `README.md` · `SECURITY.md` | Install steps per client · vulnerability reporting | People |
| `.github/` | `scanner.yml` (CI) and `dependabot.yml` (GitHub Actions only) | GitHub |
| `toda_mcp.py`, `config.json`, `package.json` | Legacy local-MCP files from before the hosted server | Left out of the Codex package (`.codexignore`); a human may run the CLI by hand |

**The one way to do each thing**
- **Agent guidance** goes in `skills/qatom/SKILL.md`, the repo's only skill. Do not add a second skill, and do not copy guidance into the manifests.
- **The hosted server is the source of truth.** SKILL.md documents only the tools the production server exposes, by their exact names and inputs.
- **Connection settings** live in `.mcp.json` and the SKILL.md frontmatter, and the two stay identical.
- **Packaging metadata** goes in the plugin manifests. `toda_mcp.py`, `config.json` and `package.json` are legacy: do not extend them, and do not add other client code to this repo. Recorded exception to the shared rule on stale files: they stay because the CLI is still one way for a human to list the live tools (see Testing). Remove them only together, in their own `chore/` PR that also updates `.codexignore` and this guide.
- **Version:** `.codex-plugin/plugin.json` holds the plugin version (1.0.0). The `2.0.0` in `package.json` and `config.json` is legacy, and SKILL.md has no version. Bump the manifest version whenever the documented tools, connection settings or manifests change.
- **Claude Code packaging:** there is no `.claude-plugin/` directory, yet README's `/plugin marketplace add` steps read `.claude-plugin/marketplace.json`. When you add it, point it at the same `skills/` and server settings and give it the same version as the Codex manifest.

**Tools the hosted server exposes**

| Tool | Public behaviour |
|---|---|
| `catalog_search` | Lists or searches the catalog. Each item has an `id`, a price, a seller and an input schema |
| `agent_checkout` | Buys an item (`catalog_item_id`, `arguments`) from the agent wallet. Unless the purchase is within the user's own auto-approve limit, it returns at once with an approval link and an `approval_id`. Once the user says they approved, call again with the same item, the same arguments and the `approval_id` |
| `guest_checkout` | Returns a checkout link the user completes by card. First collect every required input in chat and pass them in `arguments`; the checkout page asks for nothing |
| `agent_wallet_info` | Read-only: primary and agent wallet balances |
| `transfer_to_agent` | Moves funds from the primary wallet to the agent wallet. Only on the user's explicit request, and only for the amount they named. Uses the same approval flow |
| `fund_primary_wallet` | Returns a link where the user loads the primary wallet by card |
| `account_info`, `account_catalog`, `account_wallets`, `account_mcp_server`, `account_choose` | Read-only business-account tools. Only users with a business account get them |
| `account_create_item`, `account_update_item`, `account_rename_wallet`, `account_reboot_server`, `account_finish_change` | Business-account changes. Each one needs the user's approval at a link, then `account_finish_change` with the `approval_id` |

A merchant's own Qatom MCP server has the buyer tools above (not the `account_*` tools) plus `mcp_server_info`, and each item the merchant features appears there as its own tool. Sellers manage their catalog with the `account_*` tools or the Qatom dashboard. An endpoint that needs a secret is set up in the dashboard only.

SKILL.md's Auth Flow, Available Tools, Payment Flow, Agent Twin Wallet and Become a Qatom Provider sections describe an earlier tool set: `check_wallet_balance`, `fund_agent_wallet`, `transfer_to_agent_wallet`, `twin_status`, `create_twin`, `configure_twin`, `register_tool`, `list_my_tools`, `update_tool`, `deactivate_tool` and `test_tool`. The server has none of those tools. Those sections also say payments need no approval and describe server internals, and its description and When to Use list still offer provider registration. Replace them using the table above rather than editing or extending them in place. The `.codex-plugin/plugin.json` description ("automatic per-call payments") and README's Onboarding section make the same no-approval claim.

## Testing

- There are no automated tests. CI runs only the plugin scanner, which checks the package's manifests and security, not whether SKILL.md matches the server.
- Run the Commands gate. Then compare every tool name in SKILL.md with the live list from a connected client. Claude Code's `/mcp` shows the server's tools. A human can also use the legacy CLI: `python3 toda_mcp.py auth`, then `python3 toda_mcp.py tools`.
- To try a SKILL.md change before merging, install from your branch (`codex plugin marketplace add TODAQmicro/qatom-skill --ref <branch>`) or load `skills/qatom/` as a skill in your client. Then walk through each flow the change touches.
- **Real money:** `agent_checkout` and `transfer_to_agent` move real funds, and the `account_*` change tools change a live business account. Agents never call them as a test; only a human runs those flows. Agents may call `catalog_search`, `agent_wallet_info`, `fund_primary_wallet`, `guest_checkout` (nothing is charged until the user completes the link) and the read-only `account_*` tools.
- In the PR, list the clients and flows you tried and the live tool names you compared against.

## Deploy & release

| Trigger | Workflow | What it does |
|---|---|---|
| Any pull request; push to `main` | `scanner.yml` | Runs the HOL AI Plugin Scanner action (SHA-pinned) with `contents: read` |
| Weekly | Dependabot | Opens PRs to bump GitHub Actions; no other ecosystems |

- **Merging to `main` is the release.** No workflow deploys or publishes anything. README's install commands track `main` (Codex uses `--ref main`; Claude Code and Hermes install from the repo), so every new install or update picks up the merge immediately. There are no tags or changelog.
- **ClawHub:** no workflow publishes there. The `qatom` listing changes only when a maintainer publishes it.
- **Order:** a SKILL.md change for a new or changed tool merges only after that tool is live on the production server. When a tool is removed, the skill change ships in the same release window.
- **Human steps:** merging, publishing to ClawHub when the listing should change, and paid-flow checks.
- **Verify a release:** the scanner run on `main` is green. Install from `main` in at least one client. "Check my Qatom wallet balance" should call `agent_wallet_info`, and every tool SKILL.md names should appear in the client's tool list.

## Related repos & cross-checks

No code in other repos imports this one; users and their agent clients consume it. The Qatom services it describes live in private TODAQmicro repos.

| Repo | Relationship | Check when |
|---|---|---|
| mcp (private) | The hosted MCP server: every tool name, input and behaviour documented here | Before any tool or flow edit here, compare with the live tool list. When the server adds, renames or removes a tool or changes its inputs, update SKILL.md |
| payment (private) | Sign-in (OIDC) and the API behind the tools | Changing `clientId`, scopes, the sign-in host or the sign-in wording |
| checkout · charge · app (private) | The pages `guest_checkout` and `fund_primary_wallet` link to, and the mobile app where users can also approve purchases and top-ups | Wording about what those pages ask for or where the user approves |
| dashboard (private) | Where sellers manage their catalog, featured tools and endpoint secrets | Seller-flow wording |
| docs (public) | Developer documentation site. It has no MCP page yet | An MCP page added there must agree with SKILL.md |

## Rules specific to this repo

- **Approvals are not optional.** SKILL.md and the manifests never claim that payments skip the user's approval or happen automatically. They never tell an agent to approve for the user, to work around a refused or expired approval, or to call `transfer_to_agent` on its own initiative. The agent gives the user the approval link the tool returned and waits for the user to say they approved.
- **Exact names only.** Never invent or guess a tool name, argument or response field; copy them from the live tool list.
- **Public surface only.** Describe what agents and users see: tools, inputs, links, approvals and wallet roles. Leave out server internals such as internal endpoints, request bodies, token claims, cache timings and infrastructure.
- **No credentials anywhere.** These are public OAuth client settings. Never add a client secret, token, API key or account-specific hostname to any file, example or pasted transcript. Redact approval ids and balances from transcripts in PRs.
- **Links come from tools.** Do not hard-code checkout, approval or card-loading URLs; tell the agent to give the user the link from the tool result.
- **Endpoint changes** go into every file that names the host at once (`.mcp.json`, the SKILL.md frontmatter and body, README, `.codex-plugin/plugin.json`, SECURITY.md, and the legacy `config.json` and `toda_mcp.py`), and only once the new host is live in production. Installed clients keep the old URL, so coordinate with the server maintainers to keep it working.
- **Run every client flow you document.** Do not add or change install steps for a client you did not try them in.
- **Workflows:** pin every action to a full commit SHA with a version comment, keep `permissions: contents: read`, and add no step that needs secrets.
- **Legacy CLI fixes:** token files stay outside the repo and are written owner-only (0600). Refresh tokens are single-use, so refresh only near expiry and never from parallel processes.

## Where the documentation lives

| File | What it covers | Update it when |
|---|---|---|
| `README.md` | The landing page: what Qatom does, how to install or connect it in each client, and onboarding prompts. No developer setup, testing or release notes; those are in this guide. Partly out of date (the "2FA code" sign-in wording, the Claude Code steps that need the missing `.claude-plugin/`, and Onboarding's no-approval claim): this guide takes precedence until it is fixed | Install steps or supported clients change, or the server URL, sign-in host or scopes change |
| `CLAUDE.md` (`AGENTS.md` is a symlink to it) | This guide: commands, configuration, architecture, the tools the hosted server exposes, testing, release, related repos and repo rules | A command, setting, server tool, workflow or repo rule changes |
| `skills/qatom/SKILL.md` | The skill agents load: frontmatter connection settings, when to use it, how to connect, then tools and flows. Out of date after Connecting, and in its description, opening paragraph and When to Use, which describe an earlier tool set without approvals: this guide's tools table takes precedence until it is fixed | The server adds, renames or removes a tool or changes its inputs or approvals, or connection settings change (keep the frontmatter identical to `.mcp.json`) |
| `.codex-plugin/plugin.json` | The Codex and ChatGPT desktop plugin manifest: name, version, the description users see, and the `skills/` and `.mcp.json` paths. Its "automatic per-call payments" description is out of date: this guide takes precedence until it is fixed | The documented tools, connection settings or manifests change (bump `version`), or the description or paths change |
| `SECURITY.md` | How to report a vulnerability privately by email, for this package or the hosted server | The reporting address, the package's components or the server host change |

`toda_mcp.py`, `config.json` and `package.json` are legacy code and config, not documentation (see Architecture). This repo has no docs site, API spec, `.env.example` or `.claude-plugin/`; Qatom's developer documentation is in the public `docs` repo.

<!-- BEGIN QATOM ORG STANDARDS v1 (public edition) — identical in every public TODAQmicro repo; change all copies together -->
## Qatom engineering standards

These rules apply to every TODAQmicro repo. Where a repo-specific section above differs, the repo section wins: it is either stricter, or a recorded exception with its reason.

### This repo is public

Everything committed here is visible to the world. Never commit secrets, internal hostnames other than the documented public endpoints, internal architecture or infrastructure details, customer data, or code and text copied from private repos.

### Releases

- Merging to `main` is a release, whether a deploy or publish workflow ships it or users install straight from `main`. Treat it that way.
- Changes that depend on a new API or MCP tool ship only after that API or tool is live in production.
- Agents never push to `main` and never run deploys or publishes. Prepare them and hand them to a human with the exact command.

### Security

- No secrets in git, ever: keys, tokens, passwords, client secrets, private keys, `.npmrc` tokens, fixtures and doc examples. Use obvious placeholders like `<client-secret>`. If you find a committed secret, stop and tell a human. Deleting it is not enough; it must be rotated.
- Server-side secrets come from the runtime environment, never from source. Browser code is public, so only values that are meant to be public belong in client bundles.
- Client secrets, tokens and session material never go in logs, `console.*`, URLs or JS-readable cookies.
- Validate every external input at the boundary with a schema.
- Fail closed. In production, missing required config stops startup; never fall back to a default URL or an empty secret.
- When you inspect config or env files, print names, never values.
- Before committing, read `git diff --staged` for secrets. If gitleaks is installed, scan the staged changes with it too.

### Making changes

- Branch from `origin/main` with a `feat/`, `fix/`, `docs/` or `chore/` prefix. One concern per PR: keep refactors, renames and formatting out of feature and fix PRs.
- Follow the preferred pattern named in this repo's sections above. Do not introduce a new library, framework or pattern without updating this guide in the same PR.
- Run this repo's checks before pushing (commands are in the sections above). New logic ships with tests where the repo has them; a bug fix ships with a regression check.
- Delete what your change makes dead: no commented-out code, no stale files or unused dependencies left behind.
- No bare `TODO`s. Fix it now, or reference the issue that will.
- A PR description says what changed, why, and how it was verified (the commands actually run).

### Keeping documentation true

Documentation is part of the change. A PR is not done until the docs describe what it does. The "Where the documentation lives" section above lists this repo's docs.

| If you change… | Update in the same PR |
|---|---|
| a command or script | README and this file's Commands section |
| configuration (added, renamed, removed) | README configuration notes and this file |
| what a user or integrator sees (API usage, tool names, install steps) | the user-facing pages that describe it |
| the preferred way to do something | this file: name the new pattern and mark the old one legacy |
| something that cost you time to discover | one line in this file, written as a rule, not as a story |
| this shared standards block | every public repo's copy at once, with the version marker bumped |

Rules for this file:
- Accurate beats complete. Every command and path in it must work today. Run what you document.
- When something here is wrong, fix it in the same PR if it is small, otherwise flag it. Replace stale lines rather than appending corrections.
- Keep it short enough to be read every session. Review changes to it as carefully as code changes; agents follow it literally.

### Developer-friendliness

Every repo converges on these. When a change touches one of them, leave it meeting the standard.
- The README says what the repo is, then covers prerequisites with versions, setup, configuration, how to run and test, and how it is released.
- Node 24 is pinned in `.nvmrc` wherever Node is used.
- Build output is not committed.
- CLAUDE.md is the source and AGENTS.md is a symlink to it.

### Conventions

- The product is Qatom. New user-facing text says Qatom, not TODA or TODAQ.
- New and updated pages document the current public API (v4) and the current MCP tools. Never present internal or deprecated endpoints as the recommended path.
- Never hard-code `todaq.net` or `qatom.ai` where a config value or the documented public endpoint name will do.

### Working with agents

For developers:
- This file is the contract. If you want agents to behave a certain way every time, write it here, not in a chat. A correction you give twice belongs in this file.
- Give an agent a scoped task with a clear done-condition, and review its PR like a colleague's: tests actually run, docs updated, scope respected, no secrets.

For agents:
- Read before you write. Verify claims by running commands, and report results honestly, failures included. Say what you did not verify.
- Stay in scope, and flag unrelated problems rather than fixing them silently.
- If this guide and the code disagree, trust the code, then fix the guide in your PR or flag the mismatch.
<!-- END QATOM ORG STANDARDS v1 (public edition) -->
