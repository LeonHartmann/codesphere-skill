# CI Pipeline Reference

## Contents
- File location and naming
- Schema versions (single-service vs landscape v0.2)
- Step schema
- Single-service schema
- Landscape schema (v0.2) with managed services
- Complete examples (Node.js, Next.js, Python, Go, Ruby, PHP, Vue.js, LLM)
- CI profiles
- Pipeline stage behavior

## File Location

CI config files live at the **project root** (`/home/user/app/`):

| File | Purpose |
|------|---------|
| `ci.yml` | Default pipeline configuration |
| `ci.<profilename>.yml` | Named profile (e.g., `ci.production.yml`, `ci.staging.yml`) |

Profiles are created via: **Setup > CI > Add Profile** in the Codesphere IDE.

## Schema Versions

### No schemaVersion (Single-Service)

The original format. Used for single-service deployments. The `run` section uses a flat `steps` array.

### schemaVersion: v0.2 (Landscape / Multi-Service)

Required for landscape deployments (multiple services in one workspace). The `run` section uses named service keys instead of a flat `steps` array. Each service gets its own `steps`, `plan`, `replicas`, and `network` configuration.

**Key difference:** `run.steps[]` (single) vs `run.<serviceName>.steps[]` (landscape).

## Step Schema

Every step in any stage:

```yaml
steps:
  - name: "Human-readable description"    # optional but recommended
    command: "shell command to execute"     # required
```

The `command` field accepts anything you'd run from a terminal. You can reference external scripts:

```yaml
steps:
  - name: Start app
    command: "./start.sh"
```

## Single-Service Schema

```yaml
prepare:
  steps:
    - name: <string>        # optional
      command: <string>      # required
test:
  steps:
    - name: <string>
      command: <string>
run:
  steps:
    - name: <string>
      command: <string>
```

## Landscape Schema (v0.2)

```yaml
schemaVersion: v0.2          # REQUIRED
prepare:
  steps:
    - name: <string>
      command: <string>
test:
  steps:
    - name: <string>
      command: <string>
run:
  <serviceName>:             # any valid YAML key
    steps:
      - name: <string>
        command: <string>
    plan: <integer>           # resource tier
    replicas: <integer>       # horizontal scaling (default: 1, max: 10)
    isPublic: <boolean>       # externally accessible (shorthand)
    network:
      ports:
        - port: <integer>     # port number (usually 3000)
          isPublic: <boolean> # externally accessible
      paths:
        - port: <integer>     # port to forward to
          path: <string>      # URL path prefix (e.g., "/api")
          stripPath: <boolean> # strip prefix before forwarding
```

### Managed Service Schema (v0.2)

For managed databases/services within a landscape:

```yaml
run:
  <serviceName>:
    provider:
      name: <string>          # e.g., "postgres", "redis", "mysql", "mongodb"
      version: <string>       # provider schema version (e.g., "v1")
    plan:
      id: <integer>           # plan identifier
      parameters:
        storage: <integer>    # storage in MB
        cpu: <integer>        # CPU units
        memory: <integer>     # memory in MB
    config:
      version: <string>       # database version (e.g., "17.6")
      userName: <string>      # database user
      databaseName: <string>  # database name
    secrets:
      userPassword: <string>
      superuserPassword: <string>
```

## Complete Examples

### Node.js (Express)

```yaml
prepare:
  steps:
    - name: Install dependencies
      command: npm install
test:
  steps:
    - name: Run tests
      command: npm test
run:
  steps:
    - name: Start server
      command: npm start
```

### Next.js

```yaml
prepare:
  steps:
    - name: Install dependencies
      command: npm install
    - name: Build
      command: npm run build
test:
  steps: []
run:
  steps:
    - name: Start Next.js
      command: npm start
```

### Python (FastAPI with Pipenv)

```yaml
prepare:
  steps:
    - name: Install dependencies
      command: pipenv install -r requirements.txt
test:
  steps: []
run:
  steps:
    - name: Start server
      command: >
        source "$(pipenv --venv)/bin/activate" &&
        uvicorn main:app --reload --host 0.0.0.0 --port 3000
```

### Python (pip with target)

```yaml
prepare:
  steps:
    - name: Install dependencies
      command: pip install -r requirements.txt --target=/home/user/app/pipLib
test:
  steps: []
run:
  steps:
    - name: Start server
      command: PYTHONPATH=/home/user/app/pipLib python3 server.py
```

### Go

```yaml
prepare:
  steps:
    - name: Build
      command: go build -o app
test:
  steps:
    - name: Run tests
      command: go test ./...
run:
  steps:
    - name: Start server
      command: ./app
```

### Ruby on Rails

```yaml
prepare:
  steps:
    - name: Install dependencies
      command: bundle install
    - name: Setup database
      command: bundle exec rails db:setup
test:
  steps:
    - name: Run tests
      command: bundle exec rails test
run:
  steps:
    - name: Start Rails
      command: bundle exec rails server -b 0.0.0.0 -p 3000
```

### PHP (Laravel)

```yaml
prepare:
  steps:
    - name: Install dependencies
      command: composer install
test:
  steps: []
run:
  steps:
    - name: Start Laravel
      command: php artisan serve --host=0.0.0.0 --port=3000
```

### Vue.js

```yaml
prepare:
  steps:
    - name: Install dependencies
      command: yarn install
    - name: Build
      command: yarn build
test:
  steps: []
run:
  steps:
    - name: Start preview server
      command: yarn preview --host --port 3000
```

### Python LLM (Ollama/llama.cpp)

```yaml
prepare:
  steps:
    - name: Install dependencies
      command: pipenv install llama-cpp-python
    - name: Download model
      command: >
        wget -P /home/user/app/models
        https://huggingface.co/TheBloke/Llama-2-7B-Chat-GGUF/resolve/main/llama-2-7b-chat.Q4_K_M.gguf
test:
  steps: []
run:
  steps:
    - name: Start LLM server
      command: >
        pipenv run python3 -m llama_cpp.server
        --port 3000 --host 0.0.0.0
        --model /home/user/app/models/llama-2-7b-chat.Q4_K_M.gguf
```

### With Custom Node.js Version

```yaml
prepare:
  steps:
    - name: Set Node.js version
      command: sudo n 20
    - name: Install dependencies
      command: npm install
    - name: Build
      command: npm run build
test:
  steps: []
run:
  steps:
    - name: Set Node.js version
      command: sudo n 20
    - name: Start server
      command: npm start
```

**Important:** `sudo n <version>` must appear in BOTH `prepare` and `run` stages — it does not persist.

### With Nix Packages

```yaml
prepare:
  steps:
    - name: Install Node.js 24
      command: nix-env -iA nixpkgs.nodejs_24
    - name: Install dependencies
      command: npm install
test:
  steps: []
run:
  steps:
    - name: Start server
      command: npm start
```

## CI Profiles

Multiple pipeline configurations within a single workspace:

1. Navigate to **Setup > CI** in the Codesphere IDE
2. Click **Add Profile**
3. Name the profile (creates `ci.<name>.yml`)
4. Select profile when running the pipeline

Use cases:
- `ci.yml` — development/default
- `ci.staging.yml` — staging environment
- `ci.production.yml` — production with optimized builds

## Pipeline Stage Behavior

| Stage | Runs On | Restarts On Crash | Shared Across |
|-------|---------|-------------------|---------------|
| Prepare | Main replica only | No | All services share one prepare |
| Test | Main replica only | No | All services share one test |
| Run | All replicas | Yes (automatic) | Each service runs independently |

In landscape deployments, `prepare` and `test` run once globally. The `run` stage executes per service on each service's replicas.
