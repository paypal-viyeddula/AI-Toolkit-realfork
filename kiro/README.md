# PayPal Payments Power

See [plugin.json](./plugin.json) for the power manifest, [mcp.json](./mcp.json) for MCP server configuration, and [skills/paypal-best-practices/SKILL.md](./skills/paypal-best-practices/SKILL.md) for setup, configuration, and usage documentation.

## Install

In Kiro, open the **Powers** panel → **Add Custom Power** → **Import power from GitHub**, and enter `https://github.com/paypal/AI-Toolkit/tree/main/kiro`. Then set `PAYPAL_SANDBOX_ACCESS_TOKEN` on the installed power via **Open powers config** (see the setup note in [SKILL.md](./skills/paypal-best-practices/SKILL.md#using-the-paypal-sandbox-mcp-server)).

## Maintaining

`skills/paypal-best-practices/references/*.md` mirror the root [`skills/paypal-best-practices/references/`](../skills/paypal-best-practices/references/) used by the Claude Code, Codex, and Cursor plugins, minus the per-file frontmatter and agent-specific slash-command references. When you change a root reference, apply the same change here in the same PR.

**Privacy Policy:** [PayPal Privacy Statement](https://www.paypal.com/us/legalhub/paypal/privacy-full)

- Docs: https://developer.paypal.com/docs/
- Repository / issues: https://github.com/paypal/AI-Toolkit

**License:** Apache-2.0
