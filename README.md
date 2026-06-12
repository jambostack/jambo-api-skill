# jambo-api — Claude skill

A skill for **Claude Code / Claude** that teaches the agent how to work with the
[**Jambo**](https://github.com/jambostack/jambo-api) headless CMS API — reading, writing,
schema, end-user auth — with the exact request/response shapes and gotchas **verified against
the Jambo Symfony source**, not guessed.

## Why

Jambo's API has a couple of non-obvious traps that make agents silently produce broken code:

- **Flat vs nested field values.** The public content endpoint expects field values **flat**
  in the body; the studio path expects them **nested under `fields`**. Get it wrong and the
  entry is created **but every field is empty** — while still returning `201`.
- **`status` defaults to `draft`.** Forget `"status":"published"` and your content never shows
  up on the public read.
- **`per_page` is capped at 100**, schema reads need a *valid* token, `media`/`relation` are
  arrays of project UUIDs, etc.

This skill encodes all of that, plus a copy-paste reference client and an anti-bug checklist.

## What it covers

- Public content reads (anonymous when `publicApi` is on), pagination & limits
- Content writes via the recommended flat-field endpoint **and** the nested studio path
- Collection schema, token abilities (`read`/`create`/`delete`), field types
- `media`/`relation` UUID handling, end-user JWT auth
- The “static site + server proxy” security pattern, error codes, checklist

## Install

### Option A — as a personal skill (recommended)

Copy the `jambo-api/` folder into your skills directory so it’s available in every project:

```bash
cp -r jambo-api ~/.claude/skills/
```

### Option B — packaged `.skill`

Grab the `.skill` file from the [Releases](../../releases) page and install it via your
Claude Code plugin/skill installer.

## Usage

The skill triggers automatically when a task involves the Jambo API (reading/writing content,
collections, entries, fields, end-user auth, or seed scripts). In the instructions, replace the
placeholders with your values: `{API_BASE}`, `{PROJECT_UUID}`, `{collection}`, `{uuid}`.

> **Never** commit or expose an API token. Load it from the environment
> (`JAMBO_TOKEN`, `JAMBO_ADMIN_TOKEN`) or a secrets file kept out of the repo.

## License

MIT © jprud67
