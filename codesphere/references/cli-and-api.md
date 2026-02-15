# CLI and API Reference

## Contents
- cs-go CLI (installation, authentication, commands, workflows)
- Public API (base URL, authentication, endpoints, workspace creation)
- GitHub Actions integration
- GitLab CI integration
- Bitbucket Pipelines integration

## cs-go CLI

The Codesphere CLI (`cs-go`) enables command-line workspace management and CI/CD automation.

### Installation

Download binary from GitHub releases: `codesphere-cloud/cs-go` (latest: v0.17.0)

### Authentication

Set the `CS_TOKEN` environment variable with an API token from Codesphere user settings.

### Global Flags

| Flag | Env Var | Description |
|------|---------|-------------|
| `--api` / `-a` | `CS_API` | API URL (default: `https://codesphere.com/api`) |
| `--team` / `-t` | `CS_TEAM_ID` | Team ID |
| `--workspace` / `-w` | `CS_WORKSPACE_ID` | Workspace ID |

### Commands

| Command | Description |
|---------|-------------|
| `cs create` | Create a Codesphere resource (workspace, etc.) |
| `cs delete` | Delete Codesphere resources |
| `cs exec` | Run a command in a workspace |
| `cs list` | List resources (workspaces, teams) |
| `cs log` | Retrieve run logs from services |
| `cs monitor` | Monitor a command and report health |
| `cs open` | Open the Codesphere IDE in browser |
| `cs set-env` | Set environment variables on a workspace |
| `cs start` | Start workspace pipeline (prepare/test/run) |
| `cs update` | Update the CLI to latest version |
| `cs version` | Print CLI version |

### Common Workflows

**Deploy from CLI:**
```bash
export CS_TOKEN="your-api-token"
export CS_TEAM_ID="your-team-id"
export CS_WORKSPACE_ID="your-workspace-id"

# Start the full pipeline
cs start --stage prepare
cs start --stage test
cs start --stage run
```

**Execute commands remotely:**
```bash
cs exec --command "npm install"
cs exec --command "npm run build"
```

**Set environment variables:**
```bash
cs set-env --key "DATABASE_URL" --value "postgres://..."
cs set-env --key "API_KEY" --value "secret123"
```

## Public API

### Base URL
- Cloud: `https://codesphere.com/api`
- Self-hosted: `https://[Your_Base_URL]/api`

### Authentication
Bearer token from **Settings > API Keys** in Codesphere UI.

```bash
curl -H "Authorization: Bearer <token>" https://codesphere.com/api/workspaces
```

### Swagger Documentation
Available at: `https://codesphere.com/api/swagger-ui`

### Key Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/workspaces` | Create new workspace |
| `GET` | `/workspaces/{id}` | Get workspace details |
| `POST` | `/workspaces/{id}/git/pull/{remote}` | Git pull |
| `POST` | `/workspaces/{id}/pipeline/prepare/start` | Start prepare stage |
| `GET` | `/workspaces/{id}/pipeline/prepare` | Get prepare status |
| `POST` | `/workspaces/{id}/pipeline/test/start` | Start test stage |
| `GET` | `/workspaces/{id}/pipeline/test` | Get test status |
| `POST` | `/workspaces/{id}/pipeline/run/start` | Start run stage |
| `POST` | `/workspaces/{id}/pipeline/run/stop` | Stop run stage |
| `GET` | `/domains/team/{teamId}/domain/{name}/workspace-connections` | Domain routing |

### Workspace Creation via API

```bash
curl -X POST https://codesphere.com/api/workspaces \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "teamId": "team-id",
    "name": "my-workspace",
    "gitUrl": "https://github.com/user/repo.git",
    "plan": "Boost"
  }'
```

## GitHub Actions Integration

### Action

`codesphere-cloud/gh-action-deploy@main`

### Supported Triggers

`opened`, `reopened`, `synchronize`, `closed` (on `pull_request`)

### Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `email` | Yes | — | Codesphere account email |
| `password` | Yes | — | Account password |
| `team` | Yes | — | Team name |
| `plan` | No | Boost | Micro, Boost, or Pro |
| `onDemand` | No | false | Standby when unused |
| `env` | No | — | Dotenv-formatted env vars |
| `vpnConfig` | No | — | VPN config name |
| `apiUrl` | No | codesphere.com | Custom instance URL |

### Required GitHub Secrets

- `CODESPHERE_EMAIL`
- `CODESPHERE_PASSWORD`

### Example Workflow

```yaml
name: Deploy to Codesphere
on:
  pull_request:
    types: [opened, reopened, synchronize, closed]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: codesphere-cloud/gh-action-deploy@main
        with:
          email: ${{ secrets.CODESPHERE_EMAIL }}
          password: ${{ secrets.CODESPHERE_PASSWORD }}
          team: 'MyTeam'
          plan: 'Boost'
          onDemand: true
          env: |
            API_KEY=${{ secrets.API_KEY }}
            DATABASE_URL=${{ secrets.DATABASE_URL }}
```

This creates a preview deployment for each PR and tears it down when the PR is closed.

## GitLab CI Integration

Use the Public API in `.gitlab-ci.yml`:

```yaml
deploy:
  stage: deploy
  script:
    - |
      curl -X POST "https://codesphere.com/api/workspaces/${CS_WORKSPACE_ID}/git/pull/origin" \
        -H "Authorization: Bearer ${CS_TOKEN}"
    - |
      curl -X POST "https://codesphere.com/api/workspaces/${CS_WORKSPACE_ID}/pipeline/prepare/start" \
        -H "Authorization: Bearer ${CS_TOKEN}"
  variables:
    CS_TOKEN: $CODESPHERE_API_TOKEN
    CS_WORKSPACE_ID: $WORKSPACE_ID
```

## Bitbucket Pipelines Integration

Similar approach using the Public API:

```yaml
pipelines:
  default:
    - step:
        name: Deploy to Codesphere
        script:
          - |
            curl -X POST "https://codesphere.com/api/workspaces/${CS_WORKSPACE_ID}/git/pull/origin" \
              -H "Authorization: Bearer ${CS_TOKEN}"
          - |
            curl -X POST "https://codesphere.com/api/workspaces/${CS_WORKSPACE_ID}/pipeline/prepare/start" \
              -H "Authorization: Bearer ${CS_TOKEN}"
```
