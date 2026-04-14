# Finding Provider News Sources

After discovering which providers are in the project (Step 1), the skill needs to find where each provider publishes updates. This should be done dynamically — search for each discovered provider's news sources rather than relying on a pre-built list.

## Discovery Strategy

For each provider identified in the codebase scan, find their announcement channels using this approach:

### 1. Search for the provider's changelog or blog

```
"[provider name] changelog [current year]"
"[provider name] developer blog"
"[provider name] release notes"
```

Pick the most authoritative result — typically the provider's own domain, not a third-party summary.

### 2. If the provider is an open-source library, check GitHub

Construct the releases URL from the import path or package name:
- Python package `requests` → https://github.com/psf/requests/releases
- npm package `next` → https://github.com/vercel/next.js/releases
- The repo URL is often in the package manifest (`repository` field in package.json, `Homepage` in pip metadata)

You can also run:
```bash
# For Python packages
pip show [package] 2>/dev/null | grep -i "home-page\|project-url"

# For npm packages
npm info [package] repository.url 2>/dev/null
```

### 3. For cloud/SaaS providers, look for these common URL patterns

Most providers follow predictable patterns on their own domain:
- `/blog` or `/blog/` — company blog, often includes product announcements
- `/changelog` or `/changelog/` — structured list of changes
- `/docs/release-notes` or `/release-notes` — technical release notes
- `/developers/` or `/developer/news` — developer-specific announcements
- `/what-is-new` or `/new/` — AWS-style announcement feeds

### 4. For app stores and platform gatekeepers

These require policy-focused searches, not just changelogs:
```
"[platform] developer policy update [current year]"
"[platform] review guidelines changes [current year]"
```

Platform policy changes are often reported by developer news sites before they appear in official docs.

### 5. For security advisories on any dependency

```
"[package name] CVE [current year]"
"[package name] security advisory"
```

Or check GitHub's advisory database directly:
`https://github.com/advisories?query=[package name]`

## What NOT to Do

- Do not maintain a static list of provider URLs. Providers change their blog structures, merge changelogs, and move documentation. Discover fresh URLs each time.
- Do not assume a provider has a changelog. Some providers announce only via blog posts, Twitter/X, or mailing lists. If a structured changelog doesn't exist, a blog search is the fallback.
- Do not waste searches on commodity dependencies. Only run discovery for providers classified as strategic in Step 1.

## Persistence

Provider news URLs discovered during a run are stored in `.stack-intel.json` in the project root (managed by the main skill, not this reference file). This reference file only provides the *methodology* for discovering sources — it should never itself contain a static directory of URLs.
