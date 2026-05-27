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

### Option 1 — Binary

Pick the tarball for your platform from [Releases](https://github.com/ss-ayan/ayan/releases/latest):

```bash
VERSION=v4.0.15
OS=$(uname -s | tr A-Z a-z)
ARCH=$(uname -m | sed 's/x86_64/amd64/')
curl -L "https://github.com/ss-ayan/ayan/releases/download/${VERSION}/ayan-${VERSION}-${OS}-${ARCH}.tar.gz" \
  | tar -xz -C /tmp
sudo install /tmp/ayan /usr/local/bin/ayan
ayan version
```

Windows users: download the `.zip` from the Releases page and add `ayan.exe` to `PATH`.

### Option 2 — Docker

```bash
docker pull ghcr.io/ss-ayan/ayan:latest
docker run --rm ghcr.io/ss-ayan/ayan:latest version

# Wrap it so `ayan` works seamlessly against your current dir:
cat > /usr/local/bin/ayan <<'SH'
#!/bin/bash
exec docker run --rm -i \
  -v "$PWD:/workspace" -w /workspace \
  -v "$HOME/.claude:/root/.claude" \
  --entrypoint ayan ghcr.io/ss-ayan/ayan:latest "$@"
SH
chmod +x /usr/local/bin/ayan
```

The `--entrypoint ayan` override is required because the image's default entrypoint runs the webhook server (`ayan-server`); the CLI binary lives at `/usr/local/bin/ayan` inside the image.

### Option 3 — `go install` (if you have Go 1.25+)

```bash
go install github.com/ss-ayan/ayan/cmd/cli@latest
ayan version
```

Requires read access to the repo (clone via SSH if the repo is private to you).

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
| The `claude` CLI from Claude Code, signed in | ayan delegates LLM calls to your existing Claude Code subscription via a subprocess; no separate API key needed |
| For PR modes: `gh auth login` OR `GITHUB_TOKEN` | Read access to the PR; write scope only required if posting comments |
| (Optional) Per-language toolchains | `gofmt` / `go vet` are stdlib; for Python install `ruff black mypy`; for Ruby `gem install rubocop brakeman bundler-audit`. Static-analysis findings are language-specific so missing toolchains just produce fewer findings, not errors. |

## Pinning binary and plugin together

The `pragmatic-review` plugin declares a minimum `ayan` version in `.claude-plugin/plugin.json`. Each SKILL.md's first step runs `ayan version` and refuses to proceed on a mismatch. Upgrade ayan when the plugin's required version moves forward; the plugin will continue to work with newer binary releases that respect the same flag surface.

## License

(Add when the ayan repo's license is finalized.)
