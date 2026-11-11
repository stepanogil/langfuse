# Upgrading the fork from upstream

> **For the Claude Code agent reading this:** this is the maintenance runbook
> for our Langfuse fork. It produces a patched release branch and a patched web
> image. Deploying that image to the Azure VM is covered in a separate runbook,
> `docs/deploy-patched-web-image.md`. Before each step that writes to a remote
> (`git push`, running the build workflow), show the command and wait for the
> operator to confirm. Local steps and read-only steps can run without asking.
> If a cherry-pick conflicts in a way section 5 doesn't cover, stop and explain
> it. Don't guess.

## 1. Context

- **Upstream:** `https://github.com/langfuse/langfuse.git` (the git remote `origin`)
- **Fork:** `https://github.com/stepanogil/langfuse.git` (the git remote `fork`).
  Use HTTPS. The SSH URL (`git@github.com:...`) has hung on the workstation,
  while `gh` is set up for HTTPS. To switch:
  `git remote set-url fork https://github.com/stepanogil/langfuse.git`.
- **Build pipeline:** `https://github.com/stepanogil/langfuse-web-build`, a
  private repo with one workflow, `build-web.yml`. It lives outside the fork on
  purpose: the fork carries ~40 upstream workflows, most of which would run on
  every push if Actions were active there. **Keep the fork's Actions unused.**
  Don't add workflows to the fork.
- **Image:** `ghcr.io/stepanogil/langfuse-web:X.Y.Z-patched` (private,
  `linux/amd64`).
- **What the fork adds:** exactly one commit, `fix(auth): revoke JWT sessions server-side on sign-out`.
  It adds a NextAuth `events.signOut` handler in `web/src/server/auth.ts` that
  calls `advanceSessionsExpiredAtForEmail(email)`, plus one unit test in
  `web/src/__tests__/server/unit/getServerAuthSession-session-revocation.servertest.ts`.
  It closes a VAPT finding: before the fix, a copied session cookie kept working
  after sign-out.
- **Why releases, not `main`:** the VM runs the official worker and ai-gateway
  images next to our web image, and all three must be on the **same version**.
  So every build is "official release tag + the fix", never "upstream `main` +
  the fix".
- **We maintain the fork long term.** The fix won't be upstreamed. The upstream
  PR (#19242) is closed.

## 2. Branch model

| Branch | Contents | Purpose | Who changes it |
|---|---|---|---|
| `fix/revoke-jwt-session-on-sign-out` | The fix on upstream `main`, plus these runbooks | Home branch of the fork: the docs and a reference copy of the fix | Commit doc updates here |
| `release/vX.Y.Z-patched` | Tag `vX.Y.Z` plus the fix (one commit, no docs) | What gets built and deployed | Created once per release, never rebased |
| `main` (fork) | Whatever GitHub's fork copy holds | Not used by this workflow | Leave it alone |

### Current state

| Release branch | Fix commit | Image | Deployed |
|---|---|---|---|
| `release/v4.78.0-patched` | `3fddac6048` | `ghcr.io/stepanogil/langfuse-web:4.78.0-patched` (`sha256:05acc94c…`) | Not yet, as of 2026-11-11 |

Update this table whenever you add a release or deploy one.

Rules:
- **Every `release/*-patched` branch is exactly one commit ahead of its tag.**
  Check with `git rev-list --count vX.Y.Z..release/vX.Y.Z-patched`, which should
  print `1`. The build workflow refuses to build anything else.
- **The fix to cherry-pick is the tip of the newest patched branch.** It's the
  most recently conflict-resolved copy.
- **Keep old patched branches.** They record what was deployed.
- **Don't merge upstream `main` into a patched branch**, and don't use GitHub's
  "Sync fork" button on one. Either would mix unreleased upstream code into the
  build.
- To change the fix itself: amend the commit on the newest patched branch so it
  stays one commit, and make the same change on the home branch.

## 3. Prerequisites (workstation)

- Git, Node 22 (`node -v`), and **pnpm**. If pnpm is missing, run
  `corepack enable`.
- `gh` CLI logged in with push access to the fork and to `langfuse-web-build`
  (`gh auth status`). To also inspect the GHCR package from the workstation,
  add the scope with `gh auth refresh -s read:packages`.
- Working directory: the local clone, with remotes `origin` (upstream) and
  `fork` (`git remote -v`).

## 4. Per-release workflow

### 4.1 Pick the release
```bash
git fetch origin --tags
git tag --sort=-v:refname | grep -E '^v4\.[0-9]+\.[0-9]+$' | head -5
git ls-remote --heads https://github.com/stepanogil/langfuse.git 'release/*-patched'
```

Set the variables:
```bash
NEW=v4.79.0                       # the upstream release to move to
PREV=release/v4.78.0-patched      # the newest existing patched branch
git fetch https://github.com/stepanogil/langfuse.git "$PREV:$PREV"   # if it isn't local
FIX=$(git rev-parse "$PREV")      # the fix commit is the tip of PREV
git log -1 --oneline "$FIX"       # should print the fix(auth) commit
```

You can skip releases; going straight from 4.78.0 to 4.83.1 is fine. Never
pick a release older than what the VM runs, because database migrations only
move forward.

Read the upstream release notes between the deployed version and `NEW` for
breaking changes, required env vars, and infra changes:
`gh release view "$NEW" -R langfuse/langfuse`. Check `docker-compose.yml` too:

```bash
PREV_TAG=$(git rev-parse --abbrev-ref "$PREV" | sed -E 's|.*release/(v[0-9.]+)-patched|\1|')   # e.g. v4.78.0
git diff "$PREV_TAG" "$NEW" -- docker-compose.yml
```

If the diff isn't empty, tell the operator: the VM's clone needs updating in
that deploy (deploy runbook, Phase 1).

### 4.2 Create the branch and re-apply the fix
```bash
git switch -c "release/${NEW}-patched" "$NEW"
git cherry-pick "$FIX"
```

- **Clean:** go to 4.3.
- **Conflict** (usually in `web/src/server/auth.ts`, which changes often
  upstream):
  1. For every conflict hunk, take **upstream's side** (`HEAD`) as the base.
  2. Re-add only the fix's two edits:
     - Add `advanceSessionsExpiredAtForEmail` to the import from
       `@/src/features/auth/lib/sessionExpiration`.
     - In the object `getAuthOptions()` returns, add a `signOut` entry to the
       `events: { ... }` block (create the block if upstream has none; keep
       anything already in it):
       ```ts
       events: {
         // JWT sessions are stateless: clearing the cookie alone leaves a copied
         // token valid until it expires. Advancing the revocation boundary makes
         // the `session` callback reject every token issued before sign-out.
         // NextAuth logs errors thrown here and still clears the cookie.
         signOut: async (message) => {
           const email = "token" in message ? message.token?.email : undefined;
           if (email) await advanceSessionsExpiredAtForEmail(email);
         },
       },
       ```
       If upstream has added its own `signOut` event, combine the two into one
       handler. Don't define the key twice.
  3. If the test file conflicts, keep upstream's tests and add the fix's
     `"revokes the user's sessions server-side on sign-out"` test case.
  4. Make sure no conflict markers are left:
     `git diff --check && ! grep -rn '^<<<<<<<\|^>>>>>>>' web/src`
  5. `git add -A && git cherry-pick --continue`

**Stop and tell the operator** if any of these hold on the new tag:
- `grep -n "export async function advanceSessionsExpiredAtForEmail" web/src/features/auth/lib/sessionExpiration.ts`
  finds nothing, meaning the helper was renamed or removed.
- The session callback no longer checks `sessionsExpiredAt`
  (`grep -n "sessionsExpiredAt\|getSessionLoginAt" web/src/server/auth.ts`).
- Upstream now revokes sessions on sign-out itself. Look for a `signOut` event
  or a release note about it. In that case **the fork may no longer be needed**
  (see section 7).

### 4.3 Check the shape of the branch
```bash
git rev-list --count "$NEW..HEAD"         # should be 1
git diff --stat "$NEW" HEAD               # only auth.ts and the test file
```

## 5. Verify

```bash
pnpm install --frozen-lockfile
pnpm --filter=shared run db:generate && pnpm --filter=shared run build
pnpm --filter web run test getServerAuthSession-session-revocation
```

A pass prints `Tests  N passed (N)`, including
`revokes the user's sessions server-side on sign-out`. For 4.78.0 that was
`Tests  7 passed (7)`.

Then confirm the test actually catches a regression. Put back upstream's
`auth.ts`, re-run, see exactly that test fail, and restore the file:

```bash
git show "$NEW":web/src/server/auth.ts > web/src/server/auth.ts
pnpm --filter web run test getServerAuthSession-session-revocation   # expect: 1 failed
git checkout -- web/src/server/auth.ts
```

Optional and memory-heavy: `pnpm --filter web run typecheck` and
`pnpm exec turbo run lint --filter=web --force`. On the WSL workstation the
type-aware linter was killed for lack of memory (`SIGKILL` from `tsgolint`),
which is a host limit, not a lint failure. The image build in section 6
compiles the app either way. It isn't a full typecheck, but it catches broken
imports and syntax.

## 6. Publish and build

Ask before each of these.

1. **Push the branch to the fork:**
   ```bash
   git push -u fork "release/${NEW}-patched"
   ```
   The fork's Actions don't fire on this push. That's expected.

2. **Build the image** (takes about 30–35 minutes; the build cache helps on
   later runs):
   ```bash
   gh workflow run build-web.yml -R stepanogil/langfuse-web-build -f version="${NEW#v}"
   sleep 10; RUN=$(gh run list -R stepanogil/langfuse-web-build --limit 1 --json databaseId -q '.[0].databaseId')
   gh run watch "$RUN" -R stepanogil/langfuse-web-build --exit-status
   ```
   The workflow checks out `release/v<version>-patched` from the fork, fails
   unless that branch is exactly one commit on top of upstream `v<version>`,
   builds `web/Dockerfile` for `linux/amd64`, and pushes:
   - `ghcr.io/stepanogil/langfuse-web:<version>-patched`
   - `ghcr.io/stepanogil/langfuse-web:<version>-patched-<short sha>`

   A different branch can be passed with `-f ref=<branch>`.

3. **Get the digest for the hand-off:**
   ```bash
   gh run view "$RUN" -R stepanogil/langfuse-web-build --log \
     | grep -o 'langfuse-web:[^ ]*-patched@sha256:[0-9a-f]*' | sort -u
   ```

4. **Record it on the home branch:** switch to
   `fix/revoke-jwt-session-on-sign-out` and add the new row to "Current state"
   (section 2) here and to "Images available right now" in
   `docs/deploy-patched-web-image.md`. If the fix itself changed, update
   section 4.2. Then commit and `git push fork fix/revoke-jwt-session-on-sign-out`.

**Hand-off to the operator for the VM:** the version `${NEW#v}`, the image tag
and digest, and whether `docker-compose.yml` changed upstream (4.1). On the VM
they follow `docs/deploy-patched-web-image.md`: check the version, back up,
change the three tags in the override file, pull, `up -d`, verify.

### If the build fails
- `No space left on device`: the runner's disk is too small for the build. The
  workflow's "Free disk space" step removes unused preinstalled tools; it may
  need more paths added if GitHub changes the runner image.
- Killed during `next build`, or exit code 137: the runner ran out of memory.
  The workflow adds swap. If that isn't enough, use a larger runner or the
  VM-build fallback (deploy runbook, section 9).
- "HEAD^ is not upstream vX.Y.Z": the branch isn't exactly tag + one commit. Go
  back to 4.3.
- The `Legacy…` and `RedundantTargetPlatform` annotations come from upstream's
  Dockerfile. They're only warnings.

## 7. When the fork is no longer needed

We aren't upstreaming the fix, but upstream may fix the same problem on its own
(section 4.2 checks for this on every upgrade). If a release revokes sessions on
sign-out without our patch, confirm it with the cookie replay test in
`docs/deploy-patched-web-image.md` Phase 5, run against the official image.
Then:
1. On the VM, delete `docker-compose.override.yml`.
2. Run `docker compose up -d --pull always` (back on the official images).
3. Stop creating patched branches. The build repo and the GHCR package can be
   archived or deleted.

## 8. Quick reference

```bash
git fetch origin --tags
NEW=vX.Y.Z; PREV=release/vA.B.C-patched; FIX=$(git rev-parse "$PREV")
git switch -c "release/${NEW}-patched" "$NEW" && git cherry-pick "$FIX"
# resolve conflicts if any (4.2), then:
git rev-list --count "$NEW..HEAD"   # 1
pnpm install --frozen-lockfile && pnpm --filter=shared run db:generate && pnpm --filter=shared run build
pnpm --filter web run test getServerAuthSession-session-revocation
git push -u fork "release/${NEW}-patched"
gh workflow run build-web.yml -R stepanogil/langfuse-web-build -f version="${NEW#v}"
# then: record the image (6.4) and deploy with docs/deploy-patched-web-image.md
```
