# LCIT-Documentation

A Claude plugin marketplace maintained by the Low Code Integration Team (LCIT), distributing skills that analyze existing Mulesoft and Frends integrations and automatically generate standardized Level 3 sequence diagrams (Mermaid) and functional descriptions, in line with our team's documentation standards. This repo's name reflects that: `LCIT-Documentation`.

This repo is the source for our own marketplace (`lcit-documentation`, similar in spirit to, for example, the Boomi Companion marketplace): users add the marketplace once and from then on get one-click updates, instead of manually downloading and re-uploading `.skill` files.

## Plan requirements: what works where

The marketplace and its skills work on **any Claude plan, including Free** — you just need to turn on **Code execution and file creation** under Settings → Capabilities (Customize → Skills after that). On Free, Pro, and Max, that's a personal toggle; on Team/Enterprise, an org owner enables it under Organization settings → Skills.

**What does need a paid plan (Pro or higher): attaching a whole local folder.** Pointing Claude at an entire customer integration directory in one go — instead of uploading files one by one — is a **Cowork** feature, and Cowork itself is only available on paid plans (Pro, Max, Team, Enterprise), not on Free. So:

- **On any plan, including Free**: install the marketplace and chat with the skills, uploading the relevant flow-XML/JSON file(s) individually.
- **On a paid plan, via Cowork**: additionally attach the customer's whole integration folder as a workspace folder, so Claude can find the relevant files itself instead of you uploading them one at a time — much faster for anything beyond a single small file.

## Hosting: GitLab source of truth + public GitHub mirror

The source of truth is a **private repository on the company GitLab** (`repo.virtualsciences.nl`). It automatically mirrors to a **public GitHub repository**: [github.com/FKoek/LCIT-Documentation](https://github.com/FKoek/LCIT-Documentation).

**Why the public mirror exists:** Claude Code (terminal) can add a marketplace from any git host, GitLab included. But org-wide distribution to the **Claude Desktop app** (Organization settings → Plugins, so the whole team gets the plugin offered automatically without adding it themselves) currently only supports **GitHub-synced marketplaces**, and that GitHub repo needs to be **publicly reachable**. The GitLab repo stays private and remains the actual source of truth; the GitHub side is a technically necessary, public mirror purely to make Desktop-wide distribution possible — not a second place where development happens.

- **Development and review**: always in the GitLab repo.
- **Installing via Claude Code/CLI**: can use the GitLab URL directly.
- **Installing via Claude Desktop (org-wide)**: goes through the GitHub mirror, since that's the only path Claude Desktop supports for team-wide plugin distribution.

## Connecting Confluence to Claude

Only needed for the `create-confluence-documentation` skill — the two documentation skills work fine without it. This is a **personal setting**: every user who wants to publish documentation to Confluence themselves needs to do this once for their own account.

1. Go to **Settings → Connectors** (in Claude.ai or the Claude Desktop app).
2. Find **Atlassian** and click **Connect**.
3. Sign in with your own Atlassian account and approve the requested permissions. This uses OAuth — your Atlassian password is never shared with Claude.
4. If you have access to multiple Atlassian sites, pick the right one (`virtualsciences.atlassian.net`).
5. Done. Claude now uses your own Confluence permissions whenever `create-confluence-documentation` creates or updates a page — never more or less than what you already have access to yourself.

## Step by step: adding the marketplace and generating your first diagram

1. **Turn on Code execution and file creation.** Settings → Capabilities (Free, Pro, Max) or Organization settings → Skills (Team, Enterprise) — see "Plan requirements" above. Works on any plan, including Free.
2. **Add the marketplace.** In Claude Desktop: Customize → Plugins → "+" → Add marketplace, and paste `https://github.com/FKoek/LCIT-Documentation`. (In Claude Code: `claude plugin marketplace add https://github.com/FKoek/LCIT-Documentation`.)
3. **Install the plugin.** Find `integration-diagram-tools` in the marketplace and install it. This adds all three skills: `mulesoft-documentation-skill`, `frends-documentation-skill`, `create-confluence-documentation`.
4. **Want to publish to Confluence?** Connect your Atlassian account once — see "Connecting Confluence to Claude" above.
5. **Give Claude the integration to analyze.** On a paid plan, you can attach the customer's full integration folder as a workspace folder via Cowork; otherwise (or on Free), upload the specific flow-XML/JSON export file(s) directly in chat.
6. **Ask for the diagram/description.** Type a prompt describing what you want — see "Example prompts" below. You don't need to invoke a skill by name; Claude picks the right one based on your request and the uploaded/attached content.
7. **Review the output**, then optionally ask to publish it to Confluence (see the `create-confluence-documentation` example prompts).

## Example prompts

Copy-paste starting points — adjust the specifics to your situation:

**Generating documentation (Mulesoft or Frends — Claude picks the right skill automatically):**
- "Here's the flow XML for our SAP-to-Dollevoet transport order integration. Generate the Niveau 3 sequence diagram and functional description."
- "I've attached the Frends export for the order-status webhook. Can you document this integration?"
- "Analyze the integration folder I just attached and give me a sequence diagram for the customer master data sync flow."
- "Here's an existing diagram — check it against our standards and correct anything that's wrong."

**Publishing to Confluence:**
- "Publish this documentation to Confluence."
- "Can you turn the diagram and description above into a Confluence page under 'Diagram standaardisatie'?"
- "Update the existing Confluence page for this integration with the new version of the diagram."

**Combined, end to end:**
- "Here's the customer's integration folder. Document the three flows related to order processing, and once I've reviewed them, publish them to Confluence."

## Contents

```
LCIT-Documentation/
├── .claude-plugin/
│   └── marketplace.json
└── plugins/
    └── integration-diagram-tools/
        ├── .claude-plugin/
        │   └── plugin.json
        ├── shared/
        │   ├── standards.md                       ← one central copy, used by both documentation skills
        │   ├── functional-description-template.md ← idem
        │   └── assets/example-skeleton.mmd         ← idem
        └── skills/
            ├── mulesoft-documentation-skill/       ← Mulesoft-only
            ├── frends-documentation-skill/         ← Frends-only
            └── create-confluence-documentation/    ← publishes generated documentation to Confluence
```

One plugin, `integration-diagram-tools`, containing three skills:

### `mulesoft-documentation-skill`

Generates a Level 3 sequence diagram and functional description from an existing **Mulesoft** integration (flow XML). Has its own trigger phrasing and its own Mulesoft-element analysis reference (`mulesoft-analyse.md`).

### `frends-documentation-skill`

Generates a Level 3 sequence diagram and functional description from an existing **Frends** integration (process export / C# Code Tasks). Has its own trigger phrasing and its own Frends-element analysis reference (`frends-analyse.md`).

### `create-confluence-documentation`

Publishes documentation generated by either of the two skills above (or supplied directly) as a page in Confluence, reusing the same shared `functional-description-template.md` structure (no separate, duplicated page template) plus the tested Mermaid-embedding method (plain code block — no extension macro, no manual collapsible wrapper). Always asks for explicit confirmation before creating or updating a page, and never decides the target location itself. This used to be duplicated inside both documentation skills; it's now a single, separately callable skill instead.

**Why two separate skills instead of one that detects the platform:** each platform has its own analysis logic and its own vocabulary, so a dedicated, focused skill triggers more reliably than one skill that first has to guess the platform. A previous version of this marketplace combined both platforms into a single skill (and also included a `bottom-up` variant with an extra trigger-tracing step); that combined/bottom-up approach has been retired in favor of this simpler, split setup.

## Architecture: shared standard files, two platform skills

Both skills reference the **same** `standards.md`, `functional-description-template.md`, and `assets/example-skeleton.mmd`, which live at the plugin level (`plugins/integration-diagram-tools/shared/`) rather than being duplicated inside each skill folder — an earlier version of this marketplace did duplicate them, which was cleaned up. Each `SKILL.md` points to them with relative paths (e.g. `../../shared/standards.md`). This works because the whole plugin directory — including `shared/` — is copied as one unit when a user installs the plugin, so the relative paths always resolve.

This means **updating a shared file once updates both skills automatically** — there's no risk of copies drifting apart, because there's only one copy of each.

**Why `standards.md` isn't split per platform:** its content (opening config, block templates, arrow notation, common mistakes) is genuinely platform-agnostic — it describes how *any* sequence diagram should look, regardless of whether the source is Mulesoft or Frends. Splitting it would recreate the exact duplication problem this shared setup solves, for no benefit. Only content that's actually platform-specific (`mulesoft-analyse.md`, `frends-analyse.md`) stays separate, one copy per skill.

**No live connection to Confluence.** `standards.md` is a hardcoded, bundled file — no check, no sync, no prompting the user about updates.

Two places, two roles:

1. **Confluence** — remains the working document where the team maintains the standards' content (Low Code Integration → Diagram standaardisatie → Niveau 3).
2. **This repo (`shared/standards.md`)** — a deliberate, periodic copy.

## Updating the standards

This is a **deliberate, manual action, done by a single person, roughly once a month** (or sooner, if a relevant change lands on Confluence) — never automated:

1. Copy the updated content from the Confluence page "Niveau 3: Integratieproces sequence diagram" into `plugins/integration-diagram-tools/shared/standards.md`. **One copy, and it applies to both skills automatically.**
2. Check whether the change also affects the platform-specific reference files (`mulesoft-analyse.md`, `frends-analyse.md`) or the shared `functional-description-template.md`, and update those where needed.
3. Bump the version number of **both** skills (`SKILL.md` frontmatter and the readable `**Version:**` line, in both `mulesoft-documentation-skill` and `frends-documentation-skill`) and of the plugin itself (`plugin.json`) — even if only the shared standards changed and neither skill's own instructions did, because the effective content both skills use has changed. This version bump is what the marketplace uses to offer the update.
4. Validate and package.
5. Commit and push to this repo, and publish the new version to the marketplace. Briefly describe what changed in the commit message.

Users then don't need to do anything except accept the offered update.

## Installing a skill

See "Step by step" above for the full walkthrough via the marketplace (recommended — you get automatic one-click updates, including updated standards).

**Manual (alternative):** clone this repo and grab the `plugins/integration-diagram-tools` folder, and install it using the usual custom-plugin workflow. Note: this route doesn't get automatic updates — which is exactly why the marketplace is the preferred path.

## Adding a new skill

Missing functionality, or need a whole new type of diagram/documentation (e.g. a third platform)? Let Floris or Sophie know, or add a new folder under `plugins/integration-diagram-tools/skills/` yourself following the same structure (`SKILL.md`, `references/`), referencing the shared files in `../../shared/` (`standards.md`, `functional-description-template.md`, `assets/example-skeleton.mmd`) the same way the three existing skills do.

## Roles

- **Floris & Sophie** — development and maintenance of these skills.
- **Rudy de Wit** — supervisor & assessor.
- **Jan Willem van Doornspeek** — assessor.

## Related documentation

- Confluence: [Diagram standaardisatie](https://virtualsciences.atlassian.net/wiki/spaces/ACR/pages/664436738/Diagram+standaardisatie) — all diagram levels (1 through 4).
- Confluence: [Niveau 3: Integratieproces sequence diagram](https://virtualsciences.atlassian.net/wiki/spaces/ACR/pages/908328961/Niveau+3+Integratieproces+sequence+diagram) — the working document the standards are periodically copied from.
- Confluence: [Automatisch genereren van integratiedocumentatie](https://virtualsciences.atlassian.net/wiki/spaces/ACR/pages/1249738757/Automatisch+genereren+van+integratiedocumentatie) — explainer on these skills for the wider team.
