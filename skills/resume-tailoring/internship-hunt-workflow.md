# Internship Hunt Workflow

Automatically discovers newly-posted internships from a community-maintained
GitHub listings repo and tailors a resume for each one that matches, using
the normal single-job / multi-job pipeline underneath.

## Trigger

This workflow runs when:
- The user says the exact phrase **"it's go time"** (case-insensitive), or
- An external scheduler invokes Claude Code with that same phrase (see
  "Scheduling the daily run" below — this skill has no clock of its own).

No job description is required from the user for this trigger — the
workflow sources its own job postings.

## Configuration

Defaults (override if the user has said otherwise in this conversation or in
a project config):

- **Source repo:** `https://github.com/SimplifyJobs/Summer2027-Internships`
  (listings live in `README.md` on the `dev` branch)
- **Age filter:** only rows whose Age column reads exactly `0d` (posted
  today)
- **Location filter:** US-based roles only (see "Location matching" below)
- **Category scope:** all categories in the README (Software Engineering,
  Product Management, Data Science/AI/ML, Quantitative Finance, Hardware
  Engineering) unless the user has said they only want specific ones

## Phase 1: Fetch Listings

1. Shallow-clone or re-pull the source repo to a scratch location:
   ```
   git clone --depth 1 https://github.com/SimplifyJobs/Summer2027-Internships.git <scratch>/summer-internships
   ```
   If already cloned from a previous run, `git -C <scratch>/summer-internships pull --depth 1` instead of re-cloning.
2. Read `README.md` from that checkout. Listings are inside `<table>` blocks
   (not markdown pipe-tables) with columns in this order: **Company, Role,
   Location, Application, Age**.
3. Parse every `<tr>` under each `## <emoji> <Category> Internship Roles`
   heading. A company name of `↳` means "same company as the row above" —
   still treat each `↳` row as its own distinct internship (different
   role/location/application/age), just inherit the company name.
4. For each row capture: `company`, `role`, `location`, `apply_url` (the
   first `href` inside the Application cell — it's a Simplify-tracked
   redirect to the real posting, that's fine), and `age`.

## Phase 2: Filter Matches

1. **Age filter:** keep only rows where `age == "0d"`.
2. **Location filter — US only.** Use judgment, not a strict regex, since
   the column is free text (e.g. `Chicago, IL`, `NYC`, `SF`, `Multiple
   Locations`, `Remote in USA`, `Vancouver, BC, Canada`, `London, UK`):
   - **Include:** a US state abbreviation/name, "USA"/"United States",
     "Remote in USA"/"Remote - US", or an unambiguous major US city given
     without a country (NYC, SF, LA, Chicago, Seattle, Austin, Boston,
     Washington DC, etc.)
   - **Exclude:** any row that names a non-US country or a non-US city/
     province (Canada, UK, India, Vancouver BC, London, Toronto, Bangalore,
     etc.), and plain "Remote" with no country qualifier (ambiguous —
     don't assume US).
   - **"Multiple Locations":** check the linked `apply_url` job posting if
     feasible; if country can't be determined, exclude and note it as
     skipped-ambiguous in the final report rather than guessing.
3. De-duplicate against resumes already generated in a prior run for the
   same `company + role + apply_url` (check output directory / library for
   a matching filename before regenerating) so re-runs don't duplicate work
   on roles that are still `0d` because the repo hasn't refreshed yet.

## Phase 3: Generate Resumes

For the surviving set of matched internships:

1. For each match, try to fetch the real job description from `apply_url`
   (follow the redirect) to get full JD text for matching. If the page
   isn't fetchable (blocked, requires login, etc.), fall back to using just
   `company + role + location` as a lightweight JD seed and proceed with
   reduced research depth — note this in the report, don't block on it.
2. **Route by volume:**
   - 1-2 matches → run the normal **single-job workflow** (Phase 0 onward
     in `SKILL.md`) for each.
   - 3+ matches → run the **multi-job batch workflow**
     (`multi-job-workflow.md`): shared library build, aggregate gap
     analysis, one shared discovery session, then per-job
     research/template/matching/generation.
3. Skip checkpoints that require back-and-forth confirmation when this
   workflow was triggered by the scheduler (no one is watching); still
   apply all matching/scoring logic, just don't block waiting on approval.
   When triggered interactively by the user typing "it's go time", normal
   checkpoints apply.
4. Output file naming: `Resume_<Company>_<Role>_<YYYY-MM-DD>.md` (plus
   DOCX/PDF per the usual Phase 4 generation), in the usual output
   location.

## Phase 4: Report

Always end with a summary, even if triggered unattended:

```
INTERNSHIP HUNT — <date>

Fetched: <N> listings from Summer2027-Internships (dev branch)
Posted today (0d): <N>
US-based matches: <N>
  ✓ Generated: <Company> — <Role> (<location>) → <output files>
  ✓ Generated: ...
  ⚠ Skipped (ambiguous location): <Company> — <Role> (<location>)
  ⚠ Skipped (already generated in a prior run): <Company> — <Role>
  ⚠ Reduced-research (JD page unreachable): <Company> — <Role>
```

## Scheduling the daily run

Claude Code (and this skill) has no built-in clock — "every day at 3pm"
has to come from an OS-level scheduler on whatever machine runs Claude Code,
which invokes it non-interactively with the trigger phrase.

**Windows (Task Scheduler), running headless at 3:00 PM daily:**

```powershell
schtasks /create /tn "InternshipHunt" /tr "claude -p \"it's go time\" --cwd C:\Users\davin\Documents\resume-tailoring-skill" /sc daily /st 15:00
```

**macOS/Linux (cron), 3:00 PM daily:**

```cron
0 15 * * * cd ~/resume-tailoring-skill && claude -p "it's go time"
```

Notes:
- `-p` runs Claude Code non-interactively and prints the result instead of
  opening the TUI — needed for an unattended scheduled run.
- Running unattended means no one is present to approve tool calls (git
  clone, web fetch, file writes). Either pre-approve the specific tools
  this workflow needs in the project's `.claude/settings.json`, or run with
  whatever your Claude Code version's non-interactive auto-approve flag is
  — check `claude -p --help` for the exact current flag name, and only
  grant the minimum needed (network fetch to the listings repo, file
  read/write under the project and resume library) rather than blanket
  bypassing all permissions.
- Verify the scheduled task once manually (`claude -p "it's go time"`) to
  confirm resume generation actually works end-to-end before trusting the
  schedule to run unattended.
