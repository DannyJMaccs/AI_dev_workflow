# Development Workflow

A generic, plug-and-play development workflow for product teams — especially teams where much of the implementation is done by AI agents and the human's leverage is in reviewing *intent* (issues, scenarios, PRs) rather than every line of code.

Drop this file into a new repo (or keep it as its own repo and reference it), swap in your tool names where noted, and treat it as the team contract.

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
4. **A request for a PR authorises the full loop** — whoever raises the PR (human or agent) runs it end-to-end without pausing for permission at each stage:
   1. Raise the PR, linked to its issue.
   2. **Run a code review on the diff.** Fix critical findings and quick wins (small, contained, low-risk) directly on the branch and push. File anything larger as a follow-up issue rather than blocking the merge. Report the review and fixes as you go — visibility, not permission.
   3. **Wait for CI.** If it fails, diagnose from the job logs, fix, and push until green.
   4. **Merge** (squash, delete the branch) once CI is green and review findings are addressed.
5. **Stop and check with a human when:** a review finding is real but not quickly fixable (especially security, data-integrity, or money concerns); the fix would change scope beyond the issue; or the PR contains destructive migrations. Otherwise the merge is pre-authorised by the PR request.

Squash-merging keeps the default branch history one-commit-per-issue, which makes reverts and archaeology trivial.

---

## 6. Continuous Integration

Every guarantee is a required check on PRs to the default branch. A recommended baseline:

- **Typecheck** — `tsc --noEmit` (or equivalent) per package. In a monorepo, run one job per package so failures are attributed instantly.
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
- Seed scripts create known test accounts per role (admin, end user, tenant/operator) so staging is usable immediately after a reset.

---

## 10. Adoption Checklist

Bootstrapping a new project onto this workflow:

- [ ] Create the project board with **Backlog / Todo / In Progress / Done** and labels `feat` / `fix` / `chore`
- [ ] Add this file to the repo (e.g. `docs/DEV_WORKFLOW.md`) and reference it from the repo's agent/contributor instructions (`CLAUDE.md`, `CONTRIBUTING.md`)
- [ ] Protect the default branch: PRs only, required CI checks, squash merge, delete branch on merge
- [ ] Set up CI: typecheck, unit tests, build + boot smoke test, integration tests against a real DB, migration drift check
- [ ] Wire the BDD toolchain (feature files + step definitions + generator) and add the compile step to CI
- [ ] Decide the list of **critical flows** that require executable feature files
- [ ] Create the staging environment and its deploy-any-branch workflow
- [ ] Set up the scheduled production deploy + manual trigger
- [ ] Move all secrets into the CI secret store and write the `SECRETS.md` inventory
