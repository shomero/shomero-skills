---
name: codemagic
description: Drive Codemagic CI for a project through its REST API — trigger a workflow on a branch, check status, list, cancel — with workflows as config-as-code in codemagic.yaml and secrets in env groups. Use when asked to build, deploy, ship or check CI through Codemagic.
---

# ci-codemagic

## Configure (per project, in its CLAUDE.md)

- Token file: `~/.config/codemagic/token` (read at runtime, never echoed)
- App id: `<codemagic app id>`
- Workflow ids = the **keys** in `codemagic.yaml` (`pr-check`, `android-stg`, …), not UI ids

```bash
CM_TOKEN=$(cat ~/.config/codemagic/token); CM_APP=<app id>
```

## Calls

```bash
# trigger
curl -s -X POST https://api.codemagic.io/builds -H "x-auth-token: $CM_TOKEN" -H "Content-Type: application/json" \
  -d "{\"appId\":\"$CM_APP\",\"workflowId\":\"android-stg\",\"branch\":\"main\"}"
# status
curl -s -H "x-auth-token: $CM_TOKEN" https://api.codemagic.io/builds/<buildId>
# recent
curl -s -H "x-auth-token: $CM_TOKEN" "https://api.codemagic.io/builds?appId=$CM_APP&limit=10"
# cancel
curl -s -X POST -H "x-auth-token: $CM_TOKEN" https://api.codemagic.io/builds/<buildId>/cancel
```

Build page: `https://codemagic.io/app/<app id>/build/<buildId>`.

## Workflow shapes worth having

| Workflow | Does |
|---|---|
| `pr-check` | `jvmTest`, compile each platform, coverage table + ratchet, posts the PR comment |
| `<platform>-stg` | builds every app in **one** run (shared build number) → tester distribution |
| `e2e-<app>` | manual: UI flows on a headless emulator → JUnit + screenshots |

## Rules

- Gitignored config files (SDK keys, per-env JSON) are provisioned from env groups as base64 and decoded in a script step.
- Version **name** comes from a `version.properties` bumped by a release script; build number from `$PROJECT_BUILD_NUMBER`.
- A tester link printed by the build may mask the app id; never paste an install link you haven't opened.
- Trigger prod workflows only when the user asks for prod.
