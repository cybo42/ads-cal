# ADR-0001: Agenda source and ingestion pipeline

- **Status:** Accepted (spike, 2026-10-09)
- **Context:** PRD FR-DATA-1, risk R1

## Findings

`https://rsvp.withgoogle.com/events/android-dev-summit-bayview/agenda` returns an ~12 KB HTML shell. The shell is a Google Boq/Wiz app, `<rsvp-connected-guest-client>`, and contains no agenda data. The page sends `robots: noindex` and `Cache-Control: no-store`. The site has no robots.txt. A headless Chrome capture shows that the agenda loads through `batchexecute` RPCs on the same origin:

| rpcid | Payload | Size |
|---|---|---|
| `l7E3Ad` | engagement → site lookup | 0.2 KB |
| **`NdS7Dc`** | **whole site: pages, sessions, speakers, tags** | **142 KB** |
| `UsWHFd` | agenda page config (hero, track filter list) | 1.2 KB |
| `cTekQc` | activity/telemetry | 0.3 KB |

The page also opens a `signaler-pa.googleapis.com/punctual` channel, which looks like live push. We can't use it.

### Reproducible request (no cookies, no login)

```bash
curl -sS --compressed -X POST \
  'https://rsvp.withgoogle.com/_/Rsvpguest/data/batchexecute?rpcids=NdS7Dc&source-path=%2Fevents%2Fandroid-dev-summit-bayview%2Fagenda&hl=en-US' \
  -H 'Content-Type: application/x-www-form-urlencoded;charset=UTF-8' \
  -H 'X-Same-Domain: 1' \
  -H 'x-goog-ext-63847900-jspb: ["<WEB_API_KEY>"]' \
  --data-urlencode 'f.req=[[["NdS7Dc","[\"engagements/android-dev-summit-bayview\"]",null,"generic"]]]'
```

Without the `x-goog-ext-…-jspb` header the server returns error 7 ("Project for LOAS role … does not exist"). The key is the site's public web key and comes from the JS bundle. Keep it in a GitHub Actions variable rather than in code, and treat it as something that can rotate. The headless fallback reads the current key from the browser's request headers.

### Envelope

The response starts with `)]}'`, followed by length-prefixed chunks. Find the chunk `["wrb.fr","NdS7Dc","<inner JSON string>",…]` and `JSON.parse` element `[2]`.

### Field map (inner payload, positional; verified 2026-10-09)

| Path | Meaning |
|---|---|
| `[0]` | site meta; `[0][6]` = `[epochSec, nanos]` last modified |
| `[5][]` | **sessions** (49) |
| `[6][]` | **speakers** (84; 81 referenced) |
| `[7][]` | **tags** = tracks (4) |

Session `s`:

| Idx | Field | Notes |
|---|---|---|
| 0 | resource name `events/{e}/sessions/{uuid}` | stable ID |
| 2 | title | |
| 3 | description | HTML |
| 4 | slug | e.g. `ai-session-006`, `coffee-break`, `keynote` |
| 5 / 6 | start / end `[epochSec]` | |
| 7 | IANA time zone | `America/Los_Angeles` |
| 14 | speaker resource names | null for breaks |
| 29 | session code | `ADS26_035`; null for non-sessions |
| 30 | extra note | rare |
| 37 | track names (free text) | **inconsistent case; don't key on it** |
| 40 | tag resource names | use these for tracks |

Speaker `p`: 0 id · 2 first name · 15 last name · 5 title · 6 company · 7 alt photo URL · 8 X/Twitter · 9 website · 10 LinkedIn · 16[6] photo URL · 17 slug. The source has **no bio**.

Tag `t`: 0 id · 1 name.

The source has no room, level, or format fields. Derive the format from the slug and title (`keynote`, `*-workshop-*`, `[AMA]`, `hackathon`, `*break*`/`lunch`/`doors-open`/`happy-hour` → BREAK).

### Event facts

Wed 28 – Thu 29 Oct 2026, Google Bay View campus, America/Los_Angeles. The agenda has 4 parallel tracks: AI experiences, Android XR, Apps foundation, and Tools & performance.

## Decision

1. **Only our CI touches the Google RPC.** `:tools:agenda-sync` (JVM) runs on a cron in GitHub Actions. It fetches directly and falls back to a headless-Chrome network capture of the same `NdS7Dc` response. Then it parses, validates, and publishes a normalized `manifest.json` + `agenda.json` + WebP thumbnails to GitHub Pages. It commits raw snapshots only when the content changes.
2. **Apps only read our published JSON**, plus a copy bundled at build time. The app contains no Google RPC knowledge and no borrowed API key.
3. **Change detection** uses our own `contentHash` in `manifest.json` (the source has no ETag) and the Pages ETag for conditional GETs.

## Alternatives rejected

- **App calls the RPC directly.** Brittle, ships a third-party key, and a format change breaks every install until a store release.
- **HTML scraping.** There is no server-rendered content to scrape.
- **Headless browser as the primary path.** Heavier and slower than the RPC, and needed only as a fallback.
- **Manual JSON only.** Kept as the `agenda.override.json` emergency path, not the primary source.

## Consequences

- We need a GitHub Pages site and a scheduled workflow. Both are free at this scale.
- The parser has a single index-map file plus golden tests against `0001-samples/`.
- Terms of use are still unconfirmed. Low volume (≤ 96 calls/day at peak) and asking the organizers for an official feed reduce the risk.
