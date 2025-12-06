# Continuous Integration Philosophy

## What CI Actually Is (And Isn't)

"We use GitHub Actions, so we have CI" - No. That's continuous building, not continuous integration.

"We run tests on every commit" - Getting warmer, but still not there.

**Real CI**: Merging all developer work to a shared mainline multiple times per day, with automated verification at each integration point.

The distinction matters because CI isn't a tool - it's a practice designed to reduce integration entropy.

## Why CI Exists: The Cost of Delayed Integration

The longer a branch lives, the worse everything gets:

**Merge Complexity**: Hour-old branches have trivial conflicts (you changed line 10, I changed line 15). Week-old branches have semantic conflicts - we both refactored the same function in incompatible ways, and git can't tell.

**Integration Risk**: Your feature works. My feature works. Put them together? Broken. The longer we wait to discover this, the more expensive the fix.

**Feedback Latency**: You introduced a bug Monday. If you integrate Monday afternoon, you find it Monday. If you integrate Friday, you find it Friday, but you've built three more features on top of that bug.

---

## CI Starts Locally, Not on the Build Server

If CI fails but "it works on my machine," your CI isn't trustworthy - it's just noisy. Effective CI requires local environments that are high-fidelity replicas of CI.

The build shouldn't pass locally and fail in CI, or vice versa. If it does, you have an environment problem, and your CI feedback loop is broken.

### .gitattributes: The File Everyone Ignores (But Shouldn't)

Everyone knows `.gitignore`. Almost nobody knows `.gitattributes`, despite it being critical for cross-platform consistency.

`.gitignore` tells git what to ignore. `.gitattributes` tells git how to handle what it tracks. This is infrastructure-as-code for your version control behavior.

#### 1. Line Ending Normalization (The Silent Killer)

macOS/Linux use `LF`. Windows uses `CRLF`. Without `.gitattributes`, this causes:
- Massive "whitespace only" diffs that hide real changes
- Broken bash scripts (bash doesn't like `CRLF`)
- Invalidated build caches
- Failed CI builds that work locally (or vice versa)

**Solution**: Enforce `LF` everywhere in the repo.

```gitattributes
# .gitattributes

# Default behavior
* text=auto

# Explicitly enforce LF for text files
*.c     text eol=lf
*.h     text eol=lf
*.js    text eol=lf
*.ts    text eol=lf
*.json  text eol=lf
*.md    text eol=lf
*.sh    text eol=lf
```

Now Windows developers still see `CRLF` in their editor (if they want), but git stores and checks out `LF`. Consistency achieved.

#### 2. Semantic File Handling

Control how platforms interpret your files for better code review:

```gitattributes
# Don't try to diff Xcode project files as text (they're XML but unreadable)
*.pbxproj binary

# Mark generated files
# - Collapsed by default in PR diffs
# - Don't count toward language stats
package-lock.json linguist-generated
yarn.lock linguist-generated
*.min.js linguist-generated
dist/** linguist-generated
```

This keeps your PR diffs focused on actual code changes, not lockfile churn.

#### 3. Custom Merge Strategies

For files where merge conflicts are always resolved the same way:

```gitattributes
# Always use "ours" for this config file (never merge)
database.xml merge=ours

# Use a custom merge driver for package.json
package.json merge=npm-merge-driver
```

#### 4. Artifact Hygiene with export-ignore

`git archive` creates release tarballs. Exclude dev-only files:

```gitattributes
# Don't include these in release archives
.gitignore      export-ignore
.gitattributes  export-ignore
/tests          export-ignore
/docs           export-ignore
Dockerfile      export-ignore
.github/        export-ignore
```

Your release artifacts stay clean without maintaining a separate build script to delete files.

---

## Hermetic Builds: Same Inputs, Same Outputs, Every Time

A hermetic build doesn't care about the host environment. Same source code + same dependencies = same output, whether you build on your laptop, in CI, or on a server 5 years from now.

**The Anti-Pattern (Host-Dependent Builds)**:
```bash
# Uses whatever version of gcc/python/node is installed on the host
gcc main.c -o myapp
python setup.py build
npm run build
```

Your laptop has Node 18, CI has Node 20, production built with Node 16. Good luck debugging that.

**The Pattern (Hermetic Builds)**:
- Run builds inside containers (Docker, Podman)
- Use hermetic build tools (Bazel, Nix)
- Pin everything - language versions, tool versions, dependencies

### CI Should Invoke, Not Define

**Bad**: CI config contains the build logic
```yaml
# .github/workflows/ci.yml
steps:
  - run: npm install
  - run: npm run lint
  - run: npm test
  - run: npm run build
```

Now your build logic is scattered across the repo AND the CI config. Want to run CI locally? Good luck.

**Good**: CI invokes a build command defined in the repo
```yaml
# .github/workflows/ci.yml
steps:
  - run: make ci
```

Where `make ci` uses Docker to run the exact same environment as CI. **Local == Remote**. You can run the full CI pipeline on your laptop.

---

## CI and Branching Strategy: Inextricably Linked

Question: Does branching strategy fall under CI?

Answer: Absolutely. Your branching strategy either enables CI or sabotages it.

### Trunk-Based Development Needs Fast CI

TBD requires merging to main multiple times per day. This is impossible if:
- **CI takes 1 hour**: Developers can't merge multiple times per day if they're waiting an hour for feedback
- **CI is flaky**: Developers lose trust and start batching changes to reduce CI runs, destroying the "continuous" part

Fast, reliable CI is a prerequisite for TBD, not a nice-to-have.

### GitFlow Sabotages CI

GitFlow uses long-lived feature branches. You can run "CI" on feature branches, but you're doing **Continuous Building**, not **Continuous Integration**.

Integration only happens when you merge to `develop`. If your feature branch lives for two weeks, you haven't integrated for two weeks. The "I" in CI is missing.

The moment you create a long-lived branch, you've delayed integration, and integration delay is exactly what CI is designed to eliminate.

---

## The CI Pyramid: Hierarchy of Needs

Your CI needs these, in order:

1. **Determinism**: Same code + same deps = same result, every time
   - Pin dependencies (lockfiles)
   - Use containers for builds
   - Set SOURCE_DATE_EPOCH for reproducible timestamps

2. **Speed**: Feedback < 10 minutes (ideally < 5)
   - Parallelize tests
   - Cache dependencies
   - Run fast checks first (fail fast)

3. **Reliability**: Zero tolerance for flaky tests
   - Quarantine flaky tests immediately
   - Track flakiness metrics
   - Fix or delete, never ignore

4. **Comprehensive**: Catch everything
   - Unit tests
   - Integration tests
   - Linting
   - Security scans
   - Build verification

## CI Maturity Indicators

How to tell if your CI is actually mature:

### 1. The Green Build Cult
Is a red build treated as a production incident? Or does everyone just ignore it?

Mature orgs: Broken builds are "stop the line" emergencies. Everything stops until it's fixed.

Immature orgs: "Oh the build is red again, just push anyway, someone will fix it eventually."

### 2. Test Quarantine
Flaky tests are poison. They train developers to ignore test failures.

Mature approach:
```yaml
# Quarantine: Run flaky tests but don't block merges
- run: pytest tests/flaky/
  continue-on-error: true

# Create issue for flaky tests
- run: create-issue "Test xyz is flaky, quarantine or fix"
```

### 3. Intelligent Caching
```
No cache:        Install deps every run (slow)
Broken cache:    Cache invalidates too often (still slow)
Good cache:      Cache deps, invalidate only on lockfile change
Great cache:     Distributed cache shared across all developers
```

### 4. Fail Fast
```bash
# Bad: Run slow tests first, find lint error after 20 minutes
tests → integration → lint (fail!)

# Good: Run fast checks first, fail in 30 seconds
lint (fail!) → (never get here)
```
