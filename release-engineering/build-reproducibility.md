# Build Reproducibility: Same Inputs, Same Outputs, Every Time

## The Reproducibility Problem

**Definition**: Build reproducibility is the ability to recreate identical binaries from the same source code, even when built at different times, on different machines, or by different people.

### Why It Matters

**Security Verification**:
- Prove binary matches audited source
- Detect supply chain attacks
- Verify no malicious code injected during build

**Debugging**:
- Reproduce exact build for investigation
- Customer reports bug → pull exact binary → reproduce
- Bisect builds to find which introduced regression

**Compliance**:
- Regulatory requirements (SOC2, FDA, financial services)
- Audit trail of what's deployed
- Prove binary provenance

**Trust**:
- Verify published binaries haven't been tampered with
- Third-party verification (someone else can build and verify)
- Supply chain transparency

### The Core Equation

```
Same Source Code + Same Dependencies + Same Build Environment
= Same Binary (bit-for-bit identical)
```

If you rebuild from the same git commit a year later, you should get the exact same binary hash.

---

## Sources of Non-Determinism

Why builds aren't reproducible by default:

### 1. Timestamps

Build time embedded in artifacts:

```c
// C/C++ with __DATE__ and __TIME__ macros
const char* build_date = __DATE__;  // "Jan 15 2024"
const char* build_time = __TIME__;  // "14:23:05"

// Different every build!
```

```java
// Java manifest
Manifest-Version: 1.0
Build-Time: 2024-01-15T14:23:05Z  ← Changes every build
```

**Impact**: Build today vs. tomorrow → different timestamp → different binary hash.

### 2. Dependency Drift

New versions fetched between builds:

```
Monday build:    requests==2.28.0  (current version)
Tuesday release: requests 2.31.0 published
Tuesday build:   requests==2.31.0  (new version!)

Result: Different dependencies → different binary
```

### 3. Environment Differences

Different OS, tools, or library versions:

```
Developer Machine:
- Ubuntu 22.04
- gcc 11.2.0
- Python 3.10.4

CI Server (6 months later):
- Ubuntu 22.04 (security updates!)
- gcc 11.4.0 (patched)
- Python 3.10.13 (bugfix)

Result: Potentially different binaries
```

### 4. Build Tool Versions

Compiler/interpreter updates change output:

```bash
# Developer A builds today
$ cargo build --release
Binary hash: abc123def456...

# Developer B builds tomorrow (same code!)
$ cargo build --release
Binary hash: xyz789ghi012...  # DIFFERENT!

Why?
- A used Rust 1.75.0, B used Rust 1.76.0
- Dependencies updated overnight
- Different timestamps embedded
- Different build machines
```

### 5. Non-Deterministic Ordering

File system iteration order varies:

```bash
# Directory listing order depends on filesystem
ls *.rs | xargs rustc  # Order not guaranteed

# Tar archives include files in filesystem order
tar -czf release.tar.gz dir/  # Non-deterministic
```

### 6. Random Number Generation

Used in build process (obfuscation, unique IDs):

```javascript
// Generated unique ID in build
const buildId = Math.random().toString(36);
```

### 7. Network Dependencies

Fetching latest versions during build:

```bash
# Dockerfile without pinning
FROM node:18        # Which 18.x.x? Changes over time
RUN apt-get update  # Latest package versions
RUN npm install     # Latest matching package.json
```

### 8. Locale/Timezone

Different regional settings affect output:

```bash
# Date formatting depends on locale
date "+%B %d, %Y"   # "January 15, 2024" vs "15 janvier 2024"
```

---

## Strategy 1: Dependency Pinning

### The Problem: Floating Dependencies

**Bad (Allows Updates):**

```toml
# Cargo.toml (Rust)
[dependencies]
serde = "1.0"      # Could be 1.0.0, 1.0.1, 1.0.152...
tokio = "1"        # Could be any 1.x version
```

```json
// package.json (Node.js)
{
  "dependencies": {
    "express": "^4.18.0",  // ^ allows 4.18.x, 4.19.x, 4.99.x
    "lodash": "~4.17.0"    // ~ allows 4.17.x only
  }
}
```

```
# requirements.txt (Python)
requests>=2.28.0   # Any version >= 2.28.0
flask              # Any version!
```

**What Happens:**

```
Build 1 (Monday):    requests==2.28.0  (current latest)
Build 2 (Tuesday):   requests==2.31.0  (new release!)

Same source code → different dependencies → different binary
```

### The Solution: Lock Files

Lock files capture **exact** versions with checksums.

**Rust (Cargo.lock):**

```toml
[[package]]
name = "serde"
version = "1.0.152"
source = "registry+https://github.com/rust-lang/crates.io-index"
checksum = "bb7d1f0d3021d347a83e2..."

dependencies = [
  "serde_derive 1.0.152",
]
```

**Node.js (package-lock.json):**

```json
{
  "packages": {
    "node_modules/express": {
      "version": "4.18.2",
      "resolved": "https://registry.npmjs.org/express/-/express-4.18.2.tgz",
      "integrity": "sha512-5/PsL6iGPdfQ/lKM1UuielYgv3BUoJfz...",
      "dependencies": {
        "accepts": "~1.3.8",
        "body-parser": "1.20.1"
      }
    }
  }
}
```

**Python (requirements.lock or poetry.lock):**

```
# requirements.lock with hashes
requests==2.28.0 \
    --hash=sha256:64299f4909223da74... \
    --hash=sha256:7c5599b102feddaa661...

certifi==2022.12.7 \
    --hash=sha256:35824b4c3a97115964b408844d64aa14db1cc518f6562e8d7261699d1350a9e3
```

**Critical Rule**: Lock files MUST be committed to version control.

**Result:**

```
Source Code (commit abc123) + Lock File → Same Dependencies Every Time

Build today:     requests==2.28.0
Build tomorrow:  requests==2.28.0
Build next year: requests==2.28.0
```

---

## Strategy 2: Build Environment Pinning

### The Problem: Environment Drift

```
Developer Machine:
- Ubuntu 22.04
- gcc 11.2.0
- Python 3.10.4
- Node 18.12.0

CI Server (6 months later):
- Ubuntu 22.04 (security updates!)
- gcc 11.4.0 (auto-updated)
- Python 3.10.13 (patch release)
- Node 18.19.0 (patch release)

Result: Same source + lock file, but different binaries (compiler changed!)
```

### Solution: Containerized Builds

**Pin Everything with Docker:**

```dockerfile
# Dockerfile.build

# Pin base image by DIGEST (not just tag!)
FROM node:18.12.0@sha256:a6c22f7c9cab5d8d6cf104c...

# Pin OS packages with exact versions
RUN apt-get update && \
    apt-get install -y \
      gcc=4:11.2.0-1ubuntu1 \
      make=4.3-4.1build1 \
      git=1:2.34.1-1ubuntu1.9 \
    && rm -rf /var/lib/apt/lists/*

# Pin build tools
RUN npm install -g \
      webpack@5.75.0 \
      typescript@4.9.4

WORKDIR /build
COPY package.json package-lock.json ./

# Use exact dependencies from lock file
RUN npm ci  # 'ci' uses lock file exactly, 'install' can modify it

COPY . .
RUN npm run build
```

### Why Digest Instead of Tag?

**Tags can change:**

```bash
# Tags are mutable pointers
node:18.12.0 today    → sha256:abc123... (original image)
node:18.12.0 tomorrow → sha256:def456... (security patch!)

# Digests are immutable
node:18.12.0@sha256:abc123... → Always the exact same image
```

When Node.js publishes a security patch for 18.12.0, they republish the tag. Now `node:18.12.0` points to a different image. Your builds change without you changing anything.

**Best Practice:**

```dockerfile
# Pin by digest
FROM node:18.12.0@sha256:a6c22f7c9cab5d8d6cf104c...

# How to get digest:
# docker pull node:18.12.0
# docker inspect node:18.12.0 | jq '.[0].RepoDigests'
```

---

## Strategy 3: Handling Time-Dependent Builds

### The Problem: Embedded Timestamps

```c
// C/C++ with __DATE__ and __TIME__ macros
const char* build_date = __DATE__;  // "Jan 15 2024"
const char* build_time = __TIME__;  // "14:23:05"

// Result: Different binaries every build
```

```java
// Java manifest
Manifest-Version: 1.0
Build-Time: 2024-01-15T14:23:05Z  ← Different every build!
Created-By: 11.0.12 (Oracle Corporation)
```

### Solution: SOURCE_DATE_EPOCH

Set a fixed timestamp based on git commit time:

```bash
# Extract commit time (Unix timestamp)
export SOURCE_DATE_EPOCH=$(git log -1 --pretty=%ct)

# Build tools respect this environment variable
gcc -o myapp main.c
go build -o myapp
mvn package
```

**How It Works:**

```
Git commit: 2024-01-15 10:30:00 UTC
↓
git log -1 --pretty=%ct → 1705318200 (Unix timestamp)
↓
export SOURCE_DATE_EPOCH=1705318200
↓
All builds from this commit use timestamp 1705318200

Build today:      timestamp = 1705318200
Build tomorrow:   timestamp = 1705318200
Build next year:  timestamp = 1705318200

Same commit → same timestamp → same binary
```

### Deterministic Build Flags

**Go:**

```bash
go build \
  -trimpath \                    # Remove absolute paths
  -ldflags="-buildid=" \         # Remove non-deterministic build ID
  -ldflags="-X main.Version=$VERSION \
            -X main.Commit=$COMMIT_SHA \
            -X main.BuildTime=$SOURCE_DATE_EPOCH"
```

**Rust:**

```bash
# Set metadata hash to commit SHA (deterministic)
RUSTFLAGS="-C metadata=$COMMIT_SHA" cargo build --release
```

**Java (Maven):**

```bash
mvn clean package \
  -Dproject.build.outputTimestamp=$SOURCE_DATE_EPOCH
```

**Python:**

```bash
# Set build timestamp for wheel metadata
export SOURCE_DATE_EPOCH=$(git log -1 --pretty=%ct)
python setup.py bdist_wheel
```

---

## Version Metadata Strategy

### The Challenge

Binary should know its version for debugging, but adding timestamps breaks reproducibility.

### The Solution: Git-Based Metadata

Version information derived from git state is deterministic:

```bash
# These values are DETERMINISTIC for a given git commit
COMMIT_SHA=$(git rev-parse HEAD)           # Always same for commit
COMMIT_TIME=$(git log -1 --pretty=%ct)     # Commit time, not build time
VERSION=$(git describe --tags)             # Derived from tags

# Same commit → same values → same binary
```

### Build Process Example

```dockerfile
FROM golang:1.21.0@sha256:... AS builder

WORKDIR /src
COPY . .

# Embed git-based version info (all deterministic)
RUN GIT_COMMIT=$(git rev-parse HEAD) && \
    GIT_TIME=$(git log -1 --pretty=%ct) && \
    VERSION=$(git describe --tags --always) && \
    go build \
      -trimpath \
      -ldflags="-X main.GitCommit=$GIT_COMMIT \
                -X main.BuildTime=$GIT_TIME \
                -X main.Version=$VERSION"
```

### Application Code

```go
package main

import "fmt"

var (
    GitCommit = "unknown"  // Set by linker at build time
    BuildTime = "unknown"  // Git commit time (deterministic!)
    Version   = "unknown"  // Git describe (deterministic!)
)

func main() {
    fmt.Printf("Version: %s\n", Version)
    fmt.Printf("Commit: %s\n", GitCommit)
    fmt.Printf("Built: %s\n", BuildTime)  // This is commit time, not wall clock time
}
```

**Key Insight**: BuildTime is the git commit timestamp, not the current time. This is deterministic - same commit always has same commit time.

```
Commit abc123 at 2024-01-15 10:30:00 UTC
↓
BuildTime = 1705318200 (commit timestamp)
↓
Build today:      BuildTime = 1705318200
Build next year:  BuildTime = 1705318200  (SAME!)
```

---

## The Reproducibility Chain

Complete chain from source to binary:

```
1. Source Code
   ├─ Git commit: abc123def456...
   └─ Tree hash: sha256:111...
   ↓

2. Dependencies
   ├─ Lock file: package-lock.json (committed)
   └─ Checksums: sha256:222...
   ↓

3. Build Environment
   ├─ Dockerfile: Dockerfile.build (committed)
   └─ Base image: node@sha256:333... (pinned by digest)
   ↓

4. Build Process
   ├─ Commands: npm ci && npm run build
   ├─ Flags: --production
   └─ Timestamp: $SOURCE_DATE_EPOCH (from git)
   ↓

5. Output Artifact
   └─ Binary: sha256:444...

Each step deterministic → Final output deterministic
```

If any step changes, the output changes. If nothing changes, the output is identical.

---

## Multi-Architecture Reproducibility

Different CPU architectures produce different binaries (this is expected and correct):

```bash
# Build on x86_64
$ docker build --platform linux/amd64 -t myapp:amd64 .
Binary: sha256:abc123...

# Build on ARM64
$ docker build --platform linux/arm64 -t myapp:arm64 .
Binary: sha256:xyz789...  # Different (different architecture!)
```

### Version Tracking Across Architectures

```
myapp:1.2.3-amd64 → sha256:abc123... (reproducible on x86_64)
myapp:1.2.3-arm64 → sha256:xyz789... (reproducible on ARM64)

Same version, different platform = different expected hash
```

### Multi-Architecture Manifest

```yaml
version: 1.2.3
source_commit: abc123def456...

platforms:
  linux/amd64:
    digest: sha256:abc123...
    build_date_epoch: 1705318200
    reproducible: true

  linux/arm64:
    digest: sha256:xyz789...
    build_date_epoch: 1705318200
    reproducible: true

  windows/amd64:
    digest: sha256:def456...
    build_date_epoch: 1705318200
    reproducible: true
```

Each platform has a different digest (different binary), but all are reproducible on their respective platforms.

---

## Software Bill of Materials (SBOM)

Every build produces a complete inventory of components.

### SBOM Example (CycloneDX format)

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.4",
  "version": 1,
  "metadata": {
    "component": {
      "name": "myapp",
      "version": "1.2.3",
      "purl": "pkg:docker/myapp@1.2.3"
    },
    "timestamp": "2024-01-15T10:30:00Z"
  },
  "components": [
    {
      "name": "express",
      "version": "4.18.2",
      "purl": "pkg:npm/express@4.18.2",
      "hashes": [
        {
          "alg": "SHA-256",
          "content": "5/PsL6iGPdfQ/lKM1UuielYgv3BUoJfz..."
        }
      ],
      "licenses": [
        {
          "license": {
            "id": "MIT"
          }
        }
      ]
    },
    {
      "name": "lodash",
      "version": "4.17.21",
      "purl": "pkg:npm/lodash@4.17.21",
      "hashes": [
        {
          "alg": "SHA-256",
          "content": "v8gO2DZybmkAa+Hwr8pBr9nDrUoEW..."
        }
      ]
    }
  ]
}
```

### SBOM Benefits

**Security Vulnerability Tracking**:
```
🚨 CVE-2023-1234 affects lodash 4.17.20

Query SBOM: Which app versions use lodash 4.17.20?
→ myapp:1.2.3 uses lodash 4.17.20 (AFFECTED)
→ myapp:1.2.4 uses lodash 4.17.21 (SAFE)
```

**License Compliance**:
```
Query: Does myapp use any GPL-licensed dependencies?
→ Scan SBOM for GPL licenses
→ Generate compliance report
```

**Supply Chain Transparency**:
- Know exactly what's in your binary
- Track dependencies of dependencies (transitive deps)
- Verify no malicious packages

**Reproducibility Validation**:
- Rebuild binary
- Generate new SBOM
- Compare to original SBOM
- If SBOMs match → reproducible build

---

## Verification Process

### Original Build (in CI)

```bash
# Build parameters
Git commit: abc123def456...
Lock file:  package-lock.json (sha256:111...)
Dockerfile: Dockerfile.build (sha256:222...)
Env vars:   SOURCE_DATE_EPOCH=1705318200

# Build
docker build -f Dockerfile.build -t myapp:1.2.3 .

# Output
Binary hash: sha256:555...
SBOM:        sbom-1.2.3.json
```

### Verification Build (months or years later)

```bash
# Checkout exact commit
git checkout abc123def456...

# Verify lock file unchanged
sha256sum package-lock.json  # Should be sha256:111...

# Build with exact same process
docker build -f Dockerfile.build -t myapp:verify .

# Compare hashes
original_hash="sha256:555..."
verify_hash=$(docker inspect myapp:verify --format='{{.Id}}')

if [ "$verify_hash" = "$original_hash" ]; then
  echo "✅ Build is reproducible!"
else
  echo "❌ Build not reproducible"
  echo "Expected: $original_hash"
  echo "Got:      $verify_hash"

  # Debug:
  # - Check base image digest
  # - Check system package versions
  # - Check build tool versions
  # - Check SOURCE_DATE_EPOCH
fi
```

---

## Hermetic Build Systems

Tools like **Bazel** and **Nix** provide guaranteed reproducibility.

### Bazel Characteristics

- **Sandboxed**: No access to host filesystem (can't read ~/.bashrc, /tmp, etc.)
- **Content-Addressed**: Same inputs always produce same outputs
- **Declared Dependencies**: Cannot access undeclared dependencies
- **Hermetic**: No network access during build (all deps pre-fetched)
- **Deterministic**: Every build detail explicitly controlled

### Bazel Example

```python
# BUILD file
go_binary(
    name = "myapp",
    srcs = ["main.go"],
    deps = [
        "@com_github_gin_gonic_gin//:gin",
    ],
)

# WORKSPACE file
go_repository(
    name = "com_github_gin_gonic_gin",
    importpath = "github.com/gin-gonic/gin",
    tag = "v1.9.0",
    sum = "h1:Hh36tbLFtVdnVnfLQhezqGr7lD62t84...",  # Content hash
)
```

**Key Features**:
- Dependencies fetched once, cached by content hash
- Build sandboxed (no network, no access to $HOME)
- Same inputs → same outputs, guaranteed
- Can distribute build across machines (build cache shared)

### Nix Characteristics

- **Purely Functional**: Builds are functions (same inputs → same output)
- **Isolated**: Each build in its own environment
- **Content-Addressed**: Store paths include hash of inputs
- **Atomic**: Builds either complete or don't exist (no partial state)

---

## Reproducibility Levels

### Level 0: Not Reproducible

- Floating dependencies (`package.json` without lock file)
- No version pinning
- Timestamps embedded (build time varies)
- Environment dependencies (uses host gcc version)
- **Cannot rebuild old versions reliably**

### Level 1: Dependency Reproducibility

✅ Lock files committed (exact dependency versions)
✅ Base images specified (but only by tag)
❌ Build timestamps still vary
❌ Build tool versions not pinned

**Outcome**: Same dependencies, but binaries might differ due to timestamps or tool versions.

### Level 2: Build Reproducibility

✅ Lock files committed
✅ Base images pinned by digest
✅ SOURCE_DATE_EPOCH set (deterministic timestamps)
✅ Build tool versions locked
❌ Some edge cases might remain (locale, file ordering)

**Outcome**: Very likely reproducible, but not guaranteed bit-for-bit identical.

### Level 3: Bit-for-Bit Reproducibility

✅ Everything from Level 2
✅ Hermetic build system (Bazel, Nix)
✅ Sandboxed builds (no host dependencies)
✅ Content-addressed cache
✅ All sources of non-determinism eliminated

**Outcome**: Multiple independent rebuilds produce IDENTICAL binaries (same SHA256 hash).

---

## Use Cases

### Security Incident Response

```
🚨 Vulnerability in lodash 4.17.20

Questions:
1. Which app versions use this dependency?
2. What's running in production?
3. Can we rebuild with patched version?

With SBOM + Reproducibility:
1. Query SBOM → "myapp:1.2.3 uses lodash 4.17.20"
2. Check production → Running myapp:1.2.3 → AFFECTED
3. Update lock file to lodash 4.17.21
4. Rebuild → Get myapp:1.2.4 with patched dependency
5. Verify reproducibility → Hash matches
6. Deploy to production
```

### Audit Trail

```
Auditor: "Prove this binary came from this source code"

Process:
1. Provide source commit SHA (abc123...)
2. Provide build environment (Dockerfile.build)
3. Provide dependencies (package-lock.json)
4. Auditor independently rebuilds from these inputs
5. Compare binary hashes
6. ✅ If hashes match → Verified provenance
   ❌ If hashes differ → Cannot verify
```

### Debugging Production Issues

```
Customer: "Bug in v1.2.3"

Without Reproducibility:
- Pull v1.2.3 from registry
- Maybe it's the right version?
- Maybe artifact registry corrupted?
- Can't be sure

With Reproducibility:
- Checkout git tag v1.2.3
- Rebuild from source
- Compare hash to production binary
- ✅ Hashes match → Confident it's identical
- Debug with exact production binary
```

### Rollback Confidence

```
Need to rollback from v1.2.4 to v1.2.3

Questions:
- Can we rebuild v1.2.3 if artifact is lost?
- Will it be exactly the same as original?
- Can we trust it for production?

With Reproducibility:
1. Checkout git tag v1.2.3
2. Rebuild using Dockerfile.build
3. Compare hash to original build
4. ✅ Hash matches → Safe to deploy
5. Deploy with confidence
```

---

## Best Practices

1. **Pin Everything**: Dependencies (lock files), base images (by digest), build tools

2. **Lock Files in Git**: Always commit lock files (package-lock.json, Cargo.lock, poetry.lock)

3. **Use SOURCE_DATE_EPOCH**: Set from git commit time for deterministic timestamps

4. **Containerize Builds**: Isolate from host environment using Docker/Podman

5. **Content-Addressed Storage**: Store artifacts by hash, never overwrite

6. **Generate SBOMs**: Document what's in each build for security and compliance

7. **Regular Verification**: Periodically rebuild old versions to verify they're still reproducible

8. **Hermetic Tools**: Consider Bazel or Nix for critical builds requiring guaranteed reproducibility

9. **Document Process**: Make build process transparent and documented (Dockerfile.build in repo)

10. **Automate Verification**: CI should verify reproducibility as part of the build

---

## Reproducibility Checklist

Before claiming reproducible builds:

- [ ] Dependencies pinned in lock file (package-lock.json, Cargo.lock, etc.)
- [ ] Lock file committed to git
- [ ] Base images pinned by digest (@sha256:...), not just tag
- [ ] Build tool versions explicitly specified
- [ ] SOURCE_DATE_EPOCH set to git commit time
- [ ] Dockerfile.build committed and versioned
- [ ] SBOM generated for each build
- [ ] Verification build tested (rebuild produces same hash)
- [ ] Process documented (README explains how to reproduce builds)
- [ ] Automated verification in CI

---

Build reproducibility transforms builds from "black boxes" to verifiable, auditable processes. It's the foundation for supply chain security, debugging confidence, and compliance requirements.

Same source + same dependencies + same environment = same binary. Every time. Everywhere.
