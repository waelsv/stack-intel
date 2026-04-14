---
name: stack-intel
description: "Scans the current project's codebase to identify providers, platforms, and strategic dependencies (cloud infra, LLM APIs, app stores, key libraries), then searches the web for recent announcements, releases, deprecations, and policy changes from those providers — returning a prioritized briefing of what matters for THIS specific project. Use this skill whenever the user asks things like 'what's new that affects me', 'any provider updates I should know about', 'pre-sprint check', 'what changed in my stack', 'stack news', 'dependency news', 'provider updates', or any request for ecosystem intelligence about their current project. Also trigger when a user mentions a specific provider announcement (e.g. 'did you see AWS re:Invent announcements') and wants to understand its relevance to their project."
---

# Stack Intelligence

A codebase-aware provider intelligence skill. It bridges two things no existing tool combines: deep understanding of your project's architecture and real-time awareness of what your providers are shipping.

## What This Skill Does

1. **Scans the project** to build a provider/dependency map — not just package versions, but how deeply each dependency is integrated and what role it plays architecturally.
2. **Searches the web** for recent developments from those providers — announcements, releases, deprecations, policy changes, pricing shifts, security advisories.
3. **Scores and filters** each finding against the project's actual architecture to determine relevance.
4. **Produces a prioritized briefing** organized by impact level, with specific references to files and patterns in the codebase that make each finding relevant.

## Project-Level Overrides

This skill is designed to live at the user level (`~/.claude/skills/stack-intel/`) and work across all projects. Individual projects can customize its behavior without modifying the skill itself by placing a `.stack-intel-overrides.json` file in the project root.

### Loading order

On every run, before doing anything else, check for overrides:

Read `.stack-intel-overrides.json` from the project root. If the file exists, load it and apply its directives throughout the pipeline. Overrides are merged on top of the skill's default behavior — they extend, they don't replace.

### Override file structure

```json
{
  "providers": {
    "add": {
      "google-maps-api": {
        "tier": "strategic",
        "role": "Geocoding and place search — core to location features",
        "news_urls": ["https://cloud.google.com/blog/products/maps-platform"]
      },
      "gdpr": {
        "tier": "strategic",
        "role": "EU General Data Protection Regulation — compliance dependency",
        "news_urls": []
      }
    },
    "remove": ["pytest", "black"],
    "promote": {
      "express": "strategic"
    },
    "demote": {
      "sentry": "commodity"
    }
  },
  "search": {
    "window_days": 60,
    "max_searches_per_provider": 2,
    "extra_queries": [
      "HIPAA enforcement action 2026",
      "healthcare data breach notification rule changes"
    ]
  },
  "scoring": {
    "high_priority_keywords": ["PHI", "patient data", "HIPAA", "BAA"],
    "ignore_keywords": ["enterprise plan", "SOC2"]
  }
}
```

### What each section does

**providers.add** — Providers the scan won't discover from code alone. Regulatory bodies, industry standards, platform policies that don't leave import statements. These are always included in the provider map regardless of what the codebase scan finds.

**providers.remove** — Dependencies the scan might detect but the user never wants to hear about. Suppresses them from the map entirely.

**providers.promote / providers.demote** — Override the tier classification. If the scan thinks Express is commodity but it's actually load-bearing for the project, promote it to strategic. If the scan thinks Sentry is strategic but it's behind a wrapper, demote it to commodity.

**search.window_days** — Override the default 30-day lookback window. Set to 60 or 90 for projects with less frequent sprint cycles.

**search.max_searches_per_provider** — Cap search calls per provider. Useful for keeping token costs down.

**search.extra_queries** — Additional search queries to run that aren't tied to a specific provider. Useful for regulatory or industry-level monitoring (e.g., HIPAA enforcement actions, PCI-DSS requirement changes affecting payment flows).

**scoring.high_priority_keywords** — Findings containing these terms get boosted toward HIGH relevance regardless of other scoring factors.

**scoring.ignore_keywords** — Findings containing these terms get suppressed. Useful for filtering out enterprise-only features or irrelevant product tiers.

### All fields are optional

An override file with just `{"providers": {"add": {"google-maps-api": {"tier": "strategic"}}}}` is perfectly valid. Everything else falls back to the skill's default behavior.

---

## Step 1: Build the Provider Map

Before scanning the codebase from scratch, check whether a provider map already exists.

### 1a: Check for overrides and existing provider list

First, load `.stack-intel-overrides.json` if it exists (see Project-Level Overrides above). Then look for `.stack-intel.json` in the project root — the skill's persistent state file. If it exists and is recent, use it as the starting point.

Read `.stack-intel.json` from the project root if it exists. The file structure looks like this:

```json
{
  "last_full_scan": "2026-04-14",
  "last_updated": "2026-04-14",
  "providers": {
    "vercel": {
      "tier": "strategic",
      "role": "Hosting, Edge Functions, CI/CD",
      "evidence": ["vercel.json", "9 files use edge runtime"],
      "news_urls": [
        "https://vercel.com/blog",
        "https://vercel.com/changelog"
      ]
    },
    "openai": {
      "tier": "strategic",
      "role": "LLM inference API",
      "evidence": ["used in 6 files", "api_key in .env.example"],
      "news_urls": [
        "https://openai.com/blog"
      ]
    }
  }
}
```

**If the file exists and `last_full_scan` is within the last 14 days:**
- Load it as the current provider map
- Proceed to Step 1b (lightweight diff) to check for any new additions
- Skip the full codebase scan

**If the file exists but `last_full_scan` is older than 14 days:**
- Load it as a baseline, but run the full scan (Step 1c) to catch anything that's changed
- Merge results: preserve existing entries, add new discoveries, flag anything in the file that's no longer in the codebase

**If the file doesn't exist:**
- This is a first run — proceed to Step 1c (full codebase scan)

### 1b: Lightweight diff (when existing list is fresh)

Run a quick check to see if the codebase has added any providers not already in the list. This is cheaper than a full scan:

```bash
# Check for new package manifest entries since last scan
git diff --name-only HEAD~10 -- requirements.txt package.json pyproject.toml Pipfile go.mod Cargo.toml 2>/dev/null

# Check for new config files that suggest a new provider
git diff --name-only HEAD~10 -- "*.toml" "*.yml" "*.yaml" docker-compose* Dockerfile 2>/dev/null

# Check for new provider imports not in the existing list
# (construct grep patterns from providers NOT already in .stack-intel.json)
```

If the diff turns up new providers, add them to the list and discover their news sources (see Step 2). If nothing new is found, move straight to Step 2 using the existing list as-is.

### 1c: Full codebase scan (first run or stale list)

Scan the project to identify providers and dependencies. Classify each into one of two tiers:

**Strategic Dependencies** (always check for news) — providers where a change could alter the architecture, block a feature, or force a refactor. These typically include cloud/infra providers, AI/ML APIs, app store and distribution platforms, auth/identity services, payment processors, messaging/email services, primary databases, and any library that's deeply integrated across many files.

**Commodity Dependencies** (check only for security advisories) — easily replaceable, wrapped behind abstractions, or used in limited scope. Typically: utility libraries, dev-only tooling, anything with a thin integration surface (imported in 1-2 files behind a wrapper).

#### How to discover providers

Don't rely on a predefined checklist of files or provider names. Instead, explore the project dynamically:

1. **Survey the project structure.** List the root directory and key subdirectories. Every config file, dotfile, manifest, and directory name is a signal. If you see a file you don't recognize, read it — it may reveal a provider no predefined list would include.

2. **Read package manifests.** Find whatever dependency files exist (they vary by ecosystem) and read them. Assess each dependency using your general knowledge to determine what it is and whether it's strategic or commodity.

3. **Check environment variable names.** Read `.env.example`, `.env.template`, or similar files. Variable names containing `_API_KEY`, `_SECRET`, `_TOKEN`, `_URL`, or provider-specific prefixes reveal external services. Read names only — never read or output secret values.

4. **Read infrastructure and deployment configs.** Whatever CI/CD, hosting, or IaC config files exist in the project, read them. These reveal cloud providers, deployment targets, and infrastructure dependencies.

5. **Read project documentation.** Check README, CLAUDE.md, ARCHITECTURE.md, and any docs/ directory for architectural context, provider mentions, and strategic decisions the code alone won't reveal.

6. **Assess integration depth.** For each discovered provider, check how many files reference it and whether it's behind an abstraction layer. A provider called from 15 files is strategic; the same provider wrapped in a single adapter module may be commodity.

### 1d: Confirm and persist

After building or updating the provider map, present it to the user for confirmation. They may want to add providers you missed, flag something as more/less important, or remove something irrelevant.

Example output:

```
STRATEGIC DEPENDENCIES:
- Vercel (Hosting, Edge Functions, CI/CD) — vercel.json, 9 files use edge runtime
- OpenAI API — used in 6 files for inference, api_key in .env.example
- Google Play Store — android/ directory present, play-billing integration
- Twilio — SMS verification flow in auth/sms.ts, webhook handler
- Supabase — primary database, auth, and realtime subscriptions
- Mapbox — map rendering in 4 components, geocoding in search

COMMODITY DEPENDENCIES:
- express (framework, but deeply integrated — treat as strategic)
- lodash, dayjs, eslint, jest (commodity)

NEW since last scan: (none | list of newly discovered providers)
REMOVED since last scan: (none | list of providers no longer detected)
```

Once confirmed, write or update `.stack-intel.json` in the project root with the full provider map, timestamps, and any discovered news URLs from Step 2. This becomes the baseline for the next run.

## Step 2: Search for Recent Developments

For each strategic dependency, search the web for recent news. Do NOT rely on a pre-built list of URLs — discover each provider's news sources dynamically.

### Search Strategy

For each strategic provider from Step 1:

**First, find where they publish.** Search for the provider's blog, changelog, or release notes page. See `references/provider-sources.md` for discovery patterns by provider type (cloud, library, app store, etc.). For open-source libraries, the GitHub releases page is usually the best source — and you can often extract the repo URL from the package manifest.

**Then, search for recent developments.** Run 1-3 targeted searches per provider:

```
# General announcements
"[provider] announcement [current month] [current year]"

# Breaking changes and deprecations
"[provider] deprecation breaking changes [current year]"

# Pricing or policy changes
"[provider] pricing change [current year]"
"[provider] policy update [current year]"
```

**Fetch the actual pages.** Search snippets are often too sparse to assess relevance. When a search returns a promising blog post or changelog entry, fetch it to get the full content before scoring.

Aim for recency. Default to the last 30 days unless the user specifies a different window.

### What to Look For

For each finding, classify it:

| Signal Type | What It Means | Example |
|---|---|---|
| **Opportunity** | New capability that could improve or simplify something in the project | "Vercel shipped Edge Middleware v2 — could simplify your auth redirect logic" |
| **Threat** | Deprecation, breaking change, or policy shift that requires action | "Supabase deprecated auth.signIn() in v2.x — migration required" |
| **Strategic** | Pricing change, new competitor feature, or ecosystem shift that informs decisions | "AWS Lambda reduced per-request pricing by 30%" |
| **Security** | CVE, advisory, or supply chain concern | "Critical vuln in dependency X, you import it in 3 files" |

## Step 3: Score and Filter

Not everything is relevant. For each finding, ask:

1. **Does this project actually use the affected feature/API?** Check the code, not just the dependency list.
2. **How deeply integrated is the affected component?** A change to something called from 20 files matters more than something in one utility script.
3. **What's the time horizon?** A deprecation effective next month is urgent. A new feature to evaluate is not.
4. **Does this interact with a known constraint or goal?** (Check CLAUDE.md, README, or other project docs for context on what the team is trying to achieve.)
5. **Does the override file influence this finding?** If `.stack-intel-overrides.json` defines `high_priority_keywords` or `ignore_keywords`, apply those. A finding matching a high-priority keyword gets boosted; a finding matching an ignore keyword gets suppressed.

Assign two scores to each finding:

**Relevance** — how closely this finding connects to the project's actual architecture, based on codebase evidence:

- 🔴 **HIGH** — Directly affects a provider or API the project actively uses, backed by specific file/config evidence. The connection is concrete, not speculative.
- 🟡 **MEDIUM** — Affects a provider the project uses, but the specific feature or API in question isn't central to how the project uses it.
- ⚪ **LOW** — Tangentially related. The provider is in the stack but this particular change doesn't touch anything the project depends on.
- *(omit)* — Not relevant. Don't include it at all.

**Impact if ignored** — a concrete statement of what happens to the project if the user takes no action. This must be specific to the project, not a generic category. Examples:

- "Billing webhook handler in payments/webhook.ts will reject events after June 1 when the v2 payload format becomes mandatory"
- "Paying ~$340/month more than necessary on inference costs based on current call volume in lib/ai.ts"
- "No impact — awareness only"

Write the impact as a single sentence a busy developer can scan. Ground it in the project's actual files and usage patterns, not in abstract risk categories.

## Step 4: Produce the Briefing

Format the output as a prioritized briefing. Lead with what matters most.

### Briefing Format

```
## Stack Intelligence Briefing — [Project Name]
### Scan date: [date]
### Providers scanned: [list]

---

🔴 HIGH RELEVANCE

**[Provider]: [What happened]**
[1-3 sentence summary of the development]
→ Project impact: [Specific explanation referencing actual files, configs, or architectural decisions]
→ If ignored: [Concrete statement of what happens to this project if no action is taken]
→ Recommendation: [Specific, actionable next step — based on analysis you already performed, not homework for the user. If you can check compatibility, read a migration guide, or assess the blast radius yourself, do it and report what you found.]
→ Deadline: [If applicable — deprecation date, enforcement date, or "none"]
Source: [URL] ([publication date])

---

🟡 MEDIUM RELEVANCE

**[Provider]: [What happened]**
[1-2 sentence summary]
→ Project impact: [Brief connection to specific files/configs in the project]
→ If ignored: [What happens to this project — one sentence]
→ Deadline: [If applicable, or "none"]
Source: [URL] ([publication date])

---

⚪ LOW RELEVANCE

**[Provider]: [What happened]**
[One-line summary and why it's low priority]
→ If ignored: [One sentence, typically "No impact — awareness only"]
Source: [URL] ([publication date])

---

### Nothing found for:
[List providers where no recent developments were found — this is useful signal too]
```

## Modes of Operation

### On-demand sweep (default)
User asks "what's new" or "stack intel" — run the full scan.

### Targeted check
User mentions a specific provider or announcement — skip the full scan, focus on that provider, and assess relevance to the project.

### Pre-sprint check
User says "pre-sprint check" or "what should I know before I start" — run the full scan but weight findings by urgency and likelihood of impacting active work.

## Important Principles

- **Never fabricate provider news.** If a search doesn't return clear results, say "no recent developments found" — don't hallucinate announcements.
- **Every finding must have a source.** Include the URL and publication date for every item in the briefing, at every relevance tier. If you can't link to a specific, verifiable source, don't include the finding.
- **Always ground relevance in the actual codebase.** "This matters because you use X in file Y" is useful. "This might be relevant" is not.
- **Do the work, don't assign homework.** If you can check compatibility, read a migration guide, assess whether the project is affected, or verify a version — do it yourself and report what you found. Never write "check if you're affected" when you can check. The user is reading this briefing to save time, not to get a to-do list.
- **Respect the user's time.** A briefing with 3 high-relevance findings beats one with 20 medium-relevance items. Filter aggressively.
- **The provider map is the foundation.** If Step 1 is wrong, everything downstream is wrong. When in doubt, ask the user to confirm the map.
- **Strategic context from CLAUDE.md matters.** If the project's CLAUDE.md mentions goals, constraints, or architectural decisions, factor those into relevance scoring. A finding about a Stripe API deprecation is HIGH relevance if CLAUDE.md says "we process payments via direct Stripe integration with no abstraction layer."
