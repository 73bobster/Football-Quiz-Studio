# Quiz Composer

A single-page web app that generates football trivia quizzes on the fly from
a sourced, categorized fact database. This is the "Quiz Composer" component
described in the platform's [architecture spec](#background) — the piece
that takes criteria (club, category, era, question count) and assembles a
quiz from the question bank, as opposed to the Fact Miner (which finds and
scores facts) or the Question Forge (which turns facts into question
variants).

**[Live demo →](#)** *(enable GitHub Pages for this repo — Settings → Pages →
Deploy from branch `main` / root — then this link will work.)*

## How it works

- `index.html` is the entire app: no build step, no framework, no server.
  It's a static file that talks directly to a Supabase Postgres database
  over its public REST API.
- The Supabase URL and anon (public) API key are hardcoded in the file.
  This is intentional and safe: the anon key is meant to be exposed in
  client-side code. What actually protects the data is
  [Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security)
  (RLS) policies on the database itself:
  - Reference tables (clubs, countries, competitions, categories, people,
    seasons, transfers, etc.) are readable by anyone.
  - `questions` and `question_options` are only readable where
    `status = 'published'`. Draft/unreviewed questions are invisible to
    this app no matter what the client code does.
  - The underlying `facts` and `sources` tables — including provenance,
    credibility, and human-verification metadata — have **no public read
    policy at all**. The composer only ever sees finished questions, never
    the raw fact-curation data behind them.
- On load, the app queries the database for which clubs, categories, and
  decades actually have published questions right now, and builds the
  filter dropdowns from that — so as more clubs and facts get added, the
  UI updates itself with no code changes.
- "Generate Quiz" pulls all published questions matching the chosen
  filters, groups them by their underlying `fact_id` so a quiz never
  serves two phrasings of the same fact, and (optionally) avoids
  fact IDs this browser has already been quizzed on, remembered in
  `localStorage`.

## Running it locally

There's nothing to install. Either:

- Open `index.html` directly in a browser, or
- Serve the folder with any static file server, e.g. `python3 -m http.server`
  and visit `http://localhost:8000`.

## Deploying

Because it's a static file, any static host works: GitHub Pages, Netlify,
Vercel, Cloudflare Pages, or just the raw file. GitHub Pages is the
zero-config option for this repo — see the link above.

## Current data (Phase 1 — pilot)

The database currently holds Rangers FC facts and questions only, seeded
from two sources:
1. A legacy hardcoded quiz (95 questions) imported as-is, flagged
   `source_credibility: unverified`.
2. A worked example (Jorg Albertz's 1996 transfer, 35 questions across 11
   underlying facts) added with full provenance and
   `is_human_verified: true`.

All 130 current questions were bulk-set to `status = 'published'` so this
app has something to serve during development. **None of them have cleared
the two-independent-source "corroborated" bar** described in the platform
spec — they're single-source or unverified. Before treating this as a
real public-facing product, revisit which questions should actually be
publishly servable, per the spec's quality-gate section.

## Background

This app is one piece of a larger architecture: Fact Miner → Question
Forge → **Quiz Composer** → Quiz Front-End(s). See the project's
`trivia-platform-spec-v2.md` for the full design, including source
credibility tiers, the fact/question schema, and the multi-club rollout
plan.

## Extending

To point this at a different Supabase project (e.g. a fresh
multi-club database), change `SUPABASE_URL` and `SUPABASE_ANON_KEY` near
the top of the `<script type="module">` block in `index.html`. Everything
else — filters, quiz generation, scoring — is schema-driven and needs no
other changes as long as the table/column names match.
