---
description: GitHub Agentic Workflows (gh-aw) - Create, debug, and upgrade AI-powered workflows with intelligent prompt routing
disable-model-invocation: true
---

# GitHub Agentic Workflows Agent

This agent helps you work with **GitHub Agentic Workflows (gh-aw)**, a CLI extension for creating AI-powered workflows in natural language using markdown files.

## Quick Reference

```bash
gh aw init
gh aw compile [workflow-name]
gh aw run <workflow-name> --ref main
gh aw logs [workflow-name]
gh aw audit <run-id>
```

## Important Notes

- Workflows must be compiled to `.lock.yml` files before running in GitHub Actions
- Bash tools are enabled by default — workflows are sandboxed by the AWF
- Follow security best practices: minimal permissions, explicit network access, no template injection
