# CLAUDE.md

Horizon Social Network is a social network for Horizon School of Digital Technologies, built
as the 2026/2027 Integration Project. Users are students, staff or teachers. They keep a
profile, sign in and out, search for and connect with other users, publish plain-text posts
that reach their connections' news feeds in near real time, and like, share or comment on
posts. Clients are a web app (desktop browser) and a mobile app. Both talk to backend
services over APIs. The full brief is `The_Integration_Project_-_Project_Description_-_2026_2027.pdf`.

**Status: development has not started.** The API contract exists (`hsn-openapi.yml`), but no
service code, command or CI pipeline does yet. Do not invent commands, paths or tooling.
When the team makes a decision, record it in this file in the same change that introduces
it.

This repository is the hub: architecture documentation, planning and deployment
configuration. Every service lives in its own repository in the organization, one repository
per service. Service repositories copy the "Code style", "Testing" and "Working style"
sections of this file unchanged.

## Fixed by the brief

These are requirements, not choices. Do not propose designs that break them.

| Area          | Requirement                                                                          |
| ------------- | ------------------------------------------------------------------------------------ |
| Posts         | stored in a NoSQL store (MongoDB or equivalent). Plain text only                     |
| Cache         | Redis or equivalent, for faster reads. A cache, never the source of truth            |
| Repositories  | GitHub, one repository per service                                                   |
| CI/CD         | GitHub Actions (or equivalent) builds, tests and releases every service              |
| Release       | every service is containerized, except the mobile app                                |
| Orchestration | Kubernetes, or an equivalent such as Docker Compose                                  |
| Observability | Istio, Prometheus and Grafana, or equivalents                                        |
| Reliability   | SLIs (availability, latency, error rate) and SLOs for sign-in, posting and news feed |
| Planning      | GitHub Projects (or equivalent), split into epics, stories and tasks                 |

Profile fields depend on the user kind: major for a student, administrative role for staff,
speciality for a teacher. Model the kind as a real enum, never a free string.

## API contract

`hsn-openapi.yml` (OpenAPI 3.1) is the single source of truth for the public API used by the
web and mobile apps. Change the spec first, in its own pull request, then implement it. Never
let a service drift from it; when they disagree, the spec wins or gets fixed.

Every path is served under `/api/v1`, and each prefix is owned by one service so the Istio
ingress routes on prefix alone:

| Prefix                             | Service      | Data store        |
| ---------------------------------- | ------------ | ----------------- |
| `/auth`                            | user-service | PostgreSQL, Redis |
| `/users`, `/connections`           | user-service | PostgreSQL        |
| `/posts` (likes, comments, shares) | post-service | MongoDB           |
| `/feed`, `/feed/stream`            | feed-service | Redis             |

Settled by the spec. Do not reopen these without changing the spec:

- **Auth**: JWT bearer access token, short-lived, renewed with `POST /auth/refresh`.
  Self-registration creates `role: user` only. Any other role needs a caller with the
  `users:assign-role` permission. The first admin comes from a seed script.
- **Two separate enums**: profile kind (`student`, `staff`, `teacher`) decides profile fields;
  authorization role (`user`, `moderator`, `admin`) decides permissions. Never merge them.
- **Pagination** is cursor-based: `{ items, meta: { hasNext, nextCursor } }`. No offsets.
- **Errors** are RFC 9457 `application/problem+json`, from every service.
- **Likes** are `PUT` / `DELETE` and idempotent.
- **A share** is a new post owned by the sharer that embeds the original. Sharing a share
  embeds the original, never the intermediate share. A deleted original embeds as `null`.
- **Real time** is Server-Sent Events on `GET /feed/stream` with `post.created`,
  `post.deleted` and `post.stats` events, a `: ping` every 25 seconds, and resume through
  `Last-Event-ID`.
- **Health**: each service exposes `GET /healthz`, `GET /readyz` and `GET /metrics` on its own
  port. They are for Kubernetes and Prometheus, not routed through the ingress, and not in the
  spec.

## Service communication

Decided in `docs/adr/0001-rest-at-the-edge-events-between-services.md`. Read it before
proposing a protocol change.

- **Clients to services**: REST + JSON through the ingress, as in `hsn-openapi.yml`, with
  SSE for real time. No gRPC.
- **post-service to feed-service**: events on Redis Streams, never a direct call.
  post-service appends `post.created`, `post.deleted` or `post.stats` after its write
  succeeds. feed-service reads them through a consumer group and acknowledges each event
  after handling it.
- **Delivery is at least once.** Every event handler must be idempotent: handling the same
  event twice leaves the same feed as handling it once.
- **Synchronous internal calls** use plain HTTP/JSON on endpoints that are not routed through
  the ingress and not in `hsn-openapi.yml`. Document each one's request and response shape in
  the owning service's repository.
- Architecture decisions go in `docs/adr/NNNN-<slug>.md`, one decision per file, numbered in
  order.

## Decisions to record

Fill these in as they are made. Until then, ask instead of assuming.

- Web stack, mobile stack, backend stack and package manager
- Repository name of each service
- How feed-service builds feeds (fan-out on write or on read) and gets a user's connections
- Local stack details and the branch-to-environment mapping
- The CI gate per service and which check is required on `main`

## Git, branches, commits and pull requests

The professor grades each repository's commit history: authors, frequency, size and content.
Every rule here exists so that history shows small, well-described work from its real author.

### Who commits

- Never commit unless the user asks for that commit in the current request. Leave changes in
  the working tree for review.
- Commits are made from the user's own account. No `Co-Authored-By` trailer and no AI
  attribution of any kind. Never commit or push work on behalf of another team member.
- Never push to `main`, and never force-push a branch someone else is working on.

### Branches

One branch per GitHub Projects issue, cut from an up-to-date `main`:

```
<author>/<type>/<issue-number>-<short-slug>
```

- `<author>` is the author's first and last name in PascalCase, with no separators
  (`JaneDoe`). Use the same one on every branch.
- `<type>` is one of the commit types below.
- `<short-slug>` is lowercase words joined by hyphens, three to five words. Only the author
  part is PascalCase.
- Examples: `JaneDoe/feat/42-like-a-post`, `JohnSmith/fix/57-feed-skips-shared-posts`.
- Only the author pushes to their branch. Delete it after it is merged.

### Commits

[Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <summary>

<body: what changed and why, wrapped at 72 characters>

Refs #<issue-number>
```

| Type       | Use for                                            |
| ---------- | -------------------------------------------------- |
| `feat`     | a new behaviour a user or client can see           |
| `fix`      | a bug fix                                          |
| `refactor` | a code change that changes no behaviour            |
| `test`     | adding or fixing tests only                        |
| `docs`     | documentation, ADRs, the API spec                  |
| `ci`       | GitHub Actions workflows                           |
| `build`    | Dockerfiles, dependencies, build tooling           |
| `ops`      | Kubernetes manifests, Istio, Prometheus, Grafana   |
| `chore`    | anything else that touches no production behaviour |

- `<scope>` is the module or area touched: `auth`, `posts`, `feed`, `connections`, `k8s`.
  Leave it out when the change has no single area.
- The summary is imperative, lowercase, no trailing period, at most 72 characters:
  `feat(posts): add idempotent like endpoint`, not `Added likes.` or `wip`.
- The body is required whenever the summary alone does not say why.
- One concern per commit, and every commit leaves the build and tests passing. Split a
  change that needs "and" in its summary.
- Breaking API changes add `!` after the type (`feat(posts)!: ...`) and a
  `BREAKING CHANGE:` footer.

### Pull requests

- The title is a plain, capitalized sentence saying what the pull request does, with no
  `type(scope):` prefix: `Stream post stats events over SSE`.
- Keep a pull request small enough to review in one sitting, ideally under 400 changed
  lines. Split larger work into a sequence of pull requests.
- The body follows `.github/pull_request_template.md` (What, Why, How to test,
  `Closes #<issue-number>`). GitHub fills it in when the pull request is opened.

- One approving review from another team member and green CI before merging. The author
  merges, after addressing every comment.
- A change to `hsn-openapi.yml` goes in its own pull request before the code that implements
  it.
- **Merge with "Create a merge commit"**, never "Squash and merge" or "Rebase and merge".
  Squashing collapses many small commits into one large one and erases the history the
  project is graded on. When the branch is behind, merge `main` into it; never rebase a
  pushed branch.

## Code style: the contract

Non-negotiable, and they apply to every language in every repository. Treat a violation as a
bug.

**1. No magic values.** Every literal, number or string, is a named constant whose name
explains it. No exceptions for "obvious" values. Retry limits, thresholds, page sizes and
timeouts are named constants owned by the module that uses them. Enum-like sets are real
enums (or `as const` objects with a same-name type), never a bare string compared inline.

**2. No comments.** Names carry the meaning. If a block needs a comment, it needs a better
name or its own function. This covers doc-comment blocks and banner separators
(`// ===== SECTION =====`) too: a file that needs section headers needs splitting. Where a
comment would record a real constraint (a vendor quirk, or an incident the obvious change
would bring back), write it in this file or in `docs/` instead. Never leave commented-out
code. Git remembers it.

**3. Self-explanatory names.** No abbreviations: `loadConnectionFeed`, not `loadConnFeed`.
When two kinds of id are in scope, disambiguate them (`authorId` vs `viewerId`, `postId` vs
`sharedPostId`). Loop variables get real names. Domain and vendor acronyms that are proper
names stay (API, HTTP, JWT, SLO, SLI, K8s).

**4. Blank line before and after control-flow keywords:** `if`, `else`, `for`, `while`,
`return`, `throw`, `switch`, `try`. A block owns its braces, so **never** put a blank line
directly after an opening brace or a `case` label, and **never** put one directly before a
closing brace.

```ts
async function findActiveUser({
  email,
  password,
}: LoginInput): Promise<UserRow> {
  const user = await lookUpUserByEmail(email);
  const isValidPassword = await verifyPassword(
    password,
    user?.password ?? DECOY_PASSWORD_HASH,
  );

  if (!user || !isValidPassword || !user.is_active) {
    throw invalidCredentialsError();
  }

  return user;
}
```

Formatters do not enforce rule 4. Follow it by hand.

**5. DRY.** Check for an existing helper before writing one. Pure helpers go in a shared
utilities folder. Anything touching a database, the cache or an external system goes behind
one service per system. Prefer an established, maintained package over hand-rolling a
well-known algorithm (hashing, parsing, encoding, validation), and name the package and its
license when proposing it.

**6. Explainable code.** Any team member may be asked to explain any line they submitted.
Prefer the boring, readable solution over the clever one. Do not add abstractions,
configuration or scaffolding that no current feature needs.

## Backend service contract

Applies to every backend service once its stack is chosen. Each feature lives in one folder,
`src/modules/<module>/`, holding every layer it needs:

```
route  ──►  validate  ──►  service  ──►  repository  ──►  MongoDB / relational DB
                              └──────►  gateway     ──►  Redis / other services
```

- **Route** gates (authentication), validates the input against a schema matching
  `hsn-openapi.yml`, calls **one** service function, and responds. No queries and no business rules.
- **Service** receives its dependencies as arguments (`createPostService(dependencies)`),
  holds the business rules, and returns plain data or throws a domain error.
- **Repository** is the only code that touches the module's collections or tables. It
  returns rows or "not found", never a rule.
- **Errors** are domain errors with a code from the module's error codes object and an HTTP
  status. One error handler turns them into `application/problem+json` responses. Anything else is a 500.
- **Composition**: one composition root builds every module. Routes take their service from
  it, never by direct import.

Data rules that follow from the brief:

- Redis holds derived data only. Every cached value can be rebuilt from the primary store,
  and every write path that changes the source invalidates or updates the cache.
- Writes that can be retried are idempotent. A like is unique per user and post, enforced by
  a unique index, not by a check-then-insert. The feed in Redis is derived from posts and
  connections and can be rebuilt from them. A connection between two users exists once.
- Never append to a list stored in a document when a count or a separate collection with a
  unique index can express it.
- Secrets come from environment variables or the orchestrator's secret store. Never commit a
  credential, including for local development.

## Testing

Every service ships unit tests, and CI runs them on every pull request.

- **No network and no real external service from the unit suite.** Fake HTTP at the
  transport layer, and fake databases and the cache behind the repository and gateway
  boundaries, so production code needs no test hooks.
- Test names are sentences about behaviour, not restatements of the function name:
  `"a post from a connection appears in the viewer's feed"`.
- Reuse existing fixtures and doubles before writing new ones.
- Manual scripts that touch real services are not tests. Keep them out of CI.

## Working style

- Verify a fix by breaking it: revert the change and confirm the test fails. A test that
  passes both ways proves nothing.
- Before you report a regression or a pre-existing bug, check `git show HEAD:<path>` to tell
  which one it is.
- Fix the root cause, not the caller that reported it. One fix in the shared helper is a
  smaller diff than a guard in every caller, and patching one caller leaves the others broken.
- When a shared change breaks tests, fix the tests to match the new truth. Do not weaken
  assertions to make them pass.
- Report what actually happened, including what was skipped and what still needs doing.
