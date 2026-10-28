# Sip fork of Postiz

Sip's fork of [Postiz](https://github.com/gitroomhq/postiz-app): five small patches, one image workflow, and
the runbook for the dedicated VM that runs it. Values in `<ANGLE_BRACKETS>` are placeholders; no secret or real
value is ever written in this repo.

## 1. What this is

- Sip's public fork of `gitroomhq/postiz-app`, licensed AGPL-3.0. The source of every patch is published here,
  as the licence requires.
- Used only through the Public API, by the Sip portal. The Postiz UI is for the founder's admin operations only.
- One Postiz organisation, "Sip", holds every connected channel of every client.
- Everything Sip adds beyond the five patches lives in new files: this file and `.github/workflows/sip-image.yml`.

## 2. Base

- Upstream `gitroomhq/postiz-app` main @ `64f00fd6d0e9dfd55d202ff83983e7576e08bc2e` (2026-10-28), which is 10
  commits after upstream tag `v2.25.3` (`git describe --tags --match 'v2.25*' 64f00fd6` prints
  `v2.25.3-10-g64f00fd6`).
- Sip image tags are `<upstream tag>-sip.<n>`. The first one is `v2.25.3-sip.1`. Bump `<n>` for every Sip rebuild
  on the same upstream tag.
- Update this section on every upgrade (section 12).

## 3. The five patches

Rule: exactly these five, each its own commit, and nothing else changes upstream files. Each patch stays small so
an upgrade can re-apply it.

| Patch | Commit subject | Files | Why |
| --- | --- | --- | --- |
| a | `sip(patch-a): instagram-standalone asks only basic + content_publish scopes` | `libraries/nestjs-libraries/src/integrations/social/instagram.standalone.provider.ts` | Sip's Meta app only has the two publish permissions; comments and insights are unused, so the consent screen shows 2 scopes. |
| b | `sip(patch-b): tiktok asks only the inbox scopes` | `libraries/nestjs-libraries/src/integrations/social/tiktok.provider.ts` | Inbox mode (`UPLOAD`) needs `video.upload`; `video.list` matches the published video. No `video.publish`, no `user.info.stats`. |
| c | `sip(patch-c): public connect accepts an allowlisted redirectUrl and returns the new id` | `apps/backend/src/public-api/routes/v1/public.integrations.controller.ts`, `apps/frontend/src/components/launches/continue.integration.tsx` | The portal starts a connect through the Public API and needs the browser back on its own page, with the new integration id. |
| d | `sip(patch-d): instagram reels carry is_ai_generated` | `libraries/nestjs-libraries/src/dtos/posts/providers-settings/instagram.dto.ts`, `libraries/nestjs-libraries/src/integrations/social/instagram.provider.ts` | Reels must be labelled "AI info" when the content is AI generated. |
| e | `sip(patch-e): deleting a channel clears its tokens` | `libraries/nestjs-libraries/src/database/prisma/integrations/integration.repository.ts` | A deleted channel keeps its row for history but no usable credentials. |

## 4. Contract for the portal

What the patches mean to a Public API caller. Every call carries the header `Authorization: <API key>`.

- **Connect.** `GET https://<domain>/api/public/v1/social/instagram-standalone?redirectUrl=<urlencoded url>`
  returns `{ "url": "<provider OAuth url>" }`.
  - `redirectUrl`'s origin must be in the env var `PORTAL_RETURN_ORIGINS`: comma-separated exact origins
    (`scheme://host[:port]`, lowercase, no path, no trailing slash). Anything else, and every `redirectUrl` while
    the variable is unset, gets `400 { "msg": "redirectUrl is not allowed" }` and no OAuth state is created.
  - Without `redirectUrl` the call behaves exactly like upstream.
  - After a successful connect the browser ends on `<redirectUrl>` with its own query kept plus `added`, `msg` and
    `id=<integration id>`, for example `https://app.sipstudio.ai/cb?connect=n1` becomes
    `https://app.sipstudio.ai/cb?connect=n1&added=instagram-standalone&msg=Channel+Updated&id=<id>`.
  - A refused scope or a failed connect ends on Postiz's own error page, not on `redirectUrl` (stock behaviour).
  - `id` is the id that `GET /api/public/v1/integrations` lists and `DELETE /api/public/v1/integrations/<id>` takes.
- **Reconnecting the same account.** Reconnecting the SAME Instagram account, even after a delete, reuses the SAME
  integration row and id (upsert on organization + platform account id). So "connect new, then delete the old id"
  deletes the live channel when the account is the same: compare the ids first.
- **Posting a Reel.** `POST /api/public/v1/posts` with `type: "now"` and settings
  `{ "__type": "instagram-standalone", "post_type": "post", "is_ai_generated": true }`. Send a JSON boolean, not a
  string: the string `"false"` is converted to `true`. Reel containers always carry `is_ai_generated=true|false`
  (`false` when the setting is absent). The cover is the media item's `thumbnail` (an image URL), else
  `thumbnailTimestamp` (ms): upstream behaviour, no patch.
- **TikTok (wave 2).** Only inbox mode works: send `content_posting_method: "UPLOAD"`. `DIRECT_POST` and TikTok
  analytics need scopes the fork does not request.
- **Delete.** `DELETE /api/public/v1/integrations/<id>` soft-deletes the row and blanks `token` (to `''`) and
  `refreshToken` (to `null`). The dashboard and enterprise delete paths go through the same repository method, so
  they blank them too.
- **Not granted.** Instagram comments, insights and analytics: do not use Postiz analytics or comments for these
  channels.

## 5. Enable GitHub Actions + image build

GitHub Actions are disabled on a new fork; the founder enables them before the first image build.

1. [ ] Repo Settings → Actions → General → allow actions.
2. [ ] Disable the upstream workflows that cannot work on the fork:
   ```bash
   for w in build-containers.yml staging-conflicts.yml stale.yml deploy-aws.yml issue-label-triggers.yml codeql.yml; do
     gh workflow disable "$w" -R Sip-Inc/postiz
   done
   ```
   `build-containers.yml` matters most: it also fires on our tags and tries to push to `gitroomhq`. Keep
   `build.yml` (a free full build on every push). `.github/workflows/eslint` has no `.yml` extension, so GitHub
   does not run it; the two extension workflows are manual-only.
3. [ ] Build an image by pushing a tag, or by running the workflow against one:
   ```bash
   git tag v2.25.3-sip.1 <commit> && git push origin v2.25.3-sip.1
   gh workflow run sip-image.yml --ref v2.25.3-sip.1 -R Sip-Inc/postiz
   ```
   `Sip image` (`.github/workflows/sip-image.yml`) builds `Dockerfile.dev` for linux/amd64 and pushes
   `ghcr.io/sip-inc/postiz:<tag>`. It only runs on a tag; there is no `latest` tag, the VM pins an exact one.
4. [ ] Make the GHCR package `sip-inc/postiz` public (package settings → visibility) so the VM pulls without a token.

## 6. VM

The founder runs this later, after a separate yes. Nothing here is done by the delivery run.

```bash
# network: own VPC, so no project-wide default-allow-ssh applies
gcloud compute networks create postiz-net --subnet-mode=auto --project <PROJECT_ID>
gcloud compute firewall-rules create postiz-allow-web --network postiz-net --direction INGRESS \
  --allow tcp:80,tcp:443 --source-ranges 0.0.0.0/0 --target-tags postiz-web
gcloud compute firewall-rules create postiz-allow-iap-ssh --network postiz-net --direction INGRESS \
  --allow tcp:22 --source-ranges 35.235.240.0/20 --target-tags postiz-web

# static IP and service account (<REGION> is the region of <ZONE>)
gcloud compute addresses create postiz-ip --region <REGION>
gcloud iam service-accounts create postiz-vm --display-name "Postiz VM"

# the VM
gcloud compute instances create postiz --zone <ZONE> --machine-type e2-standard-2 \
  --network postiz-net --address postiz-ip --tags postiz-web \
  --image-family debian-12 --image-project debian-cloud \
  --boot-disk-size 50GB --boot-disk-type pd-balanced \
  --service-account postiz-vm@<PROJECT_ID>.iam.gserviceaccount.com --scopes cloud-platform \
  --shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring
```

Exactly two firewall rules: 80/443 from anywhere, SSH from IAP's range only. Temporal UI (8080) is not exposed.

SSH only through IAP:

```bash
gcloud compute ssh postiz --zone <ZONE> --tunnel-through-iap
```

On the VM, install Docker Engine and the compose plugin from Docker's official apt repository:

```bash
sudo apt-get update && sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
echo "deb [signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker   # containers with restart: always come back after a reboot
```

## 7. DNS

Create an `A` record `publish.sipstudio.ai` (or the founder's choice) pointing at the static IP of `postiz-ip`.
Create it before Caddy's first start so the certificate can be issued.

## 8. Compose

Start from `docker-compose.yaml` of the official `postiz-docker-compose` repository (the same file as
`docker-compose.yaml` at the root of this repo at the base), together with its `dynamicconfig/` directory, in
`/opt/postiz`. Apply these deltas (values are placeholders):

```yaml
services:
  postiz:
    image: ghcr.io/sip-inc/postiz:<tag>        # was ghcr.io/gitroomhq/postiz-app:latest
    env_file: postiz.env                       # replaces the whole `environment:` block
    # `ports: - '4007:5000'` removed: only Caddy is public

  caddy:
    image: caddy:2
    container_name: caddy
    restart: always
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy-data:/data
      - caddy-config:/config
    networks:
      - postiz-network

  temporal-postgresql:
    volumes:
      - temporal-postgres-data:/var/lib/postgresql/data   # named, was anonymous

  temporal:
    # `ports: - '7233:7233'` removed

  temporal-ui:
    ports:
      - '127.0.0.1:8080:8080'                  # loopback only

  # `spotlight` service removed

volumes:
  caddy-data:
  caddy-config:
  temporal-postgres-data:
```

Caddyfile (Caddy obtains and renews HTTPS by itself):

```
publish.sipstudio.ai {
  reverse_proxy postiz:5000
}
```

- `postiz.env` is root-owned and `chmod 600`, never committed. It uses `KEY=value` lines (no quotes, no `: `).
- Why the named Temporal volume: Postiz starts each channel's token-refresh workflow only at connect time and keeps
  it only in Temporal. If Temporal's database is lost, tokens silently stop refreshing (Instagram tokens die after
  about 60 days). Recovery if it ever happens: reconnect every channel.
- Keep `postgres-volume` and `postiz-uploads` as they are: both are persistent, and uploads must survive for
  `STORAGE_PROVIDER=local`.
- Optionally replace the default Postgres password through `POSTGRES_PASSWORD` in compose's own `.env`, consistent
  with `DATABASE_URL`.
- Reach the Temporal UI through the IAP tunnel:
  `gcloud compute ssh postiz --zone <ZONE> --tunnel-through-iap -- -L 8080:localhost:8080`, then open
  `http://localhost:8080`.

Start and watch:

```bash
cd /opt/postiz
docker compose up -d
docker compose logs -f postiz
```

## 9. Environment (names only)

The founder sets every value. Agents never print or commit them.

`postiz.env`:

| Name | Notes |
| --- | --- |
| `MAIN_URL`, `FRONTEND_URL` | `https://<domain>` |
| `NEXT_PUBLIC_BACKEND_URL` | `https://<domain>/api` |
| `JWT_SECRET` | random, unique to this install |
| `DATABASE_URL` | the compose Postgres, consistent with its user, password and database |
| `REDIS_URL` | the compose Redis |
| `BACKEND_INTERNAL_URL`, `TEMPORAL_ADDRESS` | as in the official compose |
| `IS_GENERAL` | as in the official compose |
| `DISABLE_REGISTRATION` | `false` only until the founder creates the "Sip" organisation and admin user, then `true` and `docker compose up -d` |
| `STORAGE_PROVIDER` | `local` |
| `UPLOAD_DIRECTORY`, `NEXT_PUBLIC_UPLOAD_DIRECTORY` | the persistent uploads volume (`/uploads` in the official compose) |
| `API_LIMIT` | set explicitly, e.g. 300 (the official compose sets 30; the code default is 90) |
| `PORTAL_RETURN_ORIGINS` | exact portal origins, see section 4 |
| `INSTAGRAM_APP_ID`, `INSTAGRAM_APP_SECRET` | the SAME Meta app as the portal (the app is not renamed) |
| `TIKTOK_CLIENT_ID`, `TIKTOK_CLIENT_SECRET` | later, wave 2 |

Compose's own `.env`: `POSTGRES_PASSWORD` (optional).

The Postiz API key of the "Sip" organisation (Settings → Public API) goes to Vercel as `POSTIZ_API_KEY`. It can
post to every client: handle it like `META_TOKEN_KEY`.

## 10. Meta / TikTok app settings

The founder does these, after a yes.

1. [ ] Meta app → Instagram business login → add the redirect
   `https://<domain>/integrations/social/instagram-standalone`.
2. [ ] Keep the portal's existing deauthorize and data-deletion URLs.
3. [ ] Later, for TikTok: redirect `https://<domain>/integrations/social/tiktok`, with the 4 scopes of patch b.

## 11. Backups

Nightly `pg_dump` of the Postiz database plus the uploads volume, to a bucket in the same project, kept 14 days.

```bash
gcloud storage buckets create gs://<BUCKET> --project <PROJECT_ID> --location <REGION>
echo '{"rule":[{"action":{"type":"Delete"},"condition":{"age":14}}]}' > lifecycle.json
gcloud storage buckets update gs://<BUCKET> --lifecycle-file=lifecycle.json
gcloud storage buckets add-iam-policy-binding gs://<BUCKET> \
  --member serviceAccount:postiz-vm@<PROJECT_ID>.iam.gserviceaccount.com --role roles/storage.objectCreator
```

`/opt/postiz/backup.sh` (`chmod +x`; use the DB user and name actually set in compose if they were changed):

```bash
#!/usr/bin/env bash
set -euo pipefail
TS=$(date -u +%Y%m%dT%H%M%SZ)
DIR=$(mktemp -d)
trap 'rm -rf "$DIR"' EXIT
docker exec postiz-postgres pg_dump -U postiz-user -d postiz-db-local -Fc > "$DIR/db-$TS.dump"
docker exec postiz tar czf - -C /uploads . > "$DIR/uploads-$TS.tgz"
gcloud storage cp "$DIR/db-$TS.dump" "$DIR/uploads-$TS.tgz" gs://<BUCKET>/
```

Nightly through `/etc/cron.d/postiz-backup`:

```
30 3 * * * root /opt/postiz/backup.sh >> /var/log/postiz-backup.log 2>&1
```

Restore outline: copy the dump into the `postiz-postgres` container and run
`pg_restore --clean -U postiz-user -d postiz-db-local <dump>` inside it; untar the uploads archive into `/uploads`
of the `postiz` container.

## 12. Upgrade

Manual. Postiz applies its schema (`prisma db push`) every time the container starts, so run `/opt/postiz/backup.sh`
before deploying a new image.

1. [ ] Fetch upstream: `git fetch https://github.com/gitroomhq/postiz-app.git --tags`, then pick the new upstream tag.
2. [ ] Move the Sip commits: `git rebase --onto <new upstream tag> <old base> <branch>`, where `<old base>` is the
   base commit recorded in section 2. Only the five `sip(patch-…)` commits and the `sip:` commit move. Resolve
   conflicts keeping each patch minimal.
3. [ ] Re-check that upstream did not change the meaning of the patched lines: the two `scopes` arrays, the public
   connect controller, `navigateOrShow`, the `postPending` Reel branch, `deleteChannel`.
4. [ ] Update section 2 (base commit, tag name).
5. [ ] Push the branch (`git push --force-with-lease origin <branch>`), tag `<new tag>-sip.1`, push the tag and let
   `Sip image` build (section 5).
6. [ ] On the VM, edit `postiz.image` in `docker-compose.yaml`, then:
   ```bash
   cd /opt/postiz
   /opt/postiz/backup.sh
   docker compose pull postiz && docker compose up -d postiz
   ```
7. [ ] Run the smoke test (section 13).

## 13. Smoke test

After the first deploy and after every upgrade.

1. [ ] HTTPS answers: `curl -sI https://<domain>`.
2. [ ] The API key works:
   `curl -s -H "Authorization: <API key>" https://<domain>/api/public/v1/is-connected` prints
   `{"connected":true}`.
3. [ ] Run the per-patch checks of section 14.
4. [ ] `sudo reboot`; when the VM is back, repeat steps 1 and 2, and check that `docker compose ps` shows every
   service up.
5. [ ] Run `/opt/postiz/backup.sh` once and see both files in `gcloud storage ls gs://<BUCKET>/`.

## 14. Manual checks per patch

There are no unit tests: upstream Postiz has no working test setup (`pnpm test` cannot run). Every patch has a
manual check instead. `<API key>` is the "Sip" organisation's Public API key.

- **a. Instagram scopes.**
  `curl -s -H "Authorization: <API key>" "https://<domain>/api/public/v1/social/instagram-standalone"`: the `url`
  contains `scope=instagram_business_basic%2Cinstagram_business_content_publish` and nothing more. Open it with a
  tester Instagram account: only those 2 permissions are shown, and the connect succeeds.
- **b. TikTok scopes.** The same call for `/social/tiktok`: the `url` contains
  `scope=user.info.basic%2Cuser.info.profile%2Cvideo.upload%2Cvideo.list`. When the TikTok sandbox exists, the
  connect succeeds.
- **c. redirectUrl.**
  - Allowed: `…/social/instagram-standalone?redirectUrl=https%3A%2F%2Fapp.sipstudio.ai%2F%3Fconnect%3Dtest` returns
    200 `{ url }`. Finish the login in a browser: it lands on
    `https://app.sipstudio.ai/?connect=test&added=instagram-standalone&msg=Channel+Updated&id=<id>`, and `<id>` is
    in `GET /api/public/v1/integrations`.
  - Refused: `redirectUrl=https%3A%2F%2Fevil.example%2F` and
    `redirectUrl=https%3A%2F%2Fapp.sipstudio.ai.evil.example%2F` both return 400 `{"msg":"redirectUrl is not allowed"}`.
- **d. is_ai_generated.** Upload a vertical mp4 (`POST /api/public/v1/upload-from-url`) and post it with
  `type: "now"`, `"is_ai_generated": true` and a `thumbnail` image: the Reel shows that cover and Instagram shows
  the "AI info" label. A second Reel without the setting has no label.
- **e. Token wipe.** `DELETE /api/public/v1/integrations/<id>`, then:
  ```bash
  docker exec -it postiz-postgres psql -U postiz-user -d postiz-db-local -c \
    "SELECT id, \"deletedAt\" IS NOT NULL AS deleted, token = '' AS token_cleared, \"refreshToken\" IS NULL AS refresh_cleared FROM \"Integration\" WHERE id = '<id>';"
  ```
  The row shows `t | t | t`.
