# Build Systems and Repository Composition

## What This Is

This document covers how to orchestrate builds and compose distributed codebases using Make and Git composition strategies (submodules, subtrees, worktrees). These are foundational skills for release engineering in systems with multiple components, microservices, or complex dependency graphs.

**The fundamental questions:**

- How do you build a system with 10 components in the right order?
- How do you manage shared libraries across 20 microservices?
- How do you enable distributed teams to work on tightly coupled code?
- How do you version dependencies without package registry overhead?

## Why This Matters for SRE/Platform Engineering

**Interview perspective**: When asked "How do you manage a multi-service system?", you need to discuss:

- **Build orchestration**: How components build in dependency order
- **Composition strategies**: How code is shared and versioned
- **Distributed workflows**: How teams collaborate across repos
- **CI/CD integration**: How pipelines handle complex structures

This isn't just "how to use Make" - it's about understanding the architecture of build systems and code composition.

---

## Part 1: Make - Build Orchestration Fundamentals

### What Make Actually Is

Make is a build automation tool based on:

1. **Targets**: Things to build (files or abstract goals)
2. **Dependencies**: What each target needs before it can be built
3. **Rules**: How to build each target
4. **Incremental builds**: Only rebuild what changed

**First Principles**: Make solves the **build graph problem**. In a system where A depends on B and C, B depends on D, you need to build in order: D → B → C → A. Make figures this out automatically.

### Basic Makefile Anatomy

```makefile
# Target: dependencies
#     commands (must be tab-indented)

# Example: Build a Go binary
bin/server: main.go pkg/handler.go pkg/db.go
	go build -o bin/server main.go

# Variables
GO := go
BUILD_DIR := bin
LDFLAGS := -ldflags "-X main.version=$(VERSION)"

# Phony targets (not files)
.PHONY: test clean install

test:
	$(GO) test ./...

clean:
	rm -rf $(BUILD_DIR)

install: bin/server
	cp bin/server /usr/local/bin/
```

**Key Concepts**:

- **Targets** can be files (`bin/server`) or abstract goals (`.PHONY: test`)
- **Dependencies** determine build order
- **Recipes** are the commands to execute (TAB-indented, not spaces!)
- **Variables** make Makefiles reusable

### Multi-Component Build Example

**Scenario**: You have a microservices system:

- `auth-service` (Go)
- `api-gateway` (Go)
- `frontend` (React)
- `shared` (protobuf definitions)

All depend on `shared` for API contracts.

**Makefile**:

```makefile
# Top-level orchestration Makefile
.PHONY: all build test clean proto

# Build everything
all: proto build

# Generate protobuf definitions (dependency for all services)
proto:
	@echo "==> Generating protobuf files..."
	cd shared && make proto
	@echo "==> Protobuf generation complete"

# Build all services (depends on proto)
build: proto
	@echo "==> Building auth-service..."
	cd auth-service && make build
	@echo "==> Building api-gateway..."
	cd api-gateway && make build
	@echo "==> Building frontend..."
	cd frontend && make build
	@echo "==> Build complete"

# Run tests for all services
test:
	cd auth-service && make test
	cd api-gateway && make test
	cd frontend && make test

# Clean all build artifacts
clean:
	cd auth-service && make clean
	cd api-gateway && make clean
	cd frontend && make clean
	cd shared && make clean
```

**In `auth-service/Makefile`**:

```makefile
SERVICE_NAME := auth-service
GO := go
BUILD_DIR := bin
PROTO_DIR := ../shared/proto

.PHONY: build test clean

build: $(BUILD_DIR)/$(SERVICE_NAME)

$(BUILD_DIR)/$(SERVICE_NAME): $(wildcard *.go) $(wildcard pkg/**/*.go)
	mkdir -p $(BUILD_DIR)
	$(GO) build -o $(BUILD_DIR)/$(SERVICE_NAME) ./cmd/server

test:
	$(GO) test -v ./...

clean:
	rm -rf $(BUILD_DIR)
```

**Why this works**:

- Top-level Makefile orchestrates dependencies (proto → services)
- Each service has its own Makefile (encapsulation)
- Running `make build` from root builds everything in order
- Running `make build` from `auth-service/` builds just that service

### Parallel Builds

Make can build independent targets in parallel:

```makefile
.PHONY: build

# Build services in parallel (auth and api-gateway don't depend on each other)
build: proto
	@echo "==> Building services in parallel..."
	make -j 4 build-auth build-api build-frontend

build-auth:
	cd auth-service && make build

build-api:
	cd api-gateway && make build

build-frontend:
	cd frontend && make build
```

**`make -j 4`**: Run up to 4 jobs in parallel

**When to use**:

- Independent services can build concurrently
- Reduces build time from 10 minutes to 3 minutes
- Critical for CI/CD (faster feedback)

### Make for CI/CD Integration

**Pattern**: Make targets as CI/CD interface

```makefile
# CI/CD targets
.PHONY: ci-build ci-test ci-lint ci-security

ci-build:
	make build

ci-test:
	make test
	make integration-test

ci-lint:
	golangci-lint run ./...
	eslint frontend/src

ci-security:
	gosec ./...
	npm audit --audit-level=high

# All CI checks
ci: ci-lint ci-security ci-test ci-build
```

**CI pipeline (GitHub Actions)**:

```yaml
name: CI
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: make ci
```

**Advantage**: CI logic lives in Makefile, not in CI config. Can run `make ci` locally to verify before pushing.

### Docker Integration

**Pattern**: Use Make to orchestrate Docker builds

```makefile
DOCKER_REGISTRY := ghcr.io/myorg
VERSION := $(shell git describe --tags --always)

.PHONY: docker-build docker-push

docker-build:
	docker build -t $(DOCKER_REGISTRY)/auth-service:$(VERSION) ./auth-service
	docker build -t $(DOCKER_REGISTRY)/api-gateway:$(VERSION) ./api-gateway
	docker build -t $(DOCKER_REGISTRY)/frontend:$(VERSION) ./frontend

docker-push: docker-build
	docker push $(DOCKER_REGISTRY)/auth-service:$(VERSION)
	docker push $(DOCKER_REGISTRY)/api-gateway:$(VERSION)
	docker push $(DOCKER_REGISTRY)/frontend:$(VERSION)

# Tag as latest
docker-tag-latest:
	docker tag $(DOCKER_REGISTRY)/auth-service:$(VERSION) $(DOCKER_REGISTRY)/auth-service:latest
	docker tag $(DOCKER_REGISTRY)/api-gateway:$(VERSION) $(DOCKER_REGISTRY)/api-gateway:latest
	docker tag $(DOCKER_REGISTRY)/frontend:$(VERSION) $(DOCKER_REGISTRY)/frontend:latest
```

**Usage**:

```bash
make docker-build          # Build all images
make docker-push           # Push with version tag
make docker-tag-latest     # Tag and push as latest
```

### Advanced: Makefile Best Practices

#### 1. Self-Documenting Makefiles

```makefile
.PHONY: help
help: ## Show this help message
	@echo "Available targets:"
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-20s\033[0m %s\n", $$1, $$2}'

.DEFAULT_GOAL := help

build: ## Build all services
	@echo "Building..."

test: ## Run tests
	@echo "Testing..."
```

**Output**:

```
$ make
Available targets:
  build                Build all services
  test                 Run tests
```

#### 2. Error Handling

```makefile
.PHONY: build

build:
	@echo "Building auth-service..."
	cd auth-service && make build || exit 1
	@echo "Building api-gateway..."
	cd api-gateway && make build || exit 1
	@echo "All builds successful"
```

**`|| exit 1`**: Stop the build if any component fails

#### 3. Conditional Logic

```makefile
# Detect OS
UNAME := $(shell uname -s)

ifeq ($(UNAME),Linux)
    PLATFORM := linux
endif
ifeq ($(UNAME),Darwin)
    PLATFORM := darwin
endif

# Use platform-specific commands
build:
	@echo "Building for $(PLATFORM)..."
	GOOS=$(PLATFORM) go build -o bin/server
```

#### 4. Include External Makefiles

```makefile
# Include common variables
include common.mk

# Include service-specific rules
-include auth-service.mk
-include api-gateway.mk
```

**`-include`**: Don't error if file doesn't exist

### Make vs Modern Build Tools

**When to use Make**:

- Multi-language projects (Go + Node + Python)
- Docker orchestration
- CI/CD interface (unified commands across projects)
- Simple dependency graphs
- UNIX-centric environments

**When NOT to use Make**:

- Complex dependency resolution (use Bazel, Buck)
- Cross-platform Windows + UNIX (Make syntax differs)
- Dynamic dependency graphs (use Gradle, SBT)
- Pure Go projects (consider mage - Makefiles in Go)

**Interview answer**: "We use Make as our build orchestration layer. It's simple, ubiquitous, and works with any language. We have 12 microservices (Go, Node, Python, Rust), and Make gives us a unified interface: `make build`, `make test`, `make docker-push`. Our CI/CD just calls Make targets. The alternative would be coupling our build logic to CI config, which makes local development harder."

---

## Part 2: Git Submodules - Distributed Dependency Management

### What Submodules Actually Solve

**The problem**: You have:

- **Main repo**: `platform` (your application)
- **Dependency repos**: `auth-lib`, `metrics-lib`, `config-lib`

You need:

- Specific versions of each dependency (not just latest)
- Ability to develop dependencies alongside main code
- Distributed team ownership (different teams own different libs)

**Submodules**: Reference external repos at specific commits.

### Deep Dive: How Submodules Work

When you add a submodule:

```bash
cd platform
git submodule add https://github.com/myorg/auth-lib.git libs/auth
```

**What happens**:

1. **`.gitmodules` file created**:

```ini
[submodule "libs/auth"]
    path = libs/auth
    url = https://github.com/myorg/auth-lib.git
```

2. **Submodule directory created**: `libs/auth/` (contains the auth-lib code)

3. **Special commit reference stored**:

```bash
$ git ls-tree HEAD libs/auth
160000 commit abc123... libs/auth
```

**160000 mode**: Special Git mode for submodule (not a regular directory)

**`abc123...`**: Specific commit SHA from `auth-lib` repo

**Key insight**: Your repo doesn't store the submodule code, only a pointer to a commit.

### Creating and Managing Submodules

#### Initial Setup

```bash
# In main repo
cd platform

# Add submodules
git submodule add https://github.com/myorg/auth-lib.git libs/auth
git submodule add https://github.com/myorg/metrics-lib.git libs/metrics
git submodule add https://github.com/myorg/config-lib.git libs/config

# Commit the submodule references
git commit -m "Add library submodules"
git push
```

#### Cloning a Repo with Submodules

**New developer joins, clones the repo**:

```bash
git clone https://github.com/myorg/platform.git
cd platform

# Submodule directories exist but are empty!
$ ls libs/auth
# (empty directory)

# Initialize and populate submodules
git submodule init
git submodule update

# Or clone with --recursive (does init + update automatically)
git clone --recursive https://github.com/myorg/platform.git
```

**What `submodule update` does**:

1. Looks at `.gitmodules` for URLs
2. Clones each submodule repo
3. Checks out the commit SHA recorded in the parent repo

#### Updating a Submodule to a Newer Version

**Scenario**: `auth-lib` has a new commit you want to use.

```bash
cd libs/auth
git pull origin main           # Update to latest
# Or checkout specific version:
# git checkout v1.2.3

cd ../..                        # Back to platform root

# Check status
git status
# modified:   libs/auth (new commits)

# Commit the new submodule reference
git add libs/auth
git commit -m "Update auth-lib to v1.2.3"
git push
```

**What teammates see**:

```bash
git pull                        # Get your commit
git submodule update            # Update their submodules to match
```

#### Developing Inside a Submodule

**Scenario**: You need to fix a bug in `auth-lib` while working on `platform`.

```bash
cd libs/auth

# Check current state
git status
# HEAD detached at abc123

# Create a branch (submodules default to detached HEAD)
git checkout -b fix-auth-bug

# Make changes
vim auth.go
git commit -am "fix: handle nil user gracefully"

# Push to auth-lib repo
git push origin fix-auth-bug

# Open PR in auth-lib repo
# After merge, update platform to use the new commit
cd ../..
cd libs/auth
git pull origin main
cd ../..
git add libs/auth
git commit -m "Update auth-lib with bug fix"
```

**Important**: Submodules start in detached HEAD state. Always create a branch before committing.

### Distributed Team Workflows with Submodules

#### Pattern 1: Library Teams Own Submodules

**Structure**:

- **Team A**: Owns `auth-lib` repo
- **Team B**: Owns `metrics-lib` repo
- **Team C**: Owns `platform` repo (consumes both libs)

**Workflow**:

1. Team A develops in `auth-lib` repo, tags releases (`v1.0.0`, `v1.1.0`)
2. Team C updates `platform` to new `auth-lib` version when ready
3. Team C controls exactly which version they use (no surprise updates)

**Advantages**:

- **Isolation**: Library changes don't break consumers until explicitly updated
- **Versioning**: Platform can pin to stable releases
- **Ownership**: Clear boundaries between teams

**Example**:

```bash
# Team C updates to new auth-lib release
cd platform/libs/auth
git fetch origin
git checkout v1.1.0            # Specific release tag
cd ../..
git add libs/auth
git commit -m "Upgrade auth-lib to v1.1.0"
```

#### Pattern 2: Monorepo Alternative (Submodules as Composition)

**Structure**: Multiple microservices, each in their own repo, composed into a deployment repo.

```
deployment-repo/
├── services/
│   ├── auth-service/         (submodule)
│   ├── api-gateway/          (submodule)
│   ├── payment-service/      (submodule)
├── docker-compose.yml
├── Makefile
└── .gitmodules
```

**`deployment-repo/.gitmodules`**:

```ini
[submodule "services/auth-service"]
    path = services/auth-service
    url = https://github.com/myorg/auth-service.git
[submodule "services/api-gateway"]
    path = services/api-gateway
    url = https://github.com/myorg/api-gateway.git
[submodule "services/payment-service"]
    path = services/payment-service
    url = https://github.com/myorg/payment-service.git
```

**`deployment-repo/Makefile`**:

```makefile
.PHONY: init update build deploy

init:
	git submodule init
	git submodule update

update:
	git submodule update --remote --merge

build:
	cd services/auth-service && make docker-build
	cd services/api-gateway && make docker-build
	cd services/payment-service && make docker-build

deploy:
	docker-compose up -d
```

**Workflow**:

```bash
# Developer clones deployment repo
git clone --recursive https://github.com/myorg/deployment-repo.git
cd deployment-repo

# Build everything
make build

# Deploy locally
make deploy

# Update all services to latest
make update
git commit -am "Update all services to latest"
```

**Advantage**: Each service is independently versioned in its own repo, but you have a unified deployment/testing environment.

#### Pattern 3: Sparse Submodule Checkout

**Problem**: You have 50 microservices, but you only work on 3.

**Solution**: Partial clone + sparse checkout

```bash
# Clone with no submodules
git clone https://github.com/myorg/deployment-repo.git
cd deployment-repo

# Only initialize specific submodules
git submodule init services/auth-service
git submodule init services/api-gateway
git submodule update

# Now only auth-service and api-gateway are cloned
```

**CI/CD variant**: Clone all submodules shallowly (faster)

```bash
git clone --recurse-submodules --shallow-submodules --depth=1 \
    https://github.com/myorg/deployment-repo.git
```

**Result**: Only fetch the most recent commit for each submodule (much faster for CI).

### Submodule CI/CD Integration

#### GitHub Actions Example

```yaml
name: Build with Submodules
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout with submodules
        uses: actions/checkout@v3
        with:
          submodules: recursive
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Build
        run: make build

      - name: Test
        run: make test
```

**Key**: `submodules: recursive` ensures all submodules are cloned.

#### GitLab CI Example

```yaml
variables:
  GIT_SUBMODULE_STRATEGY: recursive

build:
  script:
    - make build
    - make test
```

**`GIT_SUBMODULE_STRATEGY: recursive`**: Automatically clone submodules.

### Advanced: Nested Submodules

**Scenario**: Your submodule has its own submodules.

```
platform/
└── libs/
    └── auth/              (submodule)
        └── vendor/
            └── crypto/    (submodule within submodule)
```

**Clone with nested submodules**:

```bash
git clone --recurse-submodules https://github.com/myorg/platform.git
```

**Update nested submodules**:

```bash
git submodule update --init --recursive
```

**Warning**: Nested submodules get complex quickly. Consider flattening or using a package manager instead.

### Submodule Anti-Patterns and Gotchas

#### Anti-Pattern 1: Treating Submodules Like Regular Directories

**Wrong**:

```bash
cd libs/auth
vim file.go
git commit -am "fix bug"
git push
```

**Problem**: You pushed to `auth-lib` repo but didn't update the parent repo's reference. Your change is invisible to others.

**Right**:

```bash
cd libs/auth
git checkout -b fix-bug       # Create branch (avoid detached HEAD)
vim file.go
git commit -am "fix bug"
git push origin fix-bug
# Open PR, get it merged
git checkout main
git pull origin main
cd ../..
git add libs/auth
git commit -m "Update auth-lib with bug fix"
git push
```

#### Anti-Pattern 2: Not Documenting Submodule Workflow

**Problem**: New developers clone, see empty directories, get confused.

**Solution**: README with explicit instructions:

````markdown
# Getting Started

## Clone with Submodules

```bash
git clone --recursive https://github.com/myorg/platform.git
```
````

## If You Already Cloned Without --recursive

```bash
git submodule init
git submodule update
```

## Updating Submodules

```bash
git submodule update --remote --merge
```

#### Anti-Pattern 3: Submodule Dependency Hell

**Problem**: Service A depends on `lib v1.0`, Service B depends on `lib v2.0`, both are submodules in same deployment repo.

**You can't have two versions of the same submodule in one repo**.

**Solution**: Use package managers (Go modules, npm) for versioned dependencies, use submodules only for services/components.

### Submodules vs Alternatives

| Feature               | Submodules               | Subtrees              | Package Managers        | Monorepo                |
| --------------------- | ------------------------ | --------------------- | ----------------------- | ----------------------- |
| **Version locking**   | Exact commit             | Squashed history      | Semver ranges           | N/A (all code together) |
| **Update complexity** | Medium (init/update)     | Low (pull)            | Low (package update)    | N/A                     |
| **Repo size**         | Small (references only)  | Large (includes code) | Small (deps separate)   | Large (all code)        |
| **Independence**      | Yes (separate repos)     | No (merged into repo) | Yes (separate packages) | No (single repo)        |
| **Best for**          | Multi-repo microservices | Vendoring             | Language-specific deps  | Single-team codebases   |

---

## Part 3: Git Subtrees and Overlay Patterns

### Git Subtree: Alternative to Submodules

**Concept**: Instead of referencing another repo, copy it into your repo as a subdirectory.

#### When to Use Subtrees

**Use subtrees when**:

- You want simpler clone workflow (just `git clone`, no extra steps)
- You're vendoring a dependency you rarely update
- Contributors shouldn't need to know about external deps
- You need a self-contained repo

**Example: Vendoring a library**

```bash
# Add a subtree
git subtree add --prefix=vendor/lib https://github.com/external/lib.git main --squash

# This creates a commit in YOUR repo with all the lib code in vendor/lib/
```

**Later, update the subtree**:

```bash
git subtree pull --prefix=vendor/lib https://github.com/external/lib.git main --squash
```

**Advantage over submodules**: When someone clones your repo, `vendor/lib` is already there. No `submodule init` needed.

**Disadvantage**: Your repo contains all the lib code. Larger repo, harder to contribute changes back.

### Overlay Filesystems for Development

**Concept**: Layer multiple filesystems on top of each other. Changes go to top layer, reads fall through to lower layers.

**Use case**: Developing against production-like data without modifying it.

#### OverlayFS (Linux)

```bash
# Setup overlay
mkdir -p /tmp/overlay/upper /tmp/overlay/work /tmp/overlay/merged

sudo mount -t overlay overlay \
    -o lowerdir=/production/data,upperdir=/tmp/overlay/upper,workdir=/tmp/overlay/work \
    /tmp/overlay/merged

# Work in /tmp/overlay/merged
# Reads come from /production/data (unchanged)
# Writes go to /tmp/overlay/upper (temporary)

# When done
sudo umount /tmp/overlay/merged
rm -rf /tmp/overlay
```

**Application**: Test data migrations locally without modifying production backup.

### Git Worktree: Multiple Checkouts from Same Repo

**Problem**: You're working on a feature branch, need to quickly switch to main for a hotfix.

**Traditional approach**: Stash changes, checkout main, fix, commit, checkout feature, pop stash.

**Worktree approach**: Have multiple working directories simultaneously.

#### Creating Worktrees

```bash
# Main working directory (on feature branch)
cd /path/to/repo

# Add a worktree for hotfix
git worktree add ../repo-hotfix main

# Now you have:
# /path/to/repo          (feature branch)
# /path/to/repo-hotfix   (main branch)

# Work in hotfix directory
cd ../repo-hotfix
vim fix.go
git commit -am "fix: critical bug"
git push origin main

# Back to feature work
cd /path/to/repo
# Your changes are still here, uncommitted
```

**List worktrees**:

```bash
git worktree list
# /path/to/repo           abc123 [feature-branch]
# /path/to/repo-hotfix    def456 [main]
```

**Remove worktree when done**:

```bash
git worktree remove ../repo-hotfix
```

#### Worktree CI/CD Pattern

**Use case**: Run tests on multiple branches in parallel.

```bash
# CI script
git worktree add ../test-main main
git worktree add ../test-develop develop
git worktree add ../test-feature feature-branch

# Run tests in parallel
(cd ../test-main && make test) &
(cd ../test-develop && make test) &
(cd ../test-feature && make test) &
wait

# Cleanup
git worktree remove ../test-main
git worktree remove ../test-develop
git worktree remove ../test-feature
```

**Advantage**: No need to clone the repo 3 times. Worktrees share the same `.git` directory (efficient).

---

## Part 4: Real-World Composition Patterns

### Pattern 1: Microservices with Shared Libraries

**Structure**:

```
microservices/
├── services/
│   ├── auth-service/      (separate repo, submodule)
│   ├── api-gateway/       (separate repo, submodule)
│   └── payment-service/   (separate repo, submodule)
├── libs/
│   ├── logger/            (separate repo, submodule)
│   ├── metrics/           (separate repo, submodule)
│   └── config/            (separate repo, submodule)
├── Makefile
├── docker-compose.yml
└── .gitmodules
```

**`.gitmodules`**:

```ini
[submodule "services/auth-service"]
    path = services/auth-service
    url = git@github.com:myorg/auth-service.git
[submodule "libs/logger"]
    path = libs/logger
    url = git@github.com:myorg/logger.git
```

**Each service's `go.mod` references libs as local**:

```go
module github.com/myorg/auth-service

require (
    github.com/myorg/logger v0.0.0
)

replace github.com/myorg/logger => ../../libs/logger
```

**Makefile**:

```makefile
init:
	git submodule update --init --recursive

build:
	cd services/auth-service && go build
	cd services/api-gateway && go build

test:
	cd services/auth-service && go test ./...
	cd services/api-gateway && go test ./...
```

**Workflow**:

1. Develop services and libs together (local replace)
2. When lib stabilizes, tag it and push
3. Update service's `go.mod` to use tagged version
4. Remove replace directive for production builds

### Pattern 2: Build Orchestration with Make + Submodules

**Makefile** (root):

```makefile
.PHONY: init update build test docker-build

init:
	git submodule update --init --recursive

# Update all submodules to latest
update:
	git submodule foreach git pull origin main

# Build in dependency order
build:
	# Build shared libs first
	cd libs/logger && make build
	cd libs/metrics && make build
	# Then services
	cd services/auth-service && make build
	cd services/api-gateway && make build

test:
	cd libs/logger && make test
	cd libs/metrics && make test
	cd services/auth-service && make test
	cd services/api-gateway && make test

docker-build:
	cd services/auth-service && make docker-build
	cd services/api-gateway && make docker-build
	docker-compose build
```

### Pattern 3: Git Worktree for Multi-Version Support

**Scenario**: Support multiple product versions simultaneously.

```bash
# Main repo (latest development)
cd /path/to/product

# Add worktrees for supported versions
git worktree add ../product-v1 release/v1.x
git worktree add ../product-v2 release/v2.x
git worktree add ../product-v3 release/v3.x

# Backport fix to all versions
vim common/fix.c
git commit -am "fix: security vulnerability"
git push origin main

# Cherry-pick to v1, v2, v3
cd ../product-v1
git cherry-pick abc123
git push origin release/v1.x

cd ../product-v2
git cherry-pick abc123
git push origin release/v2.x

cd ../product-v3
git cherry-pick abc123
git push origin release/v3.x
```

---

## Interview-Ready Mental Models

### Make as Build Graph Solver

**Question**: "Why use Make when we have language-specific build tools?"

**Answer**: "Make solves the build graph problem across languages. We have Go services, Node frontends, Python ML models, and Rust CLI tools. Make gives us a single interface - `make build` - that knows the dependency order. For example, our protobuf definitions (make proto) must build before any service. Make handles that automatically. Language-specific tools (go build, npm build) are great within a language, but Make orchestrates across the entire system."

### Submodules as Versioned Composition

**Question**: "How do you manage shared code across microservices?"

**Answer**: "We use Git submodules for our shared libraries. Each service pins to a specific commit of each library. When we update a library, services opt-in to the new version explicitly. This prevents surprise breakage - if service A updates to lib v2.0 and breaks, it doesn't affect service B still on lib v1.0. The trade-off is more manual coordination, but we prioritize stability. For rapid iteration code, we use package managers with semver ranges. Submodules are for cross-cutting concerns where we want tight version control."

### Worktrees for Parallel Contexts

**Question**: "How do you handle hotfixes while working on features?"

**Answer**: "We use Git worktrees to maintain multiple working directories from the same repo. When a production issue comes in, I run `git worktree add ../repo-hotfix main`, fix the bug there, push, and switch back to my feature work without stashing. The worktrees share the same .git directory, so it's space-efficient. We also use this in CI to test multiple branches in parallel without multiple clones."

---

## Common Gotchas and Solutions

### Gotcha 1: Makefile Tab vs Spaces

**Problem**: Recipes must be indented with TABS, not spaces.

**Error**:

```
Makefile:3: *** missing separator. Stop.
```

**Solution**: Configure editor to use tabs for Makefiles.

### Gotcha 2: Submodule Detached HEAD

**Problem**: Submodules checkout to specific commit (detached HEAD).

**Symptom**:

```bash
cd libs/auth
git status
# HEAD detached at abc123
```

**Solution**: Always create a branch before committing in submodules.

```bash
git checkout -b my-feature
```

### Gotcha 3: Forgetting to Commit Submodule Reference Update

**Problem**: You update code in submodule but don't commit the new reference in parent repo.

**Result**: Others still see old version.

**Solution**: After updating submodule:

```bash
git add libs/auth
git commit -m "Update auth lib to v1.2"
```

### Gotcha 4: Make Not Rebuilding When It Should

**Problem**: Make thinks target is up-to-date when it's not.

**Solution**: Use `.PHONY` for targets that aren't files:

```makefile
.PHONY: test build clean
```

---

## Checklist: Building a Multi-Repo System

- [ ] **Makefile at root** with `init`, `build`, `test`, `clean` targets
- [ ] **Submodules documented** in README (how to clone, update)
- [ ] **CI/CD configured** to clone with `--recursive`
- [ ] **Each submodule has own Makefile** (encapsulation)
- [ ] **Phony targets declared** (`.PHONY:`)
- [ ] **Parallel builds enabled** where possible (`make -j`)
- [ ] **Help target** for discoverability (`make help`)
- [ ] **Error handling** in Makefile (fail fast on errors)
- [ ] **Submodule branches** (not detached HEAD) for development
- [ ] **Version tags** on submodules for releases

---

Build systems and repository composition are foundational to release engineering at scale. Make provides language-agnostic orchestration, submodules enable distributed team workflows with version control, and worktrees allow parallel contexts without overhead. Master these tools, and you can architect systems that scale across dozens of teams and hundreds of components.
