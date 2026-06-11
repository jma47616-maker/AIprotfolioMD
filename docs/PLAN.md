# PortfolioMD — Build Plan

AI-assisted portfolio for UK Internal Medicine Trainees (IMT Stage 1) plus the
continuous-feedback system requested by the TPD (the "Thakur module").
Plan only — no code yet. See `docs/architecture-sketch.png` for the
hand-drawn overview.

---

## 1. What we're building (two halves, one database)

**Half A — Trainee portfolio.** Trainees log SLEs, reflections, QIPs and
teaching, each entry tagged to IMT curriculum CiPs (clinical CiPs 1–8,
JRCPTB). A dashboard tracks progress against the ARCP Decision Aid targets
per training year (IMY1/2/3). No automated reflection generation.

**Half B — Feedback module (from Sachin Thakur's email).** Every doctor has a
private feedback folder. Anyone with the link opens a blank public form,
picks the doctor, writes free text, tags it positive/developmental, submits.
Feedback auto-collates into that doctor's folder, timestamped. The folder is
readable **only** by that trainee's assigned Educational Supervisor (ES) and
Clinical Supervisor (CS) — enforced by Postgres Row Level Security, never by
UI logic. Optionally the trainee can see anonymised summaries (per-trainee
toggle, set by admin/supervisor).

## 2. Stack

- Next.js 14 (App Router) + TypeScript + Tailwind
- Supabase: Postgres + Auth + Storage, all access control via RLS
- Vercel deployment later
- Quality gates on every task: `npm run lint` + `npx tsc --noEmit`
- **No real patient data, ever** — entries store reflections/learning points only;
  UI copy and validation will actively discourage identifiable details.

## 3. Data model (Postgres)

```
profiles              id (= auth.users.id), full_name, gmc_number?, role
                      role ∈ {trainee, educational_supervisor,
                              clinical_supervisor, admin}
                      training_year ∈ {IMY1, IMY2, IMY3} (trainees)
                      feedback_summary_visible boolean default false

supervision_assignments
                      trainee_id, es_id, cs_id, assigned_by, active
                      UNIQUE(trainee_id) WHERE active  → exactly one ES + one CS

curriculum_cips       code 1–8 + generic CiPs, title, descriptor (seed data)

portfolio_entries     id, trainee_id, kind ∈ {acat, mini_cex, cbd, opcat,
                      dops, reflection, qip, teaching_attendance,
                      teaching_delivered, msf, mcr, clinic, acute_take,
                      simulation, other}, title, body, occurred_on,
                      hours?, case_count?, attachment_path?, created_at

entry_cip_links       entry_id, cip_code, self_rating?   (many-to-many)

arcp_requirements     training_year, requirement_key, label, target_number,
                      unit, notes   ← seeded from the 2019 (2023 update)
                      Decision Aid (see §6)

feedback_submissions  id, trainee_id (FK profiles), body, tag ∈ {positive,
                      developmental}, submitted_at, submitter_hint?
                      (free-text "role/relationship", optional),
                      ip_hash (rate-limiting only, never displayed)
```

Storage bucket `attachments/` (certificates: ALS, MRCP, QIPAT uploads),
path-scoped to `trainee_id/`, same RLS philosophy via storage policies.

## 4. RLS policy matrix (the core of the build)

All tables: RLS enabled, `service_role` bypass used only by the public-form
API route (insert-only path).

| Table | trainee | ES/CS (assigned) | other supervisors | admin | anon (public) |
|---|---|---|---|---|---|
| profiles | read self; read doctor list (name only) | read their trainees | read own | all | name list via secure view only |
| supervision_assignments | read own row | read rows naming them | — | all + write | — |
| portfolio_entries | CRUD own | read their trainees' | — | read | — |
| entry_cip_links | CRUD own | read | — | read | — |
| feedback_submissions | **no read** (unless summary toggle → anonymised view only) | **read** their assigned trainee's | **no** | read | **insert via API only, never select** |

Key implementation details:

- A `is_supervisor_of(trainee_id)` SQL function (security definer) checks
  `supervision_assignments` — used in the ES/CS SELECT policies.
- The public form never talks to Postgres directly: a Next.js route handler
  validates + rate-limits, then inserts with the service key. The `anon` role
  has **zero** grants on `feedback_submissions`.
- Doctor picker uses a dedicated view (`public_doctor_directory`: name +
  id only) so the public form can't enumerate emails/roles.
- Anonymised trainee summary = a view stripping `submitted_at` precision
  (month only) and `submitter_hint`, gated on
  `profiles.feedback_summary_visible`.
- Tests: pgTAP (or SQL test script) asserting each cell of the matrix —
  especially "other supervisor cannot read", "trainee cannot read raw
  feedback", "anon cannot select".

## 5. App routes (App Router)

```
/                       landing + sign-in (Supabase Auth, magic link)
/feedback               PUBLIC form: pick doctor → text → tag → submit ✓
/dashboard              role-aware home
/portfolio              trainee: list/filter entries by kind & CiP
/portfolio/new          log entry (kind-specific fields), tag CiPs
/arcp                   ARCP dashboard: targets vs logged, RAG per year
/supervisor             ES/CS: their trainees, portfolio read-only view
/supervisor/[id]/feedback   the feedback folder (timeline, +/Δ filter)
/admin                  user roles, ES/CS assignment, summary toggles
```

## 6. ARCP requirements engine (seeded from the Decision Aid, 2023 update)

Seed `arcp_requirements` per year; dashboard counts qualifying
`portfolio_entries` against targets:

| Requirement | IMY1 | IMY2 | IMY3 | Notes captured in app |
|---|---|---|---|---|
| ES report | 1 | 1 | 1 | checklist item |
| MCR | 4 | 4 (≥3 acute take/post-take) | 4 (≥3 acute) | sub-flag on entry |
| MSF | 1 (≥12 raters incl 3 cons.) | 1 | 1 | rater count field |
| ACAT | 4 (≥5 cases each) | 4 | 4 | case_count ≥ 5 validation |
| CbD / mini-CEX | 4 | 4 | 4 → **2 OPCAT + 2 CbD/mini-CEX** (Sept '23) | kind split |
| MRCP(UK) | Part 1 | Full diploma | Full diploma | certificate upload |
| ALS | valid | valid | valid (expiry ≠ blocker per 2023 note) | expiry date |
| QIP + QIPAT | participation | **1 completed by end IMY2** (Sept '23) | leadership | QIPAT upload |
| Outpatient clinics | ~20 (indicative) | ~20 + 2 OPCATs | ~20, ~80 total | counter, "indicative" label |
| Acute take patients | 100 | 100 (CiP1 → L3) | 100, ≥500 total | counter |
| Ward care months | — | — | ≥24 months cumulative | placement log |
| Critical care | — | — | 10 weeks (≤2 blocks) | placement log |
| Geriatrics | — | — | 4 months + ≥1 geriatrician MCR | placement log |
| Simulation | 1 day incl. procedures | human factors | human factors | entry kind |
| Teaching attendance | 50h (≥20h IM/CPD) | 50h | 50h | hours sum |
| Clinical CiP levels | mostly 2 | CiP1,3,7 → 3 (critical progression) | CiP7 → 4, most → 3 | self-rating grid vs target |
| Procedures (DOPS) | skills lab | 5 procedures unsupervised (summative DOPS) | maintain | procedure checklist |

CiP target grid and procedure list stored as seed data so the self-rating
screen renders targets per year automatically.

## 7. Build phases

1. **Scaffold** — Next.js 14 + TS + Tailwind, Supabase client/server helpers,
   lint/typecheck wired. CLAUDE.md with the rules above.
2. **Schema + RLS** — migrations for all tables, policies, `is_supervisor_of`,
   seed CiPs + ARCP requirements. RLS test script. *(Biggest risk first.)*
3. **Auth + roles + admin** — magic-link auth, profile bootstrap, admin
   assignment UI (one ES + one CS per trainee, DB-enforced).
4. **Feedback module** — public form (validation, honeypot + IP-hash rate
   limit), insert API route, supervisor folder view, anonymised summary
   toggle. This is the TPD's headline ask — ship it early for the demo
   with Sachin.
5. **Portfolio logging** — entry forms per kind, CiP tagging, attachments.
6. **ARCP dashboard** — requirements engine, RAG status, CiP self-rating grid.
7. **Polish + deploy** — empty states, mobile, Vercel, seed demo accounts
   (fake doctors only).

Each phase ends with `npm run lint` + `npx tsc --noEmit` green.

## 8. Open questions (for the Sachin meeting / TPD)

- Should the public form require *any* identity (e.g. NHS email) or stay
  fully anonymous? Current plan: anonymous + optional relationship hint.
- Who flips the "trainee sees anonymised summary" toggle — admin, ES, or
  trainee opt-in with ES approval?
- Moderation: does the ES need a "flag/archive" action for abusive
  submissions before anything is shown to trainees?
- Data retention: delete feedback at end of training? (GDPR angle —
  feedback about a named doctor is their personal data.)
- Does this need to coexist with/export to the official JRCPTB ePortfolio,
  or is it a standalone supplement?
