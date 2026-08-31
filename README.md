# daily_counter

A repository whose only job is to tick a number up once a day.

`counter.txt` holds a single integer. A scheduled GitHub Actions workflow
increments it and commits the result, so the repo gets one commit per day
without anything running on a local machine.

## The daily commit workflow

`.github/workflows/daily-commit.yml`:

- Runs on a cron schedule at **06:00 UTC** every day, and can also be
  triggered by hand from the **Actions** tab via **Run workflow**
  (`workflow_dispatch`).
- Checks out the repo with `actions/checkout@v4`, authenticated with the
  repository secret `PAT` rather than the default `GITHUB_TOKEN`. The push is
  therefore made *as the token's owner*, not as `github-actions[bot]`.
- Reads `counter.txt` (creating it as `0` if missing), adds 1, writes it back.
- Sets the commit author to
  `NikolaLetvencuk <149421331+NikolaLetvencuk@users.noreply.github.com>`.
- Commits with the message `chore: daily counter bump` and pushes to the
  default branch.
- Checks `git status --porcelain counter.txt` first and exits successfully
  without committing if there is nothing to change.
- Fails fast with a clear error if the `PAT` secret is missing.

`permissions: contents: write` is declared as a safeguard. It is not required
while the checkout uses the PAT, but it means the workflow still works if the
token is ever switched back to `GITHUB_TOKEN`.

Scheduled runs are best-effort. GitHub queues them on shared infrastructure
and they can be delayed, especially on the hour, or skipped under load.

## Creating the PAT and adding it as a repo secret

The workflow needs a Personal Access Token stored as a repository secret
named exactly `PAT`.

### Option A — fine-grained token (recommended)

1. Go to **https://github.com/settings/personal-access-tokens/new**
   (Settings → Developer settings → Personal access tokens → Fine-grained
   tokens → Generate new token).
2. **Token name**: something recognizable, e.g. `daily-counter-bump`.
3. **Expiration**: pick the longest option available, or **No expiration**.
   See the warning below.
4. **Resource owner**: your own account.
5. **Repository access**: *Only select repositories* → choose this
   repository only.
6. **Permissions** → **Repository permissions** → **Contents**:
   set to **Read and write**. (Metadata: Read-only is added automatically.)
   Nothing else is needed.
7. **Generate token**, then copy the value. It is shown only once.

### Option B — classic token

1. Go to **https://github.com/settings/tokens/new**.
2. **Note**: `daily-counter-bump`. **Expiration**: longest available, or
   **No expiration**.
3. Tick the **`repo`** scope.
4. **Generate token** and copy it.

Classic tokens cannot be scoped to a single repository — the `repo` scope
grants access to every repo you can push to. Prefer Option A.

### Add it as a secret

1. In this repository: **Settings → Secrets and variables → Actions**.
2. **New repository secret**.
3. **Name**: `PAT` (exactly — the workflow reads `secrets.PAT`).
4. **Secret**: paste the token. **Add secret**.

To rotate later, generate a new token and use **Update** on the same secret.
The workflow needs no changes.

> **Set a long expiry, or diarize the renewal.** When the token expires the
> workflow starts failing on every run: no commit, and the repo goes quiet.
> Nothing warns you except the Actions failure emails.

## Disabling it

- **Actions tab** (reversible): **Actions → Daily commit → ⋯ → Disable
  workflow**. Re-enable from the same menu.
- **Delete the file**: remove `.github/workflows/daily-commit.yml`.
- **Comment out the schedule**: drop the `schedule:` block but keep
  `workflow_dispatch` so it can still be run by hand.
- **Repository-wide**: **Settings → Actions → General → Actions permissions →
  Disable actions**.

Revoking the PAT also stops it, but the workflow will keep running daily and
failing until it is disabled too.

## The 60-day inactivity caveat

GitHub automatically disables `schedule`-triggered workflows in a **public**
repository after **60 consecutive days with no repository activity**. The
owner is emailed, and the Actions tab shows the workflow disabled with a
banner offering to re-enable it.

Commits pushed with the default `GITHUB_TOKEN` are widely reported not to
count as the activity that resets this timer. Pushing as a real user with a
PAT is what makes this workflow keep itself alive.

To re-enable if it ever does get disabled: **Actions → Daily commit → Enable
workflow**. No YAML change is needed, and any push resets the 60-day clock.
