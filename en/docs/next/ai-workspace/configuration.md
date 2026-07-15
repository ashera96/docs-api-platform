---
title: "Configure AI Workspace"
description: "Learn how AI Workspace's config.toml can pull values from environment variables or mounted secret files, so you can change settings per environment without hardcoding secrets."
canonical_url: https://wso2.com/api-platform/docs/cloud/ai-workspace/configuration/
md_url: https://wso2.com/api-platform/docs/cloud/ai-workspace/configuration.md
tags:
  - cloud
  - ai-workspace
  - configuration
author: WSO2 API Platform Documentation Team
last_updated: 2026-07-15
content_type: "how-to"
---

# Configure AI Workspace

AI Workspace reads all of its settings from a single file, `config.toml`. The `setup.sh` script from [Getting Started](getting-started.md) fills in a working `config.toml` for you, but as you move toward staging or production you'll usually need to change some of these values per environment — for example, using a real domain name instead of `localhost`, or supplying an OIDC client secret without writing it into the file.

Instead of editing `config.toml` every time a value changes, you can write a **placeholder** for a setting and have AI Workspace fill it in automatically when it starts up. This page explains the two kinds of placeholders and when to use each.

## Why not just edit the file directly?

You can always set a value directly in `config.toml`:

```toml
domain = "app.example.com"
```

That works fine for values that don't change often or aren't sensitive. But two situations come up regularly:

- **You deploy the same file to multiple environments** (dev, staging, production) and want each one to pick up its own values automatically, without maintaining a separate `config.toml` per environment.
- **The value is a secret**, like an OIDC client secret, and you don't want it sitting in a config file that might get committed to source control or shared around.

Placeholders solve both: the setting stays visible in `config.toml`, but the actual value is supplied separately, at startup.

## Pulling a value from an environment variable

Use `{{ env "VARIABLE_NAME" "default" }}` to have a setting read from an environment variable instead of a fixed value:

```toml
domain = '{{ env "APIP_AIW_DOMAIN" "localhost:5380" }}'
```

When AI Workspace starts, it looks for an environment variable named `APIP_AIW_DOMAIN`. If that variable is set (for example, in your `docker compose` environment or Kubernetes deployment), its value is used. If it isn't set, AI Workspace falls back to `"localhost:5380"`.

This is the easiest way to change a setting per environment — for example, passing a different domain when you start the container:

```bash
docker run \
  -e APIP_AIW_DOMAIN=app.example.com \
  -v ./configs/config.toml:/etc/ai-workspace/config.toml \
  ghcr.io/wso2/api-platform/ai-workspace:<version>
```

You don't need to touch `config.toml` to do this — only the environment variable changes.

!!! note "Only settings written this way can be overridden"
    A setting only picks up an environment variable if it's written with an `{{ env }}` placeholder in `config.toml`. A plain value like `domain = "localhost:5380"` ignores the environment entirely — there's no hidden override happening in the background.

## Supplying a secret from a mounted file

For sensitive values — most importantly, the OIDC client secret — prefer `{{ file "/path/to/secret" }}` over an environment variable. Environment variables are convenient, but they can leak into process listings or logs; a file mounted from a secret store (a Kubernetes Secret, Docker secret, or similar) is a more secure way to hand over credentials.

```toml
[oidc]
client_secret = '{{ file "/secrets/ai-workspace/oidc_client_secret" }}'
```

At startup, AI Workspace reads the contents of that file and uses it as the value. The secret itself never has to appear in `config.toml`, in an environment variable, or in your shell history.

For local development, an environment variable is usually simpler:

```toml
[oidc]
client_secret = '{{ env "APIP_AIW_OIDC_CLIENT_SECRET" }}'
```

Both forms are valid — use the environment variable while you're developing locally, and switch to the mounted file once you're running in a shared or production environment.

!!! note "Where secret files can live"
    A `{{ file }}` path must point somewhere under `/etc/ai-workspace` or `/secrets/ai-workspace`. This is a safety check to stop a misconfigured path from accidentally reading an unrelated file on the host.

## What happens if a value is missing

Both placeholders **fail closed**: if `config.toml` asks for an environment variable that isn't set, or a file that doesn't exist, AI Workspace refuses to start rather than continuing with a blank or guessed value. This is intentional — it's safer to catch a missing secret at startup than to have the workspace come up silently misconfigured.

If AI Workspace won't start, check the startup logs for a message naming the missing environment variable or file path, then supply it and try again.

## Quick reference

| Placeholder | What it does | Example |
|---|---|---|
| `{{ env "VAR" "default" }}` | Reads `VAR` from the environment; uses `"default"` if `VAR` isn't set | `'{{ env "APIP_AIW_DOMAIN" "localhost:5380" }}'` |
| `{{ env "VAR" }}` | Reads `VAR` from the environment; **required** — startup fails if `VAR` isn't set | `'{{ env "APIP_AIW_OIDC_CLIENT_SECRET" }}'` |
| `{{ file "/path" }}` | Reads the contents of the file at `/path` as the value; startup fails if the file is missing | `'{{ file "/secrets/ai-workspace/oidc_client_secret" }}'` |

## Where to go next

- [Set up Asgardeo as your identity provider](authentication/asgardeo-setup.md) walks through a production OIDC setup that uses these placeholders for the client secret and other environment-specific values.
- [Getting Started](getting-started.md) covers the initial `config.toml` that `setup.sh` generates for you.
