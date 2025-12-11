# Git Practical Exercises

## How to Use This File

Each exercise is self-contained. Run the commands, observe the output, and build mental models through hands-on practice.

**Recommendation**: Type every command (don't copy-paste). The muscle memory helps in interviews and operational work.

**Time estimate**:
- Quick path: Exercises 1, 3, 6, 10 (30 minutes)
- Thorough: All exercises (2-3 hours)

---

## Exercise 1: Merge Strategies - See the Difference

**Goal**: Understand how fast-forward, no-ff, and squash affect history.

**Setup**:
```bash
mkdir ex1-merge-strategies && cd ex1-merge-strategies
git init

# Create base commits
echo "line 1" > file.txt
git add file.txt
git commit -m "A: initial commit"

echo "line 2" >> file.txt
git commit -am "B: add line 2"
```

### Part 1a: Fast-Forward Merge

```bash
# Create feature branch
git checkout -b feature-ff

echo "line 3" >> file.txt
git commit -am "C: add line 3"

echo "line 4" >> file.txt
git commit -am "D: add line 4"

# Merge with fast-forward
git checkout main
git merge feature-ff

# Observe history
git --no-pager log --oneline --graph --all
```

**Expected output**:
```
* <sha> (HEAD -> main, feature-ff) D: add line 4
* <sha> C: add line 3
* <sha> B: add line 2
* <sha> A: initial commit
```

**Question to answer**: Is there a merge commit? Can you tell commits C and D were on a branch?

**Answer**: <details><summary>Click to reveal</summary>No merge commit. Linear history. You cannot tell C and D were developed on a separate branch.</details>

### Part 1b: No-Fast-Forward Merge

```bash
# Reset main to B (2 commits back from current position)
git checkout main
git reset --hard HEAD~2

# Create new feature branch
git checkout -b feature-noff

echo "line 3" >> file.txt
git commit -am "E: add line 3"

echo "line 4" >> file.txt
git commit -am "F: add line 4"

# Merge with --no-ff
git checkout main
git merge --no-ff feature-noff -m "Merge: feature-noff branch"

# Observe history
git --no-pager log --oneline --graph --all
```

**Expected output**:
```
*   <sha> (HEAD -> main) Merge: feature-noff branch
|\
| * <sha> (feature-noff) F: add line 4
| * <sha> E: add line 3
|/
* <sha> B: add line 2
* <sha> A: initial commit
```

**Question**: How many parents does the merge commit have? What are they?

**Answer**: <details><summary>Click to reveal</summary>Two parents: B (from main) and F (tip of feature-noff). Use `git show HEAD` to see both.</details>

### Part 1c: Squash Merge

```bash
# Reset to B again (back to before the merge commit)
git checkout main
git reset --hard HEAD~1

# Create another feature branch
git checkout -b feature-squash

echo "line 3" >> file.txt
git commit -am "G: add line 3"

echo "line 4" >> file.txt
git commit -am "H: add line 4"

echo "line 5" >> file.txt
git commit -am "I: add line 5"

# Squash merge
git checkout main
git merge --squash feature-squash

# Check status
git status

# Complete the squash merge
git commit -m "feat: add lines 3, 4, and 5

Combined commits G, H, I into single commit."

# Observe history
git --no-pager log --oneline --graph --all
```

**Expected output**:
```
* <sha> (HEAD -> main) feat: add lines 3, 4, and 5
| * <sha> (feature-squash) I: add line 5
| * <sha> H: add line 4
| * <sha> G: add line 3
|/
* <sha> B: add line 2
* <sha> A: initial commit
```

**Question**: Is the squash commit connected to the feature branch? How many parents does it have?

**Answer**: <details><summary>Click to reveal</summary>No connection to feature-squash in the graph. Only one parent (B). The feature branch history is not part of main's ancestry.</details>

### Part 1d: Comparison

```bash
# Create comparison visualization
cat > /tmp/comparison.txt << 'EOF'
Fast-Forward:     No merge commit, linear history
                  Lose branch context

No-FF:            Merge commit with 2 parents
                  Preserve branch structure

Squash:           Single commit, one parent
                  Feature history exists but disconnected
EOF

cat /tmp/comparison.txt
```

**Key Takeaway**: The merge strategy fundamentally changes how history looks and what information is preserved.

---

## Exercise 2: Merge Conflicts and Resolution

**Goal**: Practice resolving conflicts and understand when they occur.

**Setup**:
```bash
cd ..
mkdir ex2-conflicts && cd ex2-conflicts
git init

echo "hello" > file.txt
git add file.txt
git commit -m "initial"
```

### Part 2a: Same Line, Different Changes

```bash
# Branch 1: Change to "goodbye"
git checkout -b branch1
echo "goodbye" > file.txt
git commit -am "branch1: change to goodbye"

# Branch 2: Change to "bonjour"
git checkout main
git checkout -b branch2
echo "bonjour" > file.txt
git commit -am "branch2: change to bonjour"

# Merge branch1 to main (no conflict)
git checkout main
git merge branch1
cat file.txt  # Should be "goodbye"

# Try to merge branch2 (CONFLICT!)
git merge branch2
```

**What you'll see**:
```
Auto-merging file.txt
CONFLICT (content): Merge conflict in file.txt
Automatic merge failed; fix conflicts and then commit the result.
```

**Examine the conflict**:
```bash
cat file.txt
```

**Output**:
```
<<<<<<< HEAD
goodbye
=======
bonjour
>>>>>>> branch2
```

**Understanding conflict markers**:
- `<<<<<<< HEAD`: Current branch (main) version
- `=======`: Separator
- `>>>>>>> branch2`: Incoming branch version

**Resolve the conflict**:
```bash
# Option 1: Keep one version
echo "goodbye" > file.txt

# Option 2: Combine both
echo "goodbye and bonjour" > file.txt

# Option 3: Choose something entirely different
echo "hello everyone" > file.txt

# For this exercise, let's combine
echo "goodbye and bonjour" > file.txt

# Mark as resolved
git add file.txt

# Complete the merge
git commit -m "Merge: resolve conflict between goodbye and bonjour"

# View history
git --no-pager log --oneline --graph --all
```

### Part 2b: File Deleted vs Modified

```bash
# Create new scenario
cd ..
mkdir ex2b-delete-modify && cd ex2b-delete-modify
git init

echo "content" > file.txt
git add file.txt
git commit -m "initial"

# Branch 1: Delete the file
git checkout -b delete-branch
git rm file.txt
git commit -m "delete file"

# Branch 2: Modify the file
git checkout main
git checkout -b modify-branch
echo "modified content" > file.txt
git commit -am "modify file"

# Merge delete-branch to main
git checkout main
git merge delete-branch

# Try to merge modify-branch (CONFLICT!)
git merge modify-branch
```

**What you'll see**:
```
CONFLICT (modify/delete): file.txt deleted in HEAD and modified in modify-branch.
```

**Resolve**:
```bash
# Decide: keep the file or delete it?

# Keep it:
git add file.txt
git commit -m "Merge: keep modified file"

# Or delete it:
# git rm file.txt
# git commit -m "Merge: confirm deletion"
```

**Key Takeaway**: Conflicts aren't just text collisions - they're semantic decisions Git can't make for you.

---

## Exercise 3: Rebase vs Merge - See the Difference

**Goal**: Understand how rebase rewrites history.

**Setup**:
```bash
cd ..
mkdir ex3-rebase-vs-merge && cd ex3-rebase-vs-merge
git init

# Create divergent branches
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

# Current state
git --no-pager log --oneline --graph --all
```

**You should see**:
```
* <sha> (HEAD -> main) F
* <sha> E
| * <sha> (feature) D
| * <sha> C
|/
* <sha> B
* <sha> A
```

### Part 3a: Merge Approach

```bash
# Save commit SHAs for later comparison
BEFORE_FEATURE_SHA=$(git rev-parse feature)
echo "Feature SHA before: $BEFORE_FEATURE_SHA"

git checkout main
git merge feature -m "Merge feature branch"

git --no-pager log --oneline --graph --all
```

**Output**:
```
*   <sha> (HEAD -> main) Merge feature branch
|\
| * <sha> (feature) D
| * <sha> C
* | <sha> F
* | <sha> E
|/
* <sha> B
* <sha> A
```

**Question**: How many parents does the merge commit have? Can you still see when branches diverged?

### Part 3b: Reset and Try Rebase

```bash
# Reset main to before merge (back 1 commit)
git reset --hard HEAD~1

# Rebase feature onto main
git checkout feature
git rebase main

# Check the SHAs
AFTER_FEATURE_SHA=$(git rev-parse feature)
echo "Feature SHA before: $BEFORE_FEATURE_SHA"
echo "Feature SHA after:  $AFTER_FEATURE_SHA"

# View history
git --no-pager log --oneline --graph --all
```

**Output**:
```
* <sha> (HEAD -> feature) D'  (note: different SHA than before!)
* <sha> C'  (note: different SHA than before!)
* <sha> (main) F
* <sha> E
* <sha> B
* <sha> A
```

**Critical observation**: The SHAs changed! C and D are new commits (C' and D').

**Verify content is the same**:
```bash
# The file content should be identical in both approaches
cat file.txt  # Should have lines 1-6
```

**Fast-forward main**:
```bash
git checkout main
git merge feature  # Fast-forward

git --no-pager log --oneline --graph
```

**Output**:
```
* <sha> (HEAD -> main, feature) D
* <sha> C
* <sha> F
* <sha> E
* <sha> B
* <sha> A
```

**Key Takeaway**:
- Merge preserves both lines of development
- Rebase creates linear history but rewrites commits (new SHAs)

---

## Exercise 4: Interactive Rebase - Crafting Clean History

**Goal**: Use interactive rebase to clean up messy commits.

**Setup**:
```bash
cd ..
mkdir ex4-interactive-rebase && cd ex4-interactive-rebase
git init

echo "1" > file.txt
git add file.txt
git commit -m "initial"

# Create messy commit history (simulating real development)
echo "2" >> file.txt
git commit -am "add feature"

echo "3" >> file.txt
git commit -am "fix typo"

echo "4" >> file.txt
git commit -am "actually add the feature"

echo "5" >> file.txt
git commit -am "fix another typo"

echo "6" >> file.txt
git commit -am "feature done"

# View messy history
git --no-pager log --oneline
```

**You'll see**:
```
<sha> feature done
<sha> fix another typo
<sha> actually add the feature
<sha> fix typo
<sha> add feature
<sha> initial
```

### Part 4a: Squash Commits

```bash
# Start interactive rebase
git rebase -i HEAD~5

# This opens your editor with:
# pick <sha> add feature
# pick <sha> fix typo
# pick <sha> actually add the feature
# pick <sha> fix another typo
# pick <sha> feature done
```

**Change to**:
```
pick <sha> add feature
fixup <sha> fix typo
fixup <sha> actually add the feature
fixup <sha> fix another typo
reword <sha> feature done
```

**What this does**:
- `pick`: Use commit as-is
- `fixup`: Meld into previous commit, discard message
- `reword`: Use commit, but edit the message

**Save and exit. You'll get another editor for reword**:
```
feat: implement user authentication

- Add login endpoint
- Implement JWT validation
- Add session management
```

**View clean history**:
```bash
git --no-pager log --oneline
```

**Output**:
```
<sha> feat: implement user authentication
<sha> initial
```

### Part 4b: Reordering Commits

```bash
# Create scenario
cd ..
mkdir ex4b-reorder && cd ex4b-reorder
git init

echo "1" > file.txt
git add file.txt
git commit -m "initial"

# Commits in wrong order
mkdir docs
echo "readme" > docs/README.md
git add docs/
git commit -m "add README"

mkdir src
echo "code" > src/app.js
git add src/
git commit -m "add application code"

mkdir tests
echo "tests" > tests/app.test.js
git add tests/
git commit -m "add tests"

# Logical order should be: code → tests → docs
git rebase -i HEAD~3
```

**Change order to**:
```
pick <sha> add application code
pick <sha> add tests
pick <sha> add README
```

**Save and view**:
```bash
git --no-pager log --oneline
```

### Part 4c: Splitting a Commit

```bash
# Create commit that does too much
cd ..
mkdir ex4c-split && cd ex4c-split
git init

echo "1" > file.txt
git add file.txt
git commit -m "initial"

# One commit with multiple changes
echo "feature A" > feature-a.txt
echo "feature B" > feature-b.txt
git add .
git commit -m "add features"

# Split it
git rebase -i HEAD~1
```

**Change to**:
```
edit <sha> add features
```

**Save, then**:
```bash
# Reset the commit but keep changes
git reset HEAD~1

# Commit separately
git add feature-a.txt
git commit -m "feat: add feature A"

git add feature-b.txt
git commit -m "feat: add feature B"

# Continue rebase
git rebase --continue

# View result
git --no-pager log --oneline
```

**Key Takeaway**: Interactive rebase is powerful for crafting professional commit history before pushing.

---

## Exercise 5: Cherry-Pick - Selective Merging

**Goal**: Apply specific commits to different branches.

**Setup**:
```bash
cd ..
mkdir ex5-cherry-pick && cd ex5-cherry-pick
git init

echo "1" > file.txt
git add file.txt
git commit -m "initial"

# Feature branch with multiple commits
git checkout -b feature
echo "feature A" > feature-a.txt
git add .
git commit -m "A: add feature A"

echo "feature B" > feature-b.txt
git add .
git commit -m "B: add feature B"

echo "feature C" > feature-c.txt
git add .
git commit -m "C: add feature C"

# View commits
git --no-pager log --oneline
```

### Part 5a: Cherry-pick One Commit

**Scenario**: Production needs feature B urgently, but A and C aren't ready.

```bash
# Get SHA of commit B (it's HEAD~1, the second-to-last commit)
COMMIT_B=$(git rev-parse HEAD~1)
echo "Commit B SHA: $COMMIT_B"

# Apply only commit B to main
git checkout main
git cherry-pick $COMMIT_B

# Check what happened
ls -la  # Should have file.txt and feature-b.txt (not A or C)
git --no-pager log --oneline --graph --all
```

**Output**:
```
* <new-sha> (HEAD -> main) B: add feature B
| * <sha> (feature) C: add feature C
| * <sha> B: add feature B
| * <sha> A: add feature A
|/
* <sha> initial
```

**Question**: Are the SHAs the same for "B" on main vs feature?

**Answer**: <details><summary>Click to reveal</summary>No! Cherry-pick creates a new commit with same changes but different SHA (different parent).</details>

### Part 5b: Cherry-pick Range

```bash
# Reset main back to initial commit
git checkout main
git reset --hard HEAD~1

# Cherry-pick commits A and B (but not C)
# A is at feature~2, B is at feature~1
COMMIT_A=$(git rev-parse feature~2)
COMMIT_B=$(git rev-parse feature~1)

git cherry-pick $COMMIT_A^..$COMMIT_B

ls -la
git --no-pager log --oneline --graph --all
```

**Key Takeaway**: Cherry-pick is useful for hotfixes and selective backporting, but creates duplicate commits.

---

## Exercise 6: Using Reflog - The Safety Net

**Goal**: Recover from mistakes using reflog.

**Setup**:
```bash
cd ..
mkdir ex6-reflog && cd ex6-reflog
git init

echo "1" > file.txt
git add file.txt
git commit -m "A"

echo "2" >> file.txt
git commit -am "B"

echo "3" >> file.txt
git commit -am "C"

echo "4" >> file.txt
git commit -am "D"

git --no-pager log --oneline
```

### Part 6a: Recover from Hard Reset

```bash
# Oops, reset too far!
git reset --hard HEAD~3

# Check log
git --no-pager log --oneline  # Only shows "A"

# File content lost
cat file.txt  # Only "1"

# Use reflog to find lost commits
git reflog
```

**Output**:
```
<sha> (HEAD -> main) HEAD@{0}: reset: moving to HEAD~3
<sha> HEAD@{1}: commit: D
<sha> HEAD@{2}: commit: C
<sha> HEAD@{3}: commit: B
<sha> HEAD@{4}: commit (initial): A
```

**Recover**:
```bash
# Go back to D
git reset --hard HEAD@{1}

# Verify
git --no-pager log --oneline
cat file.txt  # Should have lines 1-4
```

### Part 6b: Recover Deleted Branch

```bash
# Create and delete a branch
git checkout -b feature
echo "5" >> file.txt
git commit -am "E"

echo "6" >> file.txt
git commit -am "F"

# Switch to main and delete feature
git checkout main
git branch -D feature  # Force delete

# Oops, needed that branch!
# Use reflog to find it
git reflog
```

**Find the commit where feature was**:
```
<sha> HEAD@{X}: checkout: moving from feature to main
<sha> HEAD@{X+1}: commit: F
```

**Recover**:
```bash
# Recreate branch at that commit
# Look for the commit in reflog, it will be a few entries back
# Typically HEAD@{1} or HEAD@{2} depending on how many operations you've done
git checkout -b feature HEAD@{1}

git --no-pager log --oneline
```

**Key Takeaway**: Reflog is your safety net. Commits are rarely truly lost (until garbage collection after ~30 days).

---

## Exercise 7: Git Bisect - Finding Bugs

**Goal**: Use bisect to find which commit introduced a bug.

**Setup**:
```bash
cd ..
mkdir ex7-bisect && cd ex7-bisect
git init

# Create a simple script that will "break"
cat > test.sh << 'EOF'
#!/bin/bash
# Returns 0 if file.txt contains only numbers
if [[ $(cat file.txt) =~ ^[0-9]+$ ]]; then
  exit 0
else
  exit 1
fi
EOF

chmod +x test.sh

# Good commits
echo "1" > file.txt
git add .
git commit -m "A: initial"

echo "12" > file.txt
git commit -am "B: update to 12"

echo "123" > file.txt
git commit -am "C: update to 123"

echo "1234" > file.txt
git commit -am "D: update to 1234"

# Bug introduced!
echo "12345a" > file.txt
git commit -am "E: update to 12345a"

# More commits after bug
echo "12345ab" > file.txt
git commit -am "F: update to 12345ab"

echo "12345abc" > file.txt
git commit -am "G: update to 12345abc"

# Test current state
./test.sh
echo "Exit code: $?"  # Should be 1 (failure)
```

### Part 7a: Manual Bisect

```bash
# Start bisect
git bisect start

# Mark current as bad
git bisect bad HEAD

# Mark old commit as good (use tag or HEAD~N)
# Tag the initial commit first for easier reference
git tag good-version HEAD~6
git bisect good good-version

# Git checks out middle commit
# Test it
./test.sh
echo "Exit code: $?"

# Mark it (good or bad)
git bisect good  # or bad, depending on result

# Repeat until found
# Git will tell you: "X is the first bad commit"

# View the bad commit
git show

# End bisect
git bisect reset
```

### Part 7b: Automated Bisect

```bash
# Start bisect (using tag from before or HEAD~6)
git bisect start HEAD good-version

# Run automated bisect
git bisect run ./test.sh

# Git will automatically:
# 1. Check out middle commit
# 2. Run ./test.sh
# 3. Mark as good (exit 0) or bad (exit non-zero)
# 4. Repeat until found

# View result (shows first bad commit)
# Reset
git bisect reset
```

**Key Takeaway**: Bisect uses binary search. With 1000 commits, finds bug in ~10 tests (log2(1000)).

---

## Exercise 8: Handling Release Branches

**Goal**: Practice GitFlow-style release branching.

**Setup**:
```bash
cd ..
mkdir ex8-release-branches && cd ex8-release-branches
git init

# Initial release
echo "v1.0" > version.txt
git add version.txt
git commit -m "v1.0: initial release"
git tag v1.0.0

# Develop branch
git checkout -b develop
echo "v1.1-dev" > version.txt
git commit -am "start v1.1 development"
```

### Part 8a: Feature Development

```bash
# Feature 1
git checkout -b feature/auth
echo "auth code" > auth.txt
git add .
git commit -m "feat: add authentication"

git checkout develop
git merge --no-ff feature/auth -m "Merge: auth feature"

# Feature 2
git checkout -b feature/payments
echo "payment code" > payments.txt
git add .
git commit -m "feat: add payments"

git checkout develop
git merge --no-ff feature/payments -m "Merge: payments feature"

# View develop
git --no-pager log --oneline --graph
```

### Part 8b: Create Release

```bash
# Start release from develop
git checkout -b release/1.1.0
echo "v1.1.0-rc1" > version.txt
git commit -am "prepare v1.1.0 release"

# Bug found during QA
echo "auth code - fixed" > auth.txt
git commit -am "fix: auth bug found in QA"

# Release to main
git checkout main
git merge --no-ff release/1.1.0 -m "Release: v1.1.0"
git tag v1.1.0

# Merge fixes back to develop
git checkout develop
git merge --no-ff release/1.1.0 -m "Backport: v1.1.0 fixes"

# Clean up
git branch -d release/1.1.0
```

### Part 8c: Hotfix

```bash
# Production bug discovered!
git checkout main
git checkout -b hotfix/1.1.1

echo "auth code - critical fix" > auth.txt
git commit -am "fix: critical auth bug"

# Apply to main
git checkout main
git merge --no-ff hotfix/1.1.1 -m "Hotfix: v1.1.1"
git tag v1.1.1

# Apply to develop
git checkout develop
git merge --no-ff hotfix/1.1.1 -m "Backport: v1.1.1 hotfix"

# Clean up
git branch -d hotfix/1.1.1

# View full history
git --no-pager log --oneline --graph --all
```

**Key Takeaway**: GitFlow requires discipline to merge releases and hotfixes back to develop.

---

## Exercise 9: Trunk-Based Development Workflow

**Goal**: Practice TBD-style short-lived branches and feature flags.

**Setup**:
```bash
cd ..
mkdir ex9-tbd && cd ex9-tbd
git init

# Initial commit
echo "v1.0.0" > version.txt
cat > app.js << 'EOF'
function main() {
  console.log("App v1.0.0");
}
EOF

git add .
git commit -m "feat: initial release"
git tag v1.0.0
```

### Part 9a: Short-Lived Feature Branch

```bash
# Start feature
git checkout -b quick-feature

# Small, focused change
cat > feature.js << 'EOF'
function newFeature() {
  // Feature flag controlled
  if (process.env.FEATURE_ENABLED) {
    console.log("New feature!");
  }
}
EOF

git add feature.js
git commit -m "feat: add new feature behind flag"

# Merge same day
git checkout main
git merge quick-feature  # Fast-forward

git branch -d quick-feature

# Tag for release
git tag v1.1.0

git --no-pager log --oneline --graph
```

**Notice**: Linear history, feature merged within hours/days.

### Part 9b: Multiple Quick Merges

```bash
# Simulate a day of development
git checkout -b fix1
echo "fix1" > fix1.txt
git add .
git commit -m "fix: resolve bug 1"
git checkout main
git merge fix1
git branch -d fix1

git checkout -b fix2
echo "fix2" > fix2.txt
git add .
git commit -m "fix: resolve bug 2"
git checkout main
git merge fix2
git branch -d fix2

git checkout -b feat1
echo "feat1" > feat1.txt
git add .
git commit -m "feat: add small feature"
git checkout main
git merge feat1
git branch -d feat1

# Tag for release
git tag v1.2.0

# View history
git --no-pager log --oneline --graph
```

**Output**: Clean, linear history with multiple merges per day.

**Key Takeaway**: TBD emphasizes frequency of integration over isolation of work.

---

## Exercise 10: Production Workflow Simulation

**Goal**: Simulate a complete release workflow with hotfixes and rollbacks.

**Setup**:
```bash
cd ..
mkdir ex10-production && cd ex10-production
git init

# Production setup
cat > deploy.sh << 'EOF'
#!/bin/bash
VERSION=$(cat version.txt)
echo "Deploying version: $VERSION"
EOF
chmod +x deploy.sh
```

### Part 10a: Initial Release

```bash
echo "1.0.0" > version.txt
git add .
git commit -m "feat: initial release"
git tag v1.0.0

./deploy.sh  # Deploy v1.0.0
```

### Part 10b: Feature Development

```bash
# Feature branch
git checkout -b feature/new-api
echo "1.1.0-dev" > version.txt
echo "api code" > api.js
git add .
git commit -m "feat: add new API endpoint"

echo "api docs" > api-docs.md
git add .
git commit -m "docs: add API documentation"

# Clean up commits before merging
git rebase -i HEAD~2
# Squash into one commit

git checkout main
git merge feature/new-api
git tag v1.1.0

./deploy.sh  # Deploy v1.1.0
```

### Part 10c: Production Bug

```bash
# Bug discovered in v1.1.0
# Quick fix
git checkout -b hotfix
echo "api code - fixed" > api.js
git commit -am "fix: resolve API timeout"

git checkout main
git merge hotfix
git tag v1.1.1

./deploy.sh  # Deploy v1.1.1

git branch -d hotfix
```

### Part 10d: Rollback Scenario

```bash
# v1.1.1 has major issue
# Rollback to v1.1.0

# Option 1: Deploy old tag
git checkout v1.1.0
./deploy.sh

# Option 2: Revert the commit
git checkout main
git revert HEAD -m "Revert: v1.1.1 hotfix (caused issues)"
git tag v1.1.2

./deploy.sh  # Deploy v1.1.2

# View history
git --no-pager log --oneline --graph --all
```

### Part 10e: Using git bisect in Production

```bash
# Add test that would fail
cat > test.sh << 'EOF'
#!/bin/bash
# Fail if version contains "fixed"
if grep -q "fixed" api.js 2>/dev/null; then
  echo "Test failed!"
  exit 1
fi
exit 0
EOF
chmod +x test.sh

# Find when test started failing
git bisect start
git bisect bad HEAD
git bisect good v1.0.0
git bisect run ./test.sh

# Shows which commit introduced the issue
git bisect reset
```

**Key Takeaway**: Real production workflows combine branching strategies, tagging, and Git tools for operational tasks.

---

## Next Steps

After completing these exercises, you should:

1. **Practice explaining**: Teach someone else (rubber duck works too)
2. **Create your own scenarios**: Simulate your actual work environment
3. **Review real histories**: Look at open source projects
   ```bash
   git clone https://github.com/torvalds/linux.git
   cd linux
   git --no-pager log --oneline --graph --all | head -50
   ```
4. **Prepare interview stories**: Use these exercises as basis for "tell me about a time" questions

## Exercise Completion Checklist

- [ ] Exercise 1: Understand merge strategy differences
- [ ] Exercise 2: Resolve conflicts confidently
- [ ] Exercise 3: Explain rebase vs merge trade-offs
- [ ] Exercise 4: Use interactive rebase
- [ ] Exercise 5: Cherry-pick selectively
- [ ] Exercise 6: Recover with reflog
- [ ] Exercise 7: Find bugs with bisect
- [ ] Exercise 8: GitFlow release workflow
- [ ] Exercise 9: TBD workflow
- [ ] Exercise 10: Full production simulation

**Congrats!** You now have hands-on experience with Git mechanics that most engineers never get.
