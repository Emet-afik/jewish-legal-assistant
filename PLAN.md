# PLAN.md: building jewish-lawyer on top of Skills IL

Working notes for the build. Written after reading all four source skills in full
(SKILL.md, SKILL_HE.md, references/, scripts/, metadata.json, evidence.json,
optimization-log.json, CHANGELOG.md) plus `legal-tech/CLAUDE.md` and
`legal-tech/scripts/validate-skill.sh`. Sources were cloned to `_sources/`
(git-ignored, deleted before publishing).

## 1. How the source skills are built

| Piece | What it does in the source | What I keep / change |
|---|---|---|
| `SKILL.md` | English frontmatter (`name`, `description` with "Use when", `license: MIT`), then: Legal notice, Problem, Instructions split into numbered **Steps** (Step 0 classify, Step 1 collect facts, Step 2 compute, Step 3 draft, Step 4 send, Step 5 escalate), then Gotchas, Examples, Reference Links, Bundled Resources, Troubleshooting. | Same skeleton, but ONE skill with a router step and four tracks. Body in Hebrew. Cap 4,000 words (source cap is 5,000). |
| `SKILL_HE.md` | Full Hebrew mirror of SKILL.md. Their validator warns when the two drift (section-count parity) and the logs show drift happened. | Dropped. The body itself is Hebrew, so there is nothing to keep in sync. |
| `references/*.md` | Detail moved out of SKILL.md to stay under the word cap: appeal grounds, fine types, demand-letter template, evidence guide, rules-and-letters, domain checklists. Loaded only when SKILL.md points at them ("full list: `references/appeal-grounds.md`"). | Same progressive-disclosure pattern, one file per track: `fines.md`, `consumer.md`, `small-claims.md`, `website.md`, plus `sources.md`. SKILL.md says exactly when to open each. |
| `scripts/` | Offline stdlib Python helpers: `deadline-calculator.py` (receipt date + type -> 30/90 day dates), `filing-fee-calculator.py` (1%, round up to NIS 5, min 50), `claim_check.py` (technician tiers, cancellation windows/fees). Every SKILL.md says "run it if you can, otherwise apply the tables by hand". | Same rule: scripts are optional, the skill works without code. `deadlines.py` is new and much stronger (see section 3). |
| `metadata.json` | Bilingual `display_name`, `display_description`, `tags` (`{he, en}`), version, author, supported agents. | Same shape, version 1.0.0. |
| `evidence.json` | One record per factual claim: `claim_id`, `claim`, `source_url`, `raw_snippet` (verbatim), `fetched_at`, `appears_in`, sometimes `verified_via`. | Same idea plus a `numbers` array per claim, `verified_at`, and `wayback` for gov.il. The validator enforces "every number in SKILL.md and references/ is backed by a record". |
| `optimization-log.json` / `CHANGELOG.md` | Per-release lessons and fixes (Hebrew changelog). | CHANGELOG.md 1.0.0 lists what changed versus the sources. |
| `validate-skill.sh` | Frontmatter must parse as YAML; `name` kebab-case and equals folder; no "claude"/"anthropic" in name; description present, <=1023 chars raw, no `<>`, must contain a WHEN trigger; body <=5,000 words; no README.md in skill folder; secret scan. | `scripts/validate.sh` keeps the relevant checks (name, YAML, description length/triggers/no `<>`, license) and adds: body <=4,000 words, Hebrew trigger words present, closing line present, every number backed by evidence.json, every URL listed in sources.md with a date, every gov.il URL carries a Wayback capture. README.md is allowed because the repo root IS the skill folder (one-command install). |

## 2. Mistakes the sources already made and fixed (must not regress)

From `optimization-log.json` and `CHANGELOG.md`:

1. **Invented service + URL.** "צו הצו" / `gov.il/he/service/objection_traffic_offense` never existed (zero Wayback captures) and shipped anyway, because gov.il answers 403 to every script and a 403 looked like "real but blocked". Rule here: every gov.il URL is checked in the Wayback Machine; a URL with no capture is not cited. (Checked 2026-10-30: `drivers_inquiries_forms`, `filing_a_small_claim`, `filing_a_complaint_to_fair_trade_authority`, `fees_3`, `consent-2026` have 200 captures. `small_claims_courts_guide`, which the source cites, has **no** capture, so I do not cite it.)
2. **Word cap.** fines-fighter hit 5,000 words and the validator failed; they trimmed by pointing to references. Here the cap is 4,000 and detail lives in references/templates from the start.
3. **"Fine doubles after 30 days"** is wrong. Criminal track: surcharge under s.229(b) CPL + s.67 Penal Law after the 90-day payment date. Administrative track: 30% under s.10(b)(1). Never "doubles".
4. **One wrong digit on the plate does not void a ticket automatically.** Immaterial clerical errors are curable; only a material misidentification wins. Scored "medium", never "strong".
5. **Seatbelt points: driver 6, passenger 4** (the old value 2 was wrong in three files).
6. **Nat"z is not a parking ticket.** Signs 501/503, Reg. 22(a), 500 NIS, administrative track for offences from 8.2.2026: hasagah then arar, no request to be tried.
7. **No "convert to warning" for municipal tickets**, and none on the administrative track. Only a discretionary practice on the criminal track (s.229(c)).
8. **Two tracks, decided by the OFFENCE date** (s.48 transitional rule of the Administrative Traffic Violations Law): administrative = hasagah 30 / arar 90; criminal = bakasha le-bitul 30 / bakasha le-hishafet 90. Stage 1 8.2.2026 (100-500 NIS, excluding Reg. 54), stage 2 8.8.2026 (750-1,500 NIS and Reg. 54).
9. Release discipline lesson: a fix must reach body, references AND scripts in the same release (they shipped a stale "+50%" in a script once). Here the validator scans references and templates too.
10. small-claims: fee rounds UP to a multiple of NIS 5 (reg. 6(a)(3)); no "property location" venue; appeal clock 30 days from judgment; 20-day execution warning, not "30-day compliance".
11. consumer: "2 hours late = 300" is wrong (300 only 2h BEYOND the 2h window, 600 at 3h beyond); 4-month right only door-to-door or distance-with-conversation; fee cap is the LOWER of 5% / NIS 100.

## 3. What I verified fresh (2026-10-30) and what it changes

Primary texts pulled raw from he.wikisource.org and grepped verbatim:

- **CPL s.229(a)(1):** the 30-day bakasha le-bitul window becomes **90 days when the owner seeks to prove who drove (Traffic Ordinance s.27B)**. Admin law s.8(a) has the same 90. So "other driver" = 90 days on both tracks.
- **"Traffic offence" includes municipal by-laws** (Ordinance s.1), so the 90-day other-driver window also covers municipal parking.
- **CPL s.225A(a1)(1) and admin law s.6(b)(1): the 4-month camera limit counts until the notice was SENT** ("טרם נשלחו"), not received. The source skill and the brief both say "served". deadlines.py therefore takes an optional `--sent` date; without it the flag says "check the mailing date printed on the notice", and it never says "void".
- **Mail presumption:** registered mail is deemed served 15 days after sending unless the recipient proves otherwise (CPL regs 44A; admin law s.5(a)(3)).
- **Payment date / surcharge:** criminal track pay within 90 days of service (s.229(a)); admin track within 90 days (s.9(a)), or 30 days after a rejected hasagah (s.8(c)) / final arar decision (s.19(d)). The MAFNA service page (Wayback 2026-03-13) says verbatim that a request "אינה דוחה את זמן תשלום הקנס". Shown in the action card.
- **Payment on the administrative track** makes the decision final (s.9(b)) but an arar is still possible within its window (s.12; Kol Zchut, updated 10.8.2026). So the "don't pay to be safe" warning stays first and absolute for the criminal track (s.229(h): admission + conviction, every route closes, includes ALL municipal parking tickets), and is worded precisely for the administrative track. No overclaiming.
- **Interpretation Law s.10:** the day of the event is not counted; a period whose last day is a rest day ends on the next day. deadlines.py flags Saturdays and recommends filing earlier; holidays are flagged as a manual check.
- **Technician law s.18A(c)-(f)** verbatim: six covered categories only; a one-off paid call-out is not one of them. The statute counts from "השעה שתואמה"; with a window (e.g. 8:00-12:00) the safe count is from the end of the window, so 15:30 gives nothing certain. Never promise 600.
- **Spam:** Communications Law s.30A(b) express prior written consent; s.30A(c) existing-customer exception; s.30A(j)(1) up to NIS 1,000 per message without proof of damage.
- **Website law:** CPL s.14C(a) (name, ID/company no., address; price and payment terms; cancellation right), s.14C(f) (price includes delivery), s.17A/17D (total price only, incl. VAT, Israeli currency only), Special Sale reg. 6(1) 1983 (show pre-sale price; 21 consecutive days), accessibility regs 35A (IS 5568, level AA), 35E (accessibility statement in a prominent place), 35F(g) (exempt: osek patur or average turnover up to NIS 100,000; the NIS 300,000 tier the source cites expired 26.10.2020), 35A(d) (60 days to fix after notice); Privacy Law s.11 (notice at collection); PPA consent opinion 25.2.2026 (active opt-in for direct marketing; third-party ad tracking questioned). "Reject all as prominent as accept all" has **no Israeli source**: shown as ⚠️ best practice, never as ❌ law.
- Checked for changes since August 2026: no change found to the figures used (the call-recording amendment applies from 2027 and is out of scope).

## 4. What I improve over the sources

1. One entry point for four domains, one install, Hebrew from the first message.
2. Photo intake: Claude reads the ticket fields itself, shows one confirmation block, asks only for what is missing (actual receipt date, who drove).
3. Explicit score per argument (חזקה / בינונית / חלשה), one-line reason, the evidence to get NOW; losing arguments filtered out with a reason.
4. Action card with exact Gregorian dates (not "30 days"), computed by `deadlines.py` or by hand with the same rules.
5. "Don't pay to be safe" warning before everything, not on page 3.
6. Website audit: table with ✅/❌/⚠️, the section of law, what is missing, how to fix; missing documents drafted from templates.
7. Letters as separate template files so SKILL.md stays short.
8. Real tests: 8 scenarios run through `claude -p`, automated checks + manual review, and `scripts/validate.sh`.
9. Corrections to the sources found while verifying: 4-month limit counts to sending; accessibility NIS 300,000 tier expired; `small_claims_courts_guide` unverifiable; e-commerce "return shipping at buyer's expense" and "opened hygiene products" claims are not in the distance-sale statute and are not carried over.

## 5. Build order (commit after each)

1. PLAN.md + .gitignore.
2. Repo skeleton, SKILL.md, references, templates, metadata, LICENSE, evidence.json, sources.md.
3. scripts: deadlines.py, small_claims_fee.py, site_check.py, validate.sh.
4. tests/scenarios (8) + runner + checker; run through `claude -p`; fix until all pass.
5. README.md, CHANGELOG.md, delete `_sources/`, personal-data sweep, final summary.
