# Development Workflow

A generic, plug-and-play development workflow for product teams — especially teams where much of the implementation is done by AI agents and the human's leverage is in reviewing *intent* (issues, scenarios, PRs) rather than every line of code.

Drop this file into a new repo (or keep it as its own repo and reference it), swap in your tool names where noted, and treat it as the team contract.

Sections 1–10 describe the workflow at full maturity. Not all of it applies from day one — §11 stages the adoption from rapid prototyping through to production, and earlier stages deliberately relax some of the rules below.

---

## 1. Principles

- **Issue-driven.** No work without a ticket. The issue tracker is the single source of truth for what is being built and why.
- **Spec before code.** Behaviour is agreed in plain English (Gherkin scenarios) before implementation starts. Humans review scenarios; machines verify code against them.
- **Tests as you go.** Tests are written alongside the change, not retrofitted. A PR without tests for its logic is incomplete.
- **CI is the gatekeeper.** Nothing merges red. Every guarantee the team cares about (types, tests, migrations, spec coverage) is enforced by a CI check, not by convention.
- **Build for two orders of magnitude ahead.** If you have 100 users, design for 10,000. At 1,000, design for 100,000. Don't over-engineer beyond that horizon.
- **Small, single-theme changes.** One concern per branch, per PR, per merge. If a task grows a second theme, split it.

---

## 2. Issue-Driven Workflow

All work is tracked as issues on a single project board (GitHub Projects, Linear, Jira — the mechanics below assume GitHub but translate directly).

1. **Everything starts as an issue.** PRDs, feature requests, bugs, and chores are created as issues *first* — before any code. Label them by type (`feat`, `fix`, `chore`) and set status to **Todo** on the project board.
2. **Pick up work from the board.** When starting an issue, move it to **In Progress**. This keeps the board an honest, live picture of what's happening.
3. **Scope strictly to the issue.** Anything discovered mid-ticket that falls outside its scope — a nearby bug, a tempting refactor — gets split into a **new issue** in the **Backlog**, not folded into the current branch. This keeps PRs reviewable and history legible.
4. **Close issues only on merge.** Link the PR to the issue (e.g. `Closes #123`) so it auto-closes when the PR merges. Never close an issue manually while the code is still in flight — an open issue means unshipped work, always.

### Why this matters with AI agents

Agents are fast at producing code and bad at knowing when to stop. The issue is the containment boundary: it defines what "done" means, and the split-don't-expand rule stops a ticket from silently ballooning into an unreviewable mega-change.

---

## 3. Gherkin Acceptance Criteria

Scenarios are the spec layer between issues and implementation — a human-readable contract the implementation can't quietly bend. The human reviews plain-English scenarios instead of reviewing every line of code.

1. **Draft scenarios before implementing.** When picking up a `feat` issue (or a `fix` that changes behaviour), derive Given/When/Then scenarios from the issue's requirements *before* writing code. Post them as a comment on the issue for review.

   ```gherkin
   Scenario: Appointment is refused when the time slot is taken
     Given a hairdresser with an appointment confirmed for Tuesday at 10:00
     When a customer attempts to book that hairdresser for Tuesday at 10:00
     Then the booking is rejected with "slot unavailable"
     And the customer is shown the nearest free slots
   ```

2. **Ambiguity goes back to the issue.** Where requirements are silent or unclear on an edge case, ask on the issue — never assume silently. The answer becomes part of the spec.
3. **Scenarios are the acceptance contract.** Implementation and tests must satisfy them. Never rewrite a scenario to match what the code happens to do without flagging the change on the issue first — that's the spec bending to the bug.
4. **Critical flows get *executable* feature files.** For flows where a regression costs real money or trust (payments, bookings, subscriptions, wallets — define your own list), scenarios are not just comments: they live as `.feature` files in the repo, with step definitions wired via a BDD adapter for your E2E framework (e.g. [`playwright-bdd`](https://github.com/vitalets/playwright-bdd) for Playwright, or Cucumber for others). The BDD generator compiles feature files into E2E specs, and CI runs them alongside the plain test suite. Crucially, **the generator fails on any scenario step without a matching step definition** — an unimplemented scenario breaks CI, not silence.
5. **Everything else stays lightweight.** Non-critical flows keep plain unit/E2E tests; scenarios in the issue comment are enough there. Don't pay the feature-file tax where a regression is cheap.

### Why this matters with AI agents

The scenario comment is the checkpoint where a human confirms the agent understood the requirement — *before* an implementation exists to anchor on. And executable feature files mean the agreed behaviour is machine-verified forever after, independent of whoever (or whatever) wrote the code.

---

## 4. Testing Strategy

**Write unit tests before raising a PR.** Required for anything with logic or behaviour: API routes, hooks, utilities, components with state or interactions. Pure cosmetic changes (CSS, copy, static layout) are exempt.

The test layers, cheapest first:

| Layer | Tool (example) | Covers | When it runs |
|---|---|---|---|
| Typecheck | `tsc --noEmit` | Type errors across every package | Every PR |
| Unit | Vitest / Jest | Functions, routes, hooks, components | Every PR |
| Build + boot smoke test | `tsc` + start the compiled server | Catches runtime module errors (ESM/CJS mismatches) that typecheck misses | Every PR |
| Integration | Vitest against a real database (Docker service) | Data-layer behaviour, transactions, constraints | Every PR |
| E2E | Playwright | Full user journeys in a browser against a seeded stack | PRs touching app/API code (path-filtered) |
| BDD features | playwright-bdd / Cucumber | The executable Gherkin contract for critical flows | With the E2E suite |

Rules of thumb:

- Test at the cheapest layer that can catch the failure. Don't write an E2E test for logic a unit test covers.
- A bug fix ships with a test that fails without the fix.
- Flaky tests are fixed or deleted the week they flake — a suite people ignore is worse than no suite.

---

## 5. Git & PR Workflow

1. **Always branch; never commit to the default branch.** One branch per piece of work: `<type>/<description>` (e.g. `fix/auth-bug`, `feat/booking-flow`).
   - **Parallel agents must use isolated worktrees.** Agents sharing one working directory will fight over the checked-out branch and lose edits. One worktree per concurrent agent.
2. **One theme per PR.** Multiple concerns → multiple branches and PRs.
3. **PRs carry a Summary and a Test plan.** The summary says what changed and why (linking the issue); the test plan says how it was verified.
   - **New dependencies are named and justified in the summary.** Agents love solving problems by adding packages. Prefer the standard library and dependencies already in the tree; an unexplained new package is a legitimate reason to pause the merge loop for a human look.
4. **A request for a PR authorises the full loop** — whoever raises the PR (human or agent) runs it end-to-end without pausing for permission at each stage:
   1. Raise the PR, linked to its issue.
   2. **Run a code review on the diff.** Fix critical findings and quick wins (small, contained, low-risk) directly on the branch and push. File anything larger as a follow-up issue rather than blocking the merge. Report the review and fixes as you go — visibility, not permission.
   3. **Wait for CI.** If it fails, diagnose from the job logs, fix, and push until green.
   4. **Merge** (squash, delete the branch) once CI is green and review findings are addressed.
   - **Use a merge queue once agents merge in parallel.** Two PRs that are each green against a stale default branch can be red together; the queue re-validates each merge against the true tip.
5. **Stop and check with a human when:** a review finding is real but not quickly fixable (especially security, data-integrity, or money concerns); the fix would change scope beyond the issue; or the PR contains destructive migrations. Otherwise the merge is pre-authorised by the PR request.

Squash-merging keeps the default branch history one-commit-per-issue, which makes reverts and archaeology trivial.

---

## 6. Continuous Integration

Every guarantee is a required check on PRs to the default branch. A recommended baseline:

- **Typecheck** — `tsc --noEmit` (or equivalent) per package. In a monorepo, run one job per package so failures are attributed instantly.
- **Lint + format** — a zero-warnings linter and formatter check (e.g. ESLint/Biome/oxlint + Prettier). Agents drift stylistically across sessions; one enforced style keeps diffs reviewable and removes a whole category of noise from history.
- **Unit tests** — the full unit suite, per package.
- **Build + boot smoke test** — compile the production artefact and actually start it with dummy env vars, failing if the process dies within a few seconds. This catches whole classes of runtime errors (module resolution, ESM/CJS interop, missing env validation) that neither typecheck nor unit tests see.
- **Integration tests** — run against a real database spun up as a CI service container (e.g. Postgres in Docker), with migrations applied. No mocked persistence for data-layer tests.
- **E2E tests** — seed a database, boot the API and app, run the browser suite. Path-filter these to run only when relevant packages change, and give the job a hard timeout.
- **BDD spec check** — the Gherkin compile step (`bddgen` or equivalent) runs before the E2E suite, so a scenario without step definitions fails the build.
- **Migration drift check** — fail if the ORM schema has changes not covered by a committed migration file (see §7).
- **Dependency audit** — scheduled scan of dependencies for known vulnerabilities.

Optional, as the project matures:

- **Mutation testing** (e.g. Stryker) on core business logic, to measure whether tests actually assert anything.
- **Synthetic monitoring** — a scheduled workflow that exercises production's critical paths and alerts on failure.

Practical notes:

- Pin third-party actions to a commit SHA, not a tag.
- Cache dependency installs keyed on the lockfile.
- In a monorepo, use a single root lockfile and a task runner with caching (e.g. Turborepo, Nx) so `typecheck`/`test`/`build` fan out across packages in parallel and re-runs are sub-second on cache hits.

---

## 7. Database Migrations

Assumes a migration-based ORM (Prisma, Drizzle, ActiveRecord, Alembic — the rule is universal).

- **Every schema change ships with a migration file** in the same PR. Deploys run "apply pending migrations" only — a schema change without a migration file will crash production, so CI enforces the pairing with a **drift check** that fails when the schema and the migration history disagree.
- Keep the canonical schema, migration history, and seed scripts in **one shared package**; every consumer (API, workers, jobs) references it rather than holding a copy.
- **Destructive migrations** (dropping columns/tables, irreversible data rewrites) always get a human review before merge, regardless of who wrote the PR.

---

## 8. Secrets Management

- **All secrets live in the CI provider's secret store** (e.g. GitHub Actions secrets). Deploy workflows provision `.env` files on servers from those secrets at deploy time.
- **Never hand-edit env files on a server.** Manual edits are invisible, unversioned, and overwritten by the next deploy.
- To add or rotate a secret: update it in the secret store, then deploy.
- Keep a `docs/SECRETS.md` inventory: every secret's name, what it's for, and where it's used — but never its value.

---

## 9. Environments & Deployment

- **Staging mirrors production** on separate infrastructure with a fully isolated database, seeded test accounts, and provider *test-mode* keys (payments, email, push). Any branch can be deployed to staging via a manually triggered workflow — deploy feature branches to staging before merging when the change warrants a real-environment check.
- **Deploys are scheduled, not merge-triggered.** A nightly workflow deploys the default branch to production; merging alone ships nothing until then. Need it sooner? Trigger the deploy workflow manually. This decouples "merged" from "live" and batches risk into a known window.
- **Offer a scoped fast-path deploy** for isolated surfaces (e.g. a marketing site): a workflow that rebuilds and reloads only that app, skips migrations, and refuses to run if the dependency lockfile changed since the last full deploy.
- **Per-PR preview deployments** (where the platform offers them, e.g. Vercel/Netlify/Fly) are the fastest way to review UI work — the reviewer clicks a link instead of deploying a branch. Staging remains the place for full-stack checks against a real database and provider test modes.
- **Error tracking runs in every deployed environment** (e.g. Sentry). Prevention is CI's job; detection is this one — reassurance comes from finding out about breakage in minutes, not from a user's email.
- **Revert first, debug later.** When a production deploy misbehaves, roll back immediately — squash-merge history makes the revert one clean commit — then diagnose on a branch at leisure.
- Seed scripts create known test accounts per role (admin, end user, tenant/operator) so staging is usable immediately after a reset.

---

## 10. Agent Instructions

The repo's agent instructions file (`CLAUDE.md`, `AGENTS.md`, or your tool's equivalent) is the highest-leverage artefact in an AI-driven codebase: it is the one file every agent reads before touching anything, so improvements to it compound across every future session.

- **Present from the first commit**, carrying: build/test/lint commands, project structure, naming and code conventions, and the architectural decisions that aren't obvious from reading the code.
- **Reference this workflow doc from it**, so agents operate under the same contract as humans.
- **Update it whenever an agent learns something the hard way.** A gotcha hit twice is a documentation failure, not an agent failure. Encoding the lesson costs a minute and pays out on every session after.
- **Keep it short and current.** Agents trust it verbatim, so a stale instruction is worse than a missing one. Prune as aggressively as you add.

---

## 11. Staged Adoption

Each piece of machinery above earns its keep only once the thing it protects exists: the migration drift check arrives with the database, staging arrives with users, scheduled deploys arrive with users you can hurt. Adopting any of it earlier is drag; later is gambling. Where an earlier stage relaxes a rule from §1–10 (committing to the default branch, merge-triggered deploys), the stage wins — the numbered sections describe the end state.

### Stage 1 — Rapid prototyping

**Goal: maximum iteration speed. The only sin is slowing down.**

The product is exploratory, the code may be throwaway, and there may be no real backend, database, or users yet.

- **Agent instructions from the first commit** (§10). A `CLAUDE.md` (or equivalent) carrying build/test commands, conventions, and architecture decisions is the highest-leverage artefact for AI-driven speed. Update it whenever an agent learns something the hard way.
- **Minimal CI now.** Typecheck + lint + unit tests. It's ~20 lines of workflow YAML, runs in under a minute, and gives agents automatic feedback without a human re-running anything. This is the one piece of later-stage machinery cheap enough to have from day one.
- **Branching is situational.** Direct commits to the default branch are fine while one agent works at a time. The moment agents run in parallel, use branches and isolated worktrees (§5.1) — not for review ceremony, but so they don't clobber each other. No PRs or branch protection required.
- **Issues are optional.** A rough todo list is enough. But start writing Gherkin-style scenarios *in prompts* when briefing an agent on a feature — it costs nothing and builds the habit that becomes the Stage 2 checkpoint.
- **Skip entirely:** branch protection, PR ceremony, integration/E2E tests, the BDD toolchain, staging, migration checks, the secrets inventory.

**Exit when:** you commit to keeping the codebase — concretely, when a real backend/database replaces mocks, or the first person who isn't on the team uses it.

### Stage 2 — Alpha

**Goal: stop breaking your own momentum. Regressions now cost re-work.**

Real backend, real database, first test users.

- **Branch protection on:** PRs only, required CI checks, squash merge, delete branch on merge. This is the single biggest ratchet — everything else hangs off it.
- **Issue-driven workflow starts for real** (§2): every piece of work is an issue, PRs link `Closes #N`, and the split-don't-expand scope rule applies.
- **Scenario comments become the human checkpoint** (§3): Given/When/Then reviewed on the issue before the agent implements. Human leverage moves from reviewing code to reviewing intent.
- **CI grows:** build + boot smoke test (now that there is a server to boot), and the **migration drift check the same week the database arrives** — far easier to adopt at migration #1 than at migration #50.
- **Error tracking now** (e.g. Sentry), not later. The moment a test user hits a bug nobody saw, detection beats prevention.
- **Deploys stay continuous and merge-triggered.** Feedback speed is still worth more than batched risk.
- **Merges are human-approved.** The agent runs the PR loop up to the merge; a human clicks the button. CI hasn't earned auto-merge trust yet.

**Exit when:** external users you can't personally apologise to, real data you can't regenerate, or money.

### Stage 3 — Beta

**Goal: protect user trust. Breakage now has a blast radius beyond the team.**

External users, real data, first revenue.

- **Define the critical flows list** and wire the **BDD toolchain** (§3.4): executable `.feature` files, generator compile step in CI. The scenarios written since Alpha become machine-enforced.
- **Integration tests against a real database in CI** (§6); **E2E suite**, path-filtered with a hard timeout.
- **Staging environment** (§9): isolated database, seeded test accounts, provider test-mode keys, deploy-any-branch workflow.
- **Secrets into the CI store + `SECRETS.md`** (§8); destructive migrations always get human review (§7).
- **Agent auto-merge unlocks here** — CI now has a track record. Add a **merge queue** if parallel agents are merging, so two individually-green PRs can't be jointly red.
- **Scheduled dependency audit** (§6).

**Exit when:** enough paying users that a bad deploy is a support incident, not an oops.

### Stage 4 — Production

**Goal: batch risk, detect fast, keep the suite honest.**

- **Scheduled deploys + manual trigger** (§9) — "merged" and "live" now decouple, batching risk into a known window. Add the scoped fast-path deploy for isolated surfaces.
- **Synthetic monitoring** exercising critical paths on a schedule.
- **Mutation testing** on core business logic, so green tests provably assert something.
- **Formalise revert-first, debug-later** — squash history makes a revert one click — and enforce the flaky-test rule: fixed or deleted the week it flakes.

---

## 12. Adoption Checklist

The full build-out, for reference — sequence it per §11 rather than doing it all up front:

- [ ] Create the project board with **Backlog / Todo / In Progress / Done** and labels `feat` / `fix` / `chore`
- [ ] Add this file to the repo (e.g. `docs/DEV_WORKFLOW.md`) and reference it from the repo's agent/contributor instructions (`CLAUDE.md`, `CONTRIBUTING.md`)
- [ ] Protect the default branch: PRs only, required CI checks, squash merge, delete branch on merge
- [ ] Set up CI: typecheck, lint, unit tests, build + boot smoke test, integration tests against a real DB, migration drift check
- [ ] Wire up error tracking (e.g. Sentry) for every deployed environment
- [ ] Wire the BDD toolchain (feature files + step definitions + generator) and add the compile step to CI
- [ ] Decide the list of **critical flows** that require executable feature files
- [ ] Create the staging environment and its deploy-any-branch workflow
- [ ] Set up the scheduled production deploy + manual trigger
- [ ] Move all secrets into the CI secret store and write the `SECRETS.md` inventory
