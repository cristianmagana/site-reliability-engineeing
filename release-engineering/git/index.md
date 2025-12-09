# Git for Release Engineering: Deep Dive

## What This Section Covers

This is your hands-on, first-principles guide to Git mechanics in the context of release engineering. Not just "how to use Git" but "how Git actually works and why it matters when you're shipping to production."

### Why This Matters for Interviews

When someone asks "How do you handle merges?" they're not looking for "we use squash merges." They want to know:

- **What happens** to the commit graph with different strategies
- **Why** you chose that approach (trade-offs)
- **How** it affects debugging, rollbacks, and bisecting
- **What** you'd do differently in different contexts

This section gives you the mental models to answer these questions at a senior+ level.

---

## Contents

### 1. [Git Mechanics and History](./git-mechanics-and-history.md)
**The comprehensive guide to how Git actually works**

Topics:
- What Git is (content-addressed filesystem, not diff storage)
- The three trees (working dir, index, HEAD)
- Merge strategies (fast-forward, no-ff, squash, three-way)
- Rebasing (what it does, when to use, interactive rebase)
- History manipulation (amend, reset, revert)
- Reading complex Git graphs
- Connection to release engineering practices

**When to read**: Start here for foundational understanding.

### 2. [Practical Exercises](./exercises.md)
**Hands-on scenarios you can run locally**

10+ exercises covering:
- Setting up different merge scenarios
- Resolving conflicts
- Practicing rebase workflows
- Interactive rebase for cleaning history
- Cherry-picking hotfixes
- Simulating production workflows
- Using git bisect to find bugs
- Recovery from mistakes (reflog)

**When to read**: After understanding the mechanics, build muscle memory here.

### 3. [Quick Reference](./quick-reference.md)
**Cheat sheet for common scenarios**

Fast lookups for:
- Which merge strategy when?
- Branching strategy → merge strategy mapping
- Common commands with flags explained
- "Oh crap" recovery scenarios
- Interview question templates

**When to read**: Use as a refresher before interviews or when you need a quick answer.

### 4. [Advanced Topics](./advanced.md)
**Deep dives for senior+ level discussions**

Topics:
- Git internals (objects, refs, packfiles)
- Refspecs and custom fetch/push
- Hooks for enforcing team standards
- Submodules vs subtrees for monorepos
- Git worktrees for parallel work
- Performance optimization (shallow clones, sparse checkout)
- Securing Git workflows (signed commits, GPG)

**When to read**: When you want to go deeper or prepare for architect-level discussions.

---

## How to Use This Section

### For Interview Prep (Time-constrained)

1. Read **git-mechanics-and-history.md** (30-45 min)
2. Do **exercises.md** exercises 1, 3, 6, 10 (30 min)
3. Review **quick-reference.md** (10 min)
4. Practice answering interview questions (see each file's "Interview-Ready" sections)

**Total time**: ~2 hours for solid foundation

### For Deep Understanding (Thorough)

1. Read **git-mechanics-and-history.md** fully
2. Complete all **exercises.md** (run every command, observe output)
3. Read **advanced.md** for internals
4. Build your own test scenarios
5. Contribute fixes to open-source projects using these techniques

**Total time**: 8-10 hours, but you'll understand Git better than 90% of engineers

### For Quick Reference (Day-of-Interview)

1. Skim **quick-reference.md** (15 min)
2. Review the "Interview-Ready Mental Models" sections in main doc
3. Practice explaining merge vs rebase out loud

---

## Connection to Release Engineering

Git is not just version control - it's the foundation of your release pipeline:

**Branching Strategy** ↔ **Git Operations**:
- GitFlow → merge commits (--no-ff)
- Trunk-Based Development → rebase + squash
- GitHub Flow → squash or rebase

**Release Operations** ↔ **Git Commands**:
- Tagging releases → git tag
- Hotfixes → cherry-pick
- Finding regressions → git bisect
- Rollbacks → deploy previous tag
- Audit trails → clean commit history

**Operational Excellence** ↔ **Git Practices**:
- Fast incident response → clean, bisectable history
- Easy rollbacks → semantic commits
- Clear audit trail → good commit messages
- Hotfix propagation → cherry-pick strategy

---

## Prerequisites

**Required**:
- Basic Git knowledge (clone, commit, push, pull)
- Command line comfort
- Understanding of basic branching

**Helpful**:
- Read [branching-strategies.md](../branching-strategies.md) first for context
- Familiarity with a Git hosting platform (GitHub/GitLab)
- Experience with merge conflicts

---

## Learning Objectives

After completing this section, you should be able to:

### Technical Skills
- [ ] Explain what a Git commit contains and how it's addressed
- [ ] Draw commit graphs for merge, rebase, and squash scenarios
- [ ] Use interactive rebase to clean up commit history
- [ ] Recover from common mistakes using reflog
- [ ] Use git bisect to find regressions
- [ ] Choose appropriate merge strategies for different contexts
- [ ] Configure Git for team consistency

### Interview Skills
- [ ] Articulate trade-offs between merge strategies
- [ ] Map branching strategies to merge strategies
- [ ] Explain how Git choices affect operational work (debugging, hotfixes)
- [ ] Discuss Git in the context of release engineering, not just "version control"
- [ ] Tell stories about using Git to solve production problems

### Mental Models
- [ ] Git as content-addressed filesystem (not diff storage)
- [ ] Commits form DAG, branches are pointers
- [ ] Rebase rewrites history (creates new commits)
- [ ] Clean history enables better operations
- [ ] Git is infrastructure, not just developer tool

---

## Quick Start: Run Your First Exercise

```bash
# Create a test repository
mkdir git-test && cd git-test
git init

# Create some commits
echo "line 1" > file.txt
git add file.txt
git commit -m "A: initial"

echo "line 2" >> file.txt
git commit -am "B: add line 2"

# Create a branch
git checkout -b feature
echo "line 3" >> file.txt
git commit -am "C: add line 3"

# Try different merge strategies
git checkout main

# Fast-forward (default)
git merge feature
git log --oneline --graph

# Reset and try no-ff
git reset --hard B
git merge --no-ff feature -m "Merge: feature"
git log --oneline --graph

# See the difference?
```

**What you should notice**:
- First merge: linear history (fast-forward)
- Second merge: merge commit visible in graph
- Different strategies, different histories

Now go to [exercises.md](./exercises.md) to continue!

---

## Interview Question Bank

### Behavioral
- "Tell me about a time you used Git to debug a production issue"
- "How do you handle merge conflicts in a team?"
- "Describe your workflow for contributing to a shared codebase"

### Technical
- "What's the difference between merge and rebase?"
- "When would you use cherry-pick?"
- "How does git bisect work?"
- "What happens during a three-way merge?"
- "Explain fast-forward vs. no-fast-forward"

### Strategic
- "How do you enforce Git standards across a team?"
- "What's your branching strategy and why?"
- "How does your Git workflow support your release process?"
- "What are the trade-offs between squash and merge commits?"

**Answers and frameworks**: See individual documents for detailed responses.

---

## Additional Resources

**Interactive Learning**:
- https://learngitbranching.js.org/ - Visual, interactive Git branching
- https://git-school.github.io/visualizing-git/ - Visualize Git operations

**Reference**:
- Pro Git book (free): https://git-scm.com/book
- Git man pages: `git help <command>`

**Practice**:
- Create test repos and experiment
- Contribute to open source
- Review PRs and examine different merge strategies

---

## Philosophy: History is for Humans

Remember: Git history is not just a record of what happened. It's a communication tool for future developers (including you) who need to understand:

- Why was this change made?
- What was the context?
- What other changes were part of this feature?
- When did this bug get introduced?

Your Git practices should optimize for **future understanding**, not just current convenience.

Clean commits → easier debugging → faster incident response → better uptime.

That's why Git matters in release engineering.
