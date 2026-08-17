# Merge One Git Repository into Another While Preserving History

This runbook merges a source repository into a target repository with a real
Git merge commit. Every source commit remains reachable in the target history.

## Values used for this merge

```text
Source repository: https://github.com/muhammadfarazsohail/deleteThisB.git
Source branch:     master
Target repository: https://github.com/muhammadfarazsohail/deleteThisA.git
Target branch:     master
Target remote:     projectA
Integration branch: integrate-deleteThisB
```

## Before starting

- Commit or stash all local changes.
- Confirm which repository is the source and which is the target.
- Confirm both default branches; do not assume they are named `master`.
- Ensure you have permission to push to the target repository.
- Back up or protect the target branch if it contains important production work.

## Reusable procedure

Run these commands from a clean clone of the **source** repository. Replace the
placeholder values with those for your repositories.

```powershell
$TargetUrl = "https://github.com/OWNER/TARGET.git"
$TargetRemote = "target-project"
$TargetBranch = "main"
$SourceBranch = "main"
$IntegrationBranch = "integrate-source-history"

git status --short --branch
git remote add $TargetRemote $TargetUrl
git fetch $TargetRemote --prune
git switch $SourceBranch
git pull --ff-only
git switch -c $IntegrationBranch "$TargetRemote/$TargetBranch"
git merge $SourceBranch --allow-unrelated-histories --no-ff -m "Merge source repository history into target"
```

The `--allow-unrelated-histories` option is required for the first merge when
the repositories have independent roots. It is harmless to omit it for later
synchronization merges after the histories share ancestry. `--no-ff` records a
clear integration commit even when Git could otherwise fast-forward.

If the remote name already exists, update and fetch it instead:

```powershell
git remote set-url $TargetRemote $TargetUrl
git fetch $TargetRemote --prune
```

## Resolve conflicts if Git stops

Inspect conflicts:

```powershell
git status
git diff --name-only --diff-filter=U
```

Edit each conflicted file, remove the conflict markers, and then continue:

```powershell
git add <resolved-files>
git commit
```

To abandon an unresolved merge and return to the pre-merge state:

```powershell
git merge --abort
```

## Verify history before publishing

```powershell
git status --short --branch
git log --graph --oneline --decorate --all -30
git merge-base --is-ancestor "$TargetRemote/$TargetBranch" HEAD
git merge-base --is-ancestor $SourceBranch HEAD
git diff --check "$TargetRemote/$TargetBranch..HEAD"
```

Both `merge-base --is-ancestor` commands must exit with code `0`. That proves
the integration commit contains both the prior target history and the complete
source-branch history.

Publish only after verification:

```powershell
git push $TargetRemote "HEAD:$TargetBranch"
```

Finally, confirm that the remote branch points at the integration commit:

```powershell
git ls-remote $TargetUrl "refs/heads/$TargetBranch"
git rev-parse HEAD
```

The two commit hashes should match.

## Subsequent updates

After the first successful history-preserving merge, fetch both repositories
and repeat the integration from the latest target branch:

```powershell
git fetch origin --prune
git fetch $TargetRemote --prune
git switch $IntegrationBranch
git reset --hard "$TargetRemote/$TargetBranch"
git merge "origin/$SourceBranch" --no-ff -m "Merge latest source history into target"
```

`git reset --hard` discards local changes on the integration branch. Use it only
after confirming the working tree is clean. A safer alternative is to delete
and recreate the disposable integration branch after switching away from it.

Verify again, then push with the same commands from the previous sections.

## What preserves the history

Do not copy files into the target and create one new commit; that loses source
ancestry. The merge commit must have one parent from the target history and one
parent from the source history. Check its parents with:

```powershell
git show --no-patch --format="%H%n%P%n%s" HEAD
```

The `%P` line should contain two parent commit hashes.
