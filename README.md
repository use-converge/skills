# Converge Skills

Public agent skills for Converge.

## Available Skills

- `research-pulse`: run Converge Research Pulse workflows through the supported public CLI and API contract.

## Install

This repository is intended to be consumed by agents or tools that can install skills from a GitHub repository path.

The Research Pulse skill expects the public `converge` CLI to be available. On systems with Homebrew:

```bash
brew tap use-converge/cli
brew trust use-converge/cli
brew install converge
```

Converge is currently in private beta. The CLI is binary-only and requires a Converge API endpoint plus API key or reusable config profile.
