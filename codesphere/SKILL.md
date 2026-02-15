---
name: codesphere
description: >
  Generates and validates Codesphere CI pipeline configurations (ci.yml),
  guides deployment of applications to Codesphere workspaces, and configures
  landscape deployments (schemaVersion v0.2) with multi-service networking.
  Use when deploying to Codesphere, creating or fixing ci.yml files, setting up
  landscape deployments, managing custom domains, horizontal scaling, environment
  variables, or troubleshooting Codesphere deployments. Triggers: codesphere,
  ci.yml, ci pipeline, workspace, landscape, schemaVersion, deploy to codesphere.
---

# Codesphere

Codesphere is a cloud IDE and DevOps platform that combines browser-based development with integrated CI/CD, deployment, scaling, and operations — no Docker, Kubernetes, or Serverless knowledge required.

## Critical Constraints

These are **hard requirements** that cause deployment failures when violated:

| Constraint | Detail |
|------------|--------|
| **Host & Port** | Applications MUST bind to `0.0.0.0:3000`. Not `localhost`, not another port. |
| **Persistent storage** | Only `/home/user/app` and `/nix/store` survive restarts. Everything else resets. |
| **No root access** | Use Nix for OS packages: `nix-env -iA nixpkgs.<package>` |
| **TLS termination** | Codesphere handles HTTPS. Your app serves HTTP only. |
| **.gitignore** | `.codesphere-internal/` MUST be in `.gitignore`. Landscape deployments use `.gitignore` to determine what to scan — missing this breaks deployment. |
| **Node.js versions** | `sudo n <version>` is NOT persistent. Add version switch to BOTH `prepare` and `run` stages. |
| **Single public port** | One internet-facing port per service. Multiple ports require landscape deployment or separate workspaces. |

## CI Pipeline Quick Reference

CI config lives at project root (NOT in a `.codesphere/` directory):
- Default: `ci.yml`
- Profiles: `ci.<profilename>.yml` (e.g., `ci.production.yml`)

All CI pipelines use `schemaVersion: v0.2`. Even single-app deployments use this format with one service under `run`.

### CI Pipeline Format (schemaVersion: v0.2)

```yaml
schemaVersion: v0.2
prepare:
  steps:
    - name: Install dependencies
      command: npm install
test:
  steps: []
run:
  frontend:
    steps:
      - name: Start frontend
        command: npm run start:frontend
    plan: 8
    replicas: 1
    network:
      ports:
        - port: 3000
          isPublic: true
      paths:
        - port: 3000
          path: /
          stripPath: false
  backend:
    steps:
      - name: Start backend
        command: npm run start:backend
    plan: 8
    replicas: 1
    network:
      ports:
        - port: 3000
          isPublic: false
```

For the full schema specification, see [references/ci-pipeline.md](references/ci-pipeline.md).
For landscape-specific details, see [references/landscape-deployments.md](references/landscape-deployments.md).

## Generating CI Pipelines

When asked to create or fix a `ci.yml`:

1. **Detect project type** — check `package.json`, `requirements.txt`, `go.mod`, `Gemfile`, `pom.xml`, `Cargo.toml`, etc.
2. **Ask about services** — does the user need multiple services (frontend + backend, app + database)?
3. **Generate with `schemaVersion: v0.2`** — always use named services under `run`, even for single-app deployments
4. **Apply critical constraints** — host `0.0.0.0`, port `3000`, Nix for OS packages
5. **Validate .gitignore** — ensure `.codesphere-internal/` is listed
6. **Add Node.js version management** to both `prepare` and `run` if applicable

### Project Type Detection

| File Found | Framework | Prepare Command | Run Command |
|------------|-----------|-----------------|-------------|
| `package.json` (next) | Next.js | `npm install && npm run build` | `npm start` |
| `package.json` (nuxt) | Nuxt.js | `npm install && npm run build` | `node .output/server/index.mjs` |
| `package.json` (express) | Express | `npm install` | `npm start` or `node server.js` |
| `requirements.txt` | Python/Flask/Django | `pip install -r requirements.txt --target=/home/user/app/pipLib` | `PYTHONPATH=.../pipLib python3 app.py` |
| `Pipfile` | Python/Pipenv | `pipenv install` | `pipenv run python3 app.py` |
| `go.mod` | Go | `go build -o app` | `./app` |
| `Gemfile` | Ruby/Rails | `bundle install` | `bundle exec rails server -b 0.0.0.0 -p 3000` |
| `composer.json` | PHP/Laravel | `composer install` | `php artisan serve --host=0.0.0.0 --port=3000` |

## Deployment Checklist

Before deploying, verify:

- [ ] App binds to `0.0.0.0:3000`
- [ ] `.codesphere-internal/` is in `.gitignore`
- [ ] `ci.yml` is at project root (not in a subdirectory)
- [ ] All file writes go to `/home/user/app`
- [ ] OS packages installed via Nix (not apt/sudo)
- [ ] Environment variables set in Codesphere UI (Setup > Environment Variables)
- [ ] Node.js version set in both `prepare` and `run` stages (if applicable)
- [ ] For landscape: each service runs on port `3000` independently
- [ ] For landscape: `schemaVersion: v0.2` is present at top of `ci.yml`

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| App binds to `localhost:3000` | Change to `0.0.0.0:3000` (must accept external connections) |
| Missing `.codesphere-internal/` in `.gitignore` | Add it — landscape deployment scanning depends on `.gitignore` |
| Using `apt-get` or `sudo` for packages | Use `nix-env -iA nixpkgs.<package>` instead |
| Files written outside `/home/user/app` | Move to `/home/user/app` — other paths don't persist |
| Node.js version only in `prepare` | Add `sudo n <version>` to `run` stage too (not persistent across stages) |
| Using `run.steps[]` instead of named services | Use `run.<serviceName>.steps[]` — flat `run.steps[]` is not supported |
| Missing `schemaVersion: v0.2` | Add `schemaVersion: v0.2` as first line of `ci.yml` |
| Environment variable changes not taking effect | Re-run the CI Pipeline Run stage after changing env vars |
| Replicas writing to same files | Use `CS_REPLICA` env var to create per-replica file paths |
| Path routing not working | Application routes must include the path prefix in code |

## Three Pipeline Stages

| Stage | Purpose | Runs On | Auto-restarts |
|-------|---------|---------|---------------|
| **Prepare** | Install deps, build, setup | Main replica only (shared across all services in landscape) | No |
| **Test** | Run automated tests | Main replica only (shared across all services in landscape) | No |
| **Run** | Start application server | All replicas (per service in landscape) | Yes (on crash) |

## Reference Files

- **[references/ci-pipeline.md](references/ci-pipeline.md)** — Full CI schema specification (v0.2), step fields, complete examples per framework
- **[references/landscape-deployments.md](references/landscape-deployments.md)** — Multi-service v0.2 configs, private networking, managed services (Postgres, Redis), path routing
- **[references/deployment-guide.md](references/deployment-guide.md)** — Custom domains, horizontal scaling, deployment modes, environment variables, DNS setup, zero-downtime releases
- **[references/cli-and-api.md](references/cli-and-api.md)** — cs-go CLI commands, Public API endpoints, GitHub Actions integration, GitLab/Bitbucket CI
