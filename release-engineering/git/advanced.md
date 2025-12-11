# Advanced Git Topics for Senior+ Discussions

## Introduction

This file covers Git internals and advanced topics that come up in senior+ and architect-level interviews. You won't use these daily, but understanding them demonstrates deep technical knowledge and systems thinking.

---

## Part 1: Git Internals - How It Actually Works

### The Object Database

Git stores everything as objects in `.git/objects/`. There are four types:

1. **Blob**: File contents
2. **Tree**: Directory listing
3. **Commit**: Snapshot + metadata
4. **Tag**: Named reference to a commit

**Everything is content-addressed by SHA-1 hash** (moving to SHA-256).

### Examining Objects

```bash
# Create test repo
mkdir git-internals && cd git-internals
git init

# Create a file
echo "hello world" > file.txt
git add file.txt

# Find the blob object
git hash-object file.txt
# Output: 3b18e512dba79e4c8300dd08aeb37f8e728b8dad

# Examine the object
git cat-file -t 3b18e512  # Type: blob
git cat-file -p 3b18e512  # Print: hello world
git cat-file -s 3b18e512  # Size: 12

# Commit it
git commit -m "initial"

# Find commit object
git log --oneline
# abc123 initial

git cat-file -p abc123
```

**Output**:
```
tree def456
author Your Name <email> 1234567890 +0000
committer Your Name <email> 1234567890 +0000

initial
```

**Examine the tree**:
```bash
git cat-file -p def456
```

**Output**:
```
100644 blob 3b18e512... file.txt
```

**Key Insight**:
- Commits point to trees
- Trees point to blobs and other trees
- Everything is immutable and content-addressed

### Object Storage Optimization

Git uses two optimizations:

1. **Loose objects**: Individual files in `.git/objects/XX/...`
2. **Packfiles**: Delta-compressed bundles

```bash
# See loose objects
find .git/objects -type f

# Pack them
git gc

# Examine packfile
git verify-pack -v .git/objects/pack/pack-*.idx
```

**Why this matters**: Understanding packfiles helps debug large repo performance issues.

---

## Part 2: References and Refspecs

### What Are References?

References are pointers to commits, stored as files:

```bash
# Branch reference
cat .git/refs/heads/main
# Output: abc123def456... (commit SHA)

# Tag reference
cat .git/refs/tags/v1.0.0
# Output: abc123def456... (commit SHA)

# HEAD reference
cat .git/HEAD
# Output: ref: refs/heads/main (symbolic reference)
```

**Types**:
- `refs/heads/*`: Local branches
- `refs/remotes/origin/*`: Remote-tracking branches
- `refs/tags/*`: Tags
- `HEAD`: Current branch
- `FETCH_HEAD`: Last fetched commit
- `MERGE_HEAD`: Commit being merged

### Understanding Refspecs

Refspecs define how remote branches map to local references.

**Format**: `[+]<src>:<dst>`

**Example** (from `.git/config`):
```ini
[remote "origin"]
    url = https://github.com/user/repo.git
    fetch = +refs/heads/*:refs/remotes/origin/*
```

**Translation**:
- `refs/heads/*`: All branches on remote
- `refs/remotes/origin/*`: Store as remote-tracking branches locally
- `+`: Force update (even if not fast-forward)

**Custom refspecs**:
```bash
# Fetch only main branch
git fetch origin refs/heads/main:refs/remotes/origin/main

# Fetch PR branches
git fetch origin refs/pull/*/head:refs/remotes/origin/pr/*

# Now you can checkout PRs
git checkout origin/pr/123
```

**Why this matters**: Understanding refspecs helps configure custom workflows and troubleshoot fetch/push issues.

---

## Part 3: Git Hooks - Enforcing Team Standards

### What Are Hooks?

Scripts that run automatically on Git events. Located in `.git/hooks/`.

**Common hooks**:
- `pre-commit`: Before commit is created
- `commit-msg`: Validate commit message
- `pre-push`: Before pushing to remote
- `post-merge`: After successful merge
- `pre-receive`: Server-side, before accepting push
- `post-receive`: Server-side, after accepting push

### Example: Enforce Commit Message Format

**`.git/hooks/commit-msg`**:
```bash
#!/bin/bash
# Enforce conventional commits format

commit_msg=$(cat "$1")

# Pattern: type(scope): description
pattern="^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .{10,}$"

if ! echo "$commit_msg" | grep -qE "$pattern"; then
  echo "ERROR: Commit message doesn't follow conventional commits format"
  echo ""
  echo "Format: type(scope): description"
  echo "Types: feat, fix, docs, style, refactor, test, chore"
  echo "Example: feat(auth): add OAuth2 support"
  exit 1
fi
```

**Make it executable**:
```bash
chmod +x .git/hooks/commit-msg
```

**Test it**:
```bash
git commit -m "bad message"  # Fails
git commit -m "feat(api): add user endpoint"  # Succeeds
```

### Example: Prevent Commits to Main

**`.git/hooks/pre-commit`**:
```bash
#!/bin/bash
branch=$(git symbolic-ref HEAD | sed -e 's,.*/\(.*\),\1,')

if [ "$branch" = "main" ]; then
  echo "ERROR: Direct commits to main are not allowed"
  echo "Create a feature branch instead:"
  echo "  git checkout -b feature/your-feature"
  exit 1
fi
```

### Example: Run Tests Before Push

**`.git/hooks/pre-push`**:
```bash
#!/bin/bash
echo "Running tests before push..."

npm test

if [ $? -ne 0 ]; then
  echo "ERROR: Tests failed. Push aborted."
  exit 1
fi

echo "Tests passed. Pushing..."
```

### Team-Wide Hooks

Problem: `.git/hooks/` is not tracked by Git.

**Solution 1: Symlink from tracked directory**:
```bash
# Store hooks in repo
mkdir .githooks

# Create hook
cat > .githooks/pre-commit << 'EOF'
#!/bin/bash
echo "Running pre-commit checks..."
EOF

chmod +x .githooks/pre-commit

# Configure Git to use .githooks
git config core.hooksPath .githooks

# Commit the hooks directory
git add .githooks
git commit -m "Add team hooks"
```

**Solution 2: Use Husky (Node.js projects)**:
```bash
npm install --save-dev husky
npx husky install
npx husky add .husky/pre-commit "npm test"
git add .husky
```

**Why this matters**: Hooks enforce quality gates and prevent common mistakes. In interviews, discussing automated quality checks shows operational maturity.

---

## Part 4: Submodules vs Subtrees

### Problem: Managing Dependencies in Repos

You have a monorepo or need to include external repos.

### Git Submodules

**Concept**: Reference another repo at a specific commit.

**Setup**:
```bash
# Add submodule
git submodule add https://github.com/user/lib.git lib

# This creates:
# - .gitmodules file (config)
# - lib/ directory (the submodule)
# - Commit reference (not the code itself)

git commit -m "Add lib submodule"
```

**Clone repo with submodules**:
```bash
git clone https://github.com/user/main-repo.git
cd main-repo

# Submodule directories are empty!
# Initialize them:
git submodule init
git submodule update

# Or clone with submodules:
git clone --recursive https://github.com/user/main-repo.git
```

**Update submodule**:
```bash
cd lib
git pull origin main
cd ..

# Commit the new reference
git add lib
git commit -m "Update lib submodule"
```

**Pros**:
- Each repo is separate
- Clear dependency versions
- Smaller main repo

**Cons**:
- Complex workflow (init, update, checkout)
- Easy to get wrong (detached HEAD, uncommitted changes)
- CI/CD needs special handling

### Git Subtree

**Concept**: Copy another repo into a subdirectory of your repo.

**Setup**:
```bash
# Add subtree
git subtree add --prefix=lib https://github.com/user/lib.git main --squash

# This:
# - Fetches the lib repo
# - Adds it to lib/ directory
# - Creates a commit in your repo
```

**Update subtree**:
```bash
git subtree pull --prefix=lib https://github.com/user/lib.git main --squash
```

**Push changes back to lib**:
```bash
git subtree push --prefix=lib https://github.com/user/lib.git main
```

**Pros**:
- Simpler for consumers (just clone, everything is there)
- No special commands for cloning
- Works well for vendoring

**Cons**:
- History can get messy
- Larger repo size
- Harder to contribute back to dependency

### Decision Matrix

| Factor | Submodules | Subtrees |
|--------|-----------|----------|
| **Simplicity** | Complex | Simple |
| **Clone** | Need --recursive | Standard clone |
| **History** | Clean (separate) | Merged |
| **Updates** | Manual (init/update) | Automatic |
| **Contributing back** | Easy (separate repo) | Complex (subtree split) |
| **Best for** | Multiple deps, strict versions | Vendoring, simplified workflow |

**Interview discussion point**: "We evaluated submodules vs subtrees for our monorepo. We chose subtrees because developer onboarding was simpler - new engineers just clone and go. The trade-off is we can't easily contribute back to upstream, but we rarely need to since we vendor and fork."

---

## Part 5: Git Worktrees - Multiple Working Directories

### Problem: Switching Contexts Is Slow

You're working on a feature, production breaks, need to switch to main to hotfix.

**Traditional approach**:
```bash
git stash
git checkout main
# ... fix bug ...
git checkout feature
git stash pop
```

**Better: Git Worktree**

**Concept**: Multiple working directories from the same repo.

**Setup**:
```bash
# Main working directory
cd /path/to/repo

# Add worktree for hotfix
git worktree add ../repo-hotfix main

# Now you have:
# /path/to/repo (feature branch)
# /path/to/repo-hotfix (main branch)

cd ../repo-hotfix
# Make changes, commit
git commit -m "fix: critical bug"

# Push from either worktree
git push origin main
```

**List worktrees**:
```bash
git worktree list
```

**Remove worktree**:
```bash
git worktree remove ../repo-hotfix
# Or just delete the directory
rm -rf ../repo-hotfix
git worktree prune  # Clean up references
```

**Common workflow**:
```bash
# Main worktree for feature work
~/projects/app (feature/new-api)

# Worktrees for different contexts
~/projects/app-main (main) - hotfixes
~/projects/app-release (release/1.2.0) - release stabilization
~/projects/app-review (review-pr-123) - code review
```

**Why this matters**: Worktrees are a hidden productivity gem. Mentioning them shows you think about developer experience and efficiency.

---

## Part 6: Signing Commits with GPG

### Why Sign Commits?

**Problem**: Anyone can set `user.name` and `user.email`:
```bash
git config user.name "Linus Torvalds"
git config user.email "torvalds@linux-foundation.org"
git commit -m "Approve this PR"
```

**Solution**: Cryptographically sign commits with GPG.

### Setup

```bash
# Generate GPG key
gpg --full-generate-key
# Choose: RSA and RSA, 4096 bits

# List keys
gpg --list-secret-keys --keyid-format=long

# Output:
# sec   rsa4096/ABCD1234EF567890 2024-01-01 [SC]
#       ABCD1234EF567890 is your key ID

# Configure Git
git config --global user.signingkey ABCD1234EF567890
git config --global commit.gpgsign true  # Sign all commits

# Export public key
gpg --armor --export ABCD1234EF567890
# Add this to GitHub/GitLab settings
```

### Sign Commits

```bash
# Automatic (with commit.gpgsign = true)
git commit -m "feat: add feature"

# Manual
git commit -S -m "feat: add feature"

# Verify
git log --show-signature
```

**Output**:
```
commit abc123
gpg: Signature made Mon Jan 1 12:00:00 2024
gpg: using RSA key ABCD1234EF567890
gpg: Good signature from "Your Name <your@email.com>"
```

### Sign Tags

```bash
# Signed tag
git tag -s v1.0.0 -m "Release 1.0.0"

# Verify
git tag -v v1.0.0
```

**Why this matters**: Security-conscious organizations require signed commits. Shows you understand supply chain security.

---

## Part 7: Sparse Checkout - Working with Monorepos

### Problem: Monorepo is Too Large

```
monorepo/
├── frontend/     (you work here)
├── backend/
├── mobile/
├── infrastructure/
└── docs/
```

You don't want to checkout everything.

### Sparse Checkout

```bash
# Clone without checking out files
git clone --filter=blob:none --no-checkout https://github.com/user/monorepo.git
cd monorepo

# Enable sparse checkout
git sparse-checkout init --cone

# Checkout only frontend
git sparse-checkout set frontend

# Now working directory only has frontend/
ls
# Output: frontend/

# Add more directories
git sparse-checkout add docs

# View current sparse checkout
git sparse-checkout list
```

**Why this matters**: Working with large monorepos is common at scale. Showing you know sparse checkout demonstrates experience with large codebases.

---

## Part 8: Git Performance Optimization

### Large Repo Issues

**Symptoms**:
- Slow clone
- Slow fetch
- Large `.git/` directory
- Slow git status

### Techniques

**1. Shallow Clone**:
```bash
# Clone only recent history
git clone --depth=1 https://github.com/user/repo.git

# Fetch more history later
git fetch --depth=100
```

**2. Partial Clone**:
```bash
# Clone without blobs (files downloaded on-demand)
git clone --filter=blob:none https://github.com/user/repo.git

# Or clone without large blobs
git clone --filter=blob:limit=1m https://github.com/user/repo.git
```

**3. Aggressive Garbage Collection**:
```bash
git gc --aggressive --prune=now
```

**4. Remove Large Files from History**:
```bash
# Find large files
git rev-list --objects --all | \
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | \
  sed -n 's/^blob //p' | \
  sort --numeric-sort --key=2 | \
  tail -20

# Remove with BFG
bfg --strip-blobs-bigger-than 50M
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

**5. Optimize Config**:
```bash
# Faster status
git config core.untrackedCache true
git config core.fsmonitor true

# Parallel operations
git config fetch.parallel 8
git config submodule.fetchJobs 8

# Better compression
git config core.compression 9
git config pack.threads 2
```

**Interview discussion**: "When our monorepo hit 5GB, clone times became a developer experience problem. We implemented partial clones with blob filtering, reducing clone time from 10 minutes to 2 minutes. We also use sparse checkout so frontend developers only see frontend code."

---

## Part 9: Advanced Workflows

### Git Flow with Automation

**Automate version bumping**:
```bash
# .git/hooks/post-checkout
#!/bin/bash
if [ "$3" = "1" ]; then  # Branch checkout
  branch=$(git symbolic-ref --short HEAD)

  if [[ $branch =~ ^release/ ]]; then
    version=$(echo $branch | sed 's/release\///')
    echo "$version" > version.txt
    echo "Updated version.txt to $version"
  fi
fi
```

### Continuous Integration Patterns

**Prevent force push to protected branches**:
```bash
# .git/hooks/pre-push (server-side)
#!/bin/bash
while read local_ref local_sha remote_ref remote_sha; do
  if [ "$remote_ref" = "refs/heads/main" ]; then
    if [ "$remote_sha" != "0000000000000000000000000000000000000000" ]; then
      # Check if fast-forward
      merge_base=$(git merge-base "$remote_sha" "$local_sha")
      if [ "$merge_base" != "$remote_sha" ]; then
        echo "ERROR: Force push to main is not allowed"
        exit 1
      fi
    fi
  fi
done
```

### Git Bisect with Automation

**Complex bisect with build step**:
```bash
cat > bisect-test.sh << 'EOF'
#!/bin/bash
# Build the project
make clean && make || exit 125  # Exit 125 = skip this commit

# Run test
./run-test
exit $?
EOF

chmod +x bisect-test.sh

git bisect start HEAD v1.0.0
git bisect run ./bisect-test.sh
```

**Exit codes**:
- `0`: Good commit
- `1-127` (except 125): Bad commit
- `125`: Skip this commit

---

## Part 10: Security Considerations

### Preventing Secrets in Git

**Pre-commit hook**:
```bash
#!/bin/bash
# Check for common secret patterns

if git diff --cached | grep -E '(API|SECRET|TOKEN|PASSWORD|KEY).*=.*[A-Za-z0-9]{20,}'; then
  echo "ERROR: Possible secret detected in staged changes"
  echo "Please remove secrets before committing"
  exit 1
fi
```

**Tools**:
- git-secrets (AWS)
- gitleaks
- truffleHog

### Audit Log with Signed Commits

**View commit signatures**:
```bash
git log --show-signature --oneline

# Verify all commits in range
git verify-commit HEAD~10..HEAD
```

### Protecting Branches

**GitHub branch protection**:
- Require PR reviews
- Require status checks
- Require signed commits
- Prevent force push
- Prevent deletion

---

## Interview-Ready Talking Points

### When Asked About Git At Scale

**Topics to demonstrate knowledge**:

1. **Performance**: "We use partial clones and sparse checkout to handle our 10GB monorepo"

2. **Security**: "All production commits are GPG-signed and we use pre-commit hooks to scan for secrets"

3. **Quality**: "We enforce conventional commits via hooks and use git bisect in CI to automatically identify regressions"

4. **Workflow**: "We use git worktrees for parallel contexts - main worktree for features, separate ones for reviews and hotfixes"

5. **Internals**: "Understanding packfiles helped us debug why our CI was slow - we optimized GC settings and reduced pack times by 60%"

### When Asked About Managing Large Teams

**Git strategies**:
- Branch protection rules
- Required reviews
- Automated merge strategies
- Hooks for quality gates
- Signed commits for auditability

### When Asked About Monorepos

**Techniques**:
- Sparse checkout
- Partial clones
- Optimized GC
- Worktrees for context switching
- Submodules for versioned dependencies

---

## Further Learning

**Books**:
- "Pro Git" - Scott Chacon (Chapters 10-11 on internals)
- "Building Git" - James Coglan

**Resources**:
- Git source code: https://github.com/git/git
- Git documentation: https://git-scm.com/docs
- Git internals PDF: https://github.com/pluralsight/git-internals-pdf

**Practice**:
- Contribute to open source (real-world Git workflows)
- Set up your own Git server
- Implement a simple version control system to understand concepts

---

## Conclusion

These advanced topics won't come up in every interview, but having this knowledge demonstrates:

1. **Deep understanding** of tools, not just surface-level usage
2. **Systems thinking** (understanding how Git works internally)
3. **Operational maturity** (security, performance, scale)
4. **Problem-solving** (knowing when to use advanced features)

In senior+ interviews, this separates candidates who "use Git" from those who "understand Git."
