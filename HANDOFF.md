# Handoff — mini-vla / fresh-review-4

> The baton. Present tense only: where the work stands *now* and what the
> next session should do. Overwrite it each session — never append. History
> lives in git and rationale in commit bodies, not here. Before the task
> ends, promote anything durable (project status, repo instructions, commit
> body) and delete this file from the branch: a finished task hands
> nothing off, and merged, a leftover baton strays onto the default branch.

- **Repo:** mini-vla
- **Branch:** `fresh-review-4`
- **Worktree:** /root/git/worktrees/mini-vla/fresh-review-4
- **Last updated:** 2026-07-25 03:34 UTC · server (srv1841294)

## State

Not started. This is review round 4 — rounds 1-3 (PR #22, #23, #25) are all
merged into `main`. Round 1 ran the full generic-core catalog against the
pre-round-1 code (5 discovery agents + a `codebase-health` scan-only leg) and
fixed CI authorization gaps, the pre-push hook's stale-cache bug, `train.py`'s
budget-check enforcement, an E2E coverage gap, and zero test coverage on
`geometry.py`/`eval.py`. Round 2 adversarially re-reviewed round 1's own diff
(held up), dug into hardcoded defaults in emitted artifacts, and confirmed the
"sequential-guard gap auditing" dimension round 1 first named with two more
real instances (`scripts/release.mjs` had no branch/provenance check;
`.claude/skills/port-to-js/SKILL.md`'s Phase 6 instructed a raw `git tag` +
push bypassing `release.mjs` entirely — confirmed live via a real
version/tag mismatch on a past tag). Round 3 adversarially re-reviewed round
2's own diff (held up under empirical re-verification, not just reading),
confirmed the guard-gap dimension a third and fourth time (`dispatch-portfolio`
in `.github/workflows/e2e.yml` had the same provenance gap `release.mjs` used
to have — fixed), fixed a broken `pyproject.toml` console-script entry point
(verified by building and installing the wheel fresh), and fixed three
docs-vs-code drift instances (a third stale "30s budget" comment in
`mini_vla/config.py`/`js/src/config.ts` that both prior rounds had missed, a
stale `placeSide` docstring, a duplicated-instead-of-imported palette
constant). Full findings and rationale for all three rounds are in PR #22,
#23, #25's bodies — read those, not just this summary, before doing the
de-duplication pass.

Between round 3 branching and merging, a **separate, unrelated PR (#24,
`color-convergence-check`) landed on `main`** and was absorbed into round 3
via a rebase (see round 3's own PR body and this repo's `git log`) — round 3
regression-tested it (pytest/typecheck/E2E all green after the rebase) but
never applied any review dimension to its actual content, since it wasn't
round 3's own work. **This makes PR #24 the single largest block of
genuinely-unreviewed code in the repo right now**, despite `git log --stat
origin/fresh-review-3..origin/main` showing nothing changed since round 3's
branch tip — that diff is empty only because round 3's rebase already
absorbed PR #24's content into what merged; the *review* of that content
never happened. Treat it as in-scope exactly like newly-changed code, not as
already covered.

PR #24 (`mini_vla/trainer.py`, `js/src/trainer.core.ts`, `mini_vla/config.py`,
`js/src/config.ts`, `train.py`, `scripts/run_once.py`, `js/eval/main.ts`,
`js/eval/run.mjs`) adds a second convergence signal: training only reaches
"Ready" once BOTH the action loss AND a new color/language loss are
simultaneously under threshold (`ConvergeConfig.colorLoss = 0.02`, mirrored
across `config.py`/`config.ts`), fixing a bug where the demo could go "Ready"
with the color head still undertrained (it was decoding the wrong color).
The commit's own message is unusually candid about what it could NOT verify:
none of its calibration runs actually reproduced the original CI failure
signature (it's "structurally correct per the identified root cause" but not
empirically proven against the real failure), and portfolio-site's
`hero-full.spec.ts` (a separate repo) — the test that caught the original
bug — was never re-run against this fix. That second point is out of this
repo's scope to close, but is worth naming plainly in this round's report as
a known verification gap, not silently dropped.

**One concrete lead already surfaced by staging this round, worth chasing
first, not re-deriving from scratch:** `js/src/trainer.core.ts`'s convergence
loop gates advancement on a `physical` flag (`const physical =
!isNonPhysical(loss)`, around line 1040) that exists specifically to stop a
dead/lost WebGL context (which silently zeros losses) from ever satisfying
convergence. That flag is computed **only from the action loss** (`loss`) —
never from the new `colorLoss`. If a context-loss (or any other failure mode)
can zero `colorLoss` while leaving `loss` non-zero, `smoothColorLoss` could
drop under `CONVERGE_COLOR_LOSS` from zeroed values while the `physical` gate
never catches it, since it never inspects `colorLoss` at all. This may be a
non-issue if both losses always zero together under every real failure mode
(worth actually establishing, not assuming either way) — but it is exactly
the "guard patterns with blind spots" / asymmetric-guard shape this repo's
reviews keep finding, applied to a guard that didn't exist in a two-loss form
before this week. Verify whether it's a live gap or a false lead before
reporting it either way.

Given three prior rounds' coverage, this round should **not** re-sweep the
full generic-core catalog broadly against code all three rounds have already
read closely (the core model/training files' pre-PR-#24 logic, the CI
workflows' non-`dispatch-portfolio` jobs, `.githooks/pre-push`, `release.mjs`
itself, the docs/comment surfaces three rounds have already swept for drift).
Weight the round instead toward:

1. **PR #24's new code, given real adversarial depth** — the `physical`-gate
   lead above is the concrete starting point, not the whole task. Also check:
   does `js/eval/main.ts`/`js/eval/run.mjs`/`scripts/run_once.py`/`train.py`'s
   new color-loss telemetry actually get read by anything, or is any of it a
   derived-but-unused value (a field computed and logged but never acted on)?
   Is the `colorLoss = 0.02` mirrored correctly and completely between
   `config.py`/`config.ts` (both the value and the window/streak/minBatches
   reuse the commit claims)? Does `train.py`'s `_color_loss()` extraction
   (mirroring `_action_loss()`) handle every `train_on_batch` result shape
   `_action_loss()` handles, or does it diverge on an edge case (dict without
   either key, non-dict/list form, etc.)?
2. **Re-audit the live branch-protection setting.** Re-checked via `gh api
   repos/lukasmueller-dev/mini-vla/branches/main/protection` moments before
   this brief was staged: still no `required_pull_request_reviews` at all —
   unchanged from rounds 2 and 3. Re-check fresh rather than trusting this
   snapshot; report plainly either way.
3. **Re-check `perf-nightly.yml`'s run history.** Zero runs as of staging
   (`gh run list --workflow=perf-nightly.yml` — empty) — round 3 flagged this
   as expected-not-yet-fired (first scheduled Monday hadn't passed). Several
   days have now elapsed since round 3 merged; if it has fired by the time
   this round runs, actually read the result (did it go green? if not, is
   that a real regression or a CI-environment artifact?) instead of repeating
   round 3's "hasn't fired yet" framing unchanged.
4. **Round 3's other named-but-not-yet-closed items**, lower priority than 1-3
   above since they're known gaps rather than fresh discovery — worth a
   status check and, if the repo owner gives explicit go-ahead this round, an
   actual fix: `release.mjs` and `.githooks/pre-push` still have zero test
   coverage; the model/JS-port unit-test boundary still has no fast unit
   tests (round 3's PR body has concrete, scoped recommendations — a
   circular-coordinate round-trip test, a config-drift-detector test — worth
   implementing rather than re-discovering).
5. **A lighter blind pass over whatever's genuinely still unread**, weighted
   low: `mini_vla/eval.py`, `mini_vla/geometry.py`, `mini_vla/embeddings.py`,
   `js/src/rollout.ts`, `js/src/infer.ts`, `assets/`, and any doc surface not
   already swept three times. Don't manufacture findings to fill this out —
   an honest "nothing new found" is a valid, reportable result here.

Still run the full mission shape regardless of the above weighting:

1. **Discovery pass, blind** — work the generic-core dimensions (adversarial
   static read, docs-vs-code drift, test blind spots, config/permission
   safety, CI correctness, cross-file consistency, derived-but-unused
   values, guard-pattern blind spots, allowlist escapes, hardcoded
   defaults) before reading anything about prior rounds' findings — weighted
   per above, not skipped. This repo is not the toolkit that authors this
   skill, so no conditional dimensions apply; extend scope from this repo's
   own `CLAUDE.md` and `README.md` as prior rounds did.
2. **`codebase-health` scan-only leg**, after the discovery pass. Scan mode
   only: stop at its report, no approval step, no `health/*` branches or
   PRs. Prior rounds used `python.md`, `typescript.md`, `shell.md` — all
   still exist; note which this round used.
3. **De-duplicate** against all three prior rounds only now: `gh pr view 22`,
   `gh pr view 23`, `gh pr view 25` for the full list of what each fixed
   (this file's summary above is a start, not a substitute for reading the
   actual PRs). A re-found item any round already fixed is replication —
   report it separately as evidence the discovery pass works, never as a new
   finding. There is still no `PROJECT_STATUS.md` / `PROJECT_ROADMAP.md` and
   still no in-repo TODO/FIXME markers as of staging — note this rather than
   skipping the step silently.

## Next action

1. `git rebase origin/main` (never the resume verb) — this worktree was
   staged directly off the current `main` tip (which includes rounds 1-3 and
   the unrelated PR #24), so this should be a no-op unless something else
   has landed since; confirm rather than assume. This repo's `origin` remote
   resolves over SSH with no key configured in some sandboxes, which makes a
   bare `git fetch` fail silently — if `origin/main` looks stale, fetch via
   the HTTPS clone URL instead: `git fetch https://github.com/
   lukasmueller-dev/mini-vla.git main:refs/remotes/origin/main --force`.
2. Run this repo's own verification gate as the baseline before reviewing:
   `npm ci && npm run typecheck`, `pip install -e ".[dev]"` (this repo's
   system Python has no `pip` in some sandboxes — `apt-get install -y
   python3-pip python3-venv` first if so, or build a venv and install into
   that) `&& pytest`, and the Playwright E2E suite (`npx playwright install
   --with-deps chromium` then `npm run test:e2e` — `webkit-iphone` too if
   time allows, `npx playwright install --with-deps webkit` first). A
   failure here is the round's to fix or flag, not to review around. Prior
   rounds saw one known resource-contention flake class (a test that fails
   under parallel load but passes on an isolated retry, e.g.
   `demo-page.spec.ts`) — don't mistake that class for a regression, but
   don't wave away a genuinely new failure either.
3. After the rebase and baseline are green (or their failures logged),
   dispatch parallel sub-reviewers over the discovery-pass dimensions —
   independent dimensions, so fan them out rather than working the list
   serially. Weight fan-out time toward PR #24's new code (item 1 above,
   including the `physical`-gate lead) and the live branch-protection
   re-audit (item 2), since those have the least existing coverage; give
   item 5's light blind pass the least.

## Blockers

None yet — nothing has been attempted.

## Gotchas (unpromoted)

- This repo's CI is **not** opt-in per PR (unlike the toolkit repo this
  skill ships from) — `e2e.yml` triggers automatically on `push`/
  `pull_request`, so this round's own PR needs no opt-in label to get a CI
  run.
- In a network-restricted environment, this repo's `origin` remote has an
  SSH fetch URL with no key configured, so a bare `git fetch` fails
  silently. This has bitten all three prior rounds' staging in some form —
  if a newly-staged round branch doesn't contain the previous round's merge
  commit, check for exactly this before assuming something else is broken.
- Deliverable shape: a ranked findings list, each with `file:line` and a
  concrete failure scenario; label reasoned-but-unproven claims as such.
  Fixes never replace the report — this round produces a report, not
  patches, except where the owner explicitly asks (as happened for most of
  all three prior rounds' findings).
- One commit per concern if any fixes are made at all — the failure it
  prevents goes in the commit body so any single fix reverts alone.
- Get the repo owner's explicit go-ahead before any high-blast-radius
  fix — installer/uninstall safety, destructive git paths, permission or
  settings changes, or CI/workflow changes gating a real external dispatch
  (round 3 treated `dispatch-portfolio` changes this way).
- End the round's output by naming any new review-dimension class its
  findings imply. **"Sequential-guard gap auditing" is now confirmed a 4th
  time across three independent rounds (`release.mjs` twice,
  `port-to-js`'s Phase 6 once, `dispatch-portfolio` once)** — round 3
  judged it proven and recommended promoting it to
  `references/review-dimensions.md` in the `codebase-review` skill's
  toolkit repo (a separate repo, not edited as part of staging or running
  this round; that promotion is still pending as of this brief). This round
  doesn't need to re-validate the dimension itself — apply it opportunistically
  if something surfaces, but don't spend a dedicated track proving it a 5th
  time.
