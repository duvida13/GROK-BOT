# Daily rope-access digest: process spec (authoritative)

The daily routine reads this file FIRST, then `context/ledger.csv`, then `context/employers.md` and `context/learnings.md`.
Candidate: Luis Fernandes, PT citizen (EU rights), IRATA L1, GWO BST + Blade Repair, BOSIET, rigging/scaffolding. English, PT, ES usable.
Goal: every item in a digest is (a) genuinely open today, (b) not a repeat Luis has already seen, (c) ranked by how realistic it is for him.

---

## A) Ledger and repost rules

### A1. Ledger file
`context/ledger.csv`, one row per unique job, never deleted. Header:
`key,company,title,country,first_seen,last_seen,last_listed,times_listed,status,last_page_date,deadline,notes_flag`
(Optional extra columns once the routine runs: `last_checked`, `last_http`, `posting_id`, `source`.)

| Field | Meaning |
|---|---|
| key | canonical URL (A2). Primary key. |
| first_seen | date first found by us |
| last_seen | last run where the job was checked and still live |
| last_listed | last digest where it was shown as an item |
| times_listed | number of digests that showed it as an item |
| status | live / closed / unverified / expired-deadline (A3) |
| last_page_date | date printed on the ad or board card (not our run date) |
| deadline | ISO date from ad or structured data, else blank |
| notes_flag | e.g. `board re-dates` (ANETVA, net-empregos, publi24), `aging`, `multi-country`, `campaign` |

The seed file was built from the 21 day files 2026-09-11 to 2026-10-11 (246 unique jobs). Seed rows with `unknown-not-in-latest` have not been re-checked yet: the first runs must verify them (A6) and set closed/live.

### A2. Key canonicalization
1. Lowercase the host, drop `www.`, drop the scheme.
2. Strip fragments (`#...`), tracking params (`utm_*`, `fbclid`, `gclid`, `ref`, `source`, `trk*`, `trackingId`, `refId`, `campaign`, `mc_*`), and any trailing slash.
3. Keep path and meaningful query params (e.g. `?id=123`).
4. If the URL is a generic board or careers page shared by several jobs (careers home, `/vacatures/`, `/jobs`, a board listing), key = canonical URL + `#` + normalized company + `-` + normalized title (lowercase, non-alphanumerics to `-`).
5. Same job on several boards (employer page, Indeed, Jooble, Net-Empregos, LinkedIn, etc.) = ONE row. Prefer the employer's own page as key. Record the aggregator URL in a `source` column or in the digest note only.
6. A multi-country or multi-site campaign (e.g. a 5-position ad, or one employer posting the same ad for several cities) = ONE lead, one row. Several genuinely different postings with different posting ids stay separate rows.

### A3. Lifecycle states
- **live**: passed all liveness gates (B) on today's fetch.
- **closed**: a check returned 404/410, redirect to a generic page, a closed phrase, or the ad is gone from its board listing.
- **unverified**: could not be checked (blocked, timeout, 403/Cloudflare, JS-only page). Never shown as a job; counted in the footer only.
- **expired-deadline**: visible or structured deadline is before today. Treat like closed.

### A4. List-only-if rule
A row appears as an item in today's digest only if one of these is true:
1. **NEW**: key not in ledger and passes gates.
2. **REPOSTED**: ledger status was closed, expired-deadline or absent from a check, and the job passes the gates today. Also if the posting id or real posting date is newer than `last_listed` by more than 14 days, or the ad changed materially (new vacancy count, new location, new employment terms, new IRATA requirement). Label it `REPOSTED` and say what changed.
3. **DEADLINE REMINDER**: live, has a deadline, and it is 7 days away or 2 days away (show once at each point; mark in the digest line).
Everything else that is live is NOT listed. It is only counted in one footer line: `Still open from earlier days: N`.

Anti-noise rules:
- A board that re-dates cards daily (ANETVA, net-empregos, publi24, romjob, jobbland) never triggers REPOSTED on a date change alone. Set `notes_flag = board re-dates` and use only structural changes (see rule 2) or a closed-then-live transition.
- Evergreen CV-intake pages (open hire, no vacancy id) are listed once when first found and then only if their text materially changes.
- A job closed for a short time and back live within 14 days still counts as REPOSTED if our check saw it closed; otherwise it stays quiet.

### A5. Run-time updates
Each run, after verifying:
- update `last_seen` and `last_checked` for every row checked and live;
- set `last_listed` and increment `times_listed` for rows shown;
- flip status live <-> closed/expired-deadline/unverified;
- append new rows (first_seen = today);
- never delete rows; closed rows stay as history so reposts can be recognised.
Commit `context/ledger.csv` in the same run as the day file (re-fetch sha first; verify size afterwards).

### A6. Cheap re-verification schedule
- Every run: re-check ALL rows with `last_seen` within the last 7 days (HTTP GET with a short timeout, follow redirects, apply gates B1-B4 only; no browsing).
- Rows older than 7 days and still live/unverified: re-check weekly on a rotating basis (1/7 of them per day).
- Closed rows: re-check every 7 days (this is how reposts are detected). Abandon a closed row after 60 days of being closed (keep the row, stop polling).
- Sunday: full pass over every row (health check, see C6).
This is what catches closures between runs, so do not skip it when time is short; skip new-discovery depth first instead.

---

## B) Liveness gates
An item may be called Live only if ALL gates pass on today's fetch. Record the result in the ledger (`last_checked`, `last_http`).

**B1. HTTP.** Final status 2xx after redirects. 404, 410, 403, 429, 5xx, timeouts are not live (403/Cloudflare/captcha = unverified, not live).

**B2. Right page.** Compare final URL with the original:
- redirected to a home page, a generic careers/jobs/search listing, a login wall, or a different path = FAIL;
- soft-404 detection: tiny body (< ~2 KB of visible text), `<title>` unrelated to the job, a 'search results' / 'no results' / 'page not found' template, a listing with zero items.

**B3. Content match.** The page contains key tokens of the job title and the company name (at least 2 distinctive title words, e.g. 'cordiste', 'Industriekletterer', 'rope access', plus company or domain name).

**B4. No closed phrases.** Scan visible text (case-insensitive) for any of:
- EN: `no longer available`, `no longer accepting applications`, `position has been filled`, `job has expired`, `this job is closed`, `vacancy closed`, `applications closed`, `posting has been removed`, `this position is no longer`
- NL: `vacature is gesloten`, `vacature is vervuld`, `deze vacature is niet meer beschikbaar`, `niet langer beschikbaar`, `vacature bestaat niet meer`
- DE: `nicht mehr verfügbar`, `Stelle ist besetzt`, `Anzeige ist abgelaufen`, `Stellenanzeige wurde deaktiviert`, `nicht mehr aktuell`, `Bewerbungsfrist abgelaufen`
- FR: `offre expirée`, `offre pourvue`, `cette offre n'est plus disponible`, `poste pourvu`, `offre n'existe plus`, `annonce expirée`
- ES: `oferta no disponible`, `oferta cerrada`, `oferta finalizada`, `ya no está disponible`, `proceso cerrado`, `oferta caducada`
- PT: `vaga encerrada`, `oferta expirada`, `anúncio expirado`, `já não está disponível`, `candidaturas encerradas`, `vaga preenchida`
- IT: `annuncio scaduto`, `offerta scaduta`, `posizione chiusa`, `annuncio non più disponibile`, `candidature chiuse`
- PL: `oferta wygasła`, `oferta nieaktualna`, `ogłoszenie wygasło`, `rekrutacja zakończona`, `ogłoszenie nie jest już dostępne`
- SV: `tjänsten är tillsatt`, `annonsen är avslutad`, `tjänsten finns inte längre`, `ansökningstiden har gått ut`, `jobbet är inte längre tillgängligt`
- NO: `stillingen er besatt`, `stillingen er utløpt`, `annonsen er utløpt`, `ikke lenger tilgjengelig`, `søknadsfristen er utløpt`
- DA: `stillingen er besat`, `stillingen er ikke længere ledig`, `opslaget er udløbet`, `ansøgningsfristen er udløbet`
- CZ: `nabídka již není aktuální`, `inzerát byl smazán`, `pozice je obsazena`, `nabídka vypršela`, `pozice již není dostupná`
- RO: `anunțul a expirat`, `postul a fost ocupat`, `anunțul nu mai este disponibil`, `anunț expirat`, `poziție ocupată`
- LV/FI (seen in this project): `vakance nav aktīva`, `paikka on täytetty`, `hakuaika on päättynyt`
Phrase match inside navigation, cookie text or 'similar jobs' blocks does not count; check the main job container where possible.

**B5. Structured data and deadline.**
- JSON-LD `JobPosting.validThrough` in the past = FAIL. Missing validThrough is fine.
- ATS flags: Teamtailor `status: closed`, Recruitee `status: closed` / offer missing from `/api/offers`, Workable job missing from the account widget feed, Lever/Greenhouse/SmartRecruiters 404 or 'unpublished', Personio job missing from `/xml`.
- Visible deadline (`deadline`, `apply by`, `solliciteer vóór`, `Bewerbungsschluss`, `date limite`, `fecha límite`, `prazo`, `scadenza`, `termin składania`, `sista ansökningsdag`, `søknadsfrist`) earlier than today = expired-deadline.

**B6. Apply route.** An apply button, form, email address or phone number is present and works (mailto/email ok; a dead apply link = FAIL).

**B7. Source-specific rules.**
- Aggregators (Indeed, Jooble, Net-Empregos, volnamista.cz, francetravail, joblum, jobbland, publi24, olx): the detail page itself must load and still show the ad as open (not 'expired' or moved to an archive). Where possible trace to the employer page and use that URL as key if it is live.
- LinkedIn: counts as Live only if the public guest job page (`linkedin.com/jobs/view/<id>`) loads and does NOT say 'No longer accepting applications'. LinkedIn posts (`/posts/...`), recruiter posts and anything behind a login wall are NOT live; put them in the watch list in `context/` only.
- Cloudflare/captcha/403 pages: unverified, never live, say so in the 'sources blocked' footer.
- Board cards that re-date daily do not prove the job is open; verify the detail page.

**B8. Stale-age rule.** If the ad's own date (or posting id date) is older than 60 days and has no refresh, flag `aging`: it may be listed once as lower priority, never in Top picks.

**Result handling.** Failing items are written to the ledger as closed/unverified/expired-deadline. Items that were listed in yesterday's digest and now fail go in a short **Closed since last run** section (title, company, reason). Never present a failed item as live.

---

## C) Search technique upgrades

### C1. Prefer machine-readable employer sources (most reliable, cheapest to diff)
- Teamtailor: `/jobs.json` or `/jobs.rss`; Recruitee: `/api/offers`; Workable: public widget API (`/api/v1/widget/accounts/<slug>`); Lever: `api.lever.co/v0/postings/<co>?mode=json`; Greenhouse: `boards-api.greenhouse.io/v1/boards/<co>/jobs`; SmartRecruiters: `api.smartrecruiters.com/v1/companies/<co>/postings`; Personio: `<co>.jobs.personio.de/xml`; Join, BambooHR and Softgarden feeds where exposed.
- `sitemap.xml` / `sitemap_jobs.xml` of careers sites (new URL = new job; missing URL = closed).
- Use these first; fall back to HTML only when no feed exists. Note the feed URL in `context/employers.md`.

### C2. Diff-based monitoring
For each known careers page in `context/employers.md`, fetch, extract job links, and diff against ledger keys. New link not in ledger = candidate; ledger key missing from page = closed candidate (confirm with B1-B4).

### C3. Specialist boards (check every run, report which ones worked)
irata.org/jobs, ttrinternational.com employment, emploi-cordiste.fr, francetravail (cordiste), ANETVA (bolsa), Net-Empregos, volnamista.cz, finn.no, jobindex.dk, jobs.ch / jobscout24.ch, jobbland / jobbsafari (SE), ejobs.ro / publi24 / romjob, olx.pl, applavoro / jobijoba (IT), vakanser.se, careerstructure, and Indeed/Jooble when they are not returning 403. Keep the per-board checklist in `context/` (not in the digest).

### C4. Query matrix and discovery
- Multilingual query matrix: EN `rope access technician`, `IRATA`; PT `acesso por cordas`, `alpinista industrial`, `rapel`; ES `trabajos verticales`, `técnico IRATA`; FR `cordiste`; DE `Industriekletterer`, `Seilzugangstechniker`, `FISAT`; IT `lavori in fune`, `operatore su fune`, `rocciatore`; NL `rope access`, `touwtechnicus`, `bladspecialist`; NO `tilkomstteknikk`; SV `reptekniker`, `reparbetare`; DA `erhvervsklatrer`, `rebteknik`; PL `alpinista przemysłowy`; RO `alpinist utilitar`; CZ `výškové práce`, `lanař`; LV `industriālais alpīnists`.
- Company discovery: Google-style queries such as `"rope access" careers site:<tld>`, `"IRATA" vacature`, `"we are hiring" "rope access"`, plus IRATA member directory and wind/O&G contractor lists. New employers go to `context/employers.md` with their feed URL if any.
- Do not search only the same top results every day: rotate secondary countries (UK, NO, DK, CH, AT, CZ, RO, LV, FI, LU) so each is covered at least every 3 days.

### C5. Ranking and noise control
- Sort output so Luis sees **Top picks** first: max 8 items, only English or PT/ES-usable ads, L1-realistic or trainee-friendly, direct rope-access matches, live today, with a working apply route. Rank by: English/PT/ES > employer page > housing/flights/training offered > NL/BE/IE/PT/ES location > no extra certificate blockers (CSCS, VCA unmet is mentioned).
- Local-language trade-specific listings (mason, roofer, welder, painter with ropes, BTP cordiste) and L2/L3 or blade-only stretches go into a collapsed lower group **Local-language / stretch** inside each country section, one short line each.
- Never promote a stretch item to Top picks just because it is new.

### C6. Learning loop and weekly health check
- Log in `context/learnings.md` (one short line per run): which sources produced NEW discoveries, which queries yielded nothing, which techniques failed.
- Sunday health check (written into the Sunday digest footer and `context/learnings.md`): (1) broken or blocked sources list; (2) sources with zero yield for 14 days (demote or drop, never silently); (3) full re-verification of every ledger row; (4) ledger stats: live / closed / unverified counts, repost count, average times_listed of items shown (target: 1.0 per posting, apart from reminders).
- Be honest: list failed/blocked/skipped sources in a short footer line ('Not checked today: ...'). Never silently skip a source or claim coverage that was not done.

---

## D) Digest format spec

File: `finds/YYYY-MM-DD.md`. Concise, newest information only.

```
# Daily Rope Access Job Digest: YYYY-MM-DD

New: N | Reposted: N | Deadline reminders: N | Closed since last run: N | Still open from earlier days: N

## Top picks (max 8)
- Title | Company | Country | URL | why it fits | blockers (language, certs)

## Reposted / deadline reminders
- REPOSTED: ... what changed
- DEADLINE in 2 days / 7 days: ... date

## <Country>   (only countries with something to show)
- Title | Company | Country | URL | short note        (new items only, English / PT / ES first)
### Local-language / stretch (collapsed lower group)
- one-line items

## Closed since last run
- Title | Company | reason (404, expired, filled, redirected to listing)

## Footer
Still open from earlier days: N (not re-listed). Not checked / blocked today: ... Source notes: ...
```

Rules:
- Items listed are only those allowed by A4; everything else is just the count.
- Every URL listed passed the B gates today. LinkedIn-only and blocked sources never appear as jobs.
- Keep each item to one line. No paragraphs per job.
- Board checklists, employer tables and the LinkedIn/blocked watch list stay in `context/` (`employers.md`, `learnings.md`), not in the digest.
- Chat message to Luis: link to the digest, Top picks (max 5), any REPOSTED/deadline reminders, anything closed since last run. If nothing new, say so in one sentence.

---

## E) Run order checklist
1. Read this file, `context/ledger.csv`, `context/employers.md`, `context/learnings.md`.
2. Re-verify ledger rows per A6 (flip closed/live).
3. Discover: feeds and diffs (C1-C2), boards (C3), query matrix (C4).
4. Canonicalize, dedupe against ledger (A2), apply liveness gates (B) to every candidate.
5. Decide listing per A4; rank per C5.
6. Write `finds/YYYY-MM-DD.md` per D; update `context/ledger.csv` and `context/learnings.md`.
7. Message Luis with the link and Top picks.

## E. Closed-page hardening (added 2026-10-14 after Muehlhan IRATA 3 case)
- Closed phrases now include (any language): "This position is no longer active", "Either the position was filled, or the ad has expired", "no longer accepting", "position has been filled", "ad has expired". Check the raw HTML, not just the HTTP status: career.muehlhan.com (and similar ATS) serve closed ads with HTTP 200 plus that text.
- HTTP 404 and 410 = closed, always.
- Final re-check: immediately before pushing the day file, re-fetch every URL being listed and drop any that fail a gate.
- Morning-after sweep: each run first re-checks every URL listed in the previous day's file; anything now closed goes under `## Closed since last run` with the date it was last seen live, and the ledger row is set to closed.
- Fast-closing employers (Muehlhan and others seen closing within 24h): flag in notes as "apply same day".
