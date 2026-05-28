# ayan — AI code review that reads like a pragmatic senior engineer

ayan is a code-review bot that runs against your local commits, working-tree edits, or GitHub pull requests. It runs multiple parallel analyzers (idiom, slop detection, security, static analysis, pragmatism, project conventions) over a diff and surfaces findings as inline comments and a summary, either on GitHub or as a local markdown transcript.

It's designed to fit two workflows:

1. **Long Claude Code sessions** — invoke from inside Claude Code via the [pragmatic-review](https://github.com/ss-ayan/pragmatic-review) skill plugin. After finishing a chunk of work, ayan reviews the diff and Claude addresses findings.
2. **GitHub PR review** — install as a GitHub App or invoke from CI / locally against a PR URL. Posts inline comments, a summary, and (optionally) an approval.

## Why another reviewer

- **Multi-analyzer, parallel, observable.** Six analyzers run concurrently with a 30-second heartbeat showing in-flight work — no silent hangs.
- **Pragmatism is built in.** A dedicated analyzer flags over-engineering against the PR's stated intent; another catches duplication of existing functions instead of asking you to refactor every "similar" line.
- **Dry-run by default for evaluation.** Try ayan on a real PR without posting a single comment. The DryRunVCS decorator captures every would-be inline / summary / approval to a markdown file.
- **Fast partial reviews.** `--only staticanalysis` finishes in seconds for sanity checks during long plans.
- **Local mode needs no remote.** Review a feature branch with `ayan review --local --base main` before pushing. No GitHub auth, no API key, no config file required.

## Repositories

| Repo | What it is |
|---|---|
| [`ayan`](https://github.com/ss-ayan/ayan) | The bot. Go binary + Docker image. Source for the review engine. |
| [`pragmatic-review`](https://github.com/ss-ayan/pragmatic-review) | Claude Code plugin that teaches Claude when and how to invoke the ayan binary. Four skills (local / pr-dryrun / pr-post / profile). |
| [`service`](https://github.com/ss-ayan/service) | Self-hosted deployment harness — postgres, webhook receiver, GitHub App glue. |
| [`github-app`](https://github.com/ss-ayan/github-app) | The GitHub App definition (permissions, webhook URL, manifest). |

## Install

ayan ships as a Docker image. Pull from `ghcr.io/ss-ayan/ayan` and wrap it with a thin shell shim so `ayan <subcommand>` works seamlessly:

```bash
docker pull ghcr.io/ss-ayan/ayan:latest
docker run --rm ghcr.io/ss-ayan/ayan:latest version    # smoke test

# Wrap so `ayan` runs against your current directory. Pass through the
# API key env vars you'll use (uncomment whichever you've set; ayan
# interpolates `${VAR}` references in ayan.yaml from this env):
sudo tee /usr/local/bin/ayan > /dev/null <<'SH'
#!/bin/bash
exec docker run --rm -i \
  -v "$PWD:/workspace" -w /workspace \
  -e ANTHROPIC_API_KEY \
  -e OPENAI_API_KEY \
  -e OPENROUTER_API_KEY \
  -e GROQ_API_KEY \
  ghcr.io/ss-ayan/ayan:latest "$@"
SH
sudo chmod +x /usr/local/bin/ayan
ayan version
```

Self-hosted deployment? See the `service` repo — its compose/k8s manifests explicitly set `entrypoint: ayan-server` to run the webhook receiver. The image's default is the CLI.

To pin to a specific version, swap `:latest` for `:v4.0.18` (or whichever tag you want).

## Pick an LLM provider

ayan delegates LLM calls to one of these. The first run prompts you via `ayan init`; you can also write `ayan.yaml` by hand from the example yaml in the ayan repo.

| Provider | API key | When it fits |
|---|---|---|
| `anthropic` | `${ANTHROPIC_API_KEY}` (`sk-ant-…`) | You have an Anthropic Console key. Best calibration for the review pipeline. |
| `openai` | `${OPENAI_API_KEY}` (`sk-…`) | OpenAI direct. gpt-4o / gpt-4o-mini. |
| `openrouter` | `${OPENROUTER_API_KEY}` | One key, many models. Pick `anthropic/claude-…` to route to Anthropic via OpenRouter, or any of OpenRouter's catalog. |
| `openai_compatible` | provider-specific | Anything that speaks `/v1/chat/completions`. **Tested**: Groq. **Untested but should work**: Together, Fireworks, Anyscale, Cerebras, DeepSeek, Mistral, vLLM, llama.cpp. |
| `claude_cli` | none — uses your Claude Code subscription | **Native install only.** The Docker image doesn't ship the `claude` binary; the adapter refuses to start inside a container. |
| `ollama` | none | Local inference. From inside Docker your host's `localhost:11434` isn't reachable — use `http://host.docker.internal:11434` on macOS/Windows, the docker bridge IP on Linux (typically `http://172.17.0.1:11434`), or expose Ollama publicly via ngrok / Cloudflare Tunnel. |

For non-Anthropic providers you also pick a **capability tier** (`low` / `medium` / `high`) so the analyzer pipeline calibrates confidence. The example yaml lists tier mappings for common models.

## Install the Claude Code plugin

Optional but recommended — gives Claude Code the slash commands that drive ayan.

### Marketplace (recommended once submitted)

```bash
claude plugin install pragmatic-review
```

### From a clone

```bash
git clone https://github.com/ss-ayan/pragmatic-review.git
claude --plugin-dir ./pragmatic-review
```

After install, Claude Code exposes `/pragmatic-review:local`, `/pragmatic-review:pr-dryrun`, `/pragmatic-review:pr-post`, and `/pragmatic-review:profile` — and invokes them proactively when conversation context fits ("I just finished refactoring this, looks OK?" triggers `local`).

## 30-second smoke test

```bash
cd ~/some-git-repo                           # any git checkout works
ayan review --local --only staticanalysis --output -
```

Expect a markdown transcript on stdout within ~10 seconds. If your repo has uncommitted changes you want to check:

```bash
ayan review --local --working-tree --output -
```

For a GitHub PR (read-only, no comments posted):

```bash
gh auth login   # if not done already
ayan review --dry-run https://github.com/org/repo/pull/123 --output -
```

## Requirements

| What | Why |
|---|---|
| One LLM provider configured in `ayan.yaml` | Run `ayan init` interactively, or copy the example yaml from the [ayan repo](https://github.com/ss-ayan/ayan/blob/main/ayan.example.yaml). |
| For PR modes: `gh auth login` OR `GITHUB_TOKEN` | Read access to the PR; write scope only required if posting comments. |
| (Optional) Per-language toolchains | `gofmt` / `go vet` are stdlib; for Python install `ruff black mypy`; for Ruby `gem install rubocop brakeman bundler-audit`. Static-analysis findings are language-specific so missing toolchains just produce fewer findings, not errors. |

## Pinning binary and plugin together

The `pragmatic-review` plugin declares a minimum `ayan` version in `.claude-plugin/plugin.json`. Each SKILL.md's first step runs `ayan version` and refuses to proceed on a mismatch. Upgrade ayan when the plugin's required version moves forward; the plugin will continue to work with newer binary releases that respect the same flag surface.

## License

(Add when the ayan repo's license is finalized.)
