---
title: Home
description: Technical documentation of Frappe Framework version 16 for app developers, desk users and AI agents.
version: v16
sources:
  - frappe/__init__.py
  - pyproject.toml
---

# Frappe Framework

Frappe Framework is a full-stack web framework in Python and JavaScript. It is metadata driven. You describe data as DocTypes, and Frappe makes the database tables, the REST API, the desk forms, lists and reports, and the permission checks. You put your code in apps. A bench runs many sites, and each site installs a set of apps.

!!! note "Version: 16 (stable)"
    These pages describe version 16 (branch `version-16-hotfix`). Every page was verified against the code of this branch. Other versions: [v15](/frappe/v15/) (stable) and [develop](/frappe/develop/) (unreleased).

## Start here

| If you want to | Read |
|---|---|
| Install Frappe and make a site | [Installation](getting-started/installation.md) |
| Understand how the parts fit | [Architecture](getting-started/architecture.md) |
| Build a first app | [Quickstart](getting-started/quickstart.md) |
| Look up a term | [Glossary](getting-started/glossary.md) |
| See what changed since version 15 | [What's New](changes.md) |
| Move an app from version 15 | [Upgrading](developer-guide/operations/upgrading.md) |

## Sections

- **[User Guide](user-guide/index.md):** use the desk, set up users and permissions, automate work, print, report, publish web pages, and use Flow and Studio.
- **[Developer Guide](developer-guide/index.md):** build apps: DocTypes, controllers, hooks, the database API, REST, background jobs, desk scripts, website pages and tests.
- **[Reference](reference/index.md):** complete lists of hooks, CLI commands, REST endpoints, site config keys, Python and JavaScript APIs, field types and DocTypes.
- **[For AI Agents](AGENTS.md):** rules and facts for agents that write or change Frappe apps.

## Conventions

- Examples use the app `sample_app`, the DocTypes `Sample Item` and `Sample Order`, the role `Sample Role` and the user `user1@example.com`.
- File paths in `frappe/...` are relative to the frappe app repository. Paths in `sample_app/...` are files in your app.
- Commands run from the bench folder. Replace `<site>` with your site name.
