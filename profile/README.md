# ayan

AI code review for local changes and GitHub PRs. Parallel analyzers check for idiom, project conventions, slop, over-engineering, security, and static-analysis issues, then report findings as inline comments or a markdown transcript. Supports Go, Python, and Ruby.

| Repo | |
|---|---|
| [`ayan`](https://github.com/ss-ayan/ayan) | The bot: CLI, webhook server, Docker image |
| [`pragmatic-review`](https://github.com/ss-ayan/pragmatic-review) | Claude Code plugin that drives the CLI |
| [`service`](https://github.com/ss-ayan/service) | Self-hosted server deployment (Docker) |
| [`github-app`](https://github.com/ss-ayan/github-app) | GitHub App manifest |

## Install

```bash
sudo tee /usr/local/bin/ayan > /dev/null <<'SH'
#!/bin/bash
exec docker run --rm -i -v "$PWD:/workspace" -w /workspace \
  -e ANTHROPIC_API_KEY -e OPENAI_API_KEY -e OPENROUTER_API_KEY -e GROQ_API_KEY -e GITHUB_TOKEN \
  ghcr.io/ss-ayan/ayan:latest "$@"
SH
sudo chmod +x /usr/local/bin/ayan
ayan init        # writes ayan.yaml: provider, models, capability tier
```

Pin a version with `:vX.Y.Z` instead of `:latest`.

## Use

```bash
ayan review --local --working-tree --output -            # uncommitted edits
ayan review --local --base main --output -               # branch vs main
ayan review --local --only staticanalysis --output -     # lint only, ~10s
ayan review --dry-run https://github.com/org/repo/pull/1 # PR, nothing posted
ayan review https://github.com/org/repo/pull/1           # PR, posts comments
```

PR modes need `GITHUB_TOKEN`. Write scope is only needed for posting.

## Providers

Set in `ayan.yaml`: `anthropic`, `openai`, `openrouter`, `openai_compatible` (tested with Groq), `ollama`, or `claude_cli`. `claude_cli` uses your Claude Code subscription and only works with a native install, not Docker. To reach Ollama from Docker, use `host.docker.internal:11434`. Non-Anthropic providers also need `capability.tier: low|medium|high`.

## Claude Code plugin

```bash
git clone https://github.com/ss-ayan/pragmatic-review.git
claude --plugin-dir ./pragmatic-review
```

Adds `/pragmatic-review:local`, `:pr-dryrun`, `:pr-post`, and `:profile`.

Optional static-analysis toolchains: `ruff black mypy` for Python; `rubocop brakeman bundler-audit` for Ruby.
