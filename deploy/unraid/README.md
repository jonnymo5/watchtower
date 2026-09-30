# Running Watchtower on Unraid (wolverine)

This fork exists for supply-chain control: the Watchtower that runs on
wolverine — which has full access to the Docker socket — is compiled from this
repository, at a commit you chose to merge, by
`.github/workflows/unraid-image.yaml`. Upstream's images and release pipeline
are not used.

```
push to main (jonnymo5/watchtower)
  └─► GitHub Actions: unraid-image
        go test, build linux/amd64 from Dockerfile.self-local, smoke test, push
        ghcr.io/jonnymo5/watchtower:latest + :sha-<hash>
          └─► the running Watchtower sees its own :latest changed and updates itself
```

## What it manages

Only containers with the label `com.centurylinklabs.watchtower.enable=true`:

| Container       | Image                                   | Stack                                  |
| --------------- | --------------------------------------- | -------------------------------------- |
| `watchtower`    | `ghcr.io/jonnymo5/watchtower:latest`    | this directory                         |
| `actual-server` | `ghcr.io/jonnymo5/actual-server:latest` | `jonnymo5/actual-budget` deploy/unraid |

To opt another container in (e.g. `daventry-db`), add that label in its compose
file and Compose Up. Containers pinned to an immutable tag (`:sha-<hash>`) are
effectively frozen even if labelled.

## One-time setup

1. **Enable Actions on the fork** (GitHub → Actions tab → enable), then run
   the `unraid-image` workflow once (Actions → unraid-image → Run workflow).
2. **GHCR visibility**: open
   https://github.com/users/jonnymo5/packages/container/watchtower/settings and
   set visibility to **public** (the repo is public, Apache-2.0). Wolverine then
   pulls without a login.
3. **Start the stack** with Compose Manager: add a stack `watchtower`, paste in
   `deploy/unraid/docker-compose.yml`, Compose Up.
4. **Verify** on wolverine: `docker logs watchtower` should show the scheduled
   poll and the list of watched containers.

## Day-2 operations

- **Pull in upstream changes**: review, then merge and push when you want them:

  ```sh
  git fetch upstream && git log main..upstream/main   # review first
  git merge upstream/main && git push origin main
  ```

  (one-time: `git remote add upstream https://github.com/nicholas-fedor/watchtower.git`).
- **Rollback**: pin `image: ghcr.io/jonnymo5/watchtower:sha-<hash>` in the
  compose file and Compose Up. Switch back to `:latest` to resume self-updates.
- **Force a check now**: `docker restart watchtower` — it checks immediately
  on start (`WATCHTOWER_UPDATE_ON_START`), then resumes the 5-minute schedule.
- **Why not the Unraid "Auto Update Applications" plugin?** It only handles
  containers created from Unraid Docker templates, not Compose Manager stacks.
