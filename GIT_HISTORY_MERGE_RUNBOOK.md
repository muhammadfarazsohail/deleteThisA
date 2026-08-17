# Merge One Git Repository into Another While Preserving History

This runbook merges a source repository into a target repository with a real
Git merge commit. Every source commit remains reachable in the target history,
and destination-owned files are explicitly retained.

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
git merge $SourceBranch --allow-unrelated-histories --no-ff --no-commit
```

The `--allow-unrelated-histories` option is required for the first merge when
the repositories have independent roots. It is harmless to omit it for later
synchronization merges after the histories share ancestry. `--no-ff` records a
clear integration commit even when Git could otherwise fast-forward.

## Keep files from both repositories

Before completing the merge, inspect the staged result:

```powershell
git status --short
git diff --cached --name-status
```

Git may infer that a destination file was renamed or deleted by the source,
especially when files have identical content. Restore every destination-owned
path that must remain from the target branch snapshot:

```powershell
git restore --source="$TargetRemote/$TargetBranch" --staged --worktree -- file1.cs file2.cs
git status --short
git commit -m "Merge source history into target while retaining destination files"
```

Replace `file1.cs file2.cs` with the destination paths you need to retain. The
restore changes only the merge result; it does not remove either repository's
commit history.

If both repositories contain different files at the same path, both cannot
exist at that exact path. Choose a new path for one copy before committing, for
example:

```powershell
New-Item -ItemType Directory -Force source-project
git show "$SourceBranch`:shared-name.cs" | Set-Content source-project/shared-name.cs
git add source-project/shared-name.cs
```

Review line endings and binary files after extracting a file this way. For a
large repository with many collisions, place one repository under a prefix
before merging rather than resolving every collision manually.

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
git switch $SourceBranch
git pull --ff-only
git switch -c integrate-source-update-2 "$TargetRemote/$TargetBranch"
git merge "origin/$SourceBranch" --no-ff --no-commit
# Restore destination-owned paths as shown above, review, and then commit.
```

Use a new disposable integration-branch name for each update. This avoids
resetting an older integration branch and makes the operation easier to audit.

Verify again, then push with the same commands from the previous sections.

## What preserves the history

Do not copy files into the target and create one new commit; that loses source
ancestry. The merge commit must have one parent from the target history and one
parent from the source history. Check its parents with:

```powershell
git show --no-patch --format="%H%n%P%n%s" HEAD
```

The `%P` line should contain two parent commit hashes.

## Execution record for deleteThisB into deleteThisA

1. Added `deleteThisA` as remote `projectA` and fetched its `master` branch.
2. Created an integration branch from `projectA/master`.
3. Merged the independent `deleteThisB` history with
   `--allow-unrelated-histories`.
4. Verified that both source and target tips were ancestors of the merge.
5. Published the integration to `deleteThisA/master`.
6. A later source update caused Git to infer 100% renames from `file1.cs` and
   `file2.cs` to `file3.cs` and `file4.cs`.
7. The corrective update restores `file1.cs` and `file2.cs` from the previous
   destination snapshot, keeps `file3.cs`, `file4.cs`, and `file5.cs` from the
   source, and retains both histories.
