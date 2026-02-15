# Codesphere Skill for Claude Code

A comprehensive [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill for the [Codesphere](https://codesphere.com) cloud IDE and deployment platform.

## What It Does

- Generates and validates Codesphere CI pipeline configurations (`ci.yml`)
- Handles both single-service and landscape (multi-service) deployments with `schemaVersion: v0.2`
- Provides deployment guidance including domains, scaling, environment variables, and networking
- Covers the full platform: CI/CD, workspaces, CLI (`cs-go`), Public API, and GitHub Actions integration

## Installation

### Claude Code (CLI)

Copy the `codesphere/` directory to your Claude Code skills folder:

```bash
cp -r codesphere/ ~/.claude/skills/codesphere/
```

### Manual

Clone and symlink:

```bash
git clone https://github.com/LeonHartmann/codesphere-skill.git
ln -s "$(pwd)/codesphere-skill/codesphere" ~/.claude/skills/codesphere
```

## Skill Structure

```
codesphere/
  SKILL.md                              # Main skill reference
  references/
    ci-pipeline.md                      # CI schema specification, examples per framework
    landscape-deployments.md            # Multi-service v0.2, networking, managed services
    deployment-guide.md                 # Domains, scaling, modes, env vars, troubleshooting
    cli-and-api.md                      # cs-go CLI, Public API, GitHub Actions, GitLab CI
```

## Usage

Once installed, Claude Code will automatically use this skill when you:

- Ask to deploy an application to Codesphere
- Ask to create or fix a `ci.yml` file
- Ask about Codesphere configuration, scaling, or networking
- Mention landscape deployments or `schemaVersion: v0.2`
- Troubleshoot Codesphere deployment issues

### Example Prompts

```
"Create a ci.yml for my Next.js app"
"Set up a landscape deployment with a frontend, backend, and PostgreSQL database"
"Why isn't my Codesphere deployment working?"
"How do I configure path-based routing for my microservices?"
```

## Key Topics Covered

- **CI Pipeline Configuration** — Both single-service and landscape (v0.2) schemas
- **Landscape Deployments** — Multi-service architecture, private networking, managed databases
- **Critical Constraints** — Port 3000, host 0.0.0.0, persistent storage, Nix packages
- **Environment Variables** — Setting, propagation, built-in variables
- **Custom Domains** — DNS setup, path-based routing, zero-downtime releases
- **Horizontal Scaling** — Replicas, autoscaling, shared filesystem patterns
- **CLI & API** — cs-go commands, REST API endpoints, CI/CD integrations

## License

MIT
