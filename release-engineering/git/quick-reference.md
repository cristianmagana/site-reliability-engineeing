# Git Quick Reference for Release Engineering

## Decision Trees

### Which Merge Strategy Should I Use?

```
Do you want to preserve feature branch commits individually?
│
├─ YES ──> Are there multiple commits worth preserving?
│          │
│          ├─ YES ──> Use: Rebase + Merge (--ff-only)
│          │          Result: Linear history, all commits preserved
│          │          Command: git rebase main && git merge --ff-only
│          │
│          └─ NO ──> Use: Merge Commit (--no-ff)
│                    Result: One merge commit, shows branch structure
│                    Command: git merge --no-ff feature
│
└─ NO ──> Use: Squash Merge
          Result: One commit representing entire feature
          Command: git merge --squash feature && git commit
```

### When to Rebase vs Merge?

```
Has the branch been pushed to a shared repository?
│
├─ YES ──> MERGE (never rebase public branches)
│          Exception: Your own feature branch with force-push allowed
│
└─ NO ──> Is the history messy (typos, WIP commits)?
           │
           ├─ YES ──> REBASE (clean up before merging)
           │          Use: git rebase -i to squash/reorder
           │
           └─ NO ──> Either works, choose based on team preference
                      Rebase: Linear history
                      Merge: Preserve true development flow
```

### Branching Strategy → Merge Strategy

| Branching Strategy | Primary Merge Method | Why |
|-------------------|---------------------|-----|
| **GitFlow** | `--no-ff` merge commits | Preserve release/hotfix branches, audit trail |
| **Trunk-Based Development** | Rebase + squash | Clean, linear main branch |
| **GitHub Flow** | Squash or rebase | One PR = one feature, clean history |

---

## Command Cheat Sheet

### Merge Strategies

```bash
# Fast-forward (if possible)
git merge feature

# Force fast-forward only (fail if not possible)
git merge --ff-only feature

# Always create merge commit
git merge --no-ff feature -m "Merge: feature"

# Squash all commits into one (doesn't commit automatically)
git merge --squash feature
git commit -m "feat: description"

# Abort a merge in progress
git merge --abort
```

### Rebasing

```bash
# Rebase current branch onto main
git rebase main

# Interactive rebase (clean up last 5 commits)
git rebase -i HEAD~5

# Rebase and resolve conflicts
git rebase main
# ... resolve conflicts ...
git add <resolved-files>
git rebase --continue

# Abort rebase
git rebase --abort

# Rebase onto different base
git rebase --onto new-base old-base feature

# Pull with rebase (instead of merge)
git pull --rebase origin main
```

### Viewing History

```bash
# Concise graph of all branches
git log --oneline --graph --all --decorate

# With dates and authors
git log --graph --pretty=format:'%h - %s (%cr) <%an>'

# Show commits in branch1 not in branch2
git log branch2..branch1

# Show all commits that touched a file
git log --follow -- path/to/file

# Show who changed each line
git blame path/to/file

# Show changes in a commit
git show <commit-sha>

# Show files changed in a commit
git show --name-only <commit-sha>
```

### Fixing Mistakes

```bash
# Amend last commit (add changes or fix message)
git commit --amend
git commit --amend -m "New message"

# Undo last commit, keep changes staged
git reset --soft HEAD~1

# Undo last commit, keep changes unstaged
git reset --mixed HEAD~1  # or just: git reset HEAD~1

# Undo last commit, discard changes
git reset --hard HEAD~1

# Undo a public commit (creates new commit)
git revert <commit-sha>

# Undo a merge commit
git revert -m 1 <merge-commit-sha>

# Recover lost commits
git reflog
git reset --hard HEAD@{N}
```

### Cherry-Pick

```bash
# Apply specific commit to current branch
git cherry-pick <commit-sha>

# Apply range of commits
git cherry-pick start-sha..end-sha

# Cherry-pick without committing (stage only)
git cherry-pick -n <commit-sha>

# Abort cherry-pick
git cherry-pick --abort
```

### Bisecting

```bash
# Start bisecting
git bisect start

# Mark current commit as bad
git bisect bad

# Mark old commit as good
git bisect good <commit-sha>

# Mark current as good/bad after testing
git bisect good
git bisect bad

# Automate bisecting with a test
git bisect run ./test.sh

# End bisecting
git bisect reset
```

### Stashing

```bash
# Save current changes
git stash

# Save with message
git stash push -m "WIP: feature in progress"

# List stashes
git stash list

# Apply most recent stash
git stash apply

# Apply and remove most recent stash
git stash pop

# Apply specific stash
git stash apply stash@{2}

# Create branch from stash
git stash branch new-branch-name

# Delete stash
git stash drop stash@{0}
```

### Tags

```bash
# Create lightweight tag
git tag v1.2.3

# Create annotated tag
git tag -a v1.2.3 -m "Release version 1.2.3"

# Tag specific commit
git tag v1.2.3 <commit-sha>

# Push tags to remote
git push origin v1.2.3
git push origin --tags  # All tags

# Delete tag locally
git tag -d v1.2.3

# Delete tag on remote
git push origin :refs/tags/v1.2.3
git push origin --delete v1.2.3

# List tags
git tag
git tag -l "v1.*"
```

---

## "Oh Crap" Scenarios

### "I committed to the wrong branch"

```bash
# Move commit to correct branch
git checkout correct-branch
git cherry-pick <commit-sha>

git checkout wrong-branch
git reset --hard HEAD~1
```

### "I need to undo a public commit"

```bash
# Don't use reset! Use revert:
git revert <bad-commit-sha>
git push origin main
```

### "I accidentally deleted a branch"

```bash
# Find the commit
git reflog | grep "branch-name"

# Recreate branch
git branch branch-name <commit-sha>
```

### "I rebased and now everything is broken"

```bash
# Abort rebase if still in progress
git rebase --abort

# Or find previous state in reflog
git reflog
git reset --hard HEAD@{N}  # Before the rebase
```

### "I pushed a commit with secrets"

```bash
# Remove from history (DANGEROUS - rewrites history)
git filter-branch --tree-filter 'rm -f secrets.txt' HEAD

# Or use BFG Repo-Cleaner (faster)
bfg --delete-files secrets.txt

# Force push (breaks others' repos!)
git push --force origin main

# Better: Rotate the secrets immediately!
```

### "I need to sync my fork with upstream"

```bash
# Add upstream remote (once)
git remote add upstream https://github.com/original/repo.git

# Fetch and merge
git fetch upstream
git checkout main
git merge upstream/main

# Or rebase
git rebase upstream/main

git push origin main
```

### "My merge conflict is too complex"

```bash
# Abort and try differently
git merge --abort

# Try with a different strategy
git merge -X theirs feature  # Favor their changes
git merge -X ours feature    # Favor our changes

# Or use a visual tool
git mergetool
```

---

## Configuration Snippets

### Global Config for Better Git Experience

```bash
# Better log default
git config --global alias.lg "log --oneline --graph --all --decorate"

# Show diff when committing
git config --global commit.verbose true

# Rebase on pull by default
git config --global pull.rebase true

# Only allow fast-forward merges
git config --global merge.ff only

# Better diff algorithm
git config --global diff.algorithm histogram

# Auto-correct typos
git config --global help.autocorrect 20
```

### Project-Specific Config

```bash
# In your repo
cd /path/to/repo

# Enforce linear history
git config merge.ff only

# Default branch name
git config init.defaultBranch main

# Required sign-off for commits
git config commit.gpgsign true
```

---

## Interview Response Templates

### "How do you handle merges?"

**Template**:
```
We use [STRATEGY] because we optimize for [GOAL].

This gives us [BENEFIT] but trades off [TRADE-OFF].

We mitigate [TRADE-OFF] by [MITIGATION].

[Example of how this helped in production]
```

**Example**:
```
We use squash merges for PRs to main because we optimize for clean,
bisectable history.

This gives us linear main branch where each commit is a complete feature,
but trades off losing granular development history.

We mitigate this by:
1. Requiring detailed PR descriptions
2. Linking to issue tracker
3. Keeping feature branches accessible for 90 days

Last month this helped us quickly bisect a performance regression -
with 200 commits, we found the culprit in 8 iterations instead of
having to wade through merge commits and WIP commits.
```

### "Tell me about a time you used Git to debug"

**Structure**: STAR format
- **Situation**: Production issue description
- **Task**: What you needed to find
- **Action**: Git commands you used
- **Result**: How you resolved it + learning

**Example**:
```
Situation: Payment processing started failing for 2% of transactions
after a release with 50+ commits.

Task: Identify which commit introduced the regression.

Action: I used git bisect between the last known good tag (v2.3.0)
and current HEAD. I wrote a script that reproduced the failing
transaction and used 'git bisect run ./test.sh' to automate the search.

Result: Found the culprit in 7 iterations - a commit that changed
floating-point rounding. We reverted it, tagged v2.3.2, and deployed
within an hour.

Learning: This reinforced why we enforce: (1) commits should be atomic
and (2) our commit messages should explain WHY, not just WHAT, so when
bisect finds the commit, we immediately understand the context.
```

### "Merge vs Rebase - which is better?"

**Framework**:
1. Acknowledge neither is universally better
2. Explain what each does technically
3. Discuss trade-offs
4. Give context-specific recommendation

**Example**:
```
Neither is universally better - it depends on what you optimize for.

Technically: Merge creates a new commit with two parents, preserving
the true development history. Rebase replays commits onto a new base,
rewriting history to be linear.

Trade-offs:
- Merge: True history, can debug actual development flow, but creates
  non-linear history that's harder to read
- Rebase: Clean linear history, easier to bisect, but rewrites commits
  (breaks if done on public branches)

My rule: Never rebase public/shared branches. For private feature
branches, I rebase onto main before creating a PR to ensure CI runs
against current code and history is clean. After PR is approved, we
squash-merge to main for linear main branch history.

This gives us: clean main branch + preserved development history in PRs.
```

---

## Common Workflows

### GitFlow Release

```bash
# Start release
git checkout develop
git checkout -b release/1.2.0

# Bump version
echo "1.2.0" > version.txt
git commit -am "chore: bump version to 1.2.0"

# Stabilization (bug fixes only)
# ... fix bugs ...

# Merge to main
git checkout main
git merge --no-ff release/1.2.0
git tag v1.2.0

# Merge back to develop
git checkout develop
git merge --no-ff release/1.2.0

# Clean up
git branch -d release/1.2.0
git push origin main develop --tags
```

### GitFlow Hotfix

```bash
# Start hotfix
git checkout main
git checkout -b hotfix/1.2.1

# Fix bug
# ... make changes ...
git commit -am "fix: critical payment bug"

# Merge to main
git checkout main
git merge --no-ff hotfix/1.2.1
git tag v1.2.1

# Merge to develop
git checkout develop
git merge --no-ff hotfix/1.2.1

# Clean up
git branch -d hotfix/1.2.1
git push origin main develop --tags
```

### Trunk-Based Development

```bash
# Start small feature
git checkout -b quick-feature

# Make focused changes
# ... changes ...
git commit -am "feat: add user export"

# Rebase onto latest main
git fetch origin
git rebase origin/main

# Clean up commits if needed
git rebase -i HEAD~3

# Merge to main
git checkout main
git pull --rebase
git merge quick-feature  # Fast-forward

# Tag for release (if ready)
git tag v1.3.0

# Push
git push origin main --tags

# Clean up
git branch -d quick-feature
```

### GitHub Flow (PR-based)

```bash
# Create feature branch
git checkout -b feature/new-api

# Develop feature
# ... commits ...

# Push to remote
git push -u origin feature/new-api

# Create PR via web UI
# ... code review ...

# After approval, squash merge via web UI
# Or from CLI:
git checkout main
git merge --squash feature/new-api
git commit -m "feat: add new API endpoint (#123)"

# Push and delete branch
git push origin main
git push origin --delete feature/new-api
git branch -d feature/new-api
```

---

## Git Flags Explained

### Merge Flags

- `--ff` (default): Fast-forward if possible, merge commit if not
- `--ff-only`: Only merge if fast-forward is possible, fail otherwise
- `--no-ff`: Always create merge commit, even if fast-forward is possible
- `--squash`: Combine all commits into one, stage but don't commit
- `-X ours`: Favor our changes in conflicts
- `-X theirs`: Favor their changes in conflicts
- `--abort`: Cancel merge in progress

### Rebase Flags

- `-i`, `--interactive`: Interactive rebase (pick/squash/edit/etc)
- `--onto <newbase>`: Rebase onto different branch
- `--continue`: Continue after resolving conflicts
- `--skip`: Skip current commit
- `--abort`: Cancel rebase
- `-X ours`: Favor our changes in conflicts
- `-X theirs`: Favor their changes in conflicts

### Reset Flags

- `--soft`: Move HEAD, keep index and working dir
- `--mixed` (default): Move HEAD, reset index, keep working dir
- `--hard`: Move HEAD, reset index and working dir (DESTRUCTIVE)

### Log Flags

- `--oneline`: Compact one-line-per-commit format
- `--graph`: ASCII graph of branch structure
- `--all`: Show all branches
- `--decorate`: Show branch and tag names
- `--follow`: Follow file renames
- `-p`: Show patch (diff) for each commit
- `--stat`: Show file statistics
- `--since="2 weeks ago"`: Filter by date
- `--author="name"`: Filter by author

---

## Performance Tips

```bash
# Shallow clone (faster, less disk)
git clone --depth=1 https://github.com/user/repo.git

# Partial clone (large repos)
git clone --filter=blob:none https://github.com/user/repo.git

# Sparse checkout (monorepos)
git sparse-checkout init
git sparse-checkout set path/to/dir

# Optimize local repo
git gc --aggressive
git prune

# Remove untracked files
git clean -fd  # Force remove files and directories
```

---

## This Should Be Muscle Memory

By interview time, these should be instant:

```bash
# View clean history
git log --oneline --graph --all

# Undo last commit (keep changes)
git reset HEAD~1

# Amend last commit
git commit --amend

# Recover from disaster
git reflog

# Find what broke
git bisect start
git bisect bad
git bisect good <sha>

# Apply specific fix
git cherry-pick <sha>

# Clean up before merging
git rebase -i HEAD~N
```

Practice these until they're automatic. In a production incident, you don't want to be looking up syntax.
