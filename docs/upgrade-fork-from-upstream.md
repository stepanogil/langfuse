# Upgrading the fork from upstream

> **For the Claude Code agent reading this:** this is the maintenance runbook
> for our Langfuse fork. It produces a patched release branch. Deploying that
> branch to the Azure VM is covered in a separate runbook,
> `docs/deploy-patched-web-image.md`. Before each step that writes to a remote
> (`git push`, `gh repo sync`, pushing an image), show the command and wait for
> the operator to confirm. Local steps and read-only steps can run without
> asking. If a cherry-pick conflicts in a way section 5 doesn't cover, stop and
> explain it. Don't guess.

## 1. Context

- **Upstream:** `https://github.com/langfuse/langfuse.git` (the git remote `origin`)
- **Fork:** `git@github.com:stepanogil/langfuse.git` (the git remote `fork`)
- **What the fork adds:** exactly one commit, `fix(auth): revoke JWT sessions server-side on sign-out`.
  It adds a NextAuth `events.signOut` handler in `web/src/server/auth.ts` that
  calls `advanceSessionsExpiredAtForEmail(email)`, plus one unit test in
  `web/src/__tests__/server/unit/getServerAuthSession-session-revocation.servertest.ts`.
  It closes a VAPT finding: before the fix, a copied session cookie kept working
  after sign-out.
- **How it's deployed:** the VM runs the official worker and ai-gateway images,
  plus a web image we build ourselves. All three must be on the **same version**.
  That rule is why every deployed build has to be "official release tag + the
  fix", not "upstream `main` + the fix".

## 2. Branch model

| Branch | Contents | Purpose | Who changes it |
|---|---|---|---|
| `main` (fork) | Mirror of upstream `main` | Base for the upstream PR | Only `gh repo sync`; never commit to it |
| `fix/revoke-jwt-session-on-sign-out` | The fix on top of upstream `main` | Upstream pull request | Rebase onto `main` when needed |
| `release/vX.Y.Z-patched` | Tag `vX.Y.Z` plus the fix (one commit) | What gets built and deployed | Created once per release, never rebased |

Rules:
- **Every `release/*-patched` branch is exactly one commit ahead of its tag.**
  Check with `git rev-list --count vX.Y.Z..release/vX.Y.Z-patched`, which should
  print `1`.
- **Keep old patched branches.** They record what was deployed and are the
  rollback source.
- **Don't merge upstream `main` into a patched branch**, and don't use GitHub's
  "Sync fork" button on a patched branch. Either would mix unreleased upstream
  code into the build.
- To change the fix itself: amend the commit on the newest patched branch so it
  stays one commit. Make the same change on the `fix/...` branch.

## 3. Prerequisites (workstation)

- Git, Node 22 (`node -v`), and **pnpm**. If pnpm is missing, run
  `corepack enable`. pnpm was missing when this doc was written.
- `gh` CLI logged in with push access to the fork (`gh auth status`).
- Docker, if you want a local build check (optional, needs 8 GB+ of free RAM).
- Working directory: the local clone with both remotes.
  `git remote -v` should list `origin` (upstream) and `fork`.

## 4. One-time setup (if `release/v4.60.0-patched` doesn't exist yet)

Run this once. After that, section 5 is the routine.

```bash
git fetch origin --tags
git branch -r | grep release/ || true     # does the fork already have patched branches?
git checkout -b release/v4.60.0-patched v4.60.0
# The patch lives on the docs/fork-deploy-runbooks branch, so it's not in the
# working tree after checking out the tag. Read it from git instead.
git show docs/fork-deploy-runbooks:docs/fix-v4.60.0.patch > /tmp/fix-v4.60.0.patch
git apply --check /tmp/fix-v4.60.0.patch && git apply /tmp/fix-v4.60.0.patch
# The same diff is in docs/deploy-patched-web-image.md section 8.
git add web/src/server/auth.ts web/src/__tests__/server/unit/getServerAuthSession-session-revocation.servertest.ts
git commit -m "fix(auth): revoke JWT sessions server-side on sign-out"
```

Then verify it (section 6) and push it:
`git push -u fork release/v4.60.0-patched`. Also push the fix branch, which
isn't on the fork yet: `git push -u fork fix/revoke-jwt-session-on-sign-out`.

## 5. Per-release workflow

### 5.1 Pick the release
```bash
git fetch origin --tags
git tag --sort=-creatordate | grep -E '^v4\.[0-9]+\.[0-9]+$' | head -5
git branch -a --list '*release/*-patched' --sort=-creatordate | head -3
```

Set the variables:
```bash
NEW=v4.61.0                       # the upstream release to move to
PREV=release/v4.60.0-patched      # the newest existing patched branch
FIX=$(git rev-parse "$PREV")      # the fix commit is the tip of PREV
git log -1 --oneline "$FIX"       # should print the fix(auth) commit
```

You can skip releases; going straight from v4.60.0 to v4.64.2 is fine. Just
never pick a release older than what the VM runs now, because database
migrations only move forward.

Read the upstream release notes for anything between the deployed version and
`NEW`, especially breaking changes, required env vars, and infra changes:
`gh release view "$NEW" -R langfuse/langfuse`. Check `docker-compose.yml`
changes as well:
`git diff "${PREV#release/}" "$NEW" -- docker-compose.yml`. If that diff isn't
empty, the VM clone needs a `git pull` too, and the override file in the deploy
runbook may need updating.

### 5.2 Create the branch and re-apply the fix
```bash
git switch -c "release/${NEW}-patched" "$NEW"
git cherry-pick "$FIX"
```

- **Clean:** go to 5.3.
- **Conflict** (usually in `web/src/server/auth.ts`, which changes often
  upstream):
  1. For every conflict hunk, take **upstream's side** (`HEAD`) as the base.
  2. Re-add only the fix's two edits:
     - Add `advanceSessionsExpiredAtForEmail` to the import from
       `@/src/features/auth/lib/sessionExpiration`.
     - Inside the `events: { ... }` object that `getAuthOptions()` returns, keep
       everything upstream has (for example `createUser`) and add:
       ```ts
       // JWT sessions are stateless: clearing the cookie alone leaves a copied
       // token valid until it expires. Advancing the revocation boundary makes
       // the `session` callback reject every token issued before sign-out.
       // NextAuth logs errors thrown here and still clears the cookie.
       signOut: async (message) => {
         const email = "token" in message ? message.token?.email : undefined;
         if (email) await advanceSessionsExpiredAtForEmail(email);
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
  (see section 8).

### 5.3 Check the shape of the branch
```bash
git rev-list --count "$NEW..HEAD"         # should be 1
git diff --stat "$NEW" HEAD               # only auth.ts and the test file
```

## 6. Verify

```bash
pnpm install --frozen-lockfile
pnpm --filter=shared run db:generate && pnpm --filter=shared run build
pnpm --filter web run test getServerAuthSession-session-revocation
pnpm --filter web run typecheck
pnpm --filter web run lint
```

What a pass looks like:
- The vitest summary shows all tests passing (`Tests  N passed (N)`), including
  `revokes the user's sessions server-side on sign-out`.
- typecheck and lint exit with no errors. Lint fails on even one warning
  (`--max-warnings 0`).

Then confirm the test actually catches a regression. Temporarily comment out the
`signOut` handler, re-run the test, see it fail, and restore the handler with
`git checkout -- web/src/server/auth.ts`.

Optional full build check (heavy):
`docker build -f web/Dockerfile -t "langfuse-web-patched:${NEW#v}" .` from the
repo root.

## 7. Publish

Ask before running each of these:

```bash
git push -u fork "release/${NEW}-patched"
gh repo sync stepanogil/langfuse --branch main        # keep the fork's main mirrored
```

Optionally, refresh the upstream PR branch so it stays mergeable:
```bash
git switch fix/revoke-jwt-session-on-sign-out
git fetch origin && git rebase origin/main            # same conflict rules as 5.2
git push --force-with-lease fork fix/revoke-jwt-session-on-sign-out
```

Hand-off to the deploy step. Give the operator:
- the branch name `release/${NEW}-patched`
- the version `${NEW#v}` to pin in `docker-compose.override.yml`
- whether `docker-compose.yml` changed upstream (from 5.1)

On the VM, follow `docs/deploy-patched-web-image.md`:
1. Check out the branch.
2. Build `langfuse-web-patched:${NEW#v}`.
3. Back up Postgres.
4. Change all three image tags in the override file.
5. Run `docker compose pull langfuse-worker langfuse-ai-gateway && docker compose up -d`.
6. Verify.

## 8. Exit plan: retire the fork

The fork exists only until upstream has the fix. Open a PR from
`fix/revoke-jwt-session-on-sign-out` against `langfuse/langfuse:main` (follow the
repo's `CONTRIBUTING.md`). When a release includes the fix:
1. On the VM, delete `docker-compose.override.yml`.
2. Run `docker compose up -d --pull always` (back on the official images).
3. Stop creating patched branches. Keep the old ones for the record, or delete
   them.

## 9. Optional: automate with GitHub Actions

A workflow file in the fork (for example `.github/workflows/patched-release.yml`
on a branch that doesn't come from upstream) could:
1. Run daily. Compare the newest upstream `v4.*` tag with the newest
   `release/*-patched` branch.
2. If there's a newer tag, create the branch and cherry-pick the fix. If the
   cherry-pick conflicts, open an issue and stop (a human resolves it with
   section 5.2).
3. Run the checks from section 6.
4. Build `web/Dockerfile` and push it to GHCR as
   `ghcr.io/stepanogil/langfuse-web:<version>-patched`.

That would cut the VM's job down to changing the tag and running `pull`/`up`,
with no local build. The cost is one more file to maintain, plus a GHCR login on
the VM. It isn't set up yet.

## 10. Quick reference

```bash
git fetch origin --tags
NEW=vX.Y.Z; PREV=release/vA.B.C-patched; FIX=$(git rev-parse "$PREV")
git switch -c "release/${NEW}-patched" "$NEW" && git cherry-pick "$FIX"
# resolve conflicts if any (5.2), then:
git rev-list --count "$NEW..HEAD"   # 1
pnpm install --frozen-lockfile && pnpm --filter=shared run db:generate && pnpm --filter=shared run build
pnpm --filter web run test getServerAuthSession-session-revocation && pnpm --filter web run typecheck && pnpm --filter web run lint
git push -u fork "release/${NEW}-patched"
# then deploy with docs/deploy-patched-web-image.md
```
