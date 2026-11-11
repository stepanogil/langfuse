# Deploying the patched Langfuse web image on the Azure VM

> **For the Claude Code agent reading this on the VM:** this is a runbook that
> was written for you on the developer's workstation. Work through it with the
> operator one phase at a time. Before each step that changes something
> (logging in to a registry, editing compose files, pulling, restarting), show
> the command and wait for the operator to confirm it. Read-only checks can run
> without asking. If anything on the VM doesn't match the assumptions in
> section 2, stop and tell the operator before you continue.

## 1. Context

### What we run today
- Langfuse v4 is self-hosted on an Azure VM using the
  [Docker Compose method](https://langfuse.com/self-hosting/deployment/docker-compose).
- The VM has a clone of the upstream repo `https://github.com/langfuse/langfuse.git`.
  That clone is used **only for its `docker-compose.yml`**.
- Until now, updates were done with `docker compose up -d --pull always` on the
  floating `:4` tag.

### Why swapping the git clone doesn't work on its own
`docker-compose.yml` builds nothing from source. It pulls prebuilt images:

```yaml
langfuse-worker:     image: docker.langfuse.com/langfuse/langfuse-worker:${LANGFUSE_IMAGE_TAG:-4}
langfuse-web:        image: docker.langfuse.com/langfuse/langfuse:${LANGFUSE_IMAGE_TAG:-4}
langfuse-ai-gateway: image: docker.langfuse.com/langfuse/ai-gateway:${LANGFUSE_AI_GATEWAY_IMAGE_TAG:-${LANGFUSE_IMAGE_TAG:-4}}
```

So we swap the **web image**, not the clone. The fix lives in our fork and is
built into our own web image. The VM pulls that image; **the VM never builds
anything.**

### The fix we're deploying
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
- Only the web image changes. The worker, ai-gateway, Postgres, ClickHouse,
  Redis and MinIO stay on the official images. The fix adds no database
  migration.
- We maintain the fork long term; the fix won't be upstreamed.

### Where things live

| What | Where |
|---|---|
| Fork | `https://github.com/stepanogil/langfuse` |
| Patched source per release | Fork branch `release/vX.Y.Z-patched`: the upstream tag `vX.Y.Z` plus the fix as one commit |
| These runbooks | Fork branch `fix/revoke-jwt-session-on-sign-out`, folder `docs/` |
| Build pipeline | `https://github.com/stepanogil/langfuse-web-build` (private, one workflow) |
| Patched web image | `ghcr.io/stepanogil/langfuse-web:X.Y.Z-patched` (private, `linux/amd64`) |
| Making a new patched release | `docs/upgrade-fork-from-upstream.md` (workstation side) |

### Images available right now

| Version | Image | Digest | Source commit |
|---|---|---|---|
| 4.78.0 | `ghcr.io/stepanogil/langfuse-web:4.78.0-patched` | `sha256:05acc94c974933c0ddf870e1bcdaf5466cd98e64a7d5a00aa1d7f2154a4ad27f` | `3fddac6048` on `release/v4.78.0-patched` |

The immutable tag `4.78.0-patched-3fddac6` points at the same image.

### Hard rules
1. **Web, worker and ai-gateway must run the same Langfuse version.** The web
   container runs Prisma and ClickHouse migrations when it starts, and all three
   share queue and data contracts. Pin the official worker and ai-gateway to
   exactly the version of the patched web image.
2. **Never downgrade.** Database migrations only move forward. The target
   version must be **equal to or newer than** what is running now (Phase 1
   checks this).
3. Leave the upstream `docker-compose.yml` untouched. Make every change in a
   `docker-compose.override.yml` next to it, so `git pull` on the clone never
   conflicts.
4. Don't touch `.env` secrets, and don't run `docker compose down -v`. The `-v`
   flag deletes the data volumes.
5. **No more `--pull always` on the floating tag.** With the override in place,
   `--pull always` only re-pulls the pinned tags. That's harmless, but it no
   longer upgrades anything. Upgrades now go through section 10.

## 2. Assumptions to verify on the VM
- Docker Engine and the Compose v2 plugin are installed (`docker compose version`).
- The VM is **x86_64** (`uname -m` prints `x86_64`). The image is built for
  `linux/amd64` only. On an Arm VM (`aarch64`), stop: the build workflow needs a
  `linux/arm64` platform first.
- The upstream clone is in some directory. Call it `$LF_DIR`; find it with
  `docker compose ls` or by asking the operator.
- Secrets live in `$LF_DIR/.env` or are inlined in the compose file. Note which.
- Volumes: `langfuse_postgres_data`, `langfuse_clickhouse_data`,
  `langfuse_clickhouse_logs`, `langfuse_minio_data`, `langfuse_redis_data`.

## 3. Phase 1: inspect what's running (read-only)

```bash
uname -m
cd "$LF_DIR"
docker compose ls
docker compose ps
docker compose images
curl -s http://localhost:3000/api/public/health     # returns {"status":"OK","version":"4.x.y"}
df -h /var/lib/docker
git -C "$LF_DIR" log -1 --oneline
ls "$LF_DIR"/docker-compose.override.yml 2>/dev/null   # does an override already exist?
```

Write down the **running version** from the health endpoint, then pick
`TARGET` (the version without the `v`):
- **Running version ≤ the newest image in section 1** (today: ≤ 4.78.0): use
  `TARGET=4.78.0`. If the running version is older, this deploy is also an
  ordinary upgrade, the same thing `--pull always` would have done. Read the
  upstream release notes for the versions in between
  (`https://github.com/langfuse/langfuse/releases`) for breaking changes or new
  required env vars.
- **Running version newer than every available image:** stop. A patched image
  for that version has to be made first (`docs/upgrade-fork-from-upstream.md`).
  Deploying an older image would be a downgrade.

Also check the compose file against the target release. If upstream changed
`docker-compose.yml` (new services, env vars, volumes), update the clone first:

```bash
git -C "$LF_DIR" fetch --tags origin
git -C "$LF_DIR" diff HEAD "v${TARGET}" -- docker-compose.yml   # empty = nothing to do
```

If it isn't empty, show the diff to the operator. Updating the clone to the
target tag (`git -C "$LF_DIR" checkout "v${TARGET}"`) is the usual answer, but
it changes a running system, so ask first.

## 4. Phase 2: back up before you change anything

```bash
cd "$LF_DIR"
mkdir -p ~/langfuse-backups
docker compose exec -T postgres pg_dump -U "${POSTGRES_USER:-postgres}" -d "${POSTGRES_DB:-postgres}" -Fc \
  > ~/langfuse-backups/postgres-$(date +%Y%m%d-%H%M).dump
cp docker-compose.yml ~/langfuse-backups/
[ -f docker-compose.override.yml ] && cp docker-compose.override.yml ~/langfuse-backups/override.before.yml
[ -f .env ] && cp .env ~/langfuse-backups/env.bak
docker compose images > ~/langfuse-backups/images-before.txt
curl -s http://localhost:3000/api/public/health > ~/langfuse-backups/health-before.json
```

Check that the dump file isn't empty (`ls -lh ~/langfuse-backups`). If Azure
disk snapshots are available, taking one of the OS and data disks is an even
better rollback point.

## 5. Phase 3: log in to GHCR (once per VM)

The image is private. The VM needs a GitHub **classic** personal access token
with only the `read:packages` scope; GHCR doesn't accept fine-grained tokens for
this. The operator creates it at <https://github.com/settings/tokens> and
types it in themselves. Don't put it in a file, the shell history, or this doc.

```bash
docker login ghcr.io -u stepanogil      # paste the token at the password prompt
docker pull "ghcr.io/stepanogil/langfuse-web:${TARGET}-patched"
docker image inspect "ghcr.io/stepanogil/langfuse-web:${TARGET}-patched" \
  --format '{{index .RepoDigests 0}}  {{index .Config.Labels "org.opencontainers.image.revision"}}'
```

The digest and revision should match the table in section 1. Docker keeps the
login in `~/.docker/config.json`, so later deploys skip this phase until the
token expires.

## 6. Phase 4: point compose at the patched image and deploy

Create `$LF_DIR/docker-compose.override.yml`. If one already exists, **merge**
these keys into it; don't overwrite it.

```yaml
services:
  langfuse-web:
    image: ghcr.io/stepanogil/langfuse-web:4.78.0-patched
  langfuse-worker:
    image: docker.langfuse.com/langfuse/langfuse-worker:4.78.0
  langfuse-ai-gateway:
    image: docker.langfuse.com/langfuse/ai-gateway:4.78.0
```

Use `${TARGET}` in place of `4.78.0` if it differs. All three must carry the
same version. Compose merges `docker-compose.override.yml` automatically.

```bash
cd "$LF_DIR"
docker compose config | grep -E 'image:'        # confirm the three pinned images
docker compose pull langfuse-web langfuse-worker langfuse-ai-gateway
docker compose up -d
docker compose logs -f langfuse-web              # wait for migrations and "Ready"; Ctrl-C to stop following
```

## 7. Phase 5: verify

1. **Health and version:** `curl -s http://localhost:3000/api/public/health`
   should report `${TARGET}`.
2. **Confirm the patched image is running:** in `docker compose images`,
   `langfuse-web` should show `ghcr.io/stepanogil/langfuse-web` with tag
   `${TARGET}-patched`, and worker and ai-gateway should show `${TARGET}`.
3. **Smoke test:** the operator signs in through the browser, opens a project,
   and checks that traces load and new ingestion still arrives (the worker is
   processing). `docker compose logs --since 10m langfuse-worker` should be
   free of repeating errors.
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

## 8. Rollback

Roll back to the **official image of the version that's now running** (the
target), not to the version from before the deploy. If this deploy was also an
upgrade, the migrations have already run, and the older version may not work
with the newer schema.

```bash
cd "$LF_DIR"
mv docker-compose.override.yml ~/langfuse-backups/override.failed.yml
LANGFUSE_IMAGE_TAG=${TARGET} docker compose up -d
```

The patched image adds no migrations, so switching to the official image of the
same version is safe. It brings back the sign-out issue, though, so treat it as
temporary. Restore the Postgres dump only if data is actually damaged:
`pg_restore --clean -U postgres -d postgres`.

## 9. Fallback: build the image on the VM

Use this only if GHCR or the build workflow is unavailable. The Next.js build
needs about **8 GB of free RAM** and 30 GB of free disk, and competes with the
running stack.

```bash
cd ~
git clone --branch "release/v${TARGET}-patched" --depth 2 https://github.com/stepanogil/langfuse.git langfuse-build
cd langfuse-build
git log --oneline -2          # the fix commit on top of "chore: release v${TARGET}"
docker build -f web/Dockerfile -t "langfuse-web-patched:${TARGET}" .
```

Run it from the repo root (the trailing `.`). Don't pass any `NEXT_PUBLIC_*`
build args: the official image is built without them, and they get baked in at
build time. In the override, use `image: langfuse-web-patched:${TARGET}` with
`pull_policy: never`, and leave `langfuse-web` out of `docker compose pull`.

## 10. Ongoing updates (the new routine)

For each upstream release `X.Y.Z` you want to run:
1. **Workstation:** create `release/vX.Y.Z-patched` and run the build workflow
   (`docs/upgrade-fork-from-upstream.md`). The result is
   `ghcr.io/stepanogil/langfuse-web:X.Y.Z-patched`.
2. **VM:** Phase 1 (check the version), Phase 2 (back up), change all three
   tags in `docker-compose.override.yml` to `X.Y.Z`, Phase 4 (`pull`, `up -d`),
   Phase 5 (verify).
3. Optionally clean up old images: `docker image prune`.

Never bump the worker or ai-gateway tag without a patched web image of the same
version.

If an upstream release ever fixes sign-out revocation on its own, check it with
the Phase 5 test against the official image. Then delete
`docker-compose.override.yml` and go back to plain
`docker compose up -d --pull always`.
