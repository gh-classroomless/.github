# Troubleshooting

Every entry here is something that actually happened, not a hypothetical. Find your symptom.

> **⚠️ First, the trap that wastes the most time.** The API will tell you everything is fine when it
> is not. `actions/permissions.enabled` reads `true` on a repo whose fork workflows are still
> unapproved **and** on a repo with the Issues tab switched off. It is blind to both. **The only proof
> that this system works is a real issue that gets a real reply.** Two schools sat dead for weeks
> while the API said they were healthy.

---

## A student submits a form and nothing happens

No comment, no repo, the issue just stays open. Work down this list in order.

**1. Did a workflow run at all?**
`<org>/classroom` → **Actions** tab.

- **A banner asking you to approve workflows** → this is it. A fork ships with Actions disabled;
  press the green *"I understand my workflows, go ahead and enable them"*. **There is no API for
  this** — it is the single most-forgotten deployment step. Do it on `classroom` *and*
  `classroom-admin`.
- **A run exists, `conclusion: action_required`, 0 jobs, no log** → GitHub refused to schedule it.
  See "action_required with 0 jobs" below.
- **A run exists and failed** → open it; the step log names the reason.
- **No run at all, no banner** → the issue title did not match. The job only fires on titles starting
  with `register:` / `request-` / `my-repos:`. A hand-typed issue will not do; use the form.

**2. Is `ORG_PAT` set, and still valid?**
`<org>/classroom` → Settings → Secrets → Actions. Everything cross-repo runs on it: reading the
roster, creating the repo, inviting, writing to admin. **Classic PATs expire** — an expired one fails
in confusing ways. Re-create it (scopes: `repo`, `workflow`, `read:org`) and set it again.

**3. Is `config.json` there, with that course in it?**
Missing file → the bot replies "Setup error: this repo has no config.json yet". Missing course →
"Unknown course". Both are answered, so if you got *silence* the problem is above this line.

---

## "You're not registered yet" — but they did register

The roster is keyed by **lowercased GitHub username**, and registration writes
`classroom-admin/roster/<username>.json`. Check that file exists.

- **It does not exist** → their register issue failed. Did they use the right `JOIN_CODE`? A wrong
  code is answered with an explicit ❌, so ask them to look at their own closed issue.
- **It exists but requests still fail** → `ORG_PAT` cannot read the admin repo. Check the token's
  scopes and that its owner can see that repo.
- **They registered under a different account** → the username is captured automatically from
  whoever opened the issue. If they switched accounts, they must register again on the new one.

---

## `action_required` with 0 jobs, no annotation, no log

The run is created and instantly "completed" without ever starting. Two causes:

**1. The fork's workflows are not approved.** Most likely. Green button, Actions tab.

**2. The workflow file contains the literal text `toJSON(sec`+`rets)`.**
GitHub refuses to schedule any workflow whose file contains that phrase — **including inside a YAML
comment**, and with no error message anywhere. It matches on the text, not the expression.

This cost a live outage on 2026-07-31: because `request.yml` serves *every* request, adding that one
line silently blocked every student, not just the feature being built. If you are tempted to look a
secret up by a name computed at run time — don't; that construct is exactly what needs the whole
secrets context. Put the values in a file in the private admin repo instead (that is how quiz access
codes work).

**Diagnosing this class of bug:** never on a live school repo. Fork the template into a throwaway,
reproduce there, bisect one line at a time.

---

## HTTP 409 during a rush

Symptom: several students submit at once; some get repos, others get a run that dies with no comment
to them, so they retry — which makes it worse.

**Cause:** a Contents-API write is a *commit*, and a commit takes a **branch-head lock**. Concurrent
writers all read the same head and the losers get 409. One-file-per-student does **not** help — the
contention is on the branch, not the file.

**Where it is already handled:**
- `request.yml` files its log record as an **issue** (no lock); `log-build` turns N racing writes into
  one commit later.
- `register.yml` does have to write the roster immediately, so it **retries on 409** — re-reading the
  sha inside the loop, with jittered backoff, and it only retries on a head collision so a real error
  (bad token, missing repo) surfaces at once instead of hiding behind four "busy" messages.

**If you add a workflow that writes to admin:** do not commit from the student path. File an issue and
drain it, or retry with jitter. And never "fix" it by putting every run in one global `concurrency`
group — GitHub keeps only **one** pending run per group and cancels the rest, so you would silently
drop students.

---

## The bot answered, but the repo is wrong or missing

- **"template … was not found" / "is not a template repository"** → the value in `config.json` must be
  a repo **in your org** with the *Template repository* box checked. Fix the flag or the name.
- **Repo created, student cannot see it** → they have not accepted the invitation. It is in their
  email and at github.com/notifications; the confirmation comment links it too. Re-requesting is safe
  and returns the same repo.
- **"You already have this repo"** → intended. Requests are idempotent; nothing new was created.

---

## Reports show old data

Run **log-build** first. Reports read `log/*.json`, and `request.yml` only files *issues* — until
log-build drains them, a report cannot see today's provisioning. This is the single most common
"the dashboard is wrong" cause.

If a record is missing even after log-build, look for a still-open `log:` issue in the admin repo: a
closed one was already recorded, and `log-build` skips a record that already exists.

---

## A student got a quiz repo without the code

Check *when*. If the repo predates the moment you gated the quiz, the code cannot retroactively take
it back — they already have the questions.

Also note: the gate is checked **only when creating** a repo. A student who already holds it is never
asked again (by design — quizzes are multi-attempt and they come back for the link). So to actually
close it on someone, the repo has to be deleted; then their next request needs the code.

If you delete such a repo, also close its pending `log:` issue in the admin repo **before** log-build
runs, or the log will record a provisioning for a repo that no longer exists — and the later, real
request will be skipped as a duplicate, leaving the wrong timestamp on record forever.

---

## A code fix does not reach a school

`classroom` is a fork, so fixes arrive by merging the template ("Sync fork", or
`git pull https://github.com/gh-classroomless/classroom-template.git main`).

- **Merge conflict on `request.yml`** → almost always because the same fix was once written *directly*
  on the school **and** separately on the template. Two commits, same intent, different SHAs: git
  cannot see them as one change, so they collide forever and the school also misses every later
  refinement of those lines. Resolve by taking the template version, once — after that the histories
  are joined and future merges are clean.
- **The moral:** if you ever hand-edit a school's workflow in an emergency, **backport it to the
  template in the same session.** Skipping that is what creates the conflict above.
- **`classroom-admin` is not a fork** (a private template cannot be forked, so it is generated once).
  It shares no history and nothing syncs into it — copy changed workflow files over by hand from
  `gh-classroomless/classroom-admin-template`.

---

## Nothing works and the repo looks brand new

Run the checklist in order; each one is fatal on its own and none of them reports itself:

```
Issues tab ON?          Settings → Features → Issues   (a FORK ships with it OFF)
Workflows approved?     Actions tab → green button     (no API; both repos)
ORG_PAT present?        classroom → Settings → Secrets
JOIN_CODE present?      classroom → Settings → Secrets
config.json present?    classroom root
ORG_REPO_TOKEN?         classroom-admin → Settings → Secrets   (for repo-report)
```

Then smoke-test it yourself: register with your own join code, request one assignment, and confirm the
bot answers within about a minute. Do that **before** a single student sees a link.
