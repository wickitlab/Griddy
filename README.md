<p align="center">
  <img src="assets/banner.png" alt="Griddy banner" />
</p>

**An independent, tamper-evident public archive of Malta's power outages**

Enemalta's live outage map shows only the present moment, and during the July 2026 grid crisis it visibly failed while whole localities sat in the dark. Once an outage scrolls off the map, the public record of it is gone. Griddy scrapes the public feeds and writes what they said into an append-only, hash-chained archive, so the record survives and anyone can verify that nobody edited it afterwards.

This matters because Malta's compensation scheme pays households EUR 60-110 (and businesses considerably more) for outages longer than 6 hours, and the claim form asks for documentary evidence. The archive is that evidence, held by the public instead of only by the utility.

Built in Python 3.12 with `requests` and the standard library. There is no server, database or build step: a scrape cycle fetches the feeds, diffs them against the last known state and appends hash-chained JSONL lines to files in this repository, and a static HTML dashboard is generated from the archive.

> **Status: running.** A scheduled GitHub Actions workflow scrapes the feeds hourly and commits the archive, so it keeps growing. The dashboard is published at [wickitlab.github.io/Griddy](https://wickitlab.github.io/Griddy/) and is regenerated each run.

> Not affiliated with Enemalta plc. The archive records what Enemalta's own public feeds said; it adds no claims of its own.

## Contents

- [Tech stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Getting started](#getting-started)
- [Features (built)](#features-built)
- [How a scrape cycle works](#how-a-scrape-cycle-works)
- [Data model](#data-model)
- [Dashboard](#dashboard)
- [Privacy](#privacy)
- [Security](#security)
- [What's not yet built](#whats-not-yet-built)
- [License](#license)

## Tech stack

| Layer | Technology |
| --- | --- |
| Language | Python 3.12 |
| HTTP | `requests` |
| Archive | Append-only JSONL files, SHA-256 hash chain (`hashlib`, `json`) |
| Dashboard | One self-contained HTML file with inline data and vanilla JS |
| Hosting | GitHub Pages, serving the `docs/` folder |
| Tests | pytest |
| CI | GitHub Actions: tests, security scan, Dependabot |

## Prerequisites

- Python 3.12 (what CI uses) and `pip`
- Git, to clone the repository and its archive
- Network access to Enemalta's public feeds, only if you want to run a scrape cycle yourself. Verifying the archive and generating the dashboard need no network access.

## Getting started

1. Clone the repository and install the one runtime dependency:

   ```bash
   git clone https://github.com/wickitlab/Griddy.git
   cd Griddy
   pip install -r requirements.txt
   ```

2. Verify the archive. Every line of every `data/*.jsonl` file is recomputed from genesis, and the exit code is 0 when the whole chain is intact:

   ```bash
   python -m griddy.verify
   ```

3. Optional: run one scrape cycle against the live feeds. It appends to the archive in your clone only:

   ```bash
   python -m griddy.run
   python -m griddy.verify
   ```

4. Optional: regenerate the dashboard from `data/live.jsonl`:

   ```bash
   python -m griddy.dashboard
   ```

5. Run the tests:

   ```bash
   pip install pytest
   pytest
   ```

The scrape workflow (`.github/workflows/scrape.yml`) runs this on a schedule; see [How a scrape cycle works](#how-a-scrape-cycle-works).

## Features (built)

### Archive

- Scrapes three public Enemalta sources: the live outage API, the planned outage API and the planned outage HTML page (the only source carrying locality names).
- Diffs each cycle against `state/current.json` and appends `appeared`, `updated` and `cleared` events.
- Writes a `heartbeat` record every cycle with the content hash of each raw payload, even when nothing changed. This is evidence of absence: proof of what the public feeds did and did not report at that moment.
- Chains every line with `seq`, `prev` (hash of the previous line) and `hash` (hash of the line itself), so tampering, reordering or deleting a line breaks the chain at exactly that point.
- Strips personal data before anything is archived (see [Privacy](#privacy)).
- Replaces megabytes of feeder geometry with a SHA-256 digest.

### Verification

- `python -m griddy.verify` replays every archive file from genesis and prints exactly where and why a chain breaks.
- Git history is a second, independent witness to when each record appeared. The timestamps that matter for a claim are the ones inside the JSONL records themselves.

### Dashboard and statistics

- Downtime by locality, outages per week, and a table of outages longer than the 6-hour compensation threshold with the documented household range of EUR 60-110.
- Generated as one static file, `docs/index.html`, and published through GitHub Pages.

## How a scrape cycle works

`python -m griddy.run` performs one cycle:

1. Fetch the three public sources, keeping the raw bytes so the archive can store a hash of exactly what the endpoint served.
2. Normalize each case: strip personal data, digest the geometry.
3. Diff against the last known state and append events to `data/*.jsonl`.
4. Write the heartbeat record.

In the scheduled run, a GitHub Actions cron fires hourly and loops six cycles about 10 minutes apart, verifies the chain after each, regenerates the dashboard and commits once per run. GitHub throttles high-frequency schedules, which is why the job loops internally instead of using a 10-minute cron.

More detail is in [ARCHITECTURE.md](ARCHITECTURE.md) and [HOW_IT_WORKS.md](HOW_IT_WORKS.md).

## Data model

| File | Contents |
| --- | --- |
| `data/live.jsonl` | Unplanned outage events (the compensation-relevant ones) |
| `data/planned.jsonl` | Planned works from the JSON API (case IDs, transformers, feeders) |
| `data/planned_page.jsonl` | Planned works from the public HTML page (localities, streets) |
| `data/heartbeats.jsonl` | One record per cycle: source content hashes, sizes, counts |
| `state/current.json` | The snapshot of what was visible at the last cycle |

An outage's `cleared` event marks the moment it left the public feed. Combined with its `appeared` event it gives a defensible duration estimate.

## Dashboard

`python -m griddy.dashboard` writes a self-contained `docs/index.html` (no server, no build step, no extra dependency) from the current `data/live.jsonl`. It opens directly from disk or can be served by GitHub Pages from the `docs/` folder.

The compensation figures are the ones documented in this project, not Enemalta's full official schedule. Treat the eligibility flag as a starting point for a claim, not the final amount.

## Privacy

The upstream feeds can include employee names (`InCharge`) and customer account references (`AffectedAccountNos`). Neither has archival value for outage evidence, so both are stripped in `griddy/normalize.py` before anything reaches the archive. Street and locality names are kept: they are the outage's public location, published by the utility itself.

## Security

Griddy has no server, no user accounts and no inbound attack surface. The scraper makes outbound requests only, to Enemalta's public feeds, and needs no key or credential.

Known weaknesses, stated plainly:

- Records are hash-chained but not signed, so integrity is provable but not attributable to a signing key.
- Ordering between cycles trusts the host clock. There is no external trusted timestamp.
- The archive depends on Enemalta's feeds being reachable and truthful. A feed outage or falsified upstream data shows up as a gap or an incorrect record, not as a chain-verification failure.

To report a data-integrity issue, use GitHub's private vulnerability reporting on this repository. See [SECURITY.md](SECURITY.md) for the policy.

## What's not yet built

- Signing: sign each batch so the archive is attributable as well as tamper-evident (ProofLog integration).
- A verify page: paste a record hash, confirm it is in the chain.
- Certificates: locality and date in, a verifiable outage certificate out, formatted for Enemalta's Claim for Damages application.
- A chained-archive verify view inside the dashboard (verification is command-line only today).
- Broader compensation-schedule coverage beyond the documented EUR 60-110 band.

See [ROADMAP.md](ROADMAP.md) for the order these were planned in.

## License

This project is licensed under:

- [MIT](LICENSE) - free for any use. See the license for details.
