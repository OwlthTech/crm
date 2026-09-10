# Licensing — where this software may be used

This file is the practical reading of `LICENSE`. It is not legal advice.
The MIT text in `LICENSE` wins if this file and that file disagree.

Copyright (c) 2026 Comp AI. The software is MIT. `package.json` and
`apps/*/package.json` say the same.

This fork (`OwlthTech/crm`) is a copy of that software. MIT still applies.
Keep the copyright notice in every copy.

## What the licence grants

Anyone who has a copy may, free of charge:

- run it for any purpose, including commercial work
- copy it
- modify it
- merge it into other software
- publish it
- distribute it
- sublicense it
- sell copies

Those rights pass to people you give a copy to, on the same terms.

The one condition: include the copyright notice and the MIT permission
notice in all copies or substantial portions.

The software is provided "AS IS". Comp AI and later authors give no
warranty of merchantability, fitness, or non-infringement. They are not
liable for claims that arise from use.

## Where you can use it

These uses match the licence and the product's design (`SECURITY.md`).

| Use | Notes |
| --- | --- |
| Self-host for one organisation | The intended shape. One workspace. Sign-in is the auth model. |
| Internal CRM for a company you control | Set `ALLOWED_SIGN_IN` to a domain you control. |
| A solo install | `ALLOWED_SIGN_IN` may be a single address. |
| Commercial use of *your* install | Selling your product, running a paid internal tool, or using it at work is allowed. |
| Fork, white-label the code, sell a modified copy | Allowed. Keep the MIT notice. Do not claim you are Comp AI. |
| Study, teaching, and local development | Allowed. |
| Private modifications that you never publish | Allowed. |

Telemetry is on by default and sends anonymous install counts. Turn it
off with `CRM_TELEMETRY_DISABLED=1` or `DO_NOT_TRACK=1`. See
`docs/telemetry.md`. Pointing a fork at another project is an edit to
`packages/telemetry/src/project.ts`, not a licence block.

## Where you cannot use it (licence)

MIT forbids almost nothing except ignoring the notice.

You cannot:

- drop the copyright notice or the MIT text from a copy or a substantial
  portion
- pretend the authors warrant the software
- hold the authors liable under this licence for damage the software
  causes

MIT does **not** grant trademarks. "Comp AI", "Comp AI CRM", trycrm.ai,
and related marks stay with their owners. Shipping a fork that looks like
the official product, or using those names as your product name, is not
a right this licence gives you.

MIT does **not** grant rights in other people's data, mailboxes, or
vendor APIs. Those have their own contracts.

## Where you must not use it (product and law)

These are not MIT bans. They are how this codebase is built, and how
data-protection law treats a CRM that reads mail.

### Not a public or multi-tenant product

`SECURITY.md`: this CRM is for **one organisation of authenticated
internal users**. It is not a hardened public API. It is not a
multi-tenant SaaS boundary.

Do not:

- expose it to the public internet as a sign-up product for many
  customers' data
- put two companies' pipelines in one install and call that tenancy
- treat `ALLOWED_SIGN_IN=gmail.com` as a customer list. That is an open
  door
- rely on roles or per-record permissions. After sign-in, every member
  reads and writes every record

If someone must see only part of the pipeline, this is the wrong tool
today.

### Not a place to hide data from the operator

Whoever runs the deployment has the database, the environment, and the
logs. Nothing here protects customer data from the person who hosts it.

Do not use this install as if it were a vault the operator cannot open.

### Not a consumer-facing mailbox product

Gmail, Calendar, and Outlook access exist so the agent can read mail.
People on those threads did not sign up for this CRM. If you deploy it,
you are the data controller for those mailboxes.

Do not:

- connect a mailbox you do not have a lawful basis to process
- paste customer mail into third-party search (`data-boundaries.md`)
- put mailbox bodies into `/workspace` or into logs
- turn on vendor keys before you know what each query sends. A query
  typically carries a name, a domain, and an employer

With no optional keys, nothing leaves except Google or Microsoft APIs
you already connected. That is the safe default.

### Not a substitute for vendor licences

Optional keys and OAuth apps are separate products:

| Vendor | What it is not |
| --- | --- |
| Google Workspace / Microsoft 365 | MIT does not grant those APIs. Restricted Gmail scopes need Google's review for an External app. |
| Slack | Workspace install and scopes are Slack's terms. |
| Perplexity, Context.dev, Vercel AI Gateway, Blob | Metered. Their terms and bills apply. |
| ClickUp or any MCP you add later | That vendor's terms. See `adding-mcp-and-vendor-tools.md`. |

Do not ship this CRM as if it included those services.

### Not a research tool for strangers' faces or guesses

The agent must not search a face by name. It must not guess a person from
an email local part. That is product policy in `docs/agent.md`, not a
court order, and you must not strip it and call the result this CRM.

### Not a place to file security issues in public

Report vulnerabilities privately (`SECURITY.md`). Do not open a public
issue with real data.

## When the licence applies

| Moment | What happens |
| --- | --- |
| You clone or download the repo | You already have a licence to use it under MIT. |
| You run it locally or in production | Still MIT. Your data-protection duties start when real mail and contacts land. |
| You modify the fork | Your changes may use MIT or a compatible licence. You cannot relicense Comp AI's code as proprietary and drop the notice. |
| You distribute a binary or a hosted image | Include `LICENSE`. Keep the notice in the UI or docs you ship if that copy is substantial. |
| You sell access to *your* hosted instance | MIT allows it. You still need a lawful basis for the data. You still must not present it as Comp AI's cloud unless you have a separate agreement. |
| Upstream Comp AI publishes a new version | You may merge it. Their MIT notice stays. |

There is no grant of patent rights beyond what MIT is read to include in
your jurisdiction. There is no CLA. Contributions follow
`CONTRIBUTING.md`.

## What this file is not

- It is not a trademark licence.
- It is not a Data Processing Agreement.
- It is not approval to scrape LinkedIn, ignore Google's API user data
  policy, or ignore Slack's rules.
- It is not permission to run a multi-tenant CRM on this schema.

## Checklist before you deploy

1. Keep `LICENSE` in the tree you ship.
2. Set `ALLOWED_SIGN_IN` to a domain you control.
3. Generate `BETTER_AUTH_SECRET` yourself. Example values are not secrets.
4. Serve HTTPS in production.
5. Keep Postgres off the public internet.
6. Start with no optional vendor keys.
7. Read `SECURITY.md` and `docs/telemetry.md`.
8. Decide if you are the data controller for the mailboxes you connect.

## Pointers

- Binding text: `LICENSE`
- Product threat model: `SECURITY.md`
- Telemetry: `docs/telemetry.md`
- Agent egress: `apps/agent/agent/skills/data-boundaries.md`
- Contributions: `CONTRIBUTING.md`
