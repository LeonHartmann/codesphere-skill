# Deployment Guide Reference

## Contents
- Workspaces and resource plans
- Deployment modes (always-on vs standby)
- Environment variables (setting, built-in, inline)
- Custom domains and DNS setup
- Zero downtime releases
- Horizontal scaling and replicas
- Teams, monitoring, marketplace
- Version control integration
- Nix package management
- Troubleshooting

## Workspaces

A workspace is the fundamental unit in Codesphere — a cloud environment containing code, IDE, terminal, CI pipeline, and deployment. Each workspace maps to a repository/branch.

### Resource Plans

| Plan Name | Use Case |
|-----------|----------|
| Micro | Development, small apps |
| Boost | Production workloads |
| Pro | High-performance applications |

In landscape `ci.yml` configs, plans are specified as integer IDs (e.g., `plan: 8`, `plan: 22`). Higher numbers = more resources. The exact plan ID mapping is visible in the Codesphere IDE when configuring services (Setup > CI). Named plans (Micro/Boost/Pro) are used in the API and GitHub Actions; integer IDs are used in landscape `ci.yml` files.

## Deployment Modes

### Always On
- Dedicated resources continuously available
- Full-price plans
- Required for production environments needing constant availability
- Required for workspaces with replicas

### Standby When Unused (Off-When-Unused)
- Resources returned to pool when inactive
- Goes standby after ~60 minutes of inactivity
- ~10% of always-on pricing
- Re-activates in ~1 second upon domain visit or IDE access
- Auto-re-runs the CI Pipeline Run stage on activation
- Free plan workspaces are always in this mode

## Environment Variables

### Setting Variables
- **Location**: Setup > Environment Variables in the workspace UI
- **Scope**: Workspace-specific (not shared across workspaces)
- **Visibility**: Hidden by default (asterisks), eye icon to reveal
- **Propagation**: Changes require re-running the CI Pipeline Run stage

### Built-in Variables

| Variable | Description |
|----------|-------------|
| `CS_REPLICA` | Unique identifier for each replica (use to prevent file write conflicts) |
| `WORKSPACE_ID` | The workspace identifier |
| `NV_LIBCUBLAS_VERSION` | Present when GPU resources are available |

### Inline Variables in CI Commands

Set environment variables directly in pipeline commands:

```yaml
steps:
  - name: Run server
    command: PYTHONPATH=/home/user/app/pipLib PORT=3000 python3 server.py
```

Or source from `.env` files:

```yaml
steps:
  - name: Run app
    command: . .env && export MY_VAR=$MY_VAR && npm start
```

### Important Notes

- When manually creating a replica, environment variables must be copied manually
- Framework-specific naming may apply (e.g., React requires `REACT_APP_` prefix)
- Batch upload supported via Public API during workspace creation

## Custom Domains

### DNS Setup

1. Add an **A record** pointing to Codesphere's IP
2. Add a **TXT record** for domain verification
3. Verify domain in Codesphere UI
4. Connect one or more workspaces to the domain

### Multiple Workspaces on One Domain

Connecting multiple workspaces to one domain enables:
- Automatic load balancing
- A/B testing
- Blue/green deployments

### Path-Based Routing with Domains

Map different URL paths to different workspaces under a single domain:

- Applications must serve from their assigned path prefix (not root)
- Supports sub-paths (e.g., `/shop/garden`)
- Regex-based paths supported via checkbox in UI

## Zero Downtime Releases

Requires 2+ workspaces and a custom domain:

1. Deploy new version to staging workspace
2. Test staging workspace
3. Swap domain connection from production to staging
4. Instant rollback by reconnecting previous workspace

## Horizontal Scaling

### Replicas

- Up to 10 replicas per workspace/service (more in enterprise)
- **Shared filesystem** across replicas (`/home/user/app`)
- Prepare/Test run on main replica only; Run stage executes on all replicas
- Each replica has independent CPU and RAM
- SSD storage is shared (not duplicated)
- Per-replica pricing (e.g., Micro x 4 replicas = 4x cost)

### Autoscaling

- Based on CPU usage within configured ranges
- Set min and max replica count
- Codesphere scales automatically based on load

### Replica Best Practices

- Use `CS_REPLICA` env var to segregate per-replica data (e.g., log files)
- Avoid concurrent writes to the same file from multiple replicas
- All replicas share `/home/user/app` — design for shared filesystem

## Teams

- Organizational unit containing workspaces
- 3 free seats per team, extra seats $10/month
- Teams determine datacenter location
- Manage member permissions per team

## Monitoring

### Resource Monitoring
- Built-in CPU and RAM tracking
- 14-day history

### Uptime Monitoring
- For always-on workspaces with custom domains
- Periodic HTTP checks on port 3000
- 7-day display (green/yellow/red)

## Marketplace

Managed service instances available through the marketplace:
- MySQL
- Redis
- PostgreSQL
- MongoDB

These can be provisioned as managed services in landscape deployments or as standalone instances.

## Version Control Integration

Supported platforms:
- GitHub
- GitLab
- Bitbucket

### Setup
- OAuth connection grants repository access
- Create workspaces directly from repositories
- `git pull` in terminal to fetch updates
- Personal access token needed for private repos

### Best Practices
- Commit `ci.yml` to version control for workspace reuse
- Use `.gitignore` to exclude `node_modules`, `.venv`, `build`, `.codesphere-internal/`
- Use separate databases for staging vs production environments

## Nix Package Management

Since root access is restricted, use Nix for OS-level packages:

```bash
# Install a package
nix-env -iA nixpkgs.<package>

# Examples
nix-env -iA nixpkgs.nodejs_24
nix-env -iA nixpkgs.ollama
nix-env -iA nixpkgs.process-compose
nix-env -iA nixpkgs.uv
nix-env -iA nixpkgs.python311
```

Nix packages persist in `/nix/store` and are accessible to all services in a landscape.

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| App not accessible | Not binding to `0.0.0.0:3000` | Update host and port in app config |
| Packages disappear after restart | Installed outside persistent paths | Install to `/home/user/app` or use Nix |
| Permission denied | Trying to use root/sudo | Use Nix for OS packages |
| Workspace stuck rebooting | Invalid CI pipeline | Check `ci.yml` syntax and run stage |
| GitHub repos missing | OAuth token expired | Re-grant repository access |
| Env var changes not working | Run stage not restarted | Re-run the CI Pipeline Run stage |
| Deployment fails | Missing `.codesphere-internal/` in `.gitignore` | Add `.codesphere-internal/` to `.gitignore` |
| Pipeline not parsing | Missing `schemaVersion: v0.2` or using flat `run.steps[]` | Ensure `schemaVersion: v0.2` is first line and use named services under `run` |
