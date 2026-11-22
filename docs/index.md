---
title: Home
description: Technical documentation of Frappe Framework develop (17.0.0-dev) for app developers, desk users and AI agents.
version: develop
sources:
  - frappe/__init__.py
  - pyproject.toml
---

# Frappe Framework

Frappe Framework is a full-stack web framework in Python and JavaScript. It is metadata driven. You describe data as DocTypes, and Frappe makes the database tables, the REST API, the desk forms, lists and reports, and the permission checks. You put your code in apps. A bench runs many sites, and each site installs a set of apps.

!!! warning "Version: develop (unreleased)"
    These pages describe the `develop` branch (17.0.0-dev). Every page was verified against the code of this branch. For a release, read the [v16 docs](/frappe/v16/) or the [v15 docs](/frappe/v15/).

## Start here

| If you want to | Read |
|---|---|
| Install Frappe and make a site | [Installation](getting-started/installation.md) |
| Understand how the parts fit | [Architecture](getting-started/architecture.md) |
| Build a first app | [Quickstart](getting-started/quickstart.md) |
| Look up a term | [Glossary](getting-started/glossary.md) |
| See what changed since version 16 | [What's New](changes.md) |

## Sections

- **[User Guide](user-guide/index.md):** use the desk, set up users and permissions, automate work, print, report and publish web pages.
- **[Developer Guide](developer-guide/index.md):** build apps: DocTypes, controllers, hooks, the database API, REST, background jobs, desk scripts, website pages and tests.
- **[Reference](reference/index.md):** complete lists of hooks, CLI commands, REST endpoints, site config keys, Python and JavaScript APIs, field types and DocTypes.
- **[For AI Agents](AGENTS.md):** rules and facts for agents that write or change Frappe apps.

## Conventions

- Examples use the app `sample_app`, the DocTypes `Sample Item` and `Sample Order`, the role `Sample Role` and the user `user1@example.com`.
- File paths in `frappe/...` are relative to the frappe app repository. Paths in `sample_app/...` are files in your app.
- Commands run from the bench folder. Replace `<site>` with your site name.
