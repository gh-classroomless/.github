# Reference — the whole system on one page

## Repos

| Repo | Visibility | Made by | Holds | Updated by |
|---|---|---|---|---|
| `gh-classroomless/classroom-template` | public, template | — | the code everyone forks | maintainer |
| `gh-classroomless/classroom-admin-template` | public, template | — | the admin workflows | maintainer |
| `gh-classroomless/.github` | public | — | this documentation + the org landing page | maintainer |
| `<org>/classroom` | **public** | **fork** | forms, workflows, `config.json` — **no PII** | "Sync fork" / merge |
| `<org>/classroom-admin` | **private** | **template-generate** | `roster/`, `log/`, `quiz-codes.json`, reports — **all the PII** | copy files by hand |
| `<org>/<course>-<term>-<asmt>-<user>` | private | created per request | one student's work | the student |

⛔ **Generate the admin repo, never fork it** — a fork of a public repo is public by default, and that
repo holds student names, emails and exam codes.

## Files you edit

| File | Repo | What |
|---|---|---|
| `config.json` | classroom (public) | courses → semester + assignments. **The only routine edit.** |
| `quiz-codes.json` | classroom-admin (private) | `{"<course>": {"<assignment>": "<code>"}}` — exam access codes |

## Files that are generated — do not hand-edit

| File | Written by |
|---|---|
| `.github/ISSUE_TEMPLATE/request-<course>.yml` | `gen-forms` Action, from `config.json` |
| `roster/<user>.json` | `register.yml` |
| `log/<repo>.json` | `log-build` (draining `log:` issues filed by `request.yml`) |
| `roster.md`, `reports/<course>-roster.md` | `roster` workflow |
| `README.md`, `repo-report.md`, `reports/<course>-<asmt>.md` | `repo-report` workflow |
| `archive/<term>/` (`log/`, `reports/`, `roster.md`, `repo-report.md`) | `archive-term` workflow — a finished term, frozen |

## Workflows

| Workflow | Repo | Trigger | Does |
|---|---|---|---|
| `register.yml` | classroom | issue opened, title `register:` | check `JOIN_CODE` → write roster (retries on 409) → redact PII → close |
| `request.yml` | classroom | issue opened, title `request-` | must be registered → check access code if gated → create repo from template → invite → file a `log:` issue → comment + close |
| `myrepos.yml` | classroom | issue opened, title `my-repos:` | read `log/` → comment a table of that student's repos |
| `gen-forms.yml` | classroom | **push** to `config.json` | regenerate `request-*.yml` |
| `log-build.yml` | admin | manual + nightly 05:00 PT | drain `log:` issues into `log/*.json` in one commit |
| `roster.yml` | admin | manual | build the per-course roster |
| `repo-report.yml` | admin | manual + nightly 05:20 PT | build pass/fail + score per repo |
| `archive-term.yml` | admin | manual, one input: the finished term | refuse while `config.json` still has a course on it → log-build → roster → repo-report → move the term into `archive/<term>/` |

`roster` and `repo-report` read only the CURRENT term (`log/`) and take no term input; a finished
term is read from `archive/<term>/`. All admin workflows skip the template repo itself.

`gen-forms` is the only `push`-triggered workflow, and it is loop-safe: it commits with
`GITHUB_TOKEN` (which cannot re-trigger workflows) and writes a different path than it watches.

## Secrets

| Secret | Repo | For |
|---|---|---|
| `JOIN_CODE` | classroom | the class code students type to register |
| `ORG_PAT` | classroom | classic PAT (`repo`, `workflow`, `read:org`) — create repos, invite, read/write admin. **Expires; set a reminder.** |
| `ORG_REPO_TOKEN` | classroom-admin | lets `repo-report` read the student repos' run results |

Never put any of these in `config.json` — that repo is public.

## Student entry points

```
Register: https://github.com/<org>/classroom/issues/new?template=01-register.yml&join_code=<CODE>
Request:  https://github.com/<org>/classroom/issues/new?template=request-<course>.yml
My repos: https://github.com/<org>/classroom/issues/new?template=03-myrepos.yml
```
The **title prefix** decides which workflow reacts — that is why the forms, not blank issues.

## config.json shapes

```jsonc
{
  "instructor_username": "your-login",
  "courses": {
    "cs120": {
      "semester": "su26",
      "assignments": {
        "A01": "CS120A01Starter",                          // plain: always available
        "A05": { "template": "CS120A05Starter",            // time-gated
                 "available_from":  "2026-09-01T00:00:00-07:00",
                 "available_until": "2026-09-08T23:59:59-07:00" },
        "Q1":  { "template": "CS120Q1Starter",             // code-gated (code lives in the admin repo)
                 "requires_code": true }
      },
      "student_repo_pattern": "{course}-{semester}-{assignment}-{username}"   // optional; this is the default
    }
  }
}
```
Always keep `{course}` in the pattern — a literal course name collides when you copy a block.

## Ops scripts (maintainer's machine, never shipped into a school repo)

| Script | What |
|---|---|
| `ops/deploy.sh <org>` | fork `classroom`, turn Issues on, generate `classroom-admin`, print the next steps |
| `ops/config.sh <org> …` | `add-course` / `add-assignment` / `add-quiz` / `remove-assignment` / `list` |
| `ops/propagate.sh <orgs…>` | merge the template's code into each school's `classroom` |
| `ops/propagate-admin.sh [--file <name>] <orgs…>` | overwrite the admin workflow files (no shared history, so no merge), then re-read and verify each |

## Facts worth memorising

- A **fork** starts with **Issues OFF** and **Actions unapproved**. The first is scriptable; the second
  is a click with no API.
- `actions/permissions.enabled` is `true` in both broken states. It proves nothing.
- A Contents-API write is a commit; a commit locks the branch; concurrent writers get **409**. Per-file
  separation does not help.
- `GITHUB_TOKEN` commits do not re-trigger workflows. `ORG_PAT` commits **do**.
- A workflow file containing the text `toJSON(sec`+`rets)` is **never scheduled** — even in a comment.
- In workflows use REST (`gh api`), not `gh <cmd> --json` (that is GraphQL, with a much smaller quota
  shared with your own CLI).
- Requests are idempotent: re-requesting returns the existing repo and creates nothing.
- Course membership comes from **requesting a repo**, not from registering — registration carries no
  course.
