# Configuration Management: Templates, Rendering, and Source of Truth

## The Problem

You have a Kubernetes application. It needs to run in multiple environments: dev, staging, production. Each environment has different configuration:

```yaml
# Dev
replicas: 2
resources:
  memory: "256Mi"
  cpu: "100m"
image: myapp:dev-latest
ingress:
  host: dev.myapp.com

# Prod
replicas: 10
resources:
  memory: "2Gi"
  cpu: "1000m"
image: myapp:1.2.3
ingress:
  host: myapp.com
```

**The naive approach**: Maintain separate YAML files for each environment.

**The problem with naive approach**:
- Change container port: Update 3 files (dev, staging, prod)
- Add new container: Update 3 files
- Fix a typo in a label: Update 3 files
- Drift: Files diverge unintentionally

You need **parameterization** - a way to express "this is shared" and "this varies by environment."

---

## The Fundamental Question

**What is your source of truth?**

1. **Templates** (Helm charts, Kustomize bases)
2. **Rendered manifests** (Concrete YAML after substitution)
3. **Both** (Hybrid: templates + committed rendered output)

This isn't just a technical decision. It affects:
- **Auditability**: Can you see exactly what was deployed?
- **Review process**: Can reviewers understand the change?
- **GitOps workflow**: What does Git contain?
- **Debugging**: Can you reproduce exact deployed state?

---

## Approach 1: Helm (Templating Engine)

### How It Works

Helm uses Go templates to parameterize Kubernetes manifests.

**Chart structure**:
```
mychart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default values
├── values-dev.yaml     # Dev overrides
├── values-prod.yaml    # Prod overrides
└── templates/
    ├── deployment.yaml # Templated
    ├── service.yaml
    └── ingress.yaml
```

**Template example** (`templates/deployment.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app: {{ .Values.app.name }}
    version: {{ .Values.app.version }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        env:
        {{- range .Values.env }}
        - name: {{ .name }}
          value: {{ .value | quote }}
        {{- end }}
```

**Values file** (`values.yaml`):

```yaml
replicaCount: 3

image:
  repository: myapp
  tag: "1.2.3"

app:
  name: myapp
  version: "v1"

resources:
  limits:
    memory: "1Gi"
    cpu: "500m"
  requests:
    memory: "512Mi"
    cpu: "250m"

env:
  - name: LOG_LEVEL
    value: info
  - name: DATABASE_URL
    value: postgres://db:5432/myapp
```

**Environment-specific overrides** (`values-prod.yaml`):

```yaml
replicaCount: 10

image:
  tag: "1.2.3"  # Pinned version

resources:
  limits:
    memory: "2Gi"
    cpu: "1000m"

env:
  - name: LOG_LEVEL
    value: warn  # Less verbose in prod
  - name: DATABASE_URL
    value: postgres://prod-db:5432/myapp
```

**Rendering and deploying**:

```bash
# Render for dev
helm template myapp ./mychart -f values-dev.yaml

# Render for prod
helm template myapp ./mychart -f values-prod.yaml

# Install to cluster
helm install myapp ./mychart -f values-prod.yaml
```

### Pros

**1. DRY (Don't Repeat Yourself)**
- Single source of truth for shared configuration
- Change container port in one place (`templates/deployment.yaml`)
- Applies to all environments

**2. Powerful Templating**
- Conditionals: `{{ if .Values.enableFeature }}`
- Loops: `{{ range .Values.items }}`
- Functions: `{{ .Values.name | upper | quote }}`
- Includes: `{{ include "mychart.labels" . }}`

**3. Packaging**
- Charts can be versioned and distributed
- Helm repositories (Artifact Hub, private registries)
- Dependencies: Chart A depends on Chart B (e.g., app depends on PostgreSQL chart)

**4. Ecosystem**
- Huge library of pre-built charts
- Well-documented, widely adopted
- Integrated with many CD tools (ArgoCD, Flux)

### Cons

**1. Template Complexity**
- Go template syntax is not YAML (harder to validate)
- Logic in templates can become difficult to understand
- Whitespace control (`{{-`, `nindent`) is finicky

**Example of complexity**:
```yaml
{{- if and .Values.ingress.enabled (eq .Values.environment "production") }}
  {{- range .Values.ingress.hosts }}
  - host: {{ .host | quote }}
    http:
      paths:
      {{- range .paths }}
      - path: {{ .path }}
        pathType: {{ .pathType | default "Prefix" }}
        backend:
          service:
            name: {{ $.Release.Name }}-{{ $.Chart.Name }}
            port:
              number: {{ $.Values.service.port }}
      {{- end }}
  {{- end }}
{{- end }}
```

**2. Values.yaml Can Grow Large**
- Deeply nested structure
- Hard to know which values are actually used
- No schema validation (until Helm 3.6+ with JSONSchema)

**3. Rendered Output is Opaque**
- What actually gets deployed?
- Must run `helm template` to see final YAML
- Reviewers see template changes, not rendered diff

**4. Chart Versioning Complexity**
- Chart version vs. app version
- Breaking changes in chart require major version bump
- Coordinating chart updates across environments

### When to Use Helm

✅ **Complex applications** with many configurable options
✅ **Need packaging** (distribute charts to customers)
✅ **Leverage existing charts** (PostgreSQL, Redis, nginx)
✅ **DRY is priority** (minimize duplication)

❌ **Simple applications** (Kustomize is lighter)
❌ **Template logic is too complex** (consider separate chart per environment)

---

## Approach 2: Kustomize (Overlay Pattern)

### How It Works

Kustomize uses **patches** and **overlays** instead of templates. You define a **base** (shared configuration) and **overlays** (environment-specific patches).

**Directory structure**:
```
kustomize/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── prod/
        ├── kustomization.yaml
        └── resource-patch.yaml
```

**Base deployment** (`base/deployment.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1  # Default, will be patched
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:latest  # Will be overridden
        ports:
        - containerPort: 8080
        resources:
          limits:
            memory: "512Mi"
            cpu: "250m"
```

**Base kustomization** (`base/kustomization.yaml`):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml

commonLabels:
  app: myapp
  managed-by: kustomize
```

**Production overlay** (`overlays/prod/kustomization.yaml`):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

nameSuffix: -prod

replicas:
  - name: myapp
    count: 10

images:
  - name: myapp
    newTag: "1.2.3"  # Prod uses pinned version

patches:
  - path: resource-patch.yaml
```

**Production resource patch** (`overlays/prod/resource-patch.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        resources:
          limits:
            memory: "2Gi"
            cpu: "1000m"
          requests:
            memory: "1Gi"
            cpu: "500m"
```

**Rendering**:

```bash
# Render dev overlay
kubectl kustomize overlays/dev

# Render prod overlay
kubectl kustomize overlays/prod

# Apply directly
kubectl apply -k overlays/prod
```

### Pros

**1. Template-Free**
- No custom syntax, just YAML
- Easier to validate (yamllint, kubeval)
- What you see is what you get

**2. Built Into kubectl**
- No separate tool to install (since kubectl 1.14)
- Native Kubernetes integration
- Widely supported

**3. Composable**
- Multiple bases can be combined
- Patches are reusable
- Clear inheritance hierarchy

**4. Strategic Merge Patch**
- Merges patches intelligently
- Understands Kubernetes resource structure
- Can add, modify, or delete fields

### Cons

**1. Patch Syntax Can Be Verbose**

To change a single field deep in the structure:

```yaml
# Just to change container memory limit
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        resources:
          limits:
            memory: "2Gi"
```

Must specify the full path. Can use JSON patches for more precision but less readable.

**2. Less Flexible Than Templating**
- Can't do conditionals (if/else)
- Can't loop over arrays dynamically
- Can't compute values (e.g., `replicas: {{ .Values.replicas * 2 }}`)

**3. No Packaging**
- No built-in chart distribution
- Harder to version and distribute
- Must use Git for sharing

**4. Merge Conflicts**
- Strategic merge can be unpredictable
- Sometimes need JSON 6902 patches (complex)
- Learning curve for patch strategies

### When to Use Kustomize

✅ **Simpler applications** (less configuration variance)
✅ **Prefer YAML over templating** (validation, readability)
✅ **GitOps workflows** (overlays map well to Git directories)
✅ **Built-in kubectl** (no additional tooling)

❌ **Need complex logic** (Helm is more powerful)
❌ **Distributing charts** (Helm packaging is superior)

---

## Approach 3: Other Tools

### ytt (YAML Templating Tool)

**Philosophy**: Templates are data, not code.

```yaml
#@ load("@ytt:data", "data")

apiVersion: apps/v1
kind: Deployment
metadata:
  name: #@ data.values.name
spec:
  replicas: #@ data.values.replicas
  template:
    spec:
      containers:
      - name: #@ data.values.name
        image: #@ "{}/{}:{}".format(data.values.image.registry, data.values.image.name, data.values.image.tag)
```

**Pros**: Python-like syntax, overlays, validation
**Cons**: Smaller ecosystem, another tool to learn

### cue (Configure Unify Execute)

**Philosophy**: Configuration is code, with types and validation.

```cue
deployment: {
    apiVersion: "apps/v1"
    kind: "Deployment"
    metadata: name: string
    spec: {
        replicas: int & >0 & <100  // Type constraint
        template: {
            spec: containers: [...{
                name: string
                image: string
            }]
        }
    }
}
```

**Pros**: Strong typing, validation, reduces errors
**Cons**: Steep learning curve, less adoption

### jsonnet

**Philosophy**: JSON with functions, imports, and conditionals.

```jsonnet
local replicas = std.extVar('replicas');

{
  apiVersion: 'apps/v1',
  kind: 'Deployment',
  metadata: { name: 'myapp' },
  spec: {
    replicas: std.parseInt(replicas),
    template: {
      spec: {
        containers: [{
          name: 'myapp',
          image: 'myapp:1.2.3',
        }],
      },
    },
  },
}
```

**Pros**: Powerful, JSON-compatible
**Cons**: Not YAML-native, niche tool

---

## The Rendered Manifests Debate

**The question**: What do you commit to Git?

### Option 1: Templates Only

**What's in Git**:
```
manifests/
├── Chart.yaml
├── values.yaml
├── values-prod.yaml
└── templates/
    └── deployment.yaml
```

**Rendering happens**:
- In CI/CD pipeline
- By GitOps controller (ArgoCD, Flux)
- Locally (developer runs `helm template`)

**Pros**:
- DRY: Single source of truth
- Small Git repo
- Easy to change shared configuration

**Cons**:
- **Opaque deployed state**: What's actually running?
- **Harder to review**: Reviewer sees template change, not rendered diff
- **Debugging**: Must re-render to see exact deployed YAML
- **Audit trail**: Git doesn't show what was deployed

### Option 2: Rendered Manifests Only

**What's in Git**:
```
manifests/
├── dev/
│   ├── deployment.yaml   # Fully rendered
│   └── service.yaml
├── staging/
│   ├── deployment.yaml
│   └── service.yaml
└── prod/
    ├── deployment.yaml
    └── service.yaml
```

**Rendering happens**:
- Before commit (CI pre-processes templates, commits YAML)
- Locally (developer renders, commits result)

**Pros**:
- **Explicit**: Git shows exact deployed state
- **Reviewable**: Diffs show actual changes to deployed resources
- **Auditable**: Git history = deployment history
- **Debugging**: Can see exact YAML that was deployed

**Cons**:
- **Verbose**: Lots of duplicated YAML
- **Merge conflicts**: Changing shared config touches many files
- **Drift risk**: If templates and rendered disagree, which is truth?

### Option 3: Hybrid (Both Templates and Rendered)

**What's in Git**:
```
manifests/
├── helm/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       └── deployment.yaml
└── rendered/
    ├── dev/
    │   └── deployment.yaml   # Auto-generated from helm/
    ├── staging/
    │   └── deployment.yaml
    └── prod/
        └── deployment.yaml
```

**CI/CD process**:
1. Developer edits `helm/` templates or values
2. CI renders `helm/` → `rendered/`
3. CI commits `rendered/` alongside `helm/`
4. Pull request shows diffs in both `helm/` and `rendered/`
5. GitOps controller applies `rendered/` (not `helm/`)

**Pros**:
- **Best of both worlds**: DRY templates + explicit rendered state
- **Reviewable**: PR shows both template change and rendered impact
- **Auditable**: Git shows exact deployed YAML
- **Verifiable**: Can diff templates vs. rendered to detect drift

**Cons**:
- **Complexity**: CI must render and commit
- **Git churn**: Every template change generates rendered commits
- **Discipline required**: Must ensure rendered is always in sync

### Comparison Matrix

| Aspect | Templates Only | Rendered Only | Hybrid |
|--------|----------------|---------------|--------|
| **Auditability** | Low (must re-render) | High (exact state in Git) | High |
| **Review process** | Hard (template diff) | Easy (YAML diff) | Easy (both) |
| **Git size** | Small | Large | Large |
| **DRY** | Yes | No | Yes |
| **Drift detection** | Hard | Easy | Easy |
| **CI complexity** | Low | Medium (must render before commit) | High (render + commit) |

---

## Environment Promotion Strategies

How do you move from dev → staging → prod?

### Strategy 1: Git Branches

```
branches/
├── dev         # Dev environment config
├── staging     # Staging environment config
└── prod        # Prod environment config
```

**Process**:
1. Merge `dev` → `staging`
2. Test in staging
3. Merge `staging` → `prod`

**Pros**: GitOps-native, visual in Git UI
**Cons**: Merge conflicts, branch divergence

### Strategy 2: Git Directories

```
environments/
├── dev/
├── staging/
└── prod/
```

**Process**:
1. Update `dev/`
2. Test in dev
3. Copy `dev/` → `staging/` (or update staging Kustomize overlay)
4. Test in staging
5. Copy `staging/` → `prod/`

**Pros**: Single branch, easier history
**Cons**: Manual copying, can forget environments

### Strategy 3: Version Tags

```
prod/
  kustomization.yaml  # References image tag v1.2.3
```

**Process**:
1. Deploy `v1.2.3` to dev
2. Test
3. Update prod kustomization: `newTag: "v1.2.3"`
4. Commit, GitOps applies

**Pros**: Explicit version per environment
**Cons**: Still need parameterization for other config

---

## ArgoCD Rendering Strategies

ArgoCD can render templates in multiple ways.

### Server-Side Rendering (Default)

ArgoCD renders Helm/Kustomize **inside the cluster**.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  source:
    repoURL: https://github.com/myorg/myapp
    path: helm/mychart
    helm:
      valueFiles:
        - values-prod.yaml
```

**Pros**: No rendered manifests in Git (DRY)
**Cons**: Deployed state opaque

### Config Management Plugins (CMP)

Custom rendering logic.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ConfigManagementPlugin
metadata:
  name: ytt
spec:
  generate:
    command: [sh, -c]
    args:
      - ytt -f . | kbld -f - | kapp template -f -
```

**Use case**: Custom tools like ytt, cue, jsonnet

---

## Key Takeaways

1. **Helm vs. Kustomize**: Templating vs. patching. Helm for complex/packaged, Kustomize for simple/GitOps.
2. **Rendered manifests debate**: Templates-only (DRY but opaque), rendered-only (verbose but explicit), hybrid (best auditability).
3. **Environment promotion**: Branches, directories, or version tags - each has trade-offs.
4. **GitOps integration**: ArgoCD/Flux can render server-side or use pre-rendered manifests.

The right choice depends on your priorities: auditability, review process, Git size, operational complexity. There's no universal answer - understand the trade-offs and choose deliberately.
