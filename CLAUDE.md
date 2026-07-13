# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is **not** a software application. `public-apis` is a community-curated Markdown list of free public APIs. The entire product is `README.md` — a single large Markdown file containing a table per category (Animals, Finance, Machine Learning, etc.). The `scripts/` directory exists solely to validate that `README.md` is well-formed and that its links work; there is no application code to build or deploy.

Almost all contribution work is: add/edit one row in one category table in `README.md`, following the exact formatting rules enforced by `scripts/validate/format.py` and checked by CI.

## Commands

Install validation dependencies (Python, from repo root):
```bash
python -m pip install -r scripts/requirements.txt
```

Validate `README.md` Markdown/table formatting (alphabetical order, column counts, punctuation, description length, valid Auth/HTTPS/CORS values, category minimums, Index-section entries):
```bash
python scripts/validate/format.py README.md
```

Check links in `README.md` for duplicates only (fast, no network calls):
```bash
python scripts/validate/links.py README.md -odlc
```

Check links in `README.md` for duplicates **and** that every link resolves (slow — makes an HTTP request per link):
```bash
python scripts/validate/links.py README.md
```

Run the unit tests for the validation package (must `cd scripts` first):
```bash
cd scripts
python -m unittest discover tests/ --verbose
```

Run only one test module:
```bash
python -m unittest discover tests/ --verbose --pattern "test_validate_format.py"
python -m unittest discover tests/ --verbose --pattern "test_validate_links.py"
```

Run a single test case/method (standard `unittest` dotted path, from `scripts/`):
```bash
python -m unittest tests.test_validate_format.TestValidateFormat.test_some_case -v
```

## Architecture of `scripts/`

- `scripts/validate/format.py` — parses `README.md` line-by-line without a real Markdown/HTML parser: it looks for `###` category headers and `|`-delimited table rows by string prefix matching. Key regexes: `anchor_re` (category headers), `category_title_in_index_re` (Index section bullets), `link_re` (`[TITLE](URL)` entries). Enforces, per entry: 5 pipe-delimited segments (Title, Description, Auth, HTTPS, CORS), each padded with exactly one space; title not ending in "API"; description capitalized, no trailing punctuation, ≤100 chars; Auth value backtick-wrapped and one of `apiKey`/`OAuth`/`X-Mashape-Key`/`User-Agent`/`No`; HTTPS is `Yes`/`No`; CORS is `Yes`/`No`/`Unknown`. Also enforces alphabetical order within each category, a minimum of 3 entries per category, and that every `###` category has a matching bullet in the `## Index` section.
- `scripts/validate/links.py` — extracts every URL from the `## Index` section onward via regex, checks for duplicate links (ignoring trailing slash), and optionally issues a real HTTP GET to each link (with a randomized User-Agent and a Cloudflare-challenge detector, `has_cloudflare_protection`, so bot-protected sites aren't reported as broken).
- `scripts/github_pull_request.sh` — used by CI on `pull_request` events; downloads the PR's raw diff, extracts only added lines, and runs `links.py` against just those additions so unrelated pre-existing broken links in the file don't fail unrelated PRs.
- `scripts/tests/` — unit tests for the two validators above (`test_validate_format.py`, `test_validate_links.py`).

CI workflows in `.github/workflows/`:
- `test_of_validate_package.yml` — runs the `scripts/tests` unit tests on every push/PR to `master`.
- `test_of_push_and_pull.yml` — runs `format.py` on every push/PR to `master`; on PRs also runs `github_pull_request.sh` (diff-scoped link check); on direct pushes runs the full duplicate-link check.
- `validate_links.yml` — scheduled daily job that runs the full link-liveness check against all of `README.md`.

## `README.md` entry conventions (from `CONTRIBUTING.md`)

When adding or editing an API entry, follow this exact table format:
```
| [API Title](https://link-to-docs) | Description of API | Auth | HTTPS | CORS | [Run in Postman] |
```
- Keep alphabetical order within the category's table — `format.py` will fail the build otherwise.
- Each `|`-delimited column must have exactly one space padding on both sides.
- Description: capitalized first letter, no trailing punctuation, ≤100 characters.
- `Auth` must be backtick-wrapped and one of: `` `apiKey` ``, `` `OAuth` ``, `` `X-Mashape-Key` ``, `` `User-Agent` ``, or the bare word `No`.
- `HTTPS` is `Yes` or `No`. `CORS` is `Yes`, `No`, or `Unknown`.
- Entry title must not end in "API" (every entry here is already an API) and must not include the domain's TLD (e.g. "Gmail", not "Gmail.com" or "Gmail API").
- New categories need a `###` header in the body **and** a corresponding bullet added to the `## Index` section, and must have at least 3 entries.
- One API addition per pull request; PR title format `Add <Api-name> API`.
- Only list APIs with a genuinely free tier — this list is not a marketing channel for paid-only products.
- Squash all commits before opening/updating a PR.
