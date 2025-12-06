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

### 10. [Automated Rollback & Circuit Breakers](./automated-rollback.md)
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

### 11. [Release Orchestration & Coordination](./release-orchestration.md)
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

```
Developer's Laptop (Local Development)
├── Working Directory: Chaos, experimentation
├── Staging Area (Index): Curated logical changes
├── Local Repository: Committed, in the DAG
├── Pre-commit hooks: Lint, format, secret detection (< 10s)
└── Pre-push hooks: Tests, build verification (2-5 min)
    ↓
Git Push
    ↓
CI Pipeline (Continuous Integration)
├── Same checks as local (verification)
├── Platform-specific tests
├── Integration tests
├── Security scans (SAST, dependency audit)
└── Build reproducible artifacts
    ↓
Artifact Registry
├── Versioned immutably (1.2.3-dev.42+abc123)
├── Tagged with metadata (commit SHA, build time)
└── SBOM generated
    ↓
Staging Environment
├── Deploy artifact (same binary as prod will get)
├── Integration tests with real services
└── Manual QA/approval
    ↓
Production Deployment
├── Progressive rollout (canary → 10% → 50% → 100%)
├── Feature flags for controlled exposure
├── Observability-driven gates (error rates, latency)
└── Automated rollback on metric deviation
```

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

| Need | GitFlow | Trunk-Based | GitHub Flow |
|------|---------|-------------|-------------|
| Scheduled releases | ✅ | ❌ | ❌ |
| Continuous deployment | ❌ | ✅ | ✅ |
| Multiple production versions | ✅ | ❌ | ❌ |
| Simple process | ❌ | ✅ | ✅ |
| High velocity | ❌ | ✅ | ⚠️ |

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
