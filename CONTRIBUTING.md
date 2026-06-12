# Contributing to gitbankio/base-plugin

Thanks for your interest in contributing.

## What is this repo

This repo contains the Gitbank plugin for Base MCP. It has two files:

- `gitbank-base-mcp-plugin.md` -- full plugin spec for Base MCP sessions (GET endpoint approach)
- `SKILL.md` -- condensed Base MCP skill spec with frontmatter, endpoint reference, and example flows

## Good contributions

- Improve SKILL.md or gitbank-base-mcp-plugin.md (clearer endpoint descriptions, better examples)
- Correct endpoint params or response shapes if the Gitbank API changes
- Fix typos or unclear wording in README
- Report issues via GitHub Issues (include the AI client and the operation that failed)

## Workflow

1. Open an issue describing the change before opening a PR
2. Fork the repo, make changes on a branch
3. Open a PR with a clear description of what changed and why

## Rules

- No em dash in any file
- No internal tooling mentions
- Endpoint descriptions in SKILL.md must match the live behavior of `https://gitbank.io/api/public`
- All examples must use realistic amounts and valid token symbols (USDC, WETH)

## License

By contributing, you agree that your contributions will be licensed under the Apache-2.0 License.
