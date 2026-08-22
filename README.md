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
  filter pills from that — so as more clubs and facts get added, the
  UI updates itself with no code changes.
- Club, category, and era filters are all multi-select (toggle any number
  of pills) and combine with AND logic between filter types, OR within
  each one — e.g. "Rangers or Celtic" + "Transfers" + "1990s or 2000s".
- "Generate Quiz" pulls all published questions matching the chosen
  filters, groups them by their underlying `fact_id` so a quiz never
  serves two phrasings of the same fact, and (optionally) avoids fact IDs
  already served — checked against `localStorage` for anonymous play, and
  additionally against this account's own `quiz_sessions` history when
  signed in.
- Questions that don't have stored multiple-choice options (the legacy
  "direct recall" import) can still be played as multiple-choice: enable
  "Show multiple-choice options for every question" and the app builds
  3 distractors on the fly, preferring other real answers from the same
  category. This is best-effort, not hand-curated — occasionally a
  distractor will be an odd fit.
- Every question has a "Report a problem" link during play, which logs a
  row to `question_reports` (question id, optional free-text reason, and
  the reporting user if signed in). There's no public read policy on that
  table — review reports directly in the Supabase SQL editor:
  `select * from question_reports order by created_at desc;`
- Accounts are real Supabase Auth (email/password), used only so
  "avoid repeats" can work across devices, not just one browser.
- Settings (gear icon, next to your email once signed in) let you change
  the font, switch between dark/light backgrounds, or (with exactly one
  club selected in the filters) theme the accent color to that club's
  own colors — currently only Rangers has colors set
  (`clubs.primary_color` / `secondary_color`).

## Running it locally

There's nothing to install. Either:

- Open `index.html` directly in a browser, or
- Serve the folder with any static file server, e.g. `python3 -m http.server`
  and visit `http://localhost:8000`.

## Deploying

Because it's a static file, any static host works: GitHub Pages, Netlify,
Vercel, Cloudflare Pages, or just the raw file. GitHub Pages is the
zero-config option for this repo — see the link above.

**One extra step for accounts to work:** in the Supabase dashboard for this
project, go to Authentication → URL Configuration and set the **Site URL**
(and add it under **Redirect URLs** too) to wherever this is hosted, e.g.
`https://73bobster.github.io/Football-Quiz-Studio/`. Without this, the
"confirm your email" link Supabase sends on sign-up will redirect somewhere
wrong. Email/password sign-up uses Supabase's built-in email sender, which
is rate-limited on the free tier — fine for personal use and testing, not
for real public traffic.

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
publicly servable, per the spec's quality-gate section.

**Data fix applied:** the legacy 95-question import originally stored no
link between questions about the same real-world event — e.g. "which club
did Terry Butcher join Rangers from?" and "which England captain signed
from Ipswich Town in 1986?" were two unrelated facts, so a quiz could (and
did) serve both back to back. 16 such pairs were found and merged to share
one `fact_id`, so the anti-repeat logic now actually catches them. This was
a manual, one-time review of the 95 rows, not an automated process — a
future bulk import should decompose facts by predicate up front (the way
the Jorg Albertz worked example does) rather than needing this kind of
after-the-fact cleanup.

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
