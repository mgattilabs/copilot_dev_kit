---
applyTo: "**/azure-pipelines.yml,**/azure-pipelines*.yml,**/*.pipeline.yml,**/pipelines/**/*.yml"
---

# Azure DevOps Pipeline Standards

## Structure Rules

- Use **stages** to organize: Build → Test → Docker → Deploy Staging → Deploy Production
- Use **jobs** within stages to enable parallel execution
- Use **templates** for reusable steps (store in `/templates/` folder)
- 2-space indentation throughout

## Naming

- Stages: PascalCase (`BuildApi`, `DeployStaging`)
- Jobs: PascalCase (`BuildDotNet`, `BuildFrontend`)
- Steps: `displayName` always in plain English describing what the step does

## Variables

- Sensitive values (connection strings, keys, passwords) ALWAYS in Key Vault-linked variable groups — NEVER hardcoded
- Use `variables` at top level for shared values
- Parameter-ize templates for reuse across projects

## Triggers

- `trigger` on `main` and `develop` branches
- `pr` trigger for pull request validation
- Always exclude `docs/*` and `*.md` from triggering builds

## Artifacts

- Always use `PublishPipelineArtifact` to store build outputs
- Always use `DownloadPipelineArtifact` in deploy stages — never rebuild
- Never pass files between stages via workspace — always use artifacts

## Caching

- Always cache `node_modules` with `Cache@2` task using `package-lock.json` as key
- Always cache NuGet packages

## Deploy Gates

- Staging: automatic on `develop` branch success
- Production: requires `environment` with manual approval gate in ADO
- Use `dependsOn` and `condition` to control flow
- `condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))`

## Security

- Least privilege for service connections
- Use managed identity where possible
- Never echo secrets in logs
- Scan images before pushing to registry
