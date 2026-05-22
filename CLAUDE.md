# Global Project Conventions

Cross-project conventions for the apps under `~/Projects/`. This file is **authoritative for all child projects** — child `CLAUDE.md` files reference these sections rather than copying them, and add only project-specific deltas (e.g. deployed-stack outputs, milestone log, runbooks).

**This file's job:** rules a fresh agent must know to be correct in *any* project. If something only matters inside one project, it belongs in that project's `CLAUDE.md` or `docs/`. If something is user preference rather than a hard rule, it belongs in auto-memory, not here.

## Monorepo Structure

Standard layout for all TypeScript projects using npm workspaces:

```
project-name/
  apps/              # Frontend applications (Vite)
  packages/          # Shared libraries (common schemas, utilities)
  services/          # Backend services (Lambda handlers, bot clients)
  infra/             # AWS CDK infrastructure-as-code
  data/              # Canonical content/config consumed at build time
  docs/              # Architecture docs, runbooks, milestones archive
  tests/             # Cross-package test suites
  package.json       # Root: npm workspaces
  tsconfig.base.json # Shared TypeScript config
  CLAUDE.md          # Project-specific deltas (references this file)
  cdk.json           # CDK config
  .env               # API keys (gitignored)
```

**Root `package.json`:**
```json
{
  "name": "project-name",
  "private": true,
  "type": "module",
  "workspaces": ["packages/*", "apps/*", "services/*", "infra"],
  "scripts": {
    "build": "npm run build --workspaces --if-present",
    "dev": "npm run dev --workspace=apps/<main-app>",
    "deploy": "npm run build && cd infra && npx cdk deploy --require-approval never"
  }
}
```

**TypeScript config:**
- `tsconfig.base.json` at root with shared compiler options
- Each package extends it: `"extends": "../../tsconfig.base.json"`
- `composite: true` in library packages for project references
- Consumers declare: `"references": [{"path": "../../packages/common"}]`

```json
// tsconfig.base.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  }
}
```

**Package references:** `"@projectname/common": "*"` in each consumer's `package.json`. The `*` version uses npm workspace linking automatically.

## Schema-First Design

All shared types come from **Zod schemas** in `packages/common/`. No separate type definitions.

```
packages/common/
  src/
    schemas/          # Zod schema definitions (source of truth)
      index.ts        # Re-exports all schemas
    types/
      index.ts        # z.infer<typeof Schema> for all schemas
    index.ts          # Re-exports schemas + types
  package.json        # @projectname/common, depends on zod
  tsconfig.json       # composite: true
```

**Generation direction (Drizzle-backed projects):**
Drizzle defines table columns — physical shape, types, indexes, constraints. `drizzle-zod` auto-derives `*InsertSchema` / `*SelectSchema` for every table. Hand-written Zod covers tool I/O, JSONB columns, and external API payloads (places where Drizzle has nothing to say). Consumers always import Zod-derived types and use `z.infer<typeof Schema>` — never `typeof table.$inferSelect`. There is one Zod file per module under `packages/common/src/schemas/` (e.g. `vendors.ts`, `finance.ts`, `policies.ts`), exported from `index.ts`.

**Principles:**
- Schemas define validation AND types in one place
- Types exported as `z.infer<typeof Schema>` — never hand-written separately
- Validate at system boundaries (API requests, WebSocket messages, file imports)
- **Fail fast and loud at boundaries.** When reading a value whose shape is not enforced by the storage layer (JSONB columns, external API responses, file imports), validate against a Zod schema and throw on parse failure. Silent fallbacks corrupt downstream data; a loud error stops the bleed. Reserve unvalidated `raw` / `raw_payload` columns for forensic replay only — never read application logic from them.
- Use inferred types internally — no re-validation inside trusted code
- Use `z.discriminatedUnion('type', [...])` for type-safe message routing and extensible condition systems

**Example:**
```typescript
// schemas/player.ts
import { z } from 'zod';
export const PlayerSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(16),
});

// types/index.ts
import { z } from 'zod';
import { PlayerSchema } from '../schemas/player.js';
export type Player = z.infer<typeof PlayerSchema>;
```

### Migration discipline

- **Never edit a committed migration file.** Once a migration has been applied to any environment, treat its SQL as immutable. Corrections go in a new migration.
- **Commit migration SQL and `_journal.json` together.** Drizzle's runner uses the journal to decide what's applied. A SQL file without a journal entry is dead; a journal entry without SQL throws on next deploy.
- **Migrations are for schema, not seed data for new tenants.** Project-specific `INSERT … SELECT id FROM organizations LIMIT 1` style migrations run once for the original org and silently no-op for any later org. New seed data goes through a manifest + `autoProvisionOrg` flow (see Multi-client config).

## AWS Patterns

### Default Region
- All projects deploy to `us-east-1` unless otherwise specified

### CDK Infrastructure
- TypeScript CDK in `infra/`
- Single stack per project (split when complexity warrants)
- Lambda: `NodejsFunction`, Node 22.x, ARM64, 30s timeout, 256MB default
- Lambda factory construct for reusable patterns:
  ```typescript
  function createLambda(scope: Construct, id: string, props: {
    handlerFile: string;
    handlerExport?: string;
    environment?: Record<string, string>;
  }): lambdaNode.NodejsFunction
  ```

### Hosting
- S3 + CloudFront for static frontends
- Route 53 for custom domains
- OAC (Origin Access Control) for S3 privacy
- SPA: error page → index.html for client-side routing

### API
- Function URLs for simple HTTP endpoints (no API Gateway overhead)
- API Gateway WebSocket API for real-time connections
- Lambda handlers behind WebSocket: connect/disconnect/default

### Database
- DynamoDB single-table design when the data model is genuinely key-value (`pk`/`sk`, GSI for alt access patterns, TTL for cleanup, pay-per-request).
- Aurora Serverless v2 (Postgres) when the model is relational — manufacturing BOMs, inventory ledger, multi-channel order data, anything that needs joins / event-source rollups / multi-level recursion.
- Default to private isolated subnets for the DB; no public IP, no `0.0.0.0/0` ingress. Lambdas reach the DB through a VPC SG → DB SG ingress rule. Ad-hoc developer access goes through an SSM-managed bastion (cheap `t4g.nano`), not a public endpoint.

## Multi-client config

When a single codebase serves multiple deploys (one per client), client-specific values flow through a typed `ClientConfig` injected at deploy time, not hardcoded strings sprinkled across modules.

- `infra/clients/<client>.ts` exports a `ClientConfig` matching `ClientConfigSchema` in `packages/common/src/schemas/`. CDK passes it to Lambdas as `<APP>_CLIENT_CONFIG` env var (JSON-serialized).
- Per-module **policy manifests** at `packages/common/src/manifests/` declare default policy keys + values per module (manufacturing, finance, marketing, shared). `seedDefaultPolicies(orgId, modules)` is called from `autoProvisionOrg` after `seedRoles` so a lazily-provisioned org boots with the right defaults regardless of when it was created.
- **New policy seeds go in manifests, not migrations.** Migrations are for schema. The historical `INSERT … SELECT id FROM organizations LIMIT 1` style seed migrations stay as-is for the org that already exists, but they no-op for any new org — only the manifest path is canonical going forward.
- Brand-specific surfaces (system-prompt persona, vendor mapper specifics, PO email subject, etc.) take a `ClientConfig` argument; they don't read string constants.
- **Single-tenant by design is a valid choice.** If a project intentionally serves one client, say so explicitly in the project CLAUDE.md so a fresh agent doesn't flag missing `org_id` columns as a bug. The org/role/RBAC tables can still exist for intra-org RBAC without operational tables carrying `org_id`.

## Reversibility

Default to actions you can undo. When two designs satisfy a requirement, prefer the one with the smaller blast radius and a clearer rollback path. AI-driven workflows make changes faster — reversibility is what keeps that acceleration safe rather than dangerous.

This isn't theoretical risk-aversion. It's the deepest reason almost every other convention in this file works: container-isolated implement agents, additive-only migrations, soft deletes, dry-run-default scripts, event-sourced ledgers, merge-orchestrator deploy locks, hook-blocked destructive commands. The pattern across all of them is the same: never let one wrong decision compound; keep the undo button working.

### Hard rules

- **Schema migrations are additive by default.** Add new columns with a default (or backfill), let the new shape soak across a release, then drop the old shape in a separate later migration. Never combine "add column X + drop column Y" in one migration unless Y was created in the same migration. Once committed, migration SQL is immutable.
- **Destructive scripts dry-run by default.** Cleanup, prune, backfill, and migration runners require explicit `--yes` (or equivalent) to mutate state. The default invocation must be safe to run accidentally — including from a `bash <(curl ...)`-style trust-the-name mishap.
- **Soft deletes for user-visible records.** Customers, orders, vendors, materials, producibles, recipes — set `deleted_at` rather than DELETE. Joins filter on `deleted_at IS NULL`. Audit trail and accidental-deletion recovery come for free.
- **Idempotency for re-runs.** Backfills, syncs, and ingest tools must be safe to re-run. `INSERT … ON CONFLICT … DO NOTHING/UPDATE` over blind INSERT; `IS NULL` filters in UPDATEs so re-running only touches still-unfixed rows.
- **Event ledgers are append-only.** Rows are never edited or deleted. Corrections happen via counter-events. The integrity sweeper relies on this invariant.
- **Feature flags for irreversible behavior changes.** When new code replaces old behavior in production, gate it behind a config flip or DB-resident flag so it can be turned off without redeploying old code.
- **Container isolation for AI agent work.** Agent work lives on a container-agent branch until the merge orchestrator explicitly lands it. If the work is wrong, discarding the branch costs nothing. Without this, one wrong agent run can block a whole afternoon.
- **Rollback uses the same primitives as the forward path.** No separate "rollback" tooling. To undo a deployed change: `git revert <bad-sha> && deploy-landing.sh main`. Same code path, no special state machine. The forward path being safe is what makes the inverse path safe.
- **Destructive git operations require explicit user authorization in the current scope.** Force-push, hard reset on shared branches, deleting branches with unmerged work, `--no-verify` hook bypass. The hooks in each project block the routine cases; the convention here is that one-off authorization stays scoped to the one-off action — don't extrapolate.

### How to think about reversibility when designing something new

Before merging a non-trivial change, answer in your head:

1. **If this is wrong, how do I un-do it?** If the answer is "redeploy a prior SHA" — fine. If it's "manually run a SQL UPDATE on production" — design for less risk first.
2. **What's the blast radius if it's wrong?** A bad RBAC seed is a one-line revert + redeploy. A bad migration that drops a column is recoverable only from backups — that's a different class.
3. **Can a future-you (or a fresh AI agent, or a collaborator) inspect this change without breaking anything?** `git diff main...<branch>`, `git show <branch>:<file>`, dry-run flags — these are reversibility for *inspection*.
4. **If an AI agent gets it wrong, does the wrong work block other work?** Container isolation gives "yes" for file-system blast radius. The merge-orchestrator pattern gives "yes" for shared-checkout collisions.

The principle compounds: if every step is reversible, you can stack many steps confidently. If any step is irreversible, the whole sequence becomes brittle.

## Dev Loop (Agent Orchestration)

The main session is the **orchestrator** and never edits files directly. Write-capable work runs in an isolated environment; read-only work uses the `Agent` tool with `Explore` / `review` / `Plan`.

**Two isolation mechanisms exist:**

1. **Docker container isolation** (recommended for any project with frequent fan-out). A `git clone --no-hardlinks` is bind-mounted into a container as the agent's only filesystem view. The parent checkout doesn't exist in the container's mount namespace, so contamination is structurally impossible. Invoked via a project script — Bliss's `scripts/spawn-container-agent.sh` is the worked example; it fetches the agent's branch back into the parent repo after the run so the orchestrator can review the diff. See the project's `docs/dev/container-isolation.md` runbook for setup.
2. **Framework worktree isolation** (`subagent_type=implement` with `isolation: worktree` in frontmatter). The Claude Code framework creates a temporary git worktree on spawn. **Known to silently fail under contention** — the agent reports clean isolation but its commit lands on the parent's main ([anthropics/claude-code#40117](https://github.com/anthropics/claude-code/issues/40117)). A documented Bliss occurrence in May 2026 saw every defensive hook return green on manual replay while parent main was still contaminated. Use this path only when container tooling isn't set up and fan-out is rare.

### Hard rules

- **Any write task runs in isolation.** Container-isolated where the project supports it, framework worktree where it doesn't. Never use `general-purpose` for writes — no isolation, will collide with parallel agents in the main checkout.
- **Parallel write work fans out independently.** For container agents: N separate `spawn-container-agent.sh` invocations (typically one per terminal/Claude session). For framework agents: N `Agent` tool_use blocks in a single message.
- **Write-capable agents are leaf nodes.** They do not spawn other agents. The orchestrator owns parallelism decisions.
- **Write-capable agents do not merge or deploy.** They finish on their branch and return a summary. A separate merge step (often a separate orchestrator session — see below) lands the branch.
- **Review runs before merge to main, not after.** Post-merge review-fixup commits mean issues already landed on main.
- **One deploy step per session, not per task.** Deploy is a script (e.g. `deploy-landing.sh`), not an agent.
- **If any stage fails, halt and report.** Do not auto-recover. Cap fixup rounds at 2 — if review keeps rejecting after that, escalate to the user.

### Two-orchestrator model (parallel sessions)

When a project regularly runs multiple implementation sessions in parallel (each spawning its own write-capable agents), the implementation session and the merge orchestrator must be different sessions:

| Role | Allowed | Forbidden |
|---|---|---|
| **Implementation session** | spawn write-capable agents; inspect branches via `git show <branch>:<path>` and `git diff main...<branch>`; surface branch + review notes to the user | `git checkout`; `git merge`; `git add` against parent; `git push`; deploy; post-deploy cleanup |
| **Merge orchestrator** | merge to main; push; deploy; post-deploy cleanup. Serialized via a `.deploy.lock` so concurrent merge orchestrators queue rather than race | implementation work — delegate to write-capable agents instead |

Why the split: parallel implementation sessions sharing one local checkout collide on shared state (working tree, index, local main, origin push, deploy lock). Separating the merge role keeps the orchestrator pattern from breaking down under fan-out.

Single-session projects (one implementation session at a time, no parallel sessions) can fold both roles into one orchestrator without harm.

### Branch inspection without contamination

To review a candidate branch from the parent checkout:

```bash
git show <branch>:<path>                 # stream one file's content
git diff main...<branch>                 # full diff
git log --oneline main..<branch>         # commits added
```

Do NOT `git checkout <branch> -- <files>` from parent main to "see what changed" — that writes into the parent working tree and contaminates main.

### Quality over speed

When facing a non-trivial change — multi-file refactor of a load-bearing file, security-sensitive code, schema or migration design, API contract change — prefer thorough planning + phased implementation over a one-shot agent run.

1. **Spawn a Plan agent first.** Give it the constraints, the existing patterns to mirror, and explicit license to break the work into as many sub-phases as it needs. Don't compress phases to look efficient — each phase is a confidence checkpoint between the orchestrator and the user.
2. **Review the plan with the user before any write-capable agent runs.** Resolve open questions, decide cuts, then brief implementation.
3. **Ship one phase per session.** Review + merge + deploy + verify between phases. The cost of an extra deploy cycle is small; the cost of a quietly-regressed production endpoint is large.
4. **When the user says "go," they're authorizing you to do the work well, not to do it fast.** If you can't see the whole landing zone, plan more before flying. "Best possible outcome over rush" is the explicit user preference; that stance applies to the entire initiative — don't drop back into one-shot mode after the first phase succeeds.

The bar for "non-trivial" is intentionally low: if a fresh agent reading the plan would struggle to predict the failure modes without it, the work needs a plan.

### Normal-case flow

For a typical multi-task session with container isolation: pick the batch → fan out container agents in parallel (one per implementation session) → review each branch against `git diff main...<branch>` → surface to user → merge orchestrator lands approved branches sequentially via `deploy-landing.sh` (lock-serialized) → deploy → cleanup.

For projects on the framework worktree path: fan out implement agents in a single message → review each branch → merge approved branches sequentially with `npm run build` between each → deploy once at the end.

Independent tasks parallelize cleanly in either model; dependent tasks run sequentially.

### Model selection

| Task type | Model |
|---|---|
| Simple (one-liner, rename) | `haiku` |
| Standard (component, handler) | `sonnet` |
| Complex (multi-file, architectural) | `opus` |
| Review agents | `sonnet` |
| Plan agents | `sonnet` |

### Frontend-only deploy

For pure frontend changes (no infra, no Lambda code): `npm run deploy:web` (Vite build → S3 sync → CloudFront invalidation, ~15s). Skip the full CDK deploy. Predeploy gate (clean tree, build passes) still applies.

## Agent Isolation

Write-capable agents run in an isolated environment so a wrong run never contaminates the parent checkout. Two mechanisms exist, in decreasing order of safety:

| Mechanism | Where | Write-capable | Isolated | Failure mode |
|---|---|---|---|---|
| **Docker container** | `bash scripts/spawn-container-agent.sh` (project-provided) | Yes | Structurally — parent checkout not in container mount namespace | Setup overhead (one-time image build, claude setup-token in keychain) |
| **Framework worktree** | `Agent({ subagent_type: 'implement' })` with `isolation: worktree` frontmatter | Yes | Best-effort — temporary worktree per spawn | Silently fails under contention (commits land on parent main while agent reports clean) |
| **Read-only `Agent`** | `subagent_type` in `Plan` / `Explore` / `review` | No | N/A | None — read tools only |

Prefer container isolation when fan-out is frequent. Framework worktree isolation is acceptable for low-volume projects without container tooling, but the silent-failure mode is documented and recurring ([anthropics/claude-code#40117](https://github.com/anthropics/claude-code/issues/40117)).

Every project `.gitignore` should include `.claude/worktrees/` regardless of which mechanism is used — both write into that directory.

### Defensive hooks (required for either path)

Even with container isolation as the primary boundary, the parent checkout still needs hook-level defenses. The orchestrator itself runs in the parent checkout, and a single stray `git commit` or `Edit` from the orchestrator can land on main. Layers worth replicating:

1. **`pre-commit` hook** at `.git/hooks/pre-commit` (installed by `bash scripts/install-hooks.sh`, run automatically via npm postinstall) — aborts any commit on `main` from the parent checkout unless an explicit env-var bypass (e.g. `BLISS_ALLOW_MAIN_COMMIT=1`) is set. Writes a marker the pre-push hook reads to detect `--no-verify` bypass.
2. **`pre-push` hook** — refuses to push refs whose tip is newer than the precommit-ok marker. Defends against `--no-verify` bypass of pre-commit.
3. **PreToolUse hooks** in `.claude/settings.json` — block `git commit` on parent-main; block `Edit|Write|MultiEdit` against files whose branch resolves to main; verify hooks exist and match the committed source.
4. **`permissions.deny` rules** — `git commit --no-verify`, `git push --force`, `git rebase --no-verify`.
5. **`predeploy.sh`** — re-verifies hooks exist and match committed source; runs `git fetch --prune origin` to drop stale remote-tracking refs.

Use the default `.git/hooks/` location rather than `core.hooksPath`. The VSCode git extension occasionally rewrites `.git/config` (e.g. `vscode-merge-base` entries) and the resulting absolute-path override has historically pointed at an empty directory and silently disabled every hook. Default-location hooks are immune.

The Bliss project carries a worked example with the full hook scripts and the container-isolation runbook (`docs/dev/container-isolation.md`); copy from there when bootstrapping a project that needs the same defenses.

## Versioning & Deployed Code Traceability

Two version schemes coexist: **git short SHAs for apps**, **semver for shared libraries**. The settings/about UI shows both so deployed code is always traceable.

### Apps (frontend + backend in a project repo) — use git short SHAs

**Format:** `{git-short-sha}` (e.g., `a1b2c3d`). Generated at build time via `git rev-parse --short HEAD`.

**Rules:**
- **Only deploy committed code.** Deploy scripts must enforce a clean working tree (`git status --porcelain` empty) so the SHA in logs always points to a real commit.
- **Inject at build time.** Vite `define` for the frontend, Lambda env var (via CDK) for handlers. Never read from `process.env` at runtime — the value must be baked into the artifact.
- **Log version on startup.**
  ```typescript
  // Lambda
  const VERSION = process.env.APP_VERSION || 'dev';
  console.log(`[init] version=${VERSION}`);

  // Frontend (Vite)
  console.log(`[init] version=${__APP_VERSION__}`);
  ```
- **Include version in key log lines.** Error logs, API responses, chat outputs — anything you might grep — should include the version.
- **CDK wiring:** Pass `APP_VERSION` as an environment variable to all Lambdas. Generate it in the deploy script:
  ```bash
  export APP_VERSION=$(git rev-parse --short HEAD)
  ```
- **Vite wiring:**
  ```typescript
  define: { '__APP_VERSION__': JSON.stringify(process.env.APP_VERSION || 'dev') }
  ```

### Shared libraries (e.g., zachtime-ui, packages/common consumed via `file:`) — use semver

**Format:** `MAJOR.MINOR.PATCH` (e.g., `0.2.0`). Manually bumped when shipping meaningful changes.

**Why semver here, not SHAs:** shared libs aren't their own git repos — they're consumed via `file:` from multiple apps, so there's no single "deployed SHA." Different consumers may bundle different snapshots at any time. A semver string communicates intent (breaking vs. additive vs. patch) and a `CHANGELOG.md` carries the "what changed" story.

**Rules:**
- Maintain the version in **two places kept in sync**: `package.json` `version` field and `src/version.ts` (`export const VERSION = '0.2.0'`).
- Export `VERSION` from the library's index so consumers can display it.
- Maintain a `CHANGELOG.md` at the root of the library. One entry per version bump, dated, describing observable changes.
- Bump on every meaningful change (new component, prop change, visible style change). Skip pure comment/doc/test edits.
- Pre-1.0: minor bumps may include breaking changes — the changelog is authoritative.

Manual semver beats Changesets/semantic-release here because for single-developer projects with `file:`-linked libs, automated tooling adds more ceremony than value. The forcing function is that the version is visible in the consuming app's UI (next section).

### Surface all versions in the UI

Every app with a settings/about screen should display all relevant versions in one place — typically the bottom of the settings modal:

```
web v{appSha} · api v{apiSha} · ui v{libSemver}
```

- Fetch the API version via a public `GET /info` endpoint that returns `{ version, service }`.
- Highlight `web`/`api` mismatches in amber so deployment drift between frontend and backend is immediately visible.
- The `ui` version is informational (no mismatch warning) since it tracks the bundled library snapshot.

When debugging production, anyone (you, Claude Code, a future collaborator) can see deployed versions in seconds — `git show <sha>` for app code, `CHANGELOG.md` lookup for the shared lib.

## Authentication & Identity (Cognito + Google OAuth)

All apps share one identity model: **Amazon Cognito User Pools** for the user store and JWT issuance, with **one shared Google OAuth client** (umbrella brand `zachtime.xyz`) federated into every pool.

**Why this shape:** one User Pool per app keeps user data partitioned and cheap (free under 50k MAU). One shared OAuth client means users see a single consent screen across the portfolio, and Google's verification (which covers the *client*, not the *app*) carries forward to every current and future app on the same client. Cognito Hosted UI is used as the OAuth federation engine but never as user-facing UI; sign-in screens live in the shared `zachtime-ui` library.

**Per-new-app setup (one-time):**
1. Cognito User Pool in CDK with `selfSignUpEnabled`, email alias, auto-verify, Google as supplementary IdP.
2. Cognito hosted UI domain (e.g. `<app>-auth.auth.us-east-1.amazoncognito.com`).
3. Add the new hosted UI domain to the shared Google OAuth client in Google Cloud Console:
   - Redirect URI: `https://<app>-auth.auth.us-east-1.amazoncognito.com/oauth2/idpresponse`
   - JS origin: `https://<app>.zachtime.xyz`
4. Wire `SignInPage` from `zachtime-ui` with pool ID, client ID, hosted UI domain.
5. Configure Cognito's Google IdP with `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` from Secrets Manager.

**Umbrella brand requirement:** `zachtime.xyz` must host the consent-screen homepage, `/privacy`, `/terms`, and a contact email — these are non-negotiable for OAuth verification. An app that wants to live outside this umbrella needs its own OAuth client and consent screen, almost always not worth the trouble.

**Verification:** form-filling exercise (privacy URL, terms URL, app homepage, support email, scope justification). For basic scopes (`openid email profile`) it's free, no security audit, typically approved within a week. Submit once for the shared client; every current and future app on it is covered. Sensitive scopes (Gmail, Drive) require CASA — out of scope for the standard pattern.

## Delegation rules (orchestrator)

Already covered as hard rules in the Dev Loop section. Quick reference:

```bash
# Write-capable — container isolation (preferred when the project provides it)
bash scripts/spawn-container-agent.sh --task-file path/to/task.md
bash scripts/spawn-container-agent.sh --task "inline prompt"
```

```
# Write-capable — framework worktree (fallback; silent-failure mode documented)
Agent({ description: "...", prompt: "...", subagent_type: "implement" })

# Read-only planning (multi-phase design before any write)
Agent({ description: "...", prompt: "...", subagent_type: "Plan" })

# Read-only review (fresh context, bias-free)
Agent({ description: "...", prompt: "...", subagent_type: "review" })

# Read-only exploration (search, grep, file reads)
Agent({ description: "...", prompt: "...", subagent_type: "Explore" })
```

The orchestrator overrides `model` per task complexity (haiku/sonnet/opus per the table above).

## Credentials

- All API keys live in `.env` (gitignored)
- Never commit secrets to the repo
- Standard keys: `ANTHROPIC_API_KEY`
- AWS credentials via `~/.aws/credentials` or environment variables

## CLAUDE.md vs auto-memory

When a new convention or fact crystallizes, decide where it lives:

| Goes in CLAUDE.md | Goes in auto-memory |
|---|---|
| A rule a fresh agent must know to be correct in this codebase | A user preference about *how* to communicate or work |
| Architecture decisions, schema patterns, deployment shape | What the user has worked on, who collaborators are |
| Anything load-bearing for code review or design | Hedges, communication style, recurring corrections |
| Project state visible from `git log` should NOT live here | Snapshots of project state should NOT live here |

If both apply, prefer CLAUDE.md — it's checked into the repo and survives memory turnover. If neither applies cleanly, it probably doesn't need to be persisted at all.
