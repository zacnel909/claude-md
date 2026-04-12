# Global Project Conventions

Cross-project reference for bootstrapping new projects. Each project copies relevant sections into its own `CLAUDE.md` and customizes.

## Monorepo Structure

Standard layout for all TypeScript projects using npm workspaces:

```
project-name/
  apps/              # Frontend applications (Vite)
  packages/          # Shared libraries (common schemas, utilities)
  services/          # Backend services (Lambda handlers, bot clients)
  infra/             # AWS CDK infrastructure-as-code
  data/              # Canonical content/config consumed at build time
  docs/              # Architecture docs, building strategy
  tests/             # Cross-package test suites
  package.json       # Root: npm workspaces
  tsconfig.base.json # Shared TypeScript config
  CLAUDE.md          # Project-specific instructions
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

**Principles:**
- Schemas define validation AND types in one place
- Types exported as `z.infer<typeof Schema>` — never hand-written separately
- Validate at system boundaries (API requests, WebSocket messages, file imports)
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
- DynamoDB single-table design: `pk`/`sk` partition/sort keys
- GSI (`gsi1pk`/`gsi1sk`) for alternative access patterns
- TTL attribute for auto-cleanup
- Pay-per-request billing

## Dev Loop (Agent Orchestration)

Long-lived sessions use a multi-agent pipeline. The main conversation acts as the **orchestrator** — it never implements directly.

### Session Lifecycle

1. **Review docs/** — Read project docs, understand current state and priorities
2. **Select next tasks** — Pick batch, create in-session TaskCreate items
3. **Execute tasks** — Run task pipeline (below)
4. **Commit & deploy** — Merge passing work, deploy changed resources
5. **Update docs/** — Record completed work, decisions, and next steps

### Task Pipeline

```
Phase 1: Implement (1 agent, worktree)
  → explore → design → implement → write tests → run tests → commit

Phase 2: Review (1 agent, fresh context)
  → read diff → check correctness, conventions, tests → approve / request changes
```

- Phase 1 and Phase 2 run sequentially per task
- Independent tasks run in parallel (multiple worktree agents)
- Pipeline parallelism: Task A in review while Task B in implementation
- Review feedback loop: fixup agent → re-review

### Pipelines by Complexity

| Complexity | When | Pipeline |
|---|---|---|
| Simple | One-liner, typo, rename | Implement (worktree) → done |
| Standard | New component, handler, feature | Implement (worktree) → Review |
| Complex | Multi-file, architectural | Plan → user approves → Implement → Review |

### Agent Roles

**Implement agent** (`subagent_type=general-purpose`, `isolation=worktree`)
- Receives: task description + plan (if complex)
- Works in isolated git worktree
- Workflow: explore → implement → tests → commit
- Returns: branch name, summary, test results

**Review agent** (`subagent_type=general-purpose`)
- Fresh context (bias-free)
- Reads: git diff, CLAUDE.md conventions
- Checks: correctness, conventions, test coverage, security
- Returns: approve / request changes / reject

### Model Selection

| Task Type | Model |
|---|---|
| Simple (one-liner, rename) | `model=haiku` |
| Standard (component, handler) | `model=sonnet` |
| Complex (multi-file, architectural) | `model=opus` |
| Review agents | `model=sonnet` |
| Plan agents | `model=sonnet` |

### Deploy

- Only after all reviews pass and tests are green
- `npm run deploy` for full stack (CDK)
- Frontend-only: S3 sync + CloudFront invalidation
- One deploy step per session, not per task

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

**Why semver here, not SHAs:**
- Shared libs aren't their own git repos — they're consumed via `file:` from multiple apps, so there's no single "deployed SHA" to point at.
- Different consumers may bundle different snapshots of the lib at any time.
- A semver string communicates *intent* (breaking vs. additive vs. patch) and serves as a coordination signal between consumers.
- Reads from a `CHANGELOG.md` for the "what changed" story.

**Rules:**
- Maintain the version in **two places kept in sync**: `package.json` `version` field and `src/version.ts` (`export const VERSION = '0.2.0'`).
- Export `VERSION` from the library's index so consumers can display it.
- Maintain a `CHANGELOG.md` at the root of the library. One entry per version bump, dated, describing observable changes.
- Bump on every meaningful change (new component, prop change, visible style change). Skip pure comment/doc/test edits.
- Pre-1.0: minor bumps may include breaking changes — the changelog is authoritative.

**Why manual semver instead of automated tools (Changesets, semantic-release):** for a single-developer project with `file:`-linked libs, automated tooling adds more ceremony than value. Manual works as long as the version number is visible in the consuming app's UI (see below) — that visibility is the forcing function.

### Surface all versions in the UI

Every app with a settings/about screen should display all relevant versions in one place — typically the bottom of the settings modal:

```
web v{appSha} · api v{apiSha} · ui v{libSemver}
```

- Fetch the API version via a public `GET /info` endpoint that returns `{ version, service }`.
- Highlight `web`/`api` mismatches in amber so deployment drift between frontend and backend is immediately visible.
- The `ui` version is informational (no mismatch warning) since it tracks the bundled library snapshot.

**Why:** When debugging production issues, anyone (you, Claude Code, a future collaborator) can see the deployed versions in seconds — `git show <sha>` for app code, `CHANGELOG.md` lookup for the shared lib. No guessing whether the running code matches what's in the repo.

## Authentication & Identity (Cognito + Google OAuth)

All apps share a single identity model: **Amazon Cognito User Pools** for the user store and JWT issuance, with **one shared Google OAuth client** (umbrella brand) federated into every pool.

### Why this shape

- **One Cognito User Pool per app** — keeps user data partitioned (worldview users ≠ activity-tracker users), so deleting one app's pool doesn't affect the others. Cheap and free under 50k MAU per pool.
- **One shared Google OAuth client across all apps** — users see one consent screen ("zachtime") for all your apps combined. New projects only need to add their Cognito hosted UI domain to the existing client's authorized redirect URIs. Once Google verifies the client, every current and future app on the same client is verified.
- **Cognito Hosted UI is used as the OAuth federation engine, but never as the user-facing UI.** Apps construct sign-in URLs with `identity_provider=Google` to skip Cognito's branded landing page and go straight to Google. Email/password sign-in is handled directly via `amazon-cognito-identity-js` SDK (no hosted UI involved).
- **Sign-in UI lives in `zachtime-ui`.** Each app imports the shared `SignInPage` and `AuthProvider`, configured with its Cognito pool/client/domain. This guarantees consistent UX across the portfolio.

### Required setup per new app

1. **Create a Cognito User Pool** in CDK with `selfSignUpEnabled`, email sign-in alias, auto-verify, and Google as a supplementary identity provider.
2. **Create a Cognito hosted UI domain** (e.g., `worldview-auth.auth.us-east-1.amazoncognito.com`) — only used as an OAuth callback target, not for UI.
3. **Add the new hosted UI domain to the shared Google OAuth client** in Google Cloud Console (manual one-time step):
   - Authorized redirect URI: `https://<app>-auth.auth.us-east-1.amazoncognito.com/oauth2/idpresponse`
   - Authorized JavaScript origin: `https://<app>.zachtime.xyz`
4. **Wire `SignInPage` from zachtime-ui** into the app, passing pool ID, client ID, and hosted UI domain as props.
5. **Inside Cognito**, configure the Google IdP with the shared `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` (stored in Secrets Manager, fetched at deploy time).

### Umbrella brand: `zachtime.xyz`

The Google OAuth consent screen is configured against `zachtime.xyz` as the umbrella brand. This means **`zachtime.xyz` must host**:

- A real homepage that lists all the apps (already in place — `IndexPage` shows games, services, etc.)
- A privacy policy at `/privacy` covering all apps under the brand
- Terms of service at `/terms` covering all apps under the brand
- A contact email reachable from the homepage
- An authorized domain registered in the Google OAuth consent screen

These requirements are non-negotiable for OAuth verification. If a new app wants to live outside this umbrella, it needs its own Google OAuth client and its own consent screen — almost always not worth the trouble.

### Google OAuth verification

Google requires verification before an OAuth client can serve more than 100 unique users (testing-mode wall) or escape the "Google hasn't verified this app" warning screen. For basic scopes (`openid email profile`), verification is:

- Free
- No security audit (CASA only required for sensitive scopes like Gmail/Drive)
- A form-filling exercise: privacy URL, terms URL, app homepage, support email, app logo, justification of each scope
- Approval typically within a few days to a week

**Submit the verification form once for the shared OAuth client.** Every current and future app federated into Cognito pools using this client is automatically covered. New apps added later don't trigger re-verification — they just need the redirect URI added.

### Credentials

The shared Google OAuth client credentials live in AWS Secrets Manager:
- `GOOGLE_CLIENT_ID`
- `GOOGLE_CLIENT_SECRET`

They are referenced by ARN from each app's CDK stack and granted to Cognito at user pool creation.

## Credentials

- All API keys live in `.env` (gitignored)
- Never commit secrets to the repo
- Standard keys: `ANTHROPIC_API_KEY`
- AWS credentials via `~/.aws/credentials` or environment variables
