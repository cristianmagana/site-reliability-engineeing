# Git Mechanics: How History Actually Works

## Why This Matters for Release Engineering

In an interview, anyone can say "we use squash merges" or "we rebase before merging." What separates you is understanding:

- **What actually happens** to the commit graph with each approach
- **Why** different histories affect debugging, rollbacks, and bisecting
- **Trade-offs** between clean history vs. preserving context
- **How** history affects your ability to understand production incidents
- **When** to use each technique (not just "what the team does")

When production breaks and you need to `git bisect` to find the bad commit, or when you need to cherry-pick a hotfix to a release branch, understanding Git's internals becomes critical.

---

## Part 1: What Git Actually Is

### The Mental Model You Need

Git is not a version control system that stores diffs. It's a **content-addressed filesystem** with a version control interface on top.

**Key Insight**: Every commit is a snapshot of your entire working directory, addressed by the SHA-1 hash of its contents.

```
Commit Object:
├─ tree      (SHA of directory snapshot)
├─ parent    (SHA of previous commit)
├─ author    (who wrote it)
├─ committer (who committed it)
└─ message   (commit message)
```

**What this means**:

- Commits don't contain diffs - they point to complete directory snapshots
- Commits form a directed acyclic graph (DAG), not a linear history
- Branches are just pointers to commits (stored in `.git/refs/heads/`)
- Tags are also just pointers to commits (stored in `.git/refs/tags/`)
- `HEAD` is a pointer to the current branch (stored in `.git/HEAD`)

### The Three Trees

Git manages three trees:

1. **Working Directory**: What you see in your filesystem
2. **Index (Staging Area)**: What you've marked for the next commit
3. **HEAD**: The last commit on the current branch

```
Working Dir    →  git add  →    Index    →  git commit  →    HEAD
 (filesystem)                  (staging)                   (last commit)
```

**Commands map to trees**:

```bash
git diff            # Working vs Index (unstaged changes)
git diff --staged   # Index vs HEAD (staged changes)
git diff HEAD       # Working vs HEAD (all changes)
```

---

## Part 2: Merge Strategies - What Actually Happens

### Fast-Forward Merge (--ff)

**When it happens**: The target branch hasn't diverged from the feature branch.

```
Before:
main     A---B
              \
feature        C---D

After (git merge feature):
main     A---B---C---D
feature              ^
```

**What Git does**:

1. Checks if main is an ancestor of feature
2. Simply moves the main pointer forward to D
3. No new commit created
4. Linear history preserved

**Command**:

```bash
git checkout main
git merge feature  # Fast-forward if possible
```

**Actual Git output**:

```
Updating abc123..def456
Fast-forward
 file.txt | 2 ++
 1 file changed, 2 insertions(+)
```

**Trade-offs**:

- ✅ Clean, linear history
- ✅ No merge commit clutter
- ❌ Loses information about feature branch existence
- ❌ Can't tell which commits were part of the same feature

### No-Fast-Forward Merge (--no-ff)

**When to use**: You want to preserve the fact that commits were developed on a branch.

```
Before:
main     A---B
              \
feature        C---D

After (git merge --no-ff feature):
main     A---B-------M
              \     /
feature        C---D
```

**What Git does**:

1. Creates a merge commit M with two parents (B and D)
2. M's tree is the result of merging B and D
3. Feature branch topology preserved in history

**Command**:

```bash
git checkout main
git merge --no-ff feature -m "Merge feature branch"
```

**Trade-offs**:

- ✅ Preserves feature branch context
- ✅ Easy to revert entire feature (revert the merge commit)
- ✅ History shows "this group of commits was a feature"
- ❌ More merge commits in history
- ❌ Harder to read `git log` without graphing

### Squash Merge (--squash)

**When to use**: You want one commit on main representing the entire feature.

```
Before:
main     A---B
              \
feature        C---D---E
               (3 commits)

After (git merge --squash feature):
main     A---B---S

feature        C---D---E
               (still exists, but unrelated to S)
```

**What Git does**:

1. Takes all changes from C, D, E
2. Combines them into staging area
3. Creates ONE new commit S on main
4. S has only one parent (B)
5. Feature branch history is discarded

**Commands**:

```bash
git checkout main
git merge --squash feature
# At this point changes are staged, not committed
git commit -m "feat: implement user authentication

- Add login endpoint
- Implement JWT tokens
- Add password hashing"
```

**Critical Detail**: `--squash` doesn't create a commit automatically. It stages the changes, you must commit manually.

**Trade-offs**:

- ✅ Clean, linear history on main
- ✅ One commit = one feature (easy to revert)
- ✅ Main branch stays readable
- ❌ Loses granular commit history
- ❌ Can't `git bisect` within the feature
- ❌ Harder to review what happened during development

### Three-Way Merge (Default Merge Commit)

**When it happens**: Branches have diverged (both have unique commits).

```
Before:
          C---D   feature
         /
main  A---B---E---F

After (git merge feature):
          C---D
         /     \
main  A---B---E---F---M
```

**What Git does**:

1. Finds common ancestor (B)
2. Compares B→D and B→F
3. Three-way merge algorithm combines changes
4. Creates merge commit M with two parents (F and D)

**How three-way merge works**:

```
Common Ancestor (B):  line 1: hello
Branch 1 (F):         line 1: hello
                      line 2: world
Branch 2 (D):         line 1: goodbye

Result (M):           line 1: goodbye  (changed in D)
                      line 2: world    (added in F)
```

**Commands**:

```bash
git checkout main
git merge feature
# Git creates merge commit automatically
```

**Conflicts happen when**:

- Same line changed in both branches
- File deleted in one branch, modified in other
- Binary file changed in both branches

**Trade-offs**:

- ✅ Preserves complete history from both branches
- ✅ Non-destructive (doesn't rewrite commits)
- ✅ Can bisect and debug in both branches
- ❌ Creates non-linear history
- ❌ Merge commits add noise

---

## Part 3: Rebasing - Rewriting History

### What Rebase Actually Does

**Fundamental Insight**: Rebase rewrites commits. It doesn't move them - it creates new commits with new SHAs.

```
Before:
          C---D   feature (original commits)
         /
main  A---B---E---F

After (git rebase main):
                  C'--D'  feature (NEW commits, different SHAs)
                 /
main  A---B---E---F
```

**What Git does step-by-step**:

1. Find common ancestor (B)
2. Store diffs from C and D
3. Reset feature branch to F
4. Apply diff from C → creates C' (new SHA)
5. Apply diff from D → creates D' (new SHA)
6. Original C and D still exist (orphaned, will be garbage collected)

**Commands**:

```bash
git checkout feature
git rebase main

# Or in one command
git rebase main feature
```

### Interactive Rebase - The Power Tool

Interactive rebase lets you edit history before applying it.

**Use cases**:

- Combine multiple commits into one
- Split a commit into multiple
- Reorder commits
- Edit commit messages
- Delete commits
- Edit commit contents

**Command**:

```bash
git rebase -i HEAD~3  # Rebase last 3 commits
```

**The interactive editor**:

```
pick abc123 feat: add login
pick def456 fix typo
pick ghi789 feat: add logout

# Commands:
# p, pick   = use commit
# r, reword = use commit, but edit message
# e, edit   = use commit, but stop for amending
# s, squash = use commit, meld into previous commit
# f, fixup  = like squash, but discard commit message
# d, drop   = remove commit
```

**Example: Squashing commits**:

```
pick abc123 feat: add login
fixup def456 fix typo           # Squash into previous
pick ghi789 feat: add logout
```

Results in:

```
abc123' feat: add login (includes typo fix)
ghi789' feat: add logout
```

### Rebase vs Merge: The Philosophical Choice

**Merge Philosophy**: "Preserve true history, warts and all"

- History shows exactly what happened
- Including false starts, reverted changes, typos
- Can debug by following the actual development path

**Rebase Philosophy**: "History is a story you tell"

- History should be clean and logical
- Each commit should be a coherent unit
- Make it easy for future maintainers

**When to merge**:

- Public branches (never rebase published commits)
- Want to preserve context of parallel development
- Regulatory/compliance needs true history

**When to rebase**:

- Local feature branches (not yet pushed)
- Want clean, linear history
- Each commit should build and pass tests
- Preparing PR for review

**The Golden Rule of Rebasing**: Never rebase commits that have been pushed to a shared branch.

**Why?** Other developers have commits based on the old SHAs. Rebasing creates new SHAs, causing divergence.

```
Developer A (original):
main  A---B---C---D

Developer A (after rebase):
main  A---B---C'---D'

Developer B (pulls):
main  A---B---C---D (old)
          \---C'--D' (new)
# Now main has forked!
```

---

## Part 4: Practical Exercises

### Exercise 1: See How Merge Strategies Differ

**Setup**:

```bash
# Create test repository
mkdir git-merge-test && cd git-merge-test
git init

# Create initial commits
echo "line 1" > file.txt
git add file.txt
git commit -m "A: initial commit"

echo "line 2" >> file.txt
git commit -am "B: add line 2"

# Create feature branch
git checkout -b feature

echo "line 3" >> file.txt
git commit -am "C: add line 3"

echo "line 4" >> file.txt
git commit -am "D: add line 4"
```

**Exercise 1a: Fast-forward merge**:

```bash
git checkout main
git merge feature  # Should fast-forward

# View history
git --no-pager log --oneline --graph --all
```

**Expected output**:

```
* D: add line 4 (HEAD -> main, feature)
* C: add line 3
* B: add line 2
* A: initial commit
```

**Exercise 1b: No-fast-forward merge**:

```bash
# Reset to before merge
git reset --hard HEAD~2

# Merge with no-ff
git merge --no-ff feature -m "Merge: feature branch"

# View history
git --no-pager log --oneline --graph --all
```

**Expected output**:

```
*   Merge: feature branch (HEAD -> main)
|\
| * D: add line 4 (feature)
| * C: add line 3
|/
* B: add line 2
* A: initial commit
```

**Exercise 1c: Squash merge**:

```bash
# Reset
git reset --hard B  # Back to commit B

# Squash merge
git merge --squash feature
git commit -m "feat: add lines 3 and 4"

# View history
git --no-pager log --oneline --graph --all
```

**Expected output**:

```
* feat: add lines 3 and 4 (HEAD -> main)
| * D: add line 4 (feature)
| * C: add line 3
|/
* B: add line 2
* A: initial commit
```

**Notice**: Feature branch history is not connected to main.

### Exercise 2: Merge Conflicts

**Setup**:

```bash
# Clean start
cd ..
mkdir git-conflict-test && cd git-conflict-test
git init

echo "hello" > file.txt
git add file.txt
git commit -m "initial"

# Branch 1: Change line 1
git checkout -b branch1
echo "goodbye" > file.txt
git commit -am "branch1: change to goodbye"

# Branch 2: Change line 1 differently
git checkout main
git checkout -b branch2
echo "bonjour" > file.txt
git commit -am "branch2: change to bonjour"
```

**Trigger conflict**:

```bash
git checkout main
git merge branch1  # Clean merge

git merge branch2  # CONFLICT!
```

**Git's output**:

```
Auto-merging file.txt
CONFLICT (content): Merge conflict in file.txt
Automatic merge failed; fix conflicts and then commit the result.
```

**View conflict markers**:

```bash
cat file.txt
```

**Output**:

```
goodbye
```

**Resolve conflict**:

```bash
# Edit file.txt, choose one or combine
echo "hello everyone" > file.txt

git add file.txt
git commit -m "Merge: resolve conflict"

# View history
git --no-pager log --oneline --graph --all
```

**Key Learning**: Conflicts occur when same lines change in both branches. Git can't auto-resolve semantic conflicts.

### Exercise 3: Rebase vs Merge

**Setup**:

```bash
cd ..
mkdir git-rebase-test && cd git-rebase-test
git init

# Main branch commits
echo "1" > file.txt
git add file.txt
git commit -m "A"

echo "2" >> file.txt
git commit -am "B"

# Feature branch
git checkout -b feature
echo "3" >> file.txt
git commit -am "C"

echo "4" >> file.txt
git commit -am "D"

# Main continues
git checkout main
echo "5" >> file.txt
git commit -am "E"

echo "6" >> file.txt
git commit -am "F"
```

**Current state**:

```bash
git --no-pager log --oneline --graph --all
```

**Output**:

```
* F (HEAD -> main)
* E
| * D (feature)
| * C
|/
* B
* A
```

**Exercise 3a: Merge approach**:

```bash
git checkout main
git merge feature

git --no-pager log --oneline --graph --all
```

**Output**:

```
*   Merge branch 'feature' (HEAD -> main)
|\
| * D (feature)
| * C
* | F
* | E
|/
* B
* A
```

**Exercise 3b: Rebase approach**:

```bash
# Reset to before merge
git reset --hard F

# Rebase feature onto main
git checkout feature
git rebase main

git --no-pager log --oneline --graph --all
```

**Output**:

```
* D' (HEAD -> feature)  # Note: New SHA
* C'                    # Note: New SHA
* F (main)
* E
* B
* A
```

**Now fast-forward main**:

```bash
git checkout main
git merge feature  # Fast-forward

git --no-pager log --oneline --graph --all
```

**Output**:

```
* D' (HEAD -> main, feature)
* C'
* F
* E
* B
* A
```

**Key Insight**: Rebase + merge creates linear history. Pure merge preserves branching structure.

### Exercise 4: Interactive Rebase

**Setup**:

```bash
cd ..
mkdir git-interactive-test && cd git-interactive-test
git init

echo "1" > file.txt
git add file.txt
git commit -m "initial"

# Create messy commit history
echo "2" >> file.txt
git commit -am "add feature"

echo "3" >> file.txt
git commit -am "fix typo"

echo "4" >> file.txt
git commit -am "oops forgot this"

echo "5" >> file.txt
git commit -am "feature complete"
```

**View messy history**:

```bash
git --no-pager log --oneline
```

**Output**:

```
abc123 feature complete
def456 oops forgot this
ghi789 fix typo
jkl012 add feature
mno345 initial
```

**Clean it up**:

```bash
git rebase -i HEAD~4
```

**In the editor, change to**:

```
pick jkl012 add feature
fixup ghi789 fix typo
fixup def456 oops forgot this
reword abc123 feature complete
```

**Save and exit. In the reword editor**:

```
feat: implement user authentication

- Add login endpoint
- Add password hashing
- Implement session management
```

**View clean history**:

```bash
git --no-pager log --oneline
```

**Output**:

```
xyz789 feat: implement user authentication
mno345 initial
```

**Key Learning**: Interactive rebase lets you craft a clean, professional commit history before pushing.

### Exercise 5: Cherry-Pick

**Use case**: You need a specific commit from one branch on another.

**Setup**:

```bash
cd ..
mkdir git-cherry-pick-test && cd git-cherry-pick-test
git init

echo "1" > file.txt
git add file.txt
git commit -m "initial"

# Feature branch with multiple commits
git checkout -b feature
echo "2" >> file.txt
git commit -am "A: add feature A"

echo "3" >> file.txt
git commit -am "B: add feature B"

echo "4" >> file.txt
git commit -am "C: add feature C"

# We only want commit B on main
git checkout main
```

**Cherry-pick the middle commit**:

```bash
# Get the SHA of commit B
COMMIT_B=$(git --no-pager log --oneline feature | grep "B: add feature B" | cut -d' ' -f1)

git cherry-pick $COMMIT_B

cat file.txt
```

**Output**:

```
1
3
```

**View history**:

```bash
git --no-pager log --oneline --graph --all
```

**Output**:

```
* B': add feature B (HEAD -> main)  # New commit, same changes
| * C: add feature C (feature)
| * B: add feature B
| * A: add feature A
|/
* initial
```

**Key Learning**: Cherry-pick creates a new commit with the same changes but different SHA. Useful for hotfixes.

### Exercise 6: Understanding Reflog

**What is reflog?** History of HEAD movements (safety net for "oh crap" moments).

**Setup**:

```bash
cd ..
mkdir git-reflog-test && cd git-reflog-test
git init

echo "1" > file.txt
git add file.txt
git commit -m "A"

echo "2" >> file.txt
git commit -am "B"

echo "3" >> file.txt
git commit -am "C"

# Oops, reset too far
git reset --hard HEAD~2  # Back to A

git --no-pager log --oneline  # C and B are gone!
```

**Output**:

```
abc123 A
```

**Recovery with reflog**:

```bash
git reflog
```

**Output**:

```
abc123 HEAD@{0}: reset: moving to HEAD~2
def456 HEAD@{1}: commit: C
ghi789 HEAD@{2}: commit: B
abc123 HEAD@{3}: commit (initial): A
```

**Recover the lost commits**:

```bash
git reset --hard HEAD@{1}  # Back to C

git --no-pager log --oneline
```

**Output**:

```
def456 C
ghi789 B
abc123 A
```

**Key Learning**: Reflog is your safety net. Commits are rarely truly lost.

---

## Part 5: Merge Strategies in Pull Requests

### GitHub/GitLab PR Merge Options

**1. Merge Commit** (default):

```bash
git merge --no-ff feature
```

**History**:

```
*   Merge pull request #123
|\
| * D: review feedback
| * C: initial implementation
|/
* B: main branch commit
```

**When to use**:

- Want to preserve all feature branch commits
- Need to track who reviewed and approved
- Compliance requires full history

**2. Squash and Merge**:

```bash
git merge --squash feature
git commit -m "feat: add user authentication (#123)"
```

**History**:

```
* feat: add user authentication (#123)
* B: main branch commit
```

**When to use**:

- Feature branch has messy commits
- Want clean main branch history
- One commit = one PR = one feature

**3. Rebase and Merge**:

```bash
git rebase main feature
git checkout main
git merge --ff-only feature
```

**History**:

```
* D': review feedback
* C': initial implementation
* B: main branch commit
```

**When to use**:

- Want linear history
- Feature branch commits are already clean
- Each commit should be preserved

### Comparison Matrix

| Strategy         | History    | Bisect                      | Revert                    | Readability                      |
| ---------------- | ---------- | --------------------------- | ------------------------- | -------------------------------- |
| **Merge Commit** | Non-linear | Can bisect feature commits  | Revert merge commit       | Lower (many branches)            |
| **Squash**       | Linear     | Can't bisect within feature | Revert single commit      | Highest (one commit per feature) |
| **Rebase**       | Linear     | Can bisect each commit      | Revert individual commits | High (clean commits)             |

---

## Part 6: Advanced History Manipulation

### Amending Commits

**Last commit mistake?**

```bash
git commit -m "fix: typo"
# Oh no, forgot to add a file!

git add forgotten-file.txt
git commit --amend  # Adds to previous commit

# Or change commit message
git commit --amend -m "fix: correct typo in login form"
```

**What actually happens**: Creates new commit with new SHA, old commit orphaned.

**Warning**: Never amend pushed commits (breaks history for others).

### Resetting vs Reverting

**Reset** (rewrites history):

```bash
git reset --soft HEAD~1   # Undo commit, keep changes staged
git reset --mixed HEAD~1  # Undo commit, unstage changes (default)
git reset --hard HEAD~1   # Undo commit, discard changes
```

**Revert** (adds new commit that undoes changes):

```bash
git revert HEAD  # Creates new commit that undoes HEAD
```

**When to use each**:

- **Reset**: Local commits not yet pushed
- **Revert**: Public commits (preserves history)

### Rebase onto Different Base

**Scenario**: Feature branch was created from wrong branch.

```
main      A---B---C
               \
old-feature     D---E
                     \
new-feature           F---G
```

**Want to move new-feature to main**:

```bash
git rebase --onto main old-feature new-feature
```

**Result**:

```
main      A---B---C---F'---G'
               \
old-feature     D---E
```

### Splitting a Commit

**Scenario**: One commit does too many things.

```bash
git rebase -i HEAD~1
# Change 'pick' to 'edit'

git reset HEAD~1  # Undo commit, keep changes

# Stage and commit parts separately
git add file1.txt
git commit -m "feat: add feature A"

git add file2.txt
git commit -m "feat: add feature B"

git rebase --continue
```

---

## Part 7: Reading Complex Git Histories

### Understanding git --no-pager log Options

**Basic log**:

```bash
git --no-pager log --oneline --graph --all --decorate
```

**Alias this** (add to ~/.gitconfig):

```bash
git config --global alias.lg "log --oneline --graph --all --decorate"
```

**Advanced log with dates**:

```bash
git --no-pager log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit --date=relative
```

**Find when a file changed**:

```bash
git --no-pager log --follow -p -- path/to/file.txt
```

**Find who changed a line**:

```bash
git blame path/to/file.txt
git blame -L 10,20 path/to/file.txt  # Lines 10-20 only
```

**Find when a bug was introduced**:

```bash
git bisect start
git bisect bad HEAD         # Current commit is bad
git bisect good abc123      # Older commit was good
# Git checks out middle commit
# Test it, then:
git bisect good   # or bad
# Repeat until found
git bisect reset  # Done
```

### Interpreting Merge Graphs

**Simple merge**:

```
*   Merge branch 'feature'
|\
| * Feature commit
|/
* Main commit
```

**Multiple merges**:

```
*   Merge branch 'feature-2'
|\
| * Feature 2 commit
* |   Merge branch 'feature-1'
|\ \
| * | Feature 1 commit
| |/
|/|
* | Main commit
|/
* Base
```

**Octopus merge** (merging multiple branches at once):

```
*-.   Merge branches 'a', 'b', 'c'
|\ \ \
| | | * Branch C
| | * | Branch B
| * | | Branch A
|/ / /
* Base
```

---

## Part 8: Connection to Release Engineering

### Why History Matters for Production

**Scenario 1: Finding the Bad Commit**

Production breaks. With clean, linear history (rebase/squash):

```bash
git bisect start
git bisect bad v1.5.0   # Broken release
git bisect good v1.4.0  # Last known good

# Git checks out middle commit
# Run tests or deploy to staging
git bisect good  # or bad

# Repeat ~log2(n) times
# Git identifies the exact commit
```

With messy merge history, bisect is harder:

- Merge commits don't represent a single change
- Must bisect through all feature branch commits
- May need to manually track which branch caused it

**Scenario 2: Hotfix Cherry-Pick**

Production bug in v1.5.0. Main is at v1.6.0-dev with new features.

**With clean commits** (rebase):

```bash
# Find the fix commit
git --no-pager log --oneline main
# abc123 fix: resolve payment timeout

git checkout release/1.5.x
git cherry-pick abc123  # Clean pick
git tag v1.5.1
```

**With messy commits** (merge):

```bash
# The fix is buried in a merge commit with 10 other changes
git --no-pager log --oneline main
# def456 Merge PR #123 (payment fixes + new features)

# Must cherry-pick the specific commit from the branch
git --no-pager log def456^2  # Show second parent (feature branch)
# Find the actual fix commit, cherry-pick that
```

**Lesson**: Clean commits = easier operational work.

### Branching Strategy → Merge Strategy Mapping

**GitFlow** → **Merge Commits** (--no-ff):

- Release branches need to merge back to develop and main
- Want to preserve which commits were part of which release
- History shows parallel workstreams

**Trunk-Based** → **Rebase + Squash**:

- Short-lived branches rebase onto main
- Keep main linear and clean
- Feature flags handle incomplete work

**GitHub Flow** → **Squash or Rebase**:

- PRs represent features
- Squash = one commit per PR
- Rebase = preserve clean PR commits

### Configuration for Team Consistency

**Project-level config** (.gitconfig or .git/config):

```bash
# Prevent accidental merge commits
git config merge.ff only  # Only allow fast-forward

# Require explicit --no-ff for merge commits
git config alias.merge-branch "merge --no-ff"

# Set default pull behavior
git config pull.rebase true  # Rebase on pull, not merge
```

**Branch protection rules** (GitHub/GitLab):

- Require PR reviews
- Require status checks
- Enforce linear history (squash/rebase only)
- Prevent force pushes to main

---

## Part 9: Interview-Ready Mental Models

### When Asked: "How do you handle merges?"

**Bad Answer**: "We use squash merges."

**Good Answer**:
"We use squash merges for PRs to main because we optimize for a clean, linear history on our main branch. This makes `git bisect` more effective when debugging production issues - each commit on main represents a complete, tested feature.

However, for long-running release branches in our GitFlow process, we use merge commits with `--no-ff` to preserve which fixes were backported from main vs. developed directly on the release branch. This is critical for our compliance requirements.

The trade-off is we lose granular commit history from feature branches, but we mitigate this with thorough PR descriptions and linking to our issue tracker."

### When Asked: "Tell me about a time you used Git to debug an issue"

**Structure your answer**:

1. **Problem**: Production outage or bug
2. **Approach**: Used git bisect / git blame / git log
3. **Technical details**: Show you understand the commands
4. **Resolution**: Found the commit, reverted/fixed it
5. **Learning**: What you'd do differently (better commit messages, smaller commits, etc.)

**Example**:
"We had a regression in our payment processing. I used `git bisect` between the last known good release tag and the current HEAD. With about 200 commits in between, bisect found the culprit in 8 iterations (log2(200)).

The challenge was that the bad commit was a large squash merge, so I had to check out the original PR branch and bisect within that to find the specific change. This taught me that while squash merges keep main clean, we should ensure the original PR branches remain accessible until the next release is stable."

### When Asked: "Merge vs Rebase?"

**Framework**:

1. Acknowledge both are valid (shows maturity)
2. Explain what each does technically (shows understanding)
3. Discuss trade-offs (shows strategic thinking)
4. Relate to team context (shows practical experience)

**Answer**:
"The merge vs rebase decision is about what you optimize for. Merge preserves true history - you can see exactly when branches diverged and converged. Rebase creates a clean, linear history that's easier to understand.

I use merge for public/shared branches because rewriting published history breaks other developers' work. For private feature branches, I rebase onto main before creating a PR to ensure my commits apply cleanly and CI runs against current code.

The key is team agreement. We enforce this with branch protection rules that only allow squash merges to main, giving us linear history in production while preserving development history in PRs."

---

## Exercise 10: Put It All Together

**Simulate a full release workflow**:

```bash
# Setup
mkdir release-workflow && cd release-workflow
git init

# Initial release
echo "v1" > version.txt
git add version.txt
git commit -m "feat: initial release"
git tag v1.0.0

# Feature development (TBD style)
git checkout -b feature/auth
echo "auth code" > auth.txt
git commit -am "feat: add authentication"

echo "more auth" >> auth.txt
git commit -am "feat: add OAuth support"

# Rebase before merging
git checkout main
echo "hotfix" > fix.txt
git commit -am "fix: critical bug"

git checkout feature/auth
git rebase main  # Rebase feature onto latest main

# Clean up commits with interactive rebase
git rebase -i HEAD~2  # Squash the two auth commits

# Merge to main
git checkout main
git merge feature/auth  # Should fast-forward

git tag v1.1.0

# View clean history
git --no-pager log --oneline --graph

# Simulate production bug
git checkout -b hotfix
echo "hotfix code" > fix.txt
git commit -am "fix: resolve payment timeout"

git checkout main
git cherry-pick hotfix  # Apply hotfix to main

git tag v1.1.1

# Final history
git --no-pager log --oneline --graph --all
```

**Expected output**:

```
* fix: resolve payment timeout (HEAD -> main, tag: v1.1.1)
* feat: add authentication with OAuth (tag: v1.1.0, feature/auth)
* fix: critical bug
* feat: initial release (tag: v1.0.0)
```

**Key Observations**:

1. Linear history (rebase before merge)
2. Each commit is complete (interactive rebase squashed)
3. Tags mark releases
4. Hotfix cherry-picked cleanly

---

## Your Interview Prep Checklist

Can you:

- [ ] Explain what a commit actually contains (tree, parent, author, message)?
- [ ] Draw the commit graph before and after a merge/rebase?
- [ ] Explain when fast-forward happens vs. merge commit?
- [ ] Demonstrate interactive rebase (squash, reword, reorder)?
- [ ] Recover from common mistakes (wrong commit, bad rebase, lost commits)?
- [ ] Use git bisect to find a regression?
- [ ] Explain trade-offs between merge/rebase/squash?
- [ ] Map branching strategy to merge strategy (GitFlow → --no-ff, TBD → rebase)?
- [ ] Describe a time you used Git to debug a production issue?
- [ ] Configure Git for team consistency (.gitconfig, branch protection)?

---

## Recommended Deep Dives

**Books**:

- "Pro Git" by Scott Chacon (free online) - Chapter 3 (Branching), Chapter 7 (Advanced)

**Exercises**:

- https://learngitbranching.js.org/ - Interactive visualizations
- https://ohmygit.org/ - Game to learn Git

**Advanced Topics to Explore**:

- git refspec (what happens during fetch/push)
- git hooks (enforce commit message format, run tests)
- git worktree (multiple working directories)
- Submodules vs subtrees
- Sparse checkout (work with monorepos)

---

## Final Thought

Git is not just a version control tool. It's a **distributed database of code history**. Understanding its internal model - commits as content-addressed snapshots forming a DAG - makes everything else (merge, rebase, cherry-pick, bisect) logical rather than magical.

In senior+ interviews, this understanding separates you from candidates who just memorize commands.
