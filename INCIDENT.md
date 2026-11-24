# Incident: registry token committed to `configs/dev.env` (Lab 6 drill)

> Training drill. The token `ghp_THIS_IS_A_FAKE_TOKEN_FOR_THE_DRILL` is fake and was never pushed.

## What happened

1. Commit "chore: tune dev config" added `FRAUD_REGISTRY_TOKEN=ghp_...` to `configs/dev.env`.
2. Commit "chore: remove token" deleted the line.
3. `git log -p --all -S "ghp_THIS_IS" --oneline` still shows the token in the first commit.
   Deleting a line removes it from the latest version only, not from the history. Anyone with a
   clone (or a fork, a CI cache or a mirror) can still read it.

## Response, in this order

1. **Rotate the token now.** Revoke it in the registry and issue a new one. Put the new one
   only in the platform's secret store (`FRAUD_REGISTRY_TOKEN`), never in a file in the repo.
   Do this first: once a secret has been pushed, treat it as stolen. Cleaning the history
   does not un-leak it, and every minute it stays valid is a minute it can be used.
2. **Clean the history.** Remove the token from every commit (`git filter-repo --replace-text`
   or BFG), force-push, and ask everyone with a clone to clone again. Check the registry's
   audit log for any use of the old token between the leak and the revocation.
3. **Add a scanner so it cannot happen again.** Run a secret scanner (for example gitleaks)
   as a pre-commit hook and as a CI job, and turn on GitHub secret scanning and push
   protection for the repository.

## Why `git reset --hard HEAD~2` was enough in this drill

The two commits existed only on one laptop. Nobody had pulled them, CI had not seen them, and
GitHub had no copy. Dropping them locally removed the only copy. Once a commit has been pushed,
a local reset changes nothing for the remote, the forks or the caches, so step 1 (rotate) is
always the first move.
