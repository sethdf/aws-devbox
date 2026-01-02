# Personal AI Control Center

## Stack

**Host** - Terraform-provisioned EC2 with Tailscale-only access, LUKS encryption, and Bitwarden for secrets.

**CLIProxyAPI** - Wraps AI CLI tools (Gemini, Claude Code, Codex) and exposes them as standard APIs, enabling programmatic access to paid subscriptions without API keys.

**LiteLLM** - Unified interface to all AI models. Configured via YAML. Routes to direct APIs, cloud providers (Bedrock, Vertex), or CLIProxyAPI based on configuration.

**Skills** - Markdown files containing prompts, model preferences, and workflow definitions. The intelligence layer.

**Scripts** - Simple utilities in ~/bin that load skills and call LiteLLM.

## Access Paths

| Method | Auth | Use Case |
|--------|------|----------|
| Direct API | API keys from Bitwarden | Pay-per-token |
| Bedrock/Vertex | AWS/GCP credentials | Cloud billing |
| CLIProxyAPI | OAuth via CLI tools | Existing subscriptions |

## Unified Multi-Model Access

LiteLLM's batch_completion enables parallel queries to multiple models. CLIProxyAPI exposes subscription-based CLI tools as standard API endpoints. Combined, this means fanout works identically across any model regardless of authentication method.

A single batch_completion call can query Gemini through your Google subscription, Claude through your Anthropic subscription, and GPT-4 through an API key simultaneously. LiteLLM routes each request to the appropriate endpoint. The caller doesn't know or care which auth method each model uses.

This eliminates the need for custom fanout code. Multi-model comparison, racing for fastest response, or ensemble approaches all reduce to LiteLLM configuration plus a single function call.

## Directory Structure

```
~/
├── .config/litellm/config.yaml   # Model routing configuration
├── bin/                          # Simple scripts
├── skills/                       # Prompt templates and workflows
└── log/                          # Daily notes and outputs
```

## Principles

1. No custom servers when a library suffices
2. Configuration in YAML and Markdown, not code
3. Skills define what to do, scripts execute
4. Each component does one thing well
