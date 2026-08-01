# Operating it through a term

Deployment is a one-time thing ([org page](https://github.com/gh-classroomless)). This is the rest of
the year: what you touch, when, and what you must never touch.

**The one rule:** you edit **`config.json`** (public repo) and occasionally **`quiz-codes.json`**
(private admin repo). Everything else — the request forms, the roster, the reports — is generated.
If you find yourself hand-editing a generated file, stop; the next run will overwrite it.

---

## The rhythm of a term

| When | What | Where |
|---|---|---|
| Before the term | set `semester`, add the term's assignments | `config.json` |
| Each new assignment | add one line | `config.json` |
| Each quiz/exam | add it **with an access code** | `config.json` + `quiz-codes.json` |
| Before you look at any report | run **log-build** | admin → Actions |
| When you want the roster | run **roster** | admin → Actions |
| When you want pass/fail | run **repo-report** | admin → Actions |
| End of term | archive, then reset for the next one | admin |

---

## Add a course

```bash
ops/config.sh <ORG> add-course cs120 su26
```
Or edit `config.json` on the web:
```jsonc
"cs120": { "semester": "su26", "assignments": { } }
```
On push, the **gen-forms** Action rebuilds `request-cs120.yml` — a new form appears in the students'
New-issue chooser. Nothing else to do.

`semester` only feeds the repo name (`cs120-su26-A01-alice`). Change it at the start of each term.

## Add an assignment

```bash
ops/config.sh <ORG> add-assignment cs120 A01 CS120A01Starter
```
```jsonc
"A01": "CS120A01Starter"
```
The value is **the template repo in your org** that students get a copy of. It must exist and be
marked a **template repository** (Settings → check "Template repository"), or requests fail with a
setup error naming it.

The key (`A01`) is what students pick in the dropdown and what lands in the repo name. Keep it short
and stable — it is also what your grading tooling matches on.

## Add a quiz or exam (with an access code)

A window alone does **not** protect a quiz: while it is open, any registered student can request the
repo and read the questions days before sitting it. Use an access code.

```bash
ops/config.sh <ORG> add-quiz cs120 Q1 CS120Q1Starter 'CS120Q1-8xk2'
```

That one command does three things, in this order:
1. writes the code to **`classroom-admin/quiz-codes.json`** (private) — code first, so there is never a
   window where the quiz is marked gated with no code on file;
2. marks the assignment in `config.json`: `{"template": "CS120Q1Starter", "requires_code": true}` —
   **the flag only, never the code** (that repo is public);
3. regenerates the forms, so the course's request form grows an **Access code** box.

**Then put the same code inside the Canvas quiz** — in a question, not the description, so a student
only sees it once they actually start the attempt. That is the whole point: the release trigger
becomes the attempt, not the clock.

What students experience:
- right code → repo created, as usual;
- wrong or blank → ❌ "the code is inside the quiz on Canvas", **no repo created**, and they can
  retry with a new issue (no lockout — quizzes are multi-attempt);
- a student who **already has** that repo is never asked for the code (they come back for the link);
- a regular assignment **ignores** the box entirely, blank or filled.

⚠️ The code is plaintext in a private repo, and unlike a GitHub secret it is **not masked in logs** —
never echo it in a workflow. And once a student has the repo, deleting the code does not take it back.

## Time-gate an assignment (optional, combinable)

```jsonc
"A05": { "template": "CS120A05Starter",
         "available_from":  "2026-09-01T00:00:00-07:00",
         "available_until": "2026-09-08T23:59:59-07:00" }
```
Outside the window the request is rejected with the open/close time. Useful for "not yet" more than
for exam secrecy — for that, use the code.

## Remove an assignment

```bash
ops/config.sh <ORG> remove-assignment cs120 A01
```
It disappears from the dropdown. **Existing student repos are untouched** — removing the entry only
stops new requests.

---

## Reports (admin repo, all manual)

**Run `log-build` first.** `request.yml` files each provisioning record as an *issue* rather than a
commit (concurrent commits to one branch collide with HTTP 409 and students get dropped silently).
`log-build` drains those issues into `log/*.json` in a single commit. The reports read `log/`, so
without it they show stale data.

```
admin → Actions → log-build    → Run workflow     (drains the queue)
admin → Actions → roster       → Run workflow → course: cs120
admin → Actions → repo-report  → Run workflow → course: cs120
```

- **roster** → `roster.md` + `reports/cs120-roster.md`: who is in the course, each name linking to
  that student's repos. Membership comes from **having requested a repo** — registration alone carries
  no course, so a student who registered and never requested appears nowhere.
- **repo-report** → `REPORT.md` + `reports/cs120-<assignment>.md`: autograder pass/fail and score per
  repo, read from each repo's latest push run.

Both are per-course on purpose; running all courses is slow and you rarely need it.

There is also a **nightly cron** (log-build 05:00, repo-report 05:20 PT). If you looked at a report in
the morning and it seems current, that is why — but after a busy afternoon, run log-build yourself.

---

## End of term

1. Run **log-build** one last time so nothing is stuck in the issue queue.
2. Archive: move `log/*.json` into `log/archive/<term>/` (`ops/archive-term.sh` if you have the ops
   tooling). Past-term reports are then produced by passing `semester` to roster/repo-report — they
   write `roster-<term>.md` / `repo-report-<term>.md` and never overwrite the live pages.
3. Bump `semester` in `config.json` and swap in the new term's assignments.
4. Leave `roster/` alone — registration is per person, not per term, and returning students should not
   have to register again.

---

## Things that will bite you

- **Never hand-edit `request-*.yml`.** Generated from `config.json`; your edit vanishes on the next
  push. Change the config instead.
- **Never make the admin repo public**, and never *fork* the admin template (a fork of a public repo
  is public by default). Generate it private. It holds names, emails and exam codes.
- **Never put a code, token, or student email in `config.json`** — that repo is public.
- **`ORG_PAT` expires.** When it does, requests fail in ways that look like something else entirely.
  Put the expiry in your calendar.
- **A removed assignment does not remove repos**, and a deleted repo *can* be re-created by the same
  student re-requesting (with the code, if it is gated).
- **Reports lie if you skip log-build.** They are only as fresh as `log/`.
