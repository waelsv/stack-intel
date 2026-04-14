# Stack Intelligence — User Guide

A Claude Code skill that monitors your providers and dependencies for news that matters to your specific project. It reads your codebase, figures out what you depend on, and searches the web for recent announcements, deprecations, pricing changes, and security advisories — then tells you only what's relevant and why.

## Installation

### 1. Install the skill at user level

Place the `stack-intel/` folder in your user-level Claude Code skills directory so it's available across all projects:

```bash
mkdir -p ~/.claude/skills
cp -r stack-intel ~/.claude/skills/stack-intel
```

The directory structure should be:

```
~/.claude/skills/
└── stack-intel/
    ├── SKILL.md
    └── references/
        └── provider-sources.md
```

### 2. (Optional) Add project-specific overrides

If a project has providers the scan can't detect from code (regulatory bodies, platform policies, industry standards) or needs custom search behavior, create a `.stack-intel-overrides.json` in that project's root:

```json
{
  "providers": {
    "add": {
      "gdpr": {
        "tier": "strategic",
        "role": "EU General Data Protection Regulation — compliance dependency"
      }
    },
    "promote": {
      "express": "strategic"
    }
  },
  "search": {
    "extra_queries": ["PCI-DSS v4.0 compliance deadline 2026"]
  },
  "scoring": {
    "high_priority_keywords": ["PCI", "card data", "3D Secure"]
  }
}
```

This file is optional. The skill works without it — overrides just let you tune behavior per project without touching the skill itself. See the SKILL.md "Project-Level Overrides" section for the full schema.

### 3. That's it

No other configuration needed. The skill activates automatically when you ask relevant questions in Claude Code, in any project.

---

## Usage

### First run

Open Claude Code in your project and say:

```
/stack-intel
```

or just ask naturally:

```
What's new that affects my stack?
```

On the first run, the skill will:

1. **Scan your codebase** to identify providers, platforms, and key libraries
2. **Show you the provider map** and ask you to confirm it — you can add things it missed or remove things that aren't important
3. **Search the web** for recent developments from each provider
4. **Deliver a prioritized briefing** with findings scored against your actual architecture

The scan results are saved to `.stack-intel.json` in your project root. Subsequent runs load this file and skip the full scan unless the list is stale (>14 days) or your dependencies have changed.

### Subsequent runs

Just invoke the skill again. It will:

1. Load the existing provider list from `.stack-intel.json`
2. Do a quick diff to check if you've added any new providers since last time
3. Search for news — focusing on what's changed since the last check
4. Deliver the briefing

This is significantly faster than the first run since it skips the full codebase scan.

---

## Invocation Phrases

The skill responds to natural language. Any of these will trigger it:

- `/stack-intel`
- "What's new that affects me?"
- "Any provider updates I should know about?"
- "Pre-sprint check"
- "Stack news"
- "What changed in my stack?"
- "Dependency news"

### Targeted checks

You can also ask about a specific provider or announcement:

- "Did you see Vercel's latest announcement? What's relevant to us?"
- "Any recent changes to the Twilio API?"
- "Has Next.js had any breaking changes lately?"

In targeted mode, the skill skips the full scan and focuses on that one provider, scoring findings against your project.

---

## What It Produces

A prioritized briefing that looks like this:

```
## Stack Intelligence Briefing — my-saas-app
### Scan date: 2026-04-14
### Providers scanned: Vercel, OpenAI, PostgreSQL, Twilio, Datadog, Redis

🔴 HIGH RELEVANCE

**Vercel: Edge Functions now support streaming responses up to 25MB**
Persistent streaming with up to 25MB payloads, up from the previous 4MB cap.
→ Project impact: api/export.ts falls back to a serverless function because PDF
  responses exceed the 4MB edge limit. That constraint is now removed.
→ If ignored: Paying for serverless invocations on the export endpoint unnecessarily.
  Based on current traffic (~2k exports/day), estimated ~$45/month in avoidable cost.
→ Recommendation: Reviewed the migration guide — the only change needed is removing
  the `runtime: 'nodejs'` override in api/export.ts. No API changes required.
→ Deadline: none
Source: https://vercel.com/blog/edge-functions-streaming (April 10, 2026)

🟡 MEDIUM RELEVANCE

**OpenAI: GPT-4o mini price reduction and new rate limits**
50% price drop on input tokens; new tier-based rate limits take effect May 1.
→ Project impact: lib/ai.ts calls GPT-4o mini for summarization (~12k calls/day).
  Price drop saves ~$170/month automatically. New rate limit for your current tier
  is 4,000 RPM — your peak is ~3,200 RPM, so you have headroom but not much.
→ If ignored: Cost savings apply automatically. However, if traffic grows ~25%
  you'll hit the new rate limit and get 429 errors in the summarization pipeline.
→ Deadline: May 1, 2026 (new rate limits take effect)
Source: https://openai.com/blog/gpt4o-mini-pricing-update (April 8, 2026)

⚪ LOW RELEVANCE

**Datadog: New browser SDK for session replay**
Client-side session recording with heatmaps and frustration signals.
→ If ignored: No impact — project uses Datadog for server-side monitoring only,
  no browser SDK is installed.
Source: https://www.datadoghq.com/blog/session-replay-sdk (April 5, 2026)

Nothing found for: PostgreSQL, Redis
```

Each finding includes:

- **Relevance** (🔴 HIGH / 🟡 MEDIUM / ⚪ LOW) — how closely this connects to your project, grounded in specific files and configs
- **If ignored** — a concrete statement of what happens to *your* project if you do nothing, not a generic risk category
- **Recommendation** — a specific next step based on analysis the skill already performed (it reads migration guides, checks compatibility, and assesses blast radius rather than telling you to do that yourself)

---

## Files the Skill Creates

### `.stack-intel.json`

Saved in your project root after the first run. Contains:

- The provider map (names, roles, which files reference them)
- Discovered news source URLs for each provider
- Timestamps for last full scan and last update

You can commit this file to version control if you want your team to share the provider list. Or add it to `.gitignore` if you prefer each developer to maintain their own.

You can also **edit this file manually** to add providers the skill didn't detect, remove ones you don't care about, or pin specific news URLs you know are authoritative.

### `.stack-intel-overrides.json` (you create this, optional)

Project-specific configuration that customizes the skill's behavior without modifying the skill itself. Lives in the project root. Use it to:

- Add providers the codebase scan can't detect (regulatory bodies, platform policies)
- Promote or demote dependency tiers
- Adjust the search window or cap search calls
- Add extra search queries for industry/regulatory monitoring
- Boost or suppress findings by keyword

See the SKILL.md "Project-Level Overrides" section for the full schema. This file should be committed to version control — it's project configuration, not ephemeral state.

---

## Tips

**CLAUDE.md makes it smarter.** If your project has a `CLAUDE.md` with architectural context, constraints, or goals, the skill uses that to score relevance more accurately. For example, if your CLAUDE.md mentions "all user data stored in EU region only," any GDPR enforcement action or cloud provider region change gets flagged as high relevance automatically.

**The first run is the slowest.** It needs to scan the full codebase and discover news sources. After that, runs are incremental and much faster.

**You can edit the provider list.** If the skill missed something or classified a dependency wrong, just tell it during the confirmation step ("add Docker Hub as strategic" or "Next.js is critical, not just commodity") and it will update `.stack-intel.json`.

**Use targeted mode for email digests.** When a provider sends you an announcement email (like Vercel's Ship Week), paste the URL and ask the skill to assess it. It will read the announcement and map it against your project without re-scanning everything.

---

## Requirements

- **Claude Code** with web search enabled (the skill needs to search for provider news)
- A project with at least some dependency files (package.json, requirements.txt, config files, etc.)
- Works best with projects that have a `CLAUDE.md` describing architectural context
