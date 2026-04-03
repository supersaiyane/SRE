# YAML Fundamentals — Complete Reference

YAML (YAML Ain't Markup Language) is the standard configuration format for Kubernetes, Docker Compose, CI/CD pipelines, Ansible, and most DevOps tools. This guide covers everything you need to write YAML confidently.

---

## Basic Syntax Rules

```yaml
# This is a comment
key: value                    # Key-value pair
indented_key:                 # Nested structure
  child_key: child_value      # 2-space indent (standard)
```

**Rules**:
- Indentation uses **spaces only** (never tabs)
- Standard indent is **2 spaces** (Kubernetes convention)
- Keys and values are separated by `: ` (colon + space)
- YAML is **case-sensitive**
- Strings don't need quotes (unless they contain special characters)

---

## Data Types

### Strings

```yaml
# All of these are valid strings
name: hello
name: "hello"                  # Quoted (preserves special chars)
name: 'hello'                  # Single-quoted (no escape processing)
name: "line one\nline two"     # Double-quoted (processes \n, \t, etc.)
name: "yes"                    # Quote to prevent boolean interpretation
```

**When to quote**:
- Values that look like booleans: `"true"`, `"false"`, `"yes"`, `"no"`
- Values that look like numbers: `"1234"`, `"3.14"`
- Values with special characters: `"contains: colon"`, `"has # hash"`
- Empty strings: `""`

### Numbers

```yaml
integer: 42
negative: -17
float: 3.14
scientific: 1.2e+5
octal: 0o14                   # Octal (= 12)
hex: 0xFF                     # Hexadecimal (= 255)
```

### Booleans

```yaml
enabled: true
disabled: false
# Also valid: yes/no, on/off, True/False (but true/false preferred)
```

### Null

```yaml
value: null
value: ~                       # Shorthand for null
value:                         # Empty value = null
```

---

## Collections

### Lists (Sequences)

```yaml
# Block style (most common in K8s)
fruits:
  - apple
  - banana
  - cherry

# Inline style (flow)
fruits: [apple, banana, cherry]

# List of objects
containers:
  - name: web
    image: nginx
    port: 80
  - name: sidecar
    image: envoy
    port: 9901
```

### Maps (Dictionaries)

```yaml
# Block style
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
    tier: frontend

# Inline style (flow)
metadata: {name: my-app, namespace: production}
```

### Nested Structures

```yaml
# Common Kubernetes pattern: map → list → map
spec:
  containers:                  # List of maps
    - name: app
      image: myapp:v1
      ports:                   # List of maps inside a map
        - containerPort: 8080
          protocol: TCP
      env:                     # List of maps
        - name: DB_HOST
          value: "postgres"
        - name: DB_PORT
          value: "5432"
```

---

## Multi-Line Strings

### Literal Block (`|`) — preserves newlines

```yaml
script: |
  #!/bin/bash
  echo "Hello"
  echo "World"
# Result: "#!/bin/bash\necho \"Hello\"\necho \"World\"\n"
```

### Folded Block (`>`) — joins lines with spaces

```yaml
description: >
  This is a very long
  description that will
  be folded into a single line.
# Result: "This is a very long description that will be folded into a single line.\n"
```

### Modifiers

```yaml
# Strip trailing newline
script: |-
  echo "no trailing newline"

# Keep trailing newlines
script: |+
  echo "keeps all trailing newlines"


```

**Use case in Kubernetes**:
```yaml
# ConfigMap with a multi-line config file
data:
  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://backend:8080;
      }
    }
```

---

## Anchors & Aliases (DRY YAML)

Anchors (`&`) let you define a value once, and aliases (`*`) let you reuse it.

```yaml
# Define defaults with an anchor
defaults: &default-settings
  replicas: 3
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 256Mi

# Reuse with alias
production:
  <<: *default-settings          # Merge all default settings
  replicas: 5                    # Override specific values

staging:
  <<: *default-settings
  replicas: 1
```

**Result**: `production` gets all of `default-settings` but with `replicas: 5`.

**Real-world use — docker-compose.yml**:
```yaml
x-common: &common
  restart: unless-stopped
  logging:
    driver: json-file
    options:
      max-size: "10m"

services:
  web:
    <<: *common
    image: nginx
    ports:
      - "80:80"

  api:
    <<: *common
    image: myapi
    ports:
      - "3000:3000"
```

---

## Multiple Documents in One File

Use `---` to separate multiple YAML documents in a single file. Common in Kubernetes manifests.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
  namespace: my-app
data:
  LOG_LEVEL: info
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 3
  # ... rest of deployment
```

---

## Common Gotchas

### 1. Tabs vs Spaces

```yaml
# WRONG (tabs will cause parse errors)
spec:
	containers:       # This is a tab — YAML will reject it

# CORRECT
spec:
  containers:         # Two spaces
```

### 2. Unquoted Special Values

```yaml
# WRONG — interpreted as boolean true
country: NO           # Norway? Nope, YAML reads this as false

# CORRECT
country: "NO"

# WRONG — interpreted as float
version: 1.0          # Becomes 1 (loses the .0)

# CORRECT
version: "1.0"
```

### 3. Colon in Values

```yaml
# WRONG — colon followed by space starts a new mapping
message: Error: something broke

# CORRECT
message: "Error: something broke"
```

### 4. Indentation Alignment in Lists

```yaml
# WRONG — misaligned list items
containers:
  - name: web
     image: nginx     # 3 spaces instead of 4 — parse error

# CORRECT — consistent indentation
containers:
  - name: web
    image: nginx      # Aligned with 'name'
```

---

## YAML for Kubernetes — Key Patterns

### The Universal Structure

Every Kubernetes resource follows this pattern:

```yaml
apiVersion: <group/version>    # Which API to use
kind: <ResourceType>           # What to create
metadata:                      # Identity and organization
  name: <name>                 # Required: unique within namespace
  namespace: <ns>              # Optional: default is "default"
  labels:                      # Optional: for selection and grouping
    app: my-app
  annotations:                 # Optional: for tooling metadata
    description: "My application"
spec:                          # Desired state (varies by Kind)
  # ... resource-specific config
```

### Label Selectors

```yaml
# In a Service — select pods with matching labels
selector:
  app: my-app                  # Simple equality

# In a Deployment — matchLabels
selector:
  matchLabels:
    app: my-app
    tier: frontend

# Advanced — matchExpressions
selector:
  matchExpressions:
    - key: environment
      operator: In
      values: [production, staging]
    - key: tier
      operator: NotIn
      values: [test]
```

### Environment Variables

```yaml
env:
  # Direct value
  - name: LOG_LEVEL
    value: "info"

  # From ConfigMap
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: db-config
        key: host

  # From Secret
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password

  # From Pod metadata
  - name: POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name
```

### Resource Limits

```yaml
resources:
  requests:                    # Minimum guaranteed
    cpu: 100m                  # 100 millicores = 0.1 CPU
    memory: 128Mi              # 128 Mebibytes
  limits:                      # Maximum allowed
    cpu: 500m                  # 0.5 CPU
    memory: 256Mi              # 256 Mebibytes
```

**CPU units**: `1` = 1 vCPU, `100m` = 0.1 vCPU, `250m` = 0.25 vCPU

**Memory units**: `Ki` (Kibibytes), `Mi` (Mebibytes), `Gi` (Gibibytes)

---

## Validation Tools

| Tool | Usage |
|------|-------|
| `yamllint` | `yamllint myfile.yaml` — checks syntax and style |
| `kubectl` | `kubectl apply --dry-run=client -f myfile.yaml` — validates K8s manifests |
| `kubeval` | `kubeval myfile.yaml` — validates against K8s schemas |
| `yq` | `yq eval '.metadata.name' myfile.yaml` — query/transform YAML |
| Online | [YAML Lint](https://www.yamllint.com/) — quick browser-based check |

---

## Quick Reference Card

| Syntax | Meaning |
|--------|---------|
| `key: value` | Key-value pair |
| `- item` | List item |
| `#` | Comment |
| `---` | Document separator |
| `\|` | Literal block (preserve newlines) |
| `>` | Folded block (join lines) |
| `&name` | Anchor (define reusable block) |
| `*name` | Alias (reference anchor) |
| `<<:` | Merge key (merge anchor into map) |
| `~` or `null` | Null value |
| `true` / `false` | Boolean |
