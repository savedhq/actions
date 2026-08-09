# actions

The GitHub Action for saved.sh. It installs [`sctl`](https://github.com/savedhq/sctl),
verifies its checksum, and runs one command.

Deliberately thin. All the logic lives in `sctl`, so GitLab, Jenkins and CircleCI get the
same behaviour from three lines of shell rather than a second implementation that drifts.

## Two things it does

**Trigger a backup that already exists.** The definition, its source and its credentials
live in saved.sh. The pipeline only knows an id.

```yaml
- uses: savedhq/actions@v1
  with:
    api-key: ${{ secrets.SAVED_API_KEY }}
    workspace-id: ${{ vars.SAVED_WORKSPACE_ID }}
    backup-id: 8f3c1e02-0a11-4e7d-9c2b-7d1f5a2e9b40
```

**Upload something the job just produced.** The pipeline is the source.

```yaml
- run: pg_dump "$DATABASE_URL" --format=custom | gzip -9 > dump.gz
- uses: savedhq/actions@v1
  with:
    api-key: ${{ secrets.SAVED_API_KEY }}
    workspace-id: ${{ vars.SAVED_WORKSPACE_ID }}
    backup-id: ${{ vars.SAVED_MANUAL_BACKUP_ID }}
    file: dump.gz
```

## Waiting is the default

The action blocks on the run and fails the job if it did not succeed. A step that goes
green the moment the API accepts the request is reporting that the request was accepted,
not that anything was backed up, which is the failure mode a cron job already has for free.

It waits as long as the run takes. There is no default cap, because how long your pipeline
may sit here is your decision and not ours, and GitHub already gives you `timeout-minutes`
at both job and step level to make it.

Worth knowing rather than working around: **a GitHub-hosted job is killed at 6 hours.** A
large source can legitimately take longer than that, and `dumpContext` allows a dump 6
hours on its own before post-backup work even starts. If that is your situation, a
self-hosted runner or `wait: false` are the two ways out.

Three outcomes, and only one of them is red:

| Outcome | The step |
|---|---|
| Run succeeded | Passes |
| Run failed | Fails |
| Still running when a `timeout` you set elapsed | Warns and passes, because slow is not failed |

## Gating a migration

Because waiting is the default, gating is mostly about job structure: a schema change
cannot ship without a fresh copy behind it.

```yaml
jobs:
  backup:
    runs-on: ubuntu-latest
    timeout-minutes: 90
    steps:
      - uses: savedhq/actions@v1
        with:
          api-key: ${{ secrets.SAVED_API_KEY }}
          workspace-id: ${{ vars.SAVED_WORKSPACE_ID }}
          backup-id: ${{ vars.SAVED_DB_BACKUP_ID }}

  migrate:
    needs: backup
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/migrate.sh
```

Two jobs rather than two steps, so the migration is unreachable if the backup fails.

## Inputs

| Input | Required | Default | |
|---|---|---|---|
| `api-key` | yes | | Org-scoped key. Repository secret, never inline |
| `workspace-id` | yes | | Workspace the backup belongs to |
| `backup-id` | yes | | Backup to run, or to submit to |
| `file` | no | `""` | Set it to submit an artifact; empty triggers the backup's own source |
| `wait` | no | `true` | Block on the run and fail the job if it did not succeed |
| `timeout` | no | `""` | Optional cap, as a Go duration. Empty waits as long as it takes. Hitting it warns and passes; a failed run still fails |
| `api-url` | no | `https://api.saved.sh` | Override for self-hosted |
| `version` | no | `latest` | `sctl` release tag |

Output: `run-id`.

## The key

Mint one per repository, with only the permissions that repository needs:

```bash
sctl apikey create "ci-<repo>" --perms backups:write
```

Include `runs:read` too, since the action waits by default. **Never grant
`artifacts:delete`.** A refused
call names the permission it lacked, so a key that is too narrow tells you exactly what to
add rather than failing opaquely.

## Prerequisites, not yet in place

This action is written against three things `sctl` and the backend do not do yet. All three
should land before `v1` is tagged, because waiting is the default: until the trigger
endpoint returns a run id, the trigger mode fails loudly rather than working. Submitting a
file already works, because `SubmitFile` returns a run id today.

| Needed | Where | Why |
|---|---|---|
| `SCTL_API_KEY`, and skip the WorkOS refresh when it is set | `sctl` | `backendClient()` reads `access_token` and refreshes on 401 through WorkOS. An API key set there works by accident, and its first 401 fails confusingly |
| A run id in the trigger response | `backend` | `POST /v1/backups/{id}/trigger` returns `202` with no body, so nothing can tell which run it started. Polling `run list` instead would race a scheduled run and gate on the wrong one |
| `sctl run wait RUN_ID --timeout`, exiting `0` on success, `124` on timeout, non-zero otherwise | `sctl` | Putting the poll loop in `sctl` rather than this action is what gives every other CI system the same behaviour for free. The distinct timeout code is what lets a caller tell "still running" from "failed", which is the whole difference between a warning and a red build |

A fourth is optional: an org-scoped key already implies a workspace, so `requireWorkspace()`
could read it from the key and drop the `workspace-id` input entirely.

## Other CI

There is no separate integration. Install the binary, export the key, run the command.

```sh
curl -fsSL -o sctl https://github.com/savedhq/sctl/releases/latest/download/sctl-linux-amd64
chmod +x sctl
SCTL_API_KEY="$SAVED_API_KEY" ./sctl backup trigger "$BACKUP_ID"
```
