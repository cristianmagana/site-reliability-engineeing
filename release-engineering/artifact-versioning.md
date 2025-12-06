# Artifact Versioning: Identity and Traceability

## Why Versioning Matters

Every artifact your build system produces needs an identity. Not just "the app" but "which exact build of the app, from which commit, with which dependencies, built when, and deployed where."

Without proper versioning:
- You can't reproduce bugs ("which version did the customer have?")
- You can't roll back safely ("which version was stable?")
- You can't track what's deployed ("is production on v1.2.3 or v1.2.4?")
- You can't coordinate releases ("did we ship the fix?")

Versioning is your audit trail, your rollback mechanism, and your deployment coordinator.

---

## Semantic Versioning (SemVer)

The de facto standard for software versioning.

### Format

```
MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD_METADATA]

Example: 2.3.1-rc.2+20240115.abc123
         │ │ │  │       │
         │ │ │  │       └─ Build metadata
         │ │ │  └───────── Pre-release identifier
         │ │ └──────────── Patch version
         │ └─────────────── Minor version
         └───────────────── Major version
```

### Version Component Rules

**MAJOR**: Incompatible API changes (breaking changes)
- v1.x.x → v2.0.0: You changed the API in a backward-incompatible way
- Example: Removed a function, changed function signature, changed behavior of existing API

**MINOR**: New functionality (backward compatible)
- v1.2.x → v1.3.0: You added features without breaking existing code
- Example: Added new function, new optional parameter, new API endpoint

**PATCH**: Bug fixes (backward compatible)
- v1.2.3 → v1.2.4: You fixed bugs without changing API
- Example: Fixed crash, corrected calculation, resolved memory leak

**PRERELEASE**: alpha, beta, rc (release candidate)
- v1.2.0-alpha.1: Early testing, unstable
- v1.2.0-beta.1: Feature complete, stabilizing
- v1.2.0-rc.1: Release candidate, final testing

**BUILD**: Build metadata (commit hash, build number, timestamp)
- Doesn't affect version precedence
- For traceability: which commit, which build number, when built

### Version Progression Examples

```
v1.0.0          # Initial stable release
v1.1.0          # New feature added
v1.1.1          # Bug fix
v1.2.0-alpha.1  # First alpha of next minor version
v1.2.0-alpha.2  # Second alpha
v1.2.0-beta.1   # First beta
v1.2.0-rc.1     # Release candidate
v1.2.0          # Stable release
v2.0.0          # Breaking change introduced
```

### Precedence Rules

How versions are sorted:

```
1.0.0 < 1.0.1 < 1.1.0 < 2.0.0                    # Basic ordering
1.0.0-alpha < 1.0.0-alpha.1 < 1.0.0-beta < 1.0.0-rc < 1.0.0  # Prerelease
1.0.0+build.1 == 1.0.0+build.2                   # Build metadata ignored
```

---

## Version Calculation Strategies

### Manual Versioning

Developer explicitly decides version:

```bash
git tag v1.2.3
git push --tags
```

**Pros**:
- Full human control
- Can use judgment for what constitutes breaking change

**Cons**:
- Error-prone (forget to bump, bump wrong number)
- Inconsistent (different developers use different criteria)
- Manual process (slows releases)

### Automated Versioning (Conventional Commits)

Version determined automatically from commit messages.

**Conventional Commit Format:**

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types and Version Impact:**

- `feat`: New feature → **MINOR** version bump (1.2.0 → 1.3.0)
- `fix`: Bug fix → **PATCH** version bump (1.2.0 → 1.2.1)
- `feat!` or `BREAKING CHANGE`: → **MAJOR** version bump (1.2.0 → 2.0.0)
- `docs`, `style`, `refactor`, `test`, `chore`: → No version bump (or PATCH in some configs)

**Example:**

```bash
# Commit history since v1.2.2
git log v1.2.2..HEAD --oneline

feat(auth): add OAuth2 login      → 1.2.2 → 1.3.0 (MINOR bump for new feature)
fix(api): resolve memory leak     → 1.2.2 → 1.2.3 (PATCH bump for bug fix)
feat!: redesign API structure     → 1.2.2 → 2.0.0 (MAJOR bump for breaking change)
```

**Tools:**

- **semantic-release**: Fully automated (analyzes commits, calculates version, creates tag, generates changelog, publishes)
- **standard-version**: Semi-automated (calculates version, lets you review before publishing)
- **commitizen**: Interactive commit message creation (helps write conventional commits)

**Benefits**:
- Consistent versioning across team
- Automated (no manual decision)
- Traceable (version derives from commit history)
- Enables automated changelogs

**Challenges**:
- Requires team discipline (everyone must write conventional commits)
- Commit message becomes API contract
- May need tooling to enforce format

---

## Branching Strategy Version Patterns

Different branching strategies have different versioning needs.

### GitFlow Versioning

```bash
Branch Type               → Version Pattern
─────────────────────────────────────────────────────
main                      → v1.2.3 (stable releases only)
develop                   → 1.3.0-dev.{BUILD_NUMBER}
feature/user-auth         → 1.3.0-feat-user-auth.{COMMIT_SHA}
release/1.3.0             → 1.3.0-rc.{BUILD_NUMBER}
hotfix/1.2.4              → 1.2.4-hotfix.{BUILD_NUMBER}
```

**Rationale**:
- **main**: Only stable, tagged releases
- **develop**: Development builds with incrementing build numbers
- **feature branches**: Include feature name for identification
- **release branches**: Release candidates (rc.1, rc.2, ...)
- **hotfix branches**: Patch versions

### Trunk-Based Development Versioning

```bash
Context                   → Version Pattern
─────────────────────────────────────────────────────
main (untagged)           → 1.3.0-dev.{BUILD}+{SHA}
main (tagged)             → 1.3.0
feature branch            → 1.3.0-{SHORT_SHA}
```

**Key Difference from GitFlow**: No explicit alpha/beta/rc phases. Instead:
- Development versions: `-dev` suffix
- Released versions: No suffix
- Feature flags control exposure, not version numbers

**Example Progression**:
```
main untagged: 1.3.0-dev.142+abc1234
main untagged: 1.3.0-dev.143+def5678
main tagged:   1.3.0                    ← Release
main untagged: 1.4.0-dev.144+ghi9012
```

---

## Immutability Principles

### The Golden Rule

**Once published, a version NEVER changes.**

If `myapp:1.2.3` exists, it must always refer to the exact same binary. Forever.

### Good Practice (Immutable Versions)

Each build gets a unique version:

```bash
myapp:1.2.3-dev.42        # Build 42
myapp:1.2.3-dev.43        # Build 43
myapp:1.2.3-dev.44        # Build 44
myapp:1.2.3               # Final release
```

Every version is unique. You can always get back to build 42.

### Bad Practice (Mutable Versions)

Overwriting versions:

```bash
myapp:1.2.3-dev           # Build 42
myapp:1.2.3-dev           # Build 43 (OVERWRITES build 42!)
```

Now you've lost build 42. If you need to debug an issue from that build, you can't reproduce it.

### Why Immutability Matters

**Debugging**: Customer reports bug in version 1.2.3. You pull `myapp:1.2.3`, reproduce bug, fix it. If version was mutable and changed, you might not reproduce the same bug.

**Rollback**: Need to rollback from v1.2.4 to v1.2.3. Pull `myapp:1.2.3` and deploy. If it was overwritten, you're deploying something different than what was previously in production.

**Audit Trail**: Which version is deployed? If versions are mutable, you can't trust version numbers.

---

## Mutable Convenience Tags

While specific versions are immutable, convenience tags can move.

### Immutable (never change)

```
myapp:1.2.3
myapp:1.2.3-rc.1
myapp:1.2.3-dev.42+abc123
```

These are specific versions. Once published, they never change.

### Mutable (updated with releases)

```
myapp:latest           # Points to newest stable
myapp:1.2               # Points to latest 1.2.x
myapp:1                 # Points to latest 1.x.x
myapp:stable            # Alias for latest stable
myapp:edge              # Points to latest development
```

These are pointers that move as new versions are released.

### Implementation Example

```bash
# Build and push immutable version
VERSION="1.2.3"
docker build -t myapp:${VERSION} .
docker push myapp:${VERSION}

# Update mutable tags
docker tag myapp:${VERSION} myapp:latest
docker tag myapp:${VERSION} myapp:1.2
docker tag myapp:${VERSION} myapp:1

docker push myapp:latest
docker push myapp:1.2
docker push myapp:1
```

### When to Use Each

**Immutable versions** for:
- Production deployments (never `latest` in prod!)
- Audit trails
- Reproducibility
- Rollbacks

**Mutable tags** for:
- Development environments (`edge` tag)
- Documentation examples (`latest`)
- User convenience ("just pull `latest`")

---

## Multi-Dimensional Versioning

Sometimes you need to encode multiple attributes in the version string.

### By Environment

```
{VERSION}-{ENVIRONMENT}-{BUILD}+{METADATA}
```

**Examples:**
```
1.2.3-production-142+abc123.amd64
1.2.3-staging-143+def456.amd64
1.2.3-dev-144+ghi789.arm64
```

**Use Case**: Different builds for different environments (dev, staging, prod).

### By Channel

```
{VERSION}-{CHANNEL}+{METADATA}
```

**Examples:**
```
1.2.3-stable+abc123
1.2.3-beta+def456
1.2.3-alpha+ghi789
```

**Use Case**: Different stability channels (Chrome stable vs. beta vs. canary).

### By Platform

```
{VERSION}-{OS}-{ARCH}+{METADATA}
```

**Examples:**
```
1.2.3-linux-amd64+abc123
1.2.3-linux-arm64+def456
1.2.3-windows-amd64+ghi789
1.2.3-darwin-arm64+jkl012
```

**Use Case**: Multi-platform builds (different binaries for different OS/arch combinations).

---

## Version Metadata Embedding

Embed version information in artifacts for traceability.

### Docker Labels

```dockerfile
ARG VERSION=unknown
ARG BUILD_DATE=unknown
ARG VCS_REF=unknown

LABEL org.opencontainers.image.version="$VERSION" \
      org.opencontainers.image.created="$BUILD_DATE" \
      org.opencontainers.image.revision="$VCS_REF" \
      org.opencontainers.image.source="https://github.com/myorg/myrepo"
```

**Query labels**:
```bash
docker inspect myapp:1.2.3 | jq '.[0].Config.Labels'
```

### Application Version Endpoint

```javascript
// Express.js example
app.get('/version', (req, res) => {
  res.json({
    version: process.env.VERSION,
    commit: process.env.GIT_COMMIT,
    buildDate: process.env.BUILD_DATE,
    branch: process.env.GIT_BRANCH
  });
});
```

```bash
# Query running app
curl https://api.example.com/version
{
  "version": "1.2.3",
  "commit": "abc123def456",
  "buildDate": "2024-01-15T10:30:00Z",
  "branch": "main"
}
```

### Build-Time Metadata Injection

```bash
# Extract git metadata
VERSION=$(git describe --tags)
COMMIT=$(git rev-parse HEAD)
BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
BRANCH=$(git rev-parse --abbrev-ref HEAD)

# Inject at build time
docker build \
  --build-arg VERSION=$VERSION \
  --build-arg VCS_REF=$COMMIT \
  --build-arg BUILD_DATE=$BUILD_DATE \
  -t myapp:$VERSION .
```

---

## Version Promotion Workflow

How versions progress through environments.

### The Promotion Model

```
Development → Staging → Production

feature-xyz     1.3.0-dev.42    1.3.0-rc.1      1.3.0
    ↓               ↓               ↓              ↓
  Build         Dev Deploy     Stage Deploy   Prod Deploy
    ↓               ↓               ↓              ↓
  Tests         Dev Tests      Stage Tests    Monitoring
    ↓               ↓               ↓              ↓
PR Merge      Auto-Deploy    Manual Approve  Manual Deploy
```

### Key Principle: Promote, Don't Rebuild

**Bad Practice (Rebuild for Each Environment)**:
```
Dev:     Build from commit abc123 → myapp:dev
Staging: Build from commit abc123 → myapp:staging
Prod:    Build from commit abc123 → myapp:prod
```

Problem: Three different builds. Not testing what you're deploying.

**Good Practice (Build Once, Promote)**:
```
Build: commit abc123 → myapp:1.2.3-dev.42

Dev:     Deploy myapp:1.2.3-dev.42
Staging: Deploy myapp:1.2.3-dev.42 (same artifact!)
Prod:    Tag as myapp:1.2.3 and deploy (same artifact!)
```

The exact same binary moves through all environments. What you test in staging is what you deploy to production.

---

## Version Registry and Tracking

Maintain an audit trail of versions.

### Version Registry Example

```yaml
releases:
  - version: '1.2.3'
    commit: 'abc1234567890'
    branch: 'main'
    build_number: 142
    timestamp: '2024-01-15T10:30:00Z'
    artifacts:
      docker: 'registry.com/myapp:1.2.3'
      npm: 'myapp@1.2.3'
      tarball: 's3://releases/myapp-1.2.3.tar.gz'
    deployed_to:
      - environment: 'production'
        timestamp: '2024-01-15T14:00:00Z'
        deployed_by: 'user@company.com'
      - environment: 'staging'
        timestamp: '2024-01-15T11:00:00Z'
        deployed_by: 'ci-system'
```

### Queries You Can Answer

- What version is in production? → `1.2.3`
- When was it deployed? → `2024-01-15T14:00:00Z`
- Who deployed it? → `user@company.com`
- What commit is it built from? → `abc1234567890`
- Where are the artifacts? → `registry.com/myapp:1.2.3`
- What's deployed in staging? → Also `1.2.3` (deployed earlier at 11:00)

---

## Best Practices

1. **Immutable Specific Versions**: Never overwrite published versions. `1.2.3` must always refer to the same binary.

2. **Automated Calculation**: Use Conventional Commits to determine version bumps automatically. Reduces human error.

3. **Branch-Specific Patterns**: Different version formats for different branches (dev, rc, stable).

4. **Metadata Embedding**: Include version info in artifacts themselves (Docker labels, /version endpoint).

5. **Mutable Convenience Tags**: Use `latest`, `stable`, `edge` for ease of use, but never for production deployments.

6. **Audit Trail**: Track what version is deployed where, when, and by whom.

7. **Promotion Path**: Clear progression through environments. Build once, promote the same artifact.

8. **Reproducibility**: Always able to rebuild exact version from git tag + build environment + lock files.

---

Versioning is your audit trail, your debugging tool, and your deployment coordinator. Get it right, and you have traceability and confidence. Get it wrong, and you're flying blind.
