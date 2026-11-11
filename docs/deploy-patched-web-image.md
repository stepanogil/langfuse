# Deploying the patched Langfuse web image on the Azure VM

> **For the Claude Code agent reading this on the VM:** this is a runbook that
> was written for you on the developer's workstation. Work through it with the
> operator one phase at a time. Before each step that changes something
> (stopping containers, building images, editing compose files, restarting),
> show the command and wait for the operator to confirm it. Read-only checks can
> run without asking. If anything on the VM doesn't match the assumptions in
> section 2, stop and tell the operator before you continue.

## 1. Context

### What we run today
- Langfuse v4 is self-hosted on an Azure VM using the
  [Docker Compose method](https://langfuse.com/self-hosting/deployment/docker-compose).
- The VM has a clone of the upstream repo `https://github.com/langfuse/langfuse.git`.
  That clone is used **only for its `docker-compose.yml`**.
- Updates are done with `docker compose up --pull always` (usually with `-d`).

### Why swapping the git clone doesn't work on its own
`docker-compose.yml` builds nothing from source. It pulls prebuilt images:

```yaml
langfuse-worker:     image: docker.langfuse.com/langfuse/langfuse-worker:${LANGFUSE_IMAGE_TAG:-4}
langfuse-web:        image: docker.langfuse.com/langfuse/langfuse:${LANGFUSE_IMAGE_TAG:-4}
langfuse-ai-gateway: image: docker.langfuse.com/langfuse/ai-gateway:${LANGFUSE_AI_GATEWAY_IMAGE_TAG:-${LANGFUSE_IMAGE_TAG:-4}}
```

Cloning our fork in place of upstream would still run the official images. To
run the fix, we have to **build our own web image and point compose at it**.

### The fix we're deploying
- Fork: `git@github.com:stepanogil/langfuse.git` (HTTPS: `https://github.com/stepanogil/langfuse.git`)
- Original commit: `c7c201ea1577cfd9028ed8c2954ccbffc801f0f7`, on branch
  `fix/revoke-jwt-session-on-sign-out`. It sits on top of upstream `main`, not on
  a release tag, and **as of writing it has not been pushed to the fork**.
- Problem: Langfuse uses NextAuth JWT sessions. Signing out only deleted the
  session cookie, so a copied cookie kept working until it expired (14 days by
  default). That's the high finding from our VAPT.
- Fix: a NextAuth `events.signOut` handler in `web/src/server/auth.ts` calls
  `advanceSessionsExpiredAtForEmail(email)`. That sets
  `users.sessions_expired_at = now()`, the same revocation boundary that
  "Sign out of all sessions" and password reset already use. The `session`
  callback then rejects any token issued before that time.
- **Side effect the operator should know about:** signing out in one browser
  now signs that user out **everywhere** (every other browser and device). The
  revocation timestamp is per user, not per session.
- Files changed: `web/src/server/auth.ts`, plus one unit test. **Only the web
  image changes.** The worker, ai-gateway, Postgres, ClickHouse, Redis and MinIO
  stay on the official images. No database migration is added.

### Hard rules
1. **Web, worker and ai-gateway must run the same Langfuse version.** The web
   container runs Prisma and ClickHouse migrations when it starts, and all three
   share queue and data contracts. So build the patched web image from an
   upstream **release tag** and pin the official worker and ai-gateway to that
   same tag.
2. **Never downgrade.** The VM has been on the floating `:4` tag with
   `--pull always`, so it may already be on a newer release than the tag used
   here. Database migrations only move forward. Pick a tag that is **equal to or
   newer than** what is running now (Phase 1 checks this).
3. Leave the upstream `docker-compose.yml` untouched. Make every change in a
   `docker-compose.override.yml` next to it, so `git pull` on the clone never
   conflicts.
4. Don't touch `.env` secrets, and don't run `docker compose down -v`. The `-v`
   flag deletes the data volumes.

## 2. Assumptions to verify on the VM
- Docker Engine and the Compose v2 plugin are installed (`docker compose version`).
- The upstream clone is in some directory. Call it `$LF_DIR`; find it with
  `docker compose ls` or by asking the operator.
- Secrets live in `$LF_DIR/.env` or are inlined in the compose file. Note which.
- Volumes: `langfuse_postgres_data`, `langfuse_clickhouse_data`,
  `langfuse_clickhouse_logs`, `langfuse_minio_data`, `langfuse_redis_data`.

## 3. Phase 1: inspect what's running (read-only)

```bash
cd "$LF_DIR"
docker compose ls
docker compose ps
docker compose images
curl -s http://localhost:3000/api/public/health     # returns {"status":"OK","version":"4.x.y"}
free -h; nproc; df -h /var/lib/docker
git -C "$LF_DIR" log -1 --oneline
ls "$LF_DIR"/docker-compose.override.yml 2>/dev/null   # does an override already exist?
```

Write down the **running version** from the health endpoint. That sets
`TARGET_TAG`:
- If the running version is `4.60.0` or older, use `TARGET_TAG=v4.60.0`. The
  ready-made patch in section 8 applies cleanly to it. The same patch is also
  committed as `docs/fix-v4.60.0.patch` on the `docs/fork-deploy-runbooks` branch; copy it over
  with `scp` instead of pasting.
- How to create patched branches for later releases is covered in
  `docs/upgrade-fork-from-upstream.md`.
- If it's newer (for example `4.62.1`), use `TARGET_TAG=v<running version>` and
  re-apply the fix by hand (section 8 shows how).

## 4. Phase 2: back up before you change anything

```bash
cd "$LF_DIR"
mkdir -p ~/langfuse-backups
docker compose exec -T postgres pg_dump -U "${POSTGRES_USER:-postgres}" -d "${POSTGRES_DB:-postgres}" -Fc \
  > ~/langfuse-backups/postgres-$(date +%Y%m%d-%H%M).dump
cp docker-compose.yml ~/langfuse-backups/
[ -f .env ] && cp .env ~/langfuse-backups/env.bak
docker compose images > ~/langfuse-backups/images-before.txt
```

Check that the dump file isn't empty (`ls -lh ~/langfuse-backups`). If Azure
disk snapshots are available, taking one of the OS and data disks is an even
better rollback point.

## 5. Phase 3: get the patched source

Work in a separate directory, **not** in `$LF_DIR`:

```bash
cd ~
git clone https://github.com/langfuse/langfuse.git langfuse-build
cd langfuse-build
git checkout -b "release/${TARGET_TAG}-patched" "${TARGET_TAG}"
```

Then apply the fix:
- **`TARGET_TAG=v4.60.0`:** save the patch from section 8 as
  `~/fix-v4.60.0.patch`, then run
  `git apply --check ~/fix-v4.60.0.patch && git apply ~/fix-v4.60.0.patch`.
- **Newer tag:** try the patch first. If `--check` fails, make the two edits
  described in section 8 by hand.

Confirm with `git diff --stat`. It should show `web/src/server/auth.ts` and the
test file, and nothing else. Commit it:
`git commit -am "fix(auth): revoke JWT sessions server-side on sign-out"`.

If the operator wants the branch kept on the fork, push it:
`git remote add fork https://github.com/stepanogil/langfuse.git && git push fork HEAD`.
This needs credentials, so ask first.

## 6. Phase 4: build the web image

The Next.js production build is heavy. Plan for **at least 8 GB of free RAM**
(16 GB is comfortable) and 10 GB or more of free disk. Building on the VM
competes with the running stack for memory.

### Option A: build on the VM (simplest, if `free -h` shows enough headroom)
Run it from the **repo root**, because the Dockerfile needs the whole monorepo:

```bash
cd ~/langfuse-build
docker build -f web/Dockerfile -t "langfuse-web-patched:${TARGET_TAG#v}" .
```

Don't pass any `NEXT_PUBLIC_*` build args. The official self-hosted image is
built without them, and those values get baked in at build time. Runtime
settings such as `AUTH_DISABLE_SIGNUP` still come from compose and `.env` as
before.

If the VM runs out of memory during the build (the build gets killed or hangs
at `next build`), stop and use option B.

### Option B: build elsewhere and ship the image
On a machine with enough RAM, using the same `docker build` command:
- Push to a registry, either GitHub's
  (`ghcr.io/stepanogil/langfuse-web:${TARGET_TAG#v}-patched`, after
  `docker login ghcr.io`) or Azure Container Registry. Then `docker login` on the
  VM and use that image name in Phase 5.
- Or copy it over without a registry:
  `docker save langfuse-web-patched:4.60.0 | gzip | ssh <vm> 'gunzip | docker load'`.

Check that the image exists: `docker image ls | grep langfuse-web-patched`.

## 7. Phase 5: point compose at the patched image and deploy

Create `$LF_DIR/docker-compose.override.yml`. If one already exists, **merge**
these keys into it; don't overwrite it.

```yaml
services:
  langfuse-web:
    image: langfuse-web-patched:4.60.0          # or the registry name from option B
    pull_policy: never                           # local-only image; drop this line if it's in a registry
  langfuse-worker:
    image: docker.langfuse.com/langfuse/langfuse-worker:4.60.0
  langfuse-ai-gateway:
    image: docker.langfuse.com/langfuse/ai-gateway:4.60.0
```

Replace `4.60.0` with `${TARGET_TAG#v}` throughout. Compose merges
`docker-compose.override.yml` automatically.

**Don't use `--pull always` for this deploy.** It would try to pull the local
web image from Docker Hub and fail. Instead:

```bash
cd "$LF_DIR"
docker compose config | grep -E 'image:'        # confirm the three pinned images
docker compose pull langfuse-worker langfuse-ai-gateway
docker compose up -d
docker compose logs -f langfuse-web              # wait for migrations and "Ready"; Ctrl-C to stop following
```

## 8. The patch

### Ready-made patch for `v4.60.0`
This was checked on the workstation: `git apply` succeeds on a clean `v4.60.0`
checkout. It wasn't compiled or tested there, because the workstation had no
pnpm. The Docker build in Phase 4 is the compile check.

```diff
diff --git a/web/src/__tests__/server/unit/getServerAuthSession-session-revocation.servertest.ts b/web/src/__tests__/server/unit/getServerAuthSession-session-revocation.servertest.ts
index ea450b903e..25254da4fb 100644
--- a/web/src/__tests__/server/unit/getServerAuthSession-session-revocation.servertest.ts
+++ b/web/src/__tests__/server/unit/getServerAuthSession-session-revocation.servertest.ts
@@ -120,6 +120,24 @@ describe("NextAuth JWT session revocation", () => {
     expect(mockQueryRaw).toHaveBeenCalledTimes(1);
   });
 
+  it("revokes the user's sessions server-side on sign-out", async () => {
+    const signOut = (await getAuthOptions()).events?.signOut;
+    if (!signOut) throw new Error("Expected signOut event");
+    mockQueryRaw.mockResolvedValueOnce([
+      { sessionsExpiredAt: new Date("2026-09-09T12:30:00.000Z") },
+    ]);
+
+    await signOut({ token: { email: "USER@example.com" } } as {
+      session: Session;
+      token: JWT;
+    });
+
+    expect(mockQueryRaw).toHaveBeenCalledTimes(1);
+    const [query, email] = mockQueryRaw.mock.calls[0];
+    expect((query as string[]).join("")).toContain('SET "sessions_expired_at"');
+    expect(email).toBe("user@example.com");
+  });
+
   it.each([
     {
       name: "denies a token issued before the revocation timestamp",
diff --git a/web/src/server/auth.ts b/web/src/server/auth.ts
index 8a686491a9..3cee289718 100644
--- a/web/src/server/auth.ts
+++ b/web/src/server/auth.ts
@@ -20,7 +20,10 @@ import {
 import { isGatewayEnabledForOrganization } from "@/src/features/ai-gateway/server/availability";
 import { env } from "@/src/env.mjs";
 import { createProjectMembershipsOnSignup } from "@/src/features/auth/lib/createProjectMembershipsOnSignup";
-import { getSessionLoginAt } from "@/src/features/auth/lib/sessionExpiration";
+import {
+  advanceSessionsExpiredAtForEmail,
+  getSessionLoginAt,
+} from "@/src/features/auth/lib/sessionExpiration";
 import { type AdClickIds } from "@/src/features/auth/lib/signupAttribution";
 import {
   type AdapterUser,
@@ -1235,6 +1238,14 @@ export async function getAuthOptions(
           });
         }
       },
+      // JWT sessions are stateless: clearing the cookie alone leaves a copied
+      // token valid until it expires. Advancing the revocation boundary makes
+      // the `session` callback reject every token issued before sign-out.
+      // NextAuth logs errors thrown here and still clears the cookie.
+      signOut: async (message) => {
+        const email = "token" in message ? message.token?.email : undefined;
+        if (email) await advanceSessionsExpiredAtForEmail(email);
+      },
     },
   };
   return data;
```

When saving this to `~/fix-v4.60.0.patch`, copy only what's between the
```` ```diff ```` fences, and keep the trailing whitespace on the context lines.

### Re-applying the fix by hand on a newer tag
Make the two edits in `web/src/server/auth.ts`:
1. Add `advanceSessionsExpiredAtForEmail` to the existing import from
   `@/src/features/auth/lib/sessionExpiration`.
2. In the object `getAuthOptions()` returns, find the `events: { ... }` block
   (it holds `createUser`; create the block if it's missing) and add:
   ```ts
   signOut: async (message) => {
     const email = "token" in message ? message.token?.email : undefined;
     if (email) await advanceSessionsExpiredAtForEmail(email);
   },
   ```

First check that the helper exists on that tag:
`grep -n "export async function advanceSessionsExpiredAtForEmail" web/src/features/auth/lib/sessionExpiration.ts`.
If it doesn't, stop and tell the operator. The fix depends on it.

## 9. Phase 6: verify

1. **Health and version:** `curl -s http://localhost:3000/api/public/health`
   should report the target version.
2. **Confirm the patched image is running:** in `docker compose images`,
   `langfuse-web` should be the patched image, and worker and ai-gateway should
   be on the same version tag.
3. **Smoke test:** the operator signs in through the browser, opens a project,
   and checks that traces load and new ingestion still arrives (the worker is
   processing).
4. **Fix test (the VAPT scenario):**
   - Sign in. In browser devtools, copy the session cookie value. It's named
     `next-auth.session-token`, or `__Secure-next-auth.session-token` over HTTPS.
   - Check that the cookie works:
     `curl -s -H "Cookie: <name>=<value>" https://<host>/api/auth/session`
     should return a JSON body with `user`.
   - Sign out in the browser.
   - Run the same `curl` again. It should now return `{}` (no session). Before
     the fix, it still returned the user.
   - Optional database check:
     `docker compose exec postgres psql -U postgres -c "select email, sessions_expired_at from users where email='<operator email>';"`
     should show a timestamp from just now.
5. Watch `docker compose logs langfuse-web` for errors that mention
   `Failed to advance session revocation timestamp`.

## 10. Rollback

```bash
cd "$LF_DIR"
mv docker-compose.override.yml ~/langfuse-backups/override.failed.yml
LANGFUSE_IMAGE_TAG=<version that was running before> docker compose up -d
```

Roll back to the **same version that was running before** (from
`images-before.txt`), not to `:4` if `:4` has since moved forward. The patched
image adds no migrations, so going back to the official image of the same
version is safe. Restore the Postgres dump only if data is actually damaged:
`pg_restore --clean -U postgres -d postgres`.

## 11. Ongoing updates (the new routine)

`docker compose up --pull always` no longer covers the full upgrade, because the
web image is ours. For each upstream release `vX.Y.Z`:

1. `cd ~/langfuse-build && git fetch origin --tags && git checkout -b release/vX.Y.Z-patched vX.Y.Z`
2. Re-apply the fix (section 8) and build the image (Phase 4) as `langfuse-web-patched:X.Y.Z`.
3. Back up Postgres (Phase 2).
4. Bump all three tags in `docker-compose.override.yml` to `X.Y.Z`.
5. `docker compose pull langfuse-worker langfuse-ai-gateway && docker compose up -d`, then verify (Phase 6).
6. Optionally clean up old images: `docker image prune`.

Never bump the worker or ai-gateway tag without rebuilding web at the same tag.

**Exit plan:** the fix is a candidate for an upstream pull request to
`langfuse/langfuse`. Once it's merged and released, delete
`docker-compose.override.yml` and go back to plain
`docker compose up -d --pull always`.
