---
title: Flow Development
description: Ship Flow AI agents, tools and triggers from an app with frappe.flow.Agent, @tool, Trigger and the flow_agents, flow_tools and flow_triggers hooks.
version: develop
sources:
  - frappe/flow/__init__.py
  - frappe/flow/lib/agent.py
  - frappe/flow/lib/tool.py
  - frappe/flow/lib/trigger.py
  - frappe/flow/lib/model.py
  - frappe/flow/lib/session.py
  - frappe/flow/lib/resolver.py
  - frappe/flow/triggers/triggers.py
  - frappe/flow/daemon/callback.py
  - frappe/flow/daemon/lifecycle.py
  - frappe/flow/commands/__init__.py
  - frappe/flow/tools/builtins.py
  - frappe/flow/knowledge/knowledge.py
  - frappe/flow/doctype/flow_agent/flow_agent.py
  - frappe/flow/doctype/flow_session/flow_session.py
  - frappe/flow/doctype/flow_skill/flow_skill.py
  - frappe/flow/assistant/assistant.py
  - frappe/hooks.py
  - hooks.md
---

# Flow Development

Flow is the AI agent system of Frappe. An app can ship its own agents, tools and triggers in Python code. Flow writes them to the site as locked `Flow Agent`, `Flow Tool` and `Flow Trigger` documents after `bench migrate` and after app installs. Use this page when your app must give users an agent that knows your DocTypes, or must run an agent when a document changes. For the desk side of Flow (chat, models, approvals), see [Flow](../../user-guide/ai-and-app-builder/flow.md).

## Core concepts

| Name | What it is |
|---|---|
| Flow Model | A chat model and its credentials. An agent uses one Flow Model, by its title. |
| Flow Agent | Instructions, a model, tools, skills, knowledge bases and sub-agents. |
| Flow Tool | A function the model can call. Types: `Script`, `Imported`, `Server`. App tools are `Imported`. |
| Flow Trigger | Runs an agent on a document event or on a cron schedule, with a prompt template. |
| Flow Session | One conversation with an agent. |
| Flow Run | One turn of a session. Fields include `status`, `input`, `output`, `error`, `usage`. |
| Flow server | A bench-wide daemon that runs the model loop. It calls tools back on the site. |

The public API is in `frappe.flow`:

```python
from frappe.flow import Agent, Tool, ToolContext, Trigger, build_schema, is_enabled, tool
from frappe.flow.knowledge import Knowledge
```

Naming rules:

- An app agent is named `<app>/<name>`, for example `sample_app/Order Assistant`.
- An app tool is the Flow Tool `<app>/<function name>`. The model calls it `<app>__<function name>`, for example `sample_app__open_orders`.
- An app trigger is named `<app>/<title>`.
- Flow takes `<app>` from the module that constructs the `Agent` or `Trigger`. It uses the first part of the module's `__name__`.

## Write a tool

Decorate a module-level function with `@tool`. The function must take a keyword-only `ctx: ToolContext` argument. Without it, `@tool` raises `TypeError` at import.

```python
# sample_app/ai/tools.py
from typing import Annotated, Literal

import frappe
from frappe.flow import ToolContext, tool


@tool
def open_orders(
	customer_name: Annotated[str, "Customer name, as on the order"],
	limit: int = 20,
	*,
	ctx: ToolContext,
) -> list[dict]:
	"""List the open Sample Orders of one customer."""
	return frappe.get_list(
		"Sample Order",
		filters={"customer_name": customer_name, "status": "Open"},
		fields=["name", "order_date", "total"],
		limit=limit,
	)


@tool(
	requires_confirmation=True,
	confirm_prompt=lambda args: f"Mark {args['name']} as {args['status']}",
)
def set_order_status(
	name: str,
	status: Literal["Open", "Completed", "Cancelled"],
	*,
	ctx: ToolContext,
) -> str:
	"""Change the status of one Sample Order."""
	doc = frappe.get_doc("Sample Order", name)
	doc.status = status
	doc.save()
	return f"{name} is now {status}"
```

`@tool` makes the JSON schema of the arguments from the type hints (`build_schema`):

- `str`, `int`, `float`, `bool`, `list[...]`, `dict`, `Literal[...]`, `Enum` and Pydantic models map to JSON schema types.
- `Annotated[str, "text"]` adds a description to the argument.
- An argument with a default, or typed `X | None`, is not required.
- `ctx`, `self` and `cls` are left out of the schema.
- The docstring is the tool description, unless you pass `description`.

The schema of `set_order_status` is:

```json
{"type": "object", "properties": {"name": {"type": "string"}, "status": {"type": "string", "enum": ["Open", "Completed", "Cancelled"]}}, "additionalProperties": false, "required": ["name", "status"]}
```

When a tool is called, Pydantic validates the arguments against the type hints. A wrong value raises a validation error. The return value goes to the model as text. A `str` goes as is. Other values go as JSON.

### `@tool` options

| Name | Type | Default | Description |
|---|---|---|---|
| `name` | `str` | function name | The tool name. Use it when two tools in one app have the same function name. |
| `description` | `str` | docstring | What the model reads about the tool. |
| `requires_confirmation` | `bool` or `callable(args) -> bool` | `False` | Pause for approval before the call. A callable decides per call. |
| `confirm_prompt` | `callable(args) -> str` | `None` | The text of the approval card. Without it, the card shows the arguments as JSON. |
| `background` | `bool` or `callable(args) -> bool` | `False` | Run the call as a job on the `long` queue. The timeout is `flow_background_tool_timeout` in site config, else 1500 seconds. |
| `needs_user` | `bool` | `False` | The approval card takes input from the user, so no reviewer can answer it for them. |
| `display_template` | `str` | `None` | A template of the call for the transcript. It can name only argument names. Other names raise `ValueError`. |

### `ToolContext`

Flow passes `ctx` to every call. The model never sees it.

| Field | Type | Description |
|---|---|---|
| `session_id` | `str` | The `Flow Session` name. |
| `agent` | `str` | The `Flow Agent` name that called the tool. |
| `user` | `str` | The session owner. The call runs as this user. |
| `result_token` | `str` or `None` | The token of a call whose result comes later. |
| `read_only` | `bool` | `True` when the call runs unapproved in a read-only transaction. |
| `failure` | `Exception` or `None` | A failed write that was undone, or a stopped call, in this call. |

## Write an agent

Return a list of `Agent` objects from a function. Constructing an `Agent` does not touch the database.

```python
# sample_app/ai/agents.py
from frappe.flow import Agent

from sample_app.ai.tools import open_orders, set_order_status


def get_agents() -> list[Agent]:
	return [
		Agent(
			name="Order Assistant",
			model="Default Model",
			description="Answers questions about Sample Orders.",
			instructions="You help the sales team with Sample Orders. Use the tools.",
			tools=[open_orders, set_order_status],
			max_iterations=20,
		)
	]
```

### `Agent` arguments

All arguments are keyword-only.

| Name | Type | Default | Description |
|---|---|---|---|
| `name` | `str` | required | The agent name. The document is `<app>/<name>`. |
| `model` | `str` | required | The title of a `Flow Model`. Flow sets it only when it creates the document, or when the current model is gone. |
| `instructions` | `str` | required | The system instructions. |
| `description` | `str` | `""` | A short description. |
| `tools` | `list[Tool]` | `None` | `@tool` functions. Flow writes each as an `Imported` Flow Tool. |
| `knowledge` | `Knowledge` or `list[Knowledge]` | `None` | Flow Knowledge Bases the agent can search. |
| `skills` | `list[str]` | `None` | Flow Skill names. A name without an app prefix is this app's skill if it ships one, else the site's skill of that name. |
| `sub_agents` | `list[Agent]` | `None` | Agents this agent can hand a task to. Flow syncs them first. |
| `max_iterations` | `int` | `100` | The iteration limit of one run. |
| `auto_approve` | `bool` | `False` | Sessions started with `Agent.new_session()` or `Agent.run()` skip approvals. |
| `public` | `bool` | `False` | Open the agent to guests through the guest API. Their chats run tools without approval. |

!!! warning
    `Knowledge("<title>")` gets or inserts a `Flow Knowledge Base` when you construct it. Construct it inside the hook function, not at module import.

### Skills an app ships

Flow reads `<app repo>/skills/<name>/SKILL.md` of every installed app. It writes each file as a locked `Flow Skill` named `<app>/<name>`. For example, the frappe repo ships `skills/studio-app-building/SKILL.md` as `frappe/studio-app-building`. The built-in `frappe/Flow` assistant gets every app skill.

## Write a trigger

A trigger runs an agent on a document event or on a schedule. Pass `doctype` and `doc_event`, or pass `cron`. Do not pass both.

```python
# sample_app/ai/agents.py
from frappe.flow import Trigger


def get_triggers() -> list[Trigger]:
	return [
		Trigger(
			title="Review New Order",
			agent="Order Assistant",
			doctype="Sample Order",
			doc_event="after_insert",
			condition="doc.total > 1000",
			prompt="Check Sample Order {{ doc.name }} for {{ doc.customer_name }} and add a comment.",
			run_as="user1@example.com",
		),
		Trigger(
			title="Daily Open Orders",
			agent="Order Assistant",
			cron="0 9 * * *",
			prompt="Summarize the open Sample Orders as of {{ now }}.",
		),
	]
```

### `Trigger` arguments

| Name | Type | Default | Description |
|---|---|---|---|
| `title` | `str` | required | The trigger name. The document is `<app>/<title>`. |
| `agent` | `str` | required | An agent of this app by name, or another app's agent as `<app>/<name>`. |
| `prompt` | `str` | required | A Jinja template. It gets `doc` (the document, or `None`) and `now`. It renders in a sandbox without database access. |
| `doctype` | `str` | `None` | The DocType to watch. |
| `doc_event` | `str` | `None` | One of `after_insert`, `on_update`, `on_submit`, `on_cancel`, `on_trash`. |
| `cron` | `str` | `None` | A cron expression. The scheduler checks it every 5 minutes. |
| `condition` | `str` | `None` | One Python expression. It gets `doc`, `utils` and `len`. The trigger runs only when it is true. |
| `run_as` | `str` | `"Administrator"` | The email or ID of the user that runs the agent. |

How a trigger runs:

- A document event enqueues `frappe.flow.triggers.fire` after the commit. It skips events during install and migrate.
- The trigger skips a document that `run_as` cannot read. Field-level read permissions apply to `doc` in the prompt.
- Trigger sessions use the `Auto-approve` approval mode. Tools run without approval.
- A synced trigger starts enabled. The desk can only switch it on or off.

!!! warning
    A trigger runs as Administrator when you leave out `run_as`. Make a user with only the roles that the trigger needs, and set `run_as` to that user.

## Register them in hooks

```python
# sample_app/hooks.py
flow_agents = ["sample_app.ai.agents.get_agents"]
flow_triggers = ["sample_app.ai.agents.get_triggers"]
flow_tools = ["sample_app.ai.tools.open_orders"]
```

| Hook | Shape | Read by | Description |
|---|---|---|---|
| `flow_agents` | list of dotted paths | `frappe.flow.lib.agent.sync_agents` | Each path names a function that returns a list of `Agent`. |
| `flow_tools` | list of dotted paths | `frappe.flow.lib.agent.sync_tools` | Each path names a `@tool` object. Flow writes it as a locked Flow Tool that any agent can use. |
| `flow_triggers` | list of dotted paths | `frappe.flow.lib.trigger.sync_triggers` | Each path names a function that returns a list of `Trigger`. |

You do not need to list an agent's tools in `flow_tools`. `Agent.sync()` writes them. Use `flow_tools` for tools that site admins can add to their own agents.

Sync behavior:

- Frappe runs the three sync functions in `after_migrate` and `after_app_install`. It also syncs triggers in `after_app_uninstall`.
- Synced documents have `is_system_generated` set. The desk can enable or disable an agent and change its model. It cannot change anything else.
- An entry that fails is skipped and logged in Error Log. The other entries still sync.
- A trigger whose agent does not exist yet is not written. A later sync writes it.
- Flow deletes system-generated triggers that no hook declares anymore. Their Flow Runs stay.
- If a site already has a Flow Agent with the same name that code does not own, the sync of that agent fails with "Agent Conflict".

## Run an agent from code

Get the synced `Flow Agent` document and call `run()`. It starts a new session on the flow server and returns the `Flow Run` with status `Running`. It does not wait for the result.

```python
# sample_app/ai/jobs.py
import frappe


def review_order(order: str):
	agent = frappe.get_doc("Flow Agent", "sample_app/Order Assistant")
	agent.run(
		f"Review Sample Order {order}.",
		reference_doctype="Sample Order",
		reference_name=order,
		on_complete="sample_app.ai.jobs.on_review_done",
	)


def on_review_done(run):
	if run.status == "Completed":
		frappe.get_doc("Sample Order", run.reference_name).add_comment("Comment", run.output)
```

- `on_complete` is a dotted path. Flow calls it with the `Flow Run` once the run ends with `Completed`, `Denied` or `Failed`.
- Flow calls `on_complete` in a request of the session owner, inside a savepoint. An exception rolls it back and goes to Error Log.
- `new_session()` commits the database transaction, so the flow server can find the session. Do not call `run()` in the middle of a write that must stay atomic. Call it from a background job.
- The `Agent` object also has `new_session(title=None)` and `run(input, on_complete=None)`. They need the synced document.

`frappe.flow.is_enabled()` returns `True` when the site has an enabled chat model with a key or an address, or with an enabled provider. Use it to hide AI features when Flow cannot chat. It does not check the access of the user.

## How tool calls run

1. The flow server sends the call to `frappe.flow.daemon.callback.callback` on the site.
2. The callback checks the `X-Harness-Token` header against `flow_server_secret` in site config.
3. It sets the user to the session owner. The tool runs with the permissions of that user.
4. If the tool raises an exception, the whole transaction rolls back. The model gets the error text, cut to 500 characters.

The model sees only the enabled tools of the agent. When server scripts are disabled, Flow leaves out `Script` tools and the built-in `frappe/execute` tool. A session keeps the tools that it started with. Changes to an agent apply to new sessions only.

## The flow server

The flow server is the `flow` program in the bench environment. One server serves all sites of the bench.

| Command | Description |
|---|---|
| `bench flow-server setup` | Give every site a `flow_server_secret`, start the server, and sync server tools. |
| `bench flow-server start` | Start the server if it does not answer. |
| `bench flow-server stop` | Stop the server. |
| `bench flow-server restart` | Restart the server. Use it to load new extensions. |
| `bench flow-server status` | Show if the server answers and which sites it serves. |
| `bench flow-server serve` | Run the server in the foreground, for supervisor. |

Install and migrate also give the site its secret and start the server. A server that cannot start does not fail the install or the migrate.

| Config key | File | Description |
|---|---|---|
| `flow_server_secret` | `site_config.json` | The shared secret of the callbacks. Flow writes it. |
| `flow_server_manager` | `common_site_config.json` | `local` (default) or `supervisor`. With `supervisor`, `stop` and `restart` refuse to run. Use supervisorctl. |
| `flow_server_url` | `common_site_config.json` | The base URL for callbacks. Default: `http://127.0.0.1:<webserver_port>`. |
| `flow_redis_url` | `common_site_config.json` | The Redis for session events. Default: `redis_queue`. |
| `flow_background_tool_timeout` | `site_config.json` | The timeout of background tools in seconds. Default: 1500. |

Paths on the bench: the socket is `flow-server/flow.sock`, the log is `logs/flow-server.log`, and sessions of a site are in `<site>/private/flow/`.

## Pitfalls

- Construct `Agent` and `Trigger` objects in a module of your app. Flow takes the app name from the calling module. In a bench console, the name is wrong.
- Export each tool as a module-level attribute of an installed app. The callback imports it by its dotted path. Otherwise the sync fails with "Invalid Tool".
- Two tools of one app with the same function name fail with "Tool Conflict". Give one a `name=`.
- `model` must be the title of an existing `Flow Model`. Otherwise the agent fails to sync and is logged.
- The desk can change the model of a synced agent. Flow keeps that choice on later syncs.

## Related

- [Flow](../../user-guide/ai-and-app-builder/flow.md)
- [Hooks](../building-apps/hooks.md)
- [Hooks reference](../../reference/configuration/hooks.md)
- [Background Jobs](../runtime/background-jobs.md)
- [Scheduler](../runtime/scheduler.md)
- [Server and Client Scripts](server-and-client-scripts.md)
- [Configuration](../operations/configuration.md)
- [MCP and Discovery](../api/mcp-and-discovery.md)
