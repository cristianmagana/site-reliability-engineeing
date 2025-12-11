# Release Engineering: From Local Development to Production

## What This Is

Release engineering is the discipline that spans from a developer's first keystroke to running code serving production traffic. It's where software engineering meets operational reality - the complete pipeline of how code is written, validated, versioned, built, and deployed.

This guide covers the full spectrum with deep dives into theory and practice, focusing on first principles, trade-offs, and architectural decisions.

## Core Topics

### 1. [Local Development](./local-development.md)

Where reliability actually starts. Covers:

- The three-state workspace topology (Working Directory → Index → Repository)
- How git actually works (content-addressable storage, DAG, Merkle trees)
- **Deep dive into the .git folder** - understanding git's internals
- Git hooks for local CI (pre-commit, commit-msg, pre-push)
- Offloading CI work to local development
- The economics of shift-left (1-10-100 cost multiplier)
- Environment determinism and dependency isolation

**Key Concepts**: Git internals, local validation strategies, economic cost of defects at different stages

### 2. [Continuous Integration](./continuous-integration.md)

Not just "running tests" - it's about reducing integration entropy. Covers:

- What CI actually is (vs. continuous building)
- The cost of delayed integration
- .gitattributes for cross-platform consistency
- Hermetic builds (same inputs → same outputs)
- Why GitFlow sabotages CI and TBD enables it
- CI maturity indicators

**Key Concepts**: Relationship between CI and branching strategy, hermetic builds, local/CI parity

### 3. [Branching Strategies](./branching-strategies.md)

How code flows through your organization. Covers:

- **GitFlow**: Structure, workflows, when to use it
- **Trunk-Based Development**: Continuous integration, feature flags
- **GitHub Flow**: Simplified model for small teams
- Comparison matrix and decision framework
- Version mapping for each strategy
- Feature flag management and progressive delivery

**Key Concepts**: Trade-offs between strategies, context-dependent decisions, impact on CI and deployment velocity

### 4. [Artifact Versioning](./artifact-versioning.md)

What version means and how to calculate it. Covers:

- Semantic Versioning (SemVer) - MAJOR.MINOR.PATCH
- Automated versioning with Conventional Commits
- Immutability principles (never overwrite published versions)
- Mutable convenience tags (latest, stable, edge)
- Multi-dimensional versioning
- Version promotion workflows
- Metadata embedding in artifacts

**Key Concepts**: Versioning strategy design, automation, audit trails, artifact promotion vs. rebuild

### 5. [Build Reproducibility](./build-reproducibility.md)

Same inputs, same outputs, every time. Covers:

- The reproducibility problem and why it matters
- Sources of non-determinism (timestamps, dependencies, environment)
- Dependency pinning and lock files
- Build environment pinning (Docker digests)
- SOURCE_DATE_EPOCH for time-deterministic builds
- The reproducibility chain (source → deps → env → process → artifact)
- Software Bill of Materials (SBOM)
- Hermetic build systems (Bazel, Nix)

**Key Concepts**: Security implications, debugging with reproducible builds, supply chain transparency

---

## Continuous Deployment

### 6. [Deployment Strategies](./deployment-strategies.md)

How to safely release software to production. Covers:

- **Rolling Updates**: Incremental replacement of old instances
- **Blue/Green Deployment**: Atomic cutover between environments
- **Canary Releases**: Progressive rollout to subset of traffic
- **Shadow Deployment**: Run new version alongside old, compare results
- **A/B Testing**: Traffic splitting for experimentation
- **Comparison matrix**: When to use each strategy
- **Infrastructure requirements**: Load balancers, orchestration, metrics

**Key Concepts**: Blast radius reduction, deployment vs. release, rollback strategies

### 7. [Progressive Delivery](./progressive-delivery.md)

Decoupling deploy from release. Covers:

- **The Deploy vs. Release distinction**: Binary placement vs. traffic exposure
- **Ring-based deployment**: Internal → canary → early adopters → everyone
- **Percentage-based rollouts**: 1% → 5% → 25% → 50% → 100%
- **User segmentation**: Geographic, by plan tier, by risk tolerance
- **Gradual traffic shifting**: Weighted routing, sticky sessions
- **Kill switches**: Instant rollback without re-deploy
- **Rollout automation**: Automated promotion based on metrics

**Key Concepts**: Risk quantification, controlled exposure, observability-driven decisions

### 8. [Feature Flags & Runtime Configuration](./feature-flags.md)

Managing features at runtime. Covers:

- **Feature flag patterns**: Boolean toggles, multivariate flags, operational flags
- **Flag lifecycle**: Creation → rollout → cleanup (avoiding flag rot)
- **Implementation strategies**: Configuration files, databases, dedicated services (LaunchDarkly, Split)
- **Technical debt management**: Flag expiration, cleanup enforcement
- **Targeting rules**: User attributes, percentage rollouts, time-based
- **Flag evaluation**: Client-side vs. server-side, performance considerations
- **Testing with flags**: Combinatorial explosion, flag override in tests

**Key Concepts**: Decoupling deployment from release, progressive exposure, managing complexity

### 9. [Observability for Deployments](./observability-deployments.md)

Knowing what's happening during rollouts. Covers:

- **The three pillars**: Metrics, logs, traces
- **Golden signals**: Latency, traffic, errors, saturation (Google SRE)
- **RED method**: Rate, errors, duration (for services)
- **USE method**: Utilization, saturation, errors (for resources)
- **Deployment markers**: Annotating metrics with release events
- **Comparing versions**: Canary vs. baseline metrics
- **Distributed tracing**: Following requests across services
- **Real-user monitoring**: Actual user experience vs. synthetic

**Key Concepts**: Data-driven decisions, signal vs. noise, correlation vs. causation

### 10. [Validation Frameworks](./validation-frameworks.md)

Proving deployments work before, during, and after rollout. Covers:

- **The validation problem**: Why testing isn't enough, confidence gap, production reality
- **Three validation horizons**: Pre-deployment, during deployment, post-deployment
- **Validation architecture patterns**: Fail-fast pipelines, validation services, shadow traffic
- **Validation metrics**: Golden signals, RED method, business metrics, data quality
- **Integration with deployment strategies**: Canaries, blue/green, feature flags
- **Progressive validation gates**: Stage-specific criteria, statistical significance
- **Continuous validation**: Synthetic monitoring, chaos engineering, invariant checking
- **Automated decision-making**: When to rollback, when to promote

**Key Concepts**: Risk quantification, confidence ladder, observability-validation loop, progressive confidence

### 11. [Performance Validation](./performance-validation.md)

Testing and observing system performance with statistical rigor. Covers:

- **First principles**: Performance as distribution, measurement overhead, context-dependency
- **Go performance testing**: Benchmarking with testing.B, profiling (CPU, memory, trace), goroutine leak detection
- **Statistical comparison**: benchstat for significance testing, avoiding false positives
- **CI/CD integration**: Baseline establishment, regression gates, dealing with environmental noise
- **Load testing**: Types (baseline, stress, soak, spike), tools (vegeta, k6), capacity validation
- **Performance observability**: RED metrics, runtime metrics, continuous profiling, SLO alerting
- **Validation patterns**: Performance budgets, canary analysis, regression test suites

**Key Concepts**: Statistical rigor, performance budgets, progressive validation, instrumentation-first

### 12. [Automated Rollback & Circuit Breakers](./automated-rollback.md)

Failing safely and automatically. Covers:

- **Health checks**: Liveness, readiness, startup probes
- **Circuit breaker pattern**: Fail fast, prevent cascading failures
- **Automated rollback triggers**: Error rate thresholds, latency SLOs, saturation limits
- **Rollback mechanisms**: Version pinning, traffic shifting, DNS cutover
- **The decision algorithm**: When to rollback vs. rollforward
- **False positive prevention**: Statistical significance, warm-up periods
- **Human override**: Manual approval, emergency break-glass
- **Post-rollback**: Incident response, root cause analysis

**Key Concepts**: Defensive deployment, fail-safe mechanisms, observability-driven automation

### 13. [Release Orchestration & Coordination](./release-orchestration.md)

Managing complex, multi-service releases. Covers:

- **Service dependencies**: DAG of deployment order
- **Database migrations**: Schema changes, backward compatibility, dual-write patterns
- **API versioning**: Breaking changes, deprecation timelines
- **Coordinated rollouts**: Deploying multiple services in sequence
- **Release trains**: Scheduled deployment windows
- **Deployment locks**: Preventing simultaneous conflicting deploys
- **Rollback coordination**: Cascading rollbacks across services
- **Communication**: Status pages, internal notifications, customer communication

**Key Concepts**: Distributed systems deployment, dependency management, coordination overhead

## How Code Flows: The Complete Picture

### Local Development: The Shift-Left Foundation (1x Cost)

**The economic imperative**: Catching issues locally is 10-100x cheaper than finding them later.

```
┌─────────────────────────────────────────────────────────────────┐
│ DEVELOPER'S LAPTOP (Local Development)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Working Directory                                              │
│  ├─ Chaos, experimentation, work-in-progress                    │
│  ├─ Not tracked by git yet                                      │
│  └─ Local testing, debugging                                    │
│                                                                 │
│          │                                                      │
│          │ git add (stage changes)                              │
│          ↓                                                      │
│                                                                 │
│  Staging Area (Index)                                           │
│  ├─ Curated logical changes                                     │
│  ├─ Reviewed diff (git diff --staged)                           │
│  └─ Ready for commit                                            │
│                                                                 │
│          │                                                      │
│          │ git commit -m "message"                              │
│          ↓                                                      │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ PRE-COMMIT HOOK (.git/hooks/pre-commit)                   │  │
│  ├───────────────────────────────────────────────────────────┤  │
│  │ Goal: Fast feedback, prevent broken commits (< 10 seconds)│  │
│  │                                                           │  │
│  │ Runs BEFORE commit is created:                            │  │
│  │  ✓ Linting (gofmt, golint, eslint)         [1-2s]         │  │
│  │  ✓ Code formatting (auto-fix)              [1s]           │  │
│  │  ✓ Secret detection (git-secrets)          [2-3s]         │  │
│  │  ✓ Trailing whitespace removal             [<1s]          │  │
│  │  ✓ Syntax validation (go vet, tsc)         [2-3s]         │  │
│  │  ✓ Commit message format (conventional)    [<1s]          │  │
│  │                                                           │  │
│  │ If FAIL: Commit is BLOCKED, fix locally                   │  │
│  │ If PASS: Commit proceeds to local repository              │  │
│  │                                                           │  │
│  │ Why: Immediate feedback while still in context            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│          │                                                      │
│          │ Commit created and added to DAG                      │
│          ↓                                                      │
│                                                                 │
│  Local Repository (.git/)                                       │
│  ├─ Commit SHA: abc123def456                                    │
│  ├─ In the DAG, part of git history                             │
│  ├─ Can be amended, rebased (not pushed yet)                    │
│  └─ Ready to push to remote                                     │
│                                                                 │
│          │                                                      │
│          │ git push origin feature-branch                       │
│          ↓                                                      │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ PRE-PUSH HOOK (.git/hooks/pre-push)                       │  │
│  ├───────────────────────────────────────────────────────────┤  │
│  │ Goal: Comprehensive validation before sharing (2-5 min)   │  │
│  │                                                           │  │
│  │ Runs BEFORE commits are pushed to remote:                 │  │
│  │  ✓ Unit tests (go test ./...)             [30-60s]        │  │
│  │  ✓ Code coverage threshold (>80%)         [included]      │  │
│  │  ✓ Build verification (go build)          [20-40s]        │  │
│  │  ✓ Integration tests (local DB)           [30-60s]        │  │
│  │  ✓ Security analysis (gosec, semgrep)     [10-20s]        │  │
│  │  ✓ Dependency vulnerability scan          [10-20s]        │  │
│  │  ✓ License compliance check               [5-10s]         │  │
│  │  ✓ Performance benchmarks (critical)      [30-60s]        │  │
│  │                                                           │  │
│  │ If FAIL: Push is BLOCKED, investigate locally             │  │
│  │ If PASS: Push proceeds to remote repository               │  │
│  │                                                           │  │
│  │ Why: Prevent broken code from polluting shared branches   │  │
│  │ Bypass: git push --no-verify (use sparingly!)             │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

          │
          │ Push to remote (GitHub, GitLab)
          ↓

┌──────────────────────────────────────────────────────────────────┐
│ CI PIPELINE (Continuous Integration) - 10x Cost                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Why CI when we have local hooks?                                │
│  ├─ Verification: Ensure developer didn't bypass hooks           │
│  ├─ Platform-specific: Test on Linux, macOS, Windows             │
│  ├─ Matrix testing: Multiple Go versions, Node versions          │
│  ├─ Integration: Test with real external services (not mocked)   │
│  ├─ Comprehensive: Longer-running tests not feasible locally     │
│  └─ Artifact creation: Build production binaries                 │
│                                                                  │
│  Pipeline stages:                                                │
│  ├── Same checks as local (verification)       [2-3 min]         │
│  ├── Platform-specific tests (OS matrix)       [5-10 min]        │
│  ├── Integration tests (real DB, services)     [10-15 min]       │
│  ├── Security scans (SAST, SCA, containers)    [5-10 min]        │
│  ├── Build reproducible artifacts              [3-5 min]         │
│  ├── Performance benchmarks (detailed)         [10-20 min]       │
│  └── E2E tests (full system)                   [15-30 min]       │
│                                                                  │
│  Total: 20-60 minutes (parallelized)                             │
│                                                                  │
│  If FAIL: Block PR merge, notify developer (context switch cost) │
│  If PASS: Artifact promoted to registry                          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

          │
          ↓

Artifact Registry
├── Versioned immutably (1.2.3-dev.42+abc123)
├── Tagged with metadata (commit SHA, build time, CI job ID)
├── SBOM generated (supply chain transparency)
└── Signatures (cosign for container images)

          ↓

Staging Environment
├── Deploy artifact (same binary as prod will get)
├── Integration tests with real services
├── Performance load testing (smoke + full)
├── Manual QA/approval (exploratory testing)
└── Security penetration testing

          ↓

Production Deployment - 100x Cost if broken
├── Progressive rollout (canary → 10% → 50% → 100%)
├── Feature flags for controlled exposure
├── Observability-driven gates (error rates, latency, saturation)
├── Performance validation (P99 latency, throughput)
└── Automated rollback on metric deviation
```

### The Validation Pyramid: Layered Defense

**Why multiple validation stages?**

Each stage has different characteristics and catches different classes of issues:

| Stage          | Speed     | Cost | Scope                       | Catches                                     |
| -------------- | --------- | ---- | --------------------------- | ------------------------------------------- |
| **Pre-commit** | < 10s     | 1x   | Syntax, style, secrets      | Trivial errors, format issues               |
| **Pre-push**   | 2-5 min   | 1x   | Logic, unit tests, build    | Functional bugs, test failures              |
| **CI**         | 20-60 min | 10x  | Integration, platforms      | Cross-platform issues, integration failures |
| **Staging**    | Hours     | 50x  | Full system, realistic data | Configuration issues, scale problems        |
| **Canary**     | Hours     | 100x | Production traffic          | Production-only issues, edge cases          |

**The shift-left principle**: Move validation as early as possible. Fixing in pre-commit is instant, fixing in production requires incident response.

**Economic model (1-10-100 rule)**:

- **Fix locally**: 1x cost (seconds, in context, no context switch)
- **Fix in CI**: 10x cost (minutes, context switch, blocking team)
- **Fix in production**: 100x cost (incident response, customer impact, reputational damage)

### Release Pipeline Structure

**End-to-End Pipeline Architecture** (GitFlow-based workflow):

This shows how git hooks integrate with the broader CI/CD pipeline.

```
Developer's Laptop
┌───────────────────────────────────────┐
│  Working Directory                    │
│         ↓ git add                     │
│  Staging Area (Index)                 │
│         ↓ git commit                  │
│  ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓   │
│  ┃ PRE-COMMIT HOOK                ┃   │
│  ┃ - Lint, format, secrets (10s)  ┃   │
│  ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛   │
│         ↓ if pass                     │
│  Local Repository (commit created)    │
│         ↓ git push                    │
│  ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓   │
│  ┃ PRE-PUSH HOOK                  ┃   │
│  ┃ - Tests, build, scan (2-5 min) ┃   │
│  ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛   │
└───────────────────────────────────────┘
         ↓ if pass
    Push to Remote
         ↓
┌────────────────────────────────────────────┐
│   Feature Branch / develop                 │
│   ────────────────────────────────────     │
│   CI Pipeline (triggered on push):         │
│   - Verify local checks ran                │
│   - Unit tests (all platforms)             │
│   - Linting & code coverage                │
│   - Integration tests                      │
│   - Security scans (SAST/SCA)              │
│                                            │
│   Time: 10-20 minutes                      │
│   Status: Must pass before PR approval     │
└──────┬─────────────────────────────────────┘
       │ PR Review + Approval
       │ Merge to develop/main
       v
┌─────────────────────────────────────────────┐
│   Release Branch (release/v1.3.0)           │
│   ─────────────────────────────────────     │
│   CI Pipeline (triggered on branch create): │
│   - All feature branch checks               │
│   - Integration tests (full suite)          │
│   - Build artifacts (multi-platform)        │
│   - Container images (multi-arch)           │
│   - Security scanning (full depth)          │
│   - SBOM generation                         │
│   - Artifact signing                        │
│                                             │
│   Time: 30-45 minutes                       │
│   Output: Versioned artifacts in registry   │
└──────┬──────────────────────────────────────┘
       │ Create RC Tag
       v
┌─────────────────────────────────────────────┐
│   Tag (v1.3.0-rc.1)                         │
│   ─────────────────────────────────────     │
│   CD Pipeline (triggered on RC tag):        │
│   - Deploy to staging environment           │
│   - Smoke tests (critical paths)            │
│   - Performance load tests (full suite)     │
│   - Security penetration testing            │
│   - Manual QA/exploratory testing           │
│   - Performance budget validation           │
│   - Integration with external systems       │
│                                             │
│   Time: 1-4 hours                           │
│   Gate: Manual approval required            │
└──────┬──────────────────────────────────────┘
       │ QA Approval
       │ Create Production Tag
       v
┌─────────────────────────────────────────────┐
│   Production Tag (v1.3.0)                   │
│   ─────────────────────────────────────     │
│   Production Deployment:                    │
│   - Canary deployment (5% traffic)          │
│      └─ Validation: 10 min, golden signals  │
│   - Expand to 25% traffic                   │
│      └─ Validation: 15 min, metrics + logs  │
│   - Expand to 50% traffic                   │
│      └─ Validation: 30 min, full observ.    │
│   - Full rollout (100% traffic)             │
│      └─ Continuous monitoring (24h)         │
│                                             │
│   Automated gates:                          │
│   - Error rate < baseline + 0.5%            │
│   - P99 latency < SLO + 10%                 │
│   - No crashes, goroutine leaks             │
│                                             │
│   Automated rollback triggers:              │
│   - SLO violation sustained > 5 min         │
│   - Error budget burn rate > 10x            │
│   - Crash or panic detected                 │
│                                             │
│   Time: 2-6 hours for full rollout          │
└─────────────────────────────────────────────┘
```

### Key Stages and Their Purpose

**Local (Pre-commit/Pre-push)**:

- **Goal**: Fast, private feedback
- **Cost**: 1x (developer's time, in-context)
- **Validation**: Syntax, style, unit tests, critical paths
- **Failure**: Developer fixes immediately, no team visibility

**Feature Branch CI**:

- **Goal**: Verify local hooks, platform-specific testing
- **Cost**: 10x (shared CI resources, context switch)
- **Validation**: All platforms, full integration suite
- **Failure**: Blocks PR, team visible, requires investigation

**Release Branch CI**:

- **Goal**: Create shippable artifacts
- **Cost**: 10x (longer pipeline, artifact creation)
- **Validation**: Comprehensive, security-focused, artifact integrity
- **Failure**: Blocks release creation, escalation to team leads

**RC Tag (Staging)**:

- **Goal**: Validate in production-like environment
- **Cost**: 50x (staging infrastructure, manual QA time)
- **Validation**: Performance, integration, exploratory testing
- **Failure**: Blocks production deployment, requires fix + re-test

**Production Tag (Canary)**:

- **Goal**: Safe production rollout with real traffic
- **Cost**: 100x (production impact, customer-facing)
- **Validation**: Real user traffic, observability-driven
- **Failure**: Automated rollback, incident response, customer communication

### The Shift-Left Philosophy in Action

**Notice the pattern**:

1. **Fastest checks first** (pre-commit: 10s)
2. **Comprehensive checks local** (pre-push: 2-5 min)
3. **Verification + platform testing** (CI: 20-60 min)
4. **Realistic environment** (staging: hours)
5. **Production validation** (canary: hours with progressive confidence)

**Why this works**:

- 90% of issues caught locally (1x cost)
- 9% caught in CI (10x cost, but before staging)
- 0.9% caught in staging (50x cost, but before production)
- 0.1% caught in production canary (100x cost, but limited blast radius)

**The alternative (no local hooks)**:

- Wait 30-60 minutes for CI to discover linting errors
- Context switch while waiting
- Block CI resources for trivial issues
- Slow down entire team

## Designing a Release Pipeline

A complete release pipeline addresses:

1. **Local development setup** - Environment parity, hooks for early validation
2. **CI strategy** - Hermetic builds, caching, fail-fast checks
3. **Branching model** - Trunk-Based for velocity, GitFlow for stability, trade-offs
4. **Versioning scheme** - SemVer, automation, immutability
5. **Build reproducibility** - Lock files, containerized builds, SOURCE_DATE_EPOCH
6. **Deployment strategy** - Canary, blue/green, feature flags
7. **Observability gates** - Automated rollback on golden signal deviations

## Core Principles

Understanding these concepts is fundamental:

- **Git internals**: How objects are stored, why branches are cheap, what the reflog is
- **The 1-10-100 rule**: Cost multiplier for fixing issues (local vs CI vs production)
- **Hermetic builds**: Why same code can produce different binaries, how to prevent it
- **Conventional Commits**: How to automate versioning from commit messages
- **Feature flags vs branches**: When to use each, lifecycle management
- **Error budgets and release velocity**: The optimization function between CoD and CoF

## Key Mental Models

### Release Engineering is Risk Management

Not "zero defects" but "optimal velocity within acceptable risk":

- Cost of Delay (CoD): Value lost by not shipping
- Cost of Failure (CoF): Damage from shipping broken code
- Error budgets: Automate the velocity/stability trade-off

### Shift-Left is Economics, Not Just Best Practice

Fix issues earlier = exponentially cheaper:

- Local: 1x cost (seconds, in context)
- CI: 10x cost (minutes, context switch)
- Production: 100x cost (incident response, customer impact)

### Reproducibility Enables Everything

Same source + same deps = same binary:

- Security: Verify binaries match audited source
- Debugging: Reproduce exact build for investigation
- Compliance: Audit trail of what's deployed
- Rollback: Rebuild old versions with confidence

## Domain-Specific Considerations

Different domains have unique release engineering challenges:

**Systems Software (Drivers, Kernels)**:

- Multi-architecture builds (AMD64, ARM64, specialized hardware)
- Critical reproducibility (bugs are expensive, exact reproduction essential)
- Complex version matrices (driver × kernel × OS combinations)
- Compiler sensitivity (version changes can affect performance)
- Long support cycles (builds from years ago must be reproducible)

**SaaS Applications**:

- Continuous deployment (multiple releases per day)
- Feature flags for progressive rollout
- Rapid iteration and experimentation
- Single production version

**Mobile Apps**:

- Scheduled releases (app store approval processes)
- Multiple versions in wild (users don't update)
- Can't force updates or instant rollback

**Enterprise Software**:

- Long release cycles (quarters, not days)
- Multiple versions supported simultaneously
- Formal change control processes
- Extensive backwards compatibility

## Quick Reference

### Branching Strategy Decision Matrix

| Need                         | GitFlow | Trunk-Based | GitHub Flow |
| ---------------------------- | ------- | ----------- | ----------- |
| Scheduled releases           | ✅      | ❌          | ❌          |
| Continuous deployment        | ❌      | ✅          | ✅          |
| Multiple production versions | ✅      | ❌          | ❌          |
| Simple process               | ❌      | ✅          | ✅          |
| High velocity                | ❌      | ✅          | ⚠️          |

### Version Format Patterns

```
Release:            1.2.3
Pre-release:        1.2.3-rc.1
Development:        1.2.3-dev.42+abc123
Feature branch:     1.2.3-feat-name.abc123
Multi-arch:         1.2.3-linux-amd64
```

### Reproducibility Checklist

- [ ] Dependencies pinned in lock file
- [ ] Lock file committed to git
- [ ] Base images pinned by digest (not just tag)
- [ ] Build tool versions specified
- [ ] SOURCE_DATE_EPOCH set to git commit time
- [ ] Dockerfile.build committed and versioned
- [ ] SBOM generated for each build
- [ ] Verification build tested

---

Release engineering is the discipline that makes software delivery reliable, repeatable, and efficient. Understanding these principles - from git's content-addressable storage to progressive delivery strategies to the economics of shift-left testing - is fundamental to building robust software delivery systems.
