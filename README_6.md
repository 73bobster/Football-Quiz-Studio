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
- Settings (gear icon, always visible in the header now — previously it only
  appeared once signed in, which was a bug) let you change the font, switch
  between dark/light backgrounds, or (with exactly one club selected in the
  filters) theme the accent color to that club's own colors — currently only
  Rangers has colors set (`clubs.primary_color` / `secondary_color`).
- The sign-in/register modal has a "Show"/"Hide" toggle on the password
  field.
- The sign-in tab has a **"Forgot password?"** link (calls
  `supabase.auth.resetPasswordForEmail`), and the app now handles the
  resulting `PASSWORD_RECOVERY` auth event with a "Set a new password"
  modal (`supabase.auth.updateUser({ password })`). Previously there was no
  way to recover a forgotten password at all — trying to "sign up" again
  with an already-registered email silently no-ops (Supabase's anti-
  enumeration behavior: it does *not* create a new account or change the
  password), which is what caused a real lockout during this project.
- Filters now have three more dimensions beyond club/category/era: a
  **Filter by: Clubs / Countries** toggle above the club pills lets you
  narrow by nationality/country as well as (or instead of) club — useful
  now that the database covers more than one national context; a
  **Difficulty** row (Easy / Medium / Hard) filters on `difficulty_tier`;
  and eras below 1950 are grouped into a single **Pre-1950s** pill instead
  of one pill per individual decade, since pre-1950s coverage is thin.
  A live **"N questions available"** count updates under the filters as you
  change any of them, and disables Generate Quiz at 0.
- Mid-quiz, a small &times; next to the score opens a confirmation prompt
  ("Exit this quiz?") before leaving — it no longer silently loses your
  place if you tap it by accident.
- The app can be added to a phone's home screen (Safari/Chrome "Add to Home
  Screen") with its own icon and name, via `manifest.json` and the
  `apple-touch-icon`/`apple-mobile-web-app-*` tags in `index.html`.
- The **"N questions available"** counter now lives at the top of the setup
  screen, right under the header, in a larger sticky panel (`position:
  sticky`) that stays visible while you scroll down through the filters.
- Club and country filtering is now a single hierarchy instead of a
  Clubs/Countries mode toggle: **Country pills come first**, and selecting
  one or more countries narrows the **Club pills** below to clubs in those
  countries (selecting a country alone already filters questions at country
  level — you don't have to also pick a club). Club labels use each club's
  short name (Rangers, Hearts, Hibernian) rather than the full legal/official
  name.
- A **Tournament** filter (Domestic / Non-domestic) filters on a new
  `is_domestic` column — see "Domestic vs. non-domestic" below.
- Non-multiple-choice ("direct recall") questions now offer a
  **"Get a hint (show multiple choice)"** button alongside "Reveal answer".
  Clicking it generates the same auto-distractor multiple-choice options
  used elsewhere and turns that question into a normal MC question for
  scoring purposes — the old "I got it right / I got it wrong"
  self-assessment step only appears if you reveal the answer outright
  instead of asking for the hint.
- Each question now shows its **difficulty badge** (Easy/Medium/Hard/Unrated)
  above the answer options, with a "Suggest a different difficulty" link
  that logs a structured note (current tier → suggested tier) to the same
  `question_reports` table used for problem reports — there's no separate
  moderation queue for these yet, they show up in `question_reports` like
  any other flag.
- Settings now has a **Custom colours** background option (separate
  background and font colour pickers, `<input type="color">`), in addition
  to Dark/Light/Club-themed, plus **Favourite club** and **favourite
  country** selectors. Your favourite club's colours become the Club-themed
  default palette when no single club is selected in the filters, and your
  favourite club/country are sorted to the top of their respective pill
  lists (alphabetical thereafter).
- The "N questions available" indicator is now a small single-line pill
  (not a big prominent panel) — still sticky at the top of setup, just
  much less visually dominant.
- The "Suggest a different difficulty" control now sits on the same line
  as the difficulty badge during play, instead of below it.
- Settings has a new **How to play** note, including that difficulty
  ratings assume you're a fan of the relevant club/country — an "Easy"
  Rangers question assumes Rangers-level familiarity, not general
  football knowledge.
- **Report a problem** is now a structured choice instead of free text
  only: "Answer is wrong", "Multiple-choice options are wrong / not in
  context", "Question or answer is inappropriate", "Question not related
  to criteria set" — plus an optional free-text field for extra detail.
  Still logs to `question_reports`.
- The **"Avoid facts I've already been quizzed on"** checkbox is gone —
  that behaviour is now always on: a quiz always prefers facts you
  haven't seen yet, and only reuses previously-served facts once every
  fact matching your filters has come up at least once.
- Settings shows your **signed-in email** (or "Not signed in") near the
  top, and a new **Your stats** section: quizzes generated, questions
  answered, and % correct, tracked in `localStorage` on this device (not
  synced across devices or accounts) with a "Reset questions/correct
  count" button. Quizzes-generated isn't reset by that button.
- The header blurb is now a short one-liner ("Select what you want, then
  create the quiz") instead of a longer description.
- There's no separate Tournament section any more — **Domestic
  Tournaments** / **Non-domestic Tournaments** are now two pills inside
  the **Categories** field itself (still mutually exclusive with each
  other, and independent of which categories are picked).
- Field labels simplified: "Countr(y/ies)" → **Countries**; "Club(s)" →
  **Clubs / Countries** (a reminder that the countries picked above
  narrow this list); "Category(ies)" → **Categories**.
- Every club/national team now has real `primary_color` /
  `secondary_color` values (previously only Rangers and a placeholder
  Hamburger SV row did) — Celtic, Hearts, Hibernian, Aberdeen, Chelsea,
  Scotland, and England were backfilled from cross-referenced kit-color
  sources (Wikipedia's stated color name plus a color-code site for the
  hex value — nobody publishes an "official" brand hex for most of these,
  so treat them as good-enough-for-UI, not pixel-perfect brand compliance;
  see the SQL in this session's changelog for exact values and sourcing
  notes). This is what makes the Settings "Club themed" background option
  and the favourite-club default palette actually show each club's own
  colors now, instead of falling back to blank/default for everyone but
  Rangers.

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
from four sources:
1. A legacy hardcoded quiz (95 questions) imported as-is, flagged
   `source_credibility: unverified`.
2. A worked example (Jorg Albertz's 1996 transfer, 35 questions across 11
   underlying facts) added with full provenance and
   `is_human_verified: true`.
3. A full Rangers managerial history (William Wilton, 1899, through the
   current manager, Derek McInnes) researched from Wikipedia and a
   fan-maintained managers archive (gersnet.co.uk), corroborated with news
   reporting for 2022 onwards: 36 managerial spells recorded as `people` /
   `affiliations` / `facts` rows (including brief caretaker spells, for a
   complete record), with 16 questions generated only for spells long
   enough to be genuine quiz material (roughly 90+ days in charge).
4. A "Club Info" batch from the main Rangers Wikipedia article: kit
   supplier history (8 suppliers, 10 date ranges, 1978–present), founding
   details, nicknames, stadium heritage facts, attendance records,
   rivalries (Celtic/Old Firm, Aberdeen, Queen's Park), and ownership
   history (Craig Whyte's 2011 takeover and 2012 administration, the 2025
   Cavenagh/49ers Enterprises stake purchase) — 20 questions in total,
   5 of them true multiple-choice using other real kit suppliers as
   distractors.
5. A 50-player biographical batch, individually researched from each
   player's own Wikipedia biography page (not the unreliable squad-list
   wikitables), spanning Alan Morton in the 1920s through Madjid
   Bougherra's 2011 departure. Adds 7 new countries (Ukraine, Italy,
   Spain, Australia, Croatia, Algeria, Romania), 45 new `people` rows, and
   50 new biographical facts/questions. Five players — Willie Waddell,
   Willie Thornton, Tommy McLean, Ian Durrant and Stuart McCall — served
   Rangers as both player and manager; rather than creating duplicate
   people, this batch added a second `affiliations` row (role='player')
   against the same `people.id` already created for their managerial
   spell. Two players (Colin Stein, Derek Johnstone, Willie Johnston,
   Trevor Steven, Craig Moore and Kyle Lafferty among them) have two
   non-contiguous spells at the club recorded as separate `affiliations`
   rows. As with the managerial batch, candidate questions were
   cross-checked against every existing question in the Players category
   first, so this batch adds no reciprocal duplicates.

6. A multi-club "starter batch" adding Celtic, Heart of Midlothian, Hibernian,
   Aberdeen, Chelsea, and the Scotland national team as first-class entities
   (`clubs` rows), each with roughly 10-13 sourced facts/questions in the
   same Club-Info style as the Rangers batch above (founding, nickname,
   stadium, honours count, a signature historic result, rivalry, and
   ownership where relevant) — 69 questions in total, researched via
   parallel web-search agents (mostly Wikipedia, plus Sky Sports/Britannica/
   official club pages for specific ownership facts). This is deliberately a
   *starter* batch, not full parity with Rangers — no manager history or
   player-by-player research was done for these six yet.
7. The England national football team, added the same way as Scotland: a
   `clubs` row with `is_national_team = true`, and a 12-fact starter batch
   (Wembley, "Three Lions" nickname, the 1966 World Cup win, Euro 2020/2024
   final defeats, the 2018 World Cup semi-final, the 1953 "Match of the
   Century" loss to Hungary, record caps/goals holders, and managers).

All 297 current questions were bulk-set to `status = 'published'` so this
app has something to serve during development. **None of them have cleared
the two-independent-source "corroborated" bar** described in the platform
spec — they're single-source or unverified. Before treating this as a
real public-facing product, revisit which questions should actually be
publicly servable, per the spec's quality-gate section.

**Duplicate-avoidance note on the Wikipedia-sourced batches:** several
facts Wikipedia surfaces about well-known managers (e.g. Bill Struth's
trophy count, John Greig's start decade, the 2020–21 unbeaten title) were
already covered by existing legacy questions from a different angle. Those
were deliberately *not* re-added as new questions — the underlying fact
was still recorded, but no second question was generated — to avoid
reintroducing the same "two questions about the same fact" problem that
was fixed earlier in the legacy 95-question set. Wikidata and Wikidata's
SPARQL endpoint were not accessible from this environment; Wikipedia's own
wikitables (managers list, players list) don't reliably extract via this
environment's fetch tool either — the managerial timeline was sourced from
a fan-maintained archive site instead, with prose-style Wikipedia articles
and news reporting used for everything else. Individual player biography
pages (prose, not tables) do fetch reliably, which is how the 50-player
batch above was researched — a curated, sourced selection across eras
rather than an exhaustive import. Wikipedia's own squad list runs to
roughly 243 Rangers players meeting its 100-appearance threshold, so this
50-player batch is a deliberately-scoped subset, not full coverage; a
further batch covering more of that list remains a good candidate for a
future session.

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

**Aberdeen/Rangers rivalry discrepancy flagged, not yet fixed:** the existing
Rangers-side Club Info fact says the Rangers-Aberdeen rivalry "intensified
after an incident at the 1979 Scottish League Cup final." Researching
Aberdeen's own batch turned up that the actual 1979-80 League Cup final was
an Aberdeen v Dundee United match (the "New Firm" derby), not a Rangers
fixture — the Rangers-Aberdeen friction is better associated with a 1980
league match incident at Ibrox. The Aberdeen researcher correctly declined
to guess and used a verified Dundee United fact instead, but the original
Rangers-side fact was left as-is rather than silently rewritten. Worth a
manual look before this one gets served much.

## Difficulty tiers

Facts and questions both have a `difficulty_tier` column (1 = easy, 2 =
medium, 3 = hard), which existed in the schema from the start but was never
populated or exposed in the app — every row was `null` and there was no way
to filter by it. This session backfilled all 285 questions with a tier via
a rule-based pass (by `predicate`, e.g. `nickname`/`founding_year`/simple
`trophy_count` → tier 1; `record`/`historic_win`/named-match trivia → tier
3; everything else, including all biographical player facts and the legacy
95-question import, → tier 2 by default), rather than a fact-by-fact manual
review. It's a reasonable first pass, not a rigorous editorial judgment call
per question — treat the boundaries as approximate, and feel free to hand-
correct individual `questions.difficulty_tier` / `facts.difficulty_tier`
values directly in the Supabase SQL editor as you notice ones that feel
mis-tiered. The app now has a Difficulty filter (Easy/Medium/Hard) using
this column.

## Domestic vs. non-domestic

Facts and questions both have an `is_domestic` boolean (default `true`).
Domestic means a local, country-specific club competition or fact;
non-domestic means an international/continental competition, or anything
belonging to a national team at all (`clubs.is_national_team = true`) — per
the rule this was built to: *international teams are all non-domestic*.
There's no `competitions` table backing this (that table exists in the
schema but isn't populated or referenced by any current fact/question), so
this is a plain column, not a foreign key into a tournament registry.

This was backfilled for all pre-existing rows with a two-step heuristic,
the same "reasonable first pass, not per-row editorial review" spirit as
the difficulty-tier backfill:
1. Every fact/question belonging to a national team (Scotland, England) was
   set `is_domestic = false` outright, regardless of predicate.
2. For club-level facts, rows whose *entire subject* is a European/
   international competition result were set `is_domestic = false` by id —
   e.g. Aberdeen's 1983 Cup Winners' Cup and Super Cup wins, Celtic's 1967
   European Cup, Chelsea's Champions League wins, Hibernian's 1955 European
   Cup run, and Rangers' European final history (1961, 1972, 2008, 2022).
   Facts that are primarily about a domestic transfer or career milestone
   but merely *mention* a player's past European exploits elsewhere (e.g.
   "signed from Juventus, where he'd won the Champions League") were left
   `is_domestic = true`, since the fact itself is a domestic signing, not a
   European result. This distinction was judged by reading each candidate
   fact's text, not purely by predicate — a handful of edge cases may still
   be mis-classified; treat the boundary the same way as difficulty tiers
   and hand-correct in the SQL editor as you notice one.

## Finding duplicate facts and questions

There's no automated duplicate detection yet — every batch so far has been
de-duplicated by hand: before generating new questions, query the existing
`questions` table for the same category/club and read through them, then
skip anything that's a reciprocal or near-verbatim restatement of an
existing fact (the underlying `fact` and `people`/`affiliations` rows still
get inserted for completeness; only the redundant *question* is skipped).
That's how all six data batches in this project have avoided repeats.

For a quick automated pass, `pg_trgm` (installed in the `extensions` schema
as part of this session's admin work) can flag likely near-duplicate
question text by trigram similarity — e.g. in the Supabase SQL editor:

```sql
select a.id, a.question_text, b.id, b.question_text,
       extensions.similarity(a.question_text, b.question_text) as sim
from questions a
join questions b on a.id < b.id
where extensions.similarity(a.question_text, b.question_text) > 0.5
order by sim desc;
```

This catches near-identical *phrasing*, not two differently-worded questions
about the same real-world fact (which is the harder, more common case) — for
that, matching on `facts.predicate` + `facts.person_id`/`club_id`/
`object_value` is more useful than text similarity:

```sql
select person_id, club_id, predicate, count(*), array_agg(fact_text)
from facts
where person_id is not null or club_id is not null
group by person_id, club_id, predicate
having count(*) > 1;
```

Neither query is wired into the app — they're things to run by hand in the
SQL editor when auditing for repeats, e.g. before a big new import.

## Admin dashboard

`admin.html` is a second static page (same folder, same Supabase project)
showing operational stats: total facts, published/draft question counts,
flagged "bad answer" reports, registered users, quizzes played, clubs,
players, and managers on record — with a club/country filter on the
content counts. It's restricted to one account (`73bobster@googlemail.com`)
via Postgres Row Level Security, not just a client-side check: new `SELECT`
policies were added on `facts`, `sources`, `fact_sources`, `questions`,
`question_reports`, `quiz_sessions`, and `quiz_session_questions` that only
match that one user id, so anyone else's anon or authenticated session gets
empty results from those tables regardless of what the page's JavaScript
does. A new `public.profiles` table (id, email, created_at) mirrors
`auth.users` via a trigger on signup, since client libraries should never
query `auth.users` directly — this is what "registered users" reads from.
Signed in as the admin account, Settings in the main app shows an "Open
admin dashboard" link.

One pre-existing, unrelated item the security advisor flags: **leaked
password protection is disabled** on this project's Auth settings (checks
new passwords against HaveIBeenPwned) — nothing this session touched, but
worth turning on in Authentication → Providers → Email in the Supabase
dashboard.

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
