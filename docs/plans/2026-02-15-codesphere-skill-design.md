# Codesphere Skill Design

## Goal

Create a comprehensive Claude Code skill for the Codesphere cloud IDE and deployment platform. The skill should:
- Cover the full platform (CI pipelines, deployment, scaling, domains, CLI, API)
- Actively generate and validate CI pipeline configurations
- Emphasize schemaVersion v0.2 (landscape deployments) as the primary pain point
- Follow Claude skill best practices (SKILL.md + references/ pattern)

## Structure

```
codesphere/
  SKILL.md                              # Main skill (~300 lines)
  references/
    ci-pipeline.md                      # CI schema reference, both versions, examples
    landscape-deployments.md            # Multi-service v0.2, networking, managed services
    deployment-guide.md                 # Domains, scaling, modes, env vars, constraints
    cli-and-api.md                      # cs-go CLI, Public API, GitHub Actions
```

## SKILL.md Sections

1. **Overview** - What Codesphere is (1 sentence)
2. **Critical Constraints** - Hard rules: port 3000, 0.0.0.0, /home/user/app, no root/use Nix, .codesphere-internal/ in .gitignore
3. **CI Pipeline Quick Reference** - Decision flow for single vs landscape, both schemas
4. **Generating CI Pipelines** - Workflow for creating/fixing ci.yml files
5. **Deployment Checklist** - Quick validation before deploying
6. **Common Mistakes** - Top pitfalls with fixes
7. **Reference Files** - Links to 4 reference docs

## Key Design Decisions

- `.codesphere-internal/` MUST be in `.gitignore` (landscape uses gitignore for deployment scanning)
- v0.2 schema gets prominent placement as main pain point
- Critical constraints (port, host, persistence) shown early to prevent common failures
- CI generation workflow detects project type and asks single vs landscape
- Follows existing skill patterns (gwi-research, picnic-mcp) for consistency

## User Requirements

- Target audience: Both developers deploying apps AND DevOps/platform teams
- Behavior: Actively generate ci.yml files + provide deployment guidance
- Push to GitHub as open-source skill
