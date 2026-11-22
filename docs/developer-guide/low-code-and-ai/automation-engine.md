---
title: Automation Engine
description: Extend Automation Flows from an app with the automation_events, automation_actions and automation_relationships hooks, emit() and skip_automations.
version: develop
sources:
  - frappe/automation_engine/__init__.py
  - frappe/automation_engine/events.py
  - frappe/automation_engine/settings.py
  - frappe/automation_engine/dispatch.py
  - frappe/automation_engine/registry.py
  - frappe/automation_engine/runner.py
  - frappe/automation_engine/actions/base.py
  - frappe/automation_engine/actions/core.py
  - frappe/automation_engine/relationships.py
  - frappe/automation_engine/schema_relationships.py
  - frappe/automation_engine/scheduler.py
  - frappe/automation_engine/drainer.py
  - frappe/automation_engine/queue.py
  - frappe/automation_engine/api.py
  - frappe/automation/doctype/automation_flow/automation_flow.json
  - frappe/automation/doctype/automation_action/automation_action.json
  - frappe/automation/doctype/automation_settings/automation_settings.json
  - frappe/model/document.py
  - frappe/hooks.py
---

# Automation Engine

The automation engine runs `Automation Flow` documents. A flow has a trigger and a list of steps. Users build flows in the desk. An app extends the engine with its own events, actions and relationships, so users can build flows about the app's domain. Use this page when your app must add a trigger such as "Order was paid", or a step such as "Add Order Comment". For the flow builder, see [Automation Flow](../../user-guide/automation/automation-flow.md).

## Core concepts

| Name | What it is |
|---|---|
| `Automation Flow` | The flow. Fields include `document_type`, `trigger_type`, `custom_event`, `filters`, `condition`, `relationships`, `actions`, `run_as`, `enabled`. |
| `Automation Action` | One step (child table `actions`). Fields: `step_key`, `step_type` (`Action`, `Wait`, `WaitForEvent`, `If`), `action_type`, `target`, `output_alias`, `params`, `step_condition`. |
| Trigger type | `Doc Created`, `Doc Updated`, `Field Value Changed`, `Doc Deleted`, `Doc Submitted`, `Doc Cancelled`, `Date Based`, `Scheduled`, `Custom Event`, `Manual`. |
| Event | A named domain event that an app emits. It starts `Custom Event` flows and resumes `WaitForEvent` steps. |
| Action | A Python class that does one step. |
| Relationship | A named path from a document to related records. Steps can target the records. |

The public API is in `frappe.automation_engine`:

```python
from frappe.automation_engine import emit, is_enabled, skip_automations
```

## How a flow runs

1. `Document.run_method` calls `frappe.automation_engine.dispatch.run_automations` after document hooks, notifications, webhooks and server scripts.
2. Dispatch matches enabled flows of the DocType. It writes a queue row for each match.
3. After the commit, dispatch enqueues one deduplicated drain job on the `default` queue.
4. The drainer runs each queued flow. Each step runs in a savepoint. A failed step rolls back its own changes.

Dispatch does not run during install, patch or migrate. It also stops at a depth of `max_depth` (default 3) when flows trigger other flows.

Scheduler entries in `frappe/hooks.py`:

| Schedule | Method | Description |
|---|---|---|
| every 5 minutes | `frappe.automation_engine.scheduler.process_cron` | Queues `Scheduled` flows that are due. |
| every 5 minutes | `frappe.automation_engine.drainer.drain_due` | Runs queue rows that are due, such as resumed waits. |
| hourly | `frappe.automation_engine.scheduler.process_date_based` | Queues `Date Based` flows. |
| daily maintenance | `frappe.automation_engine.drainer.purge_queue` | Deletes old queue rows. |

## Add events

Declare event names in the `automation_events` hook. An entry is a name, or a dict of names to a schema.

```python
# sample_app/hooks.py
automation_events = [
	{
		"sample_app.order_paid": {
			"label": "Order was paid",
			"description": "A payment closed the order.",
			"subject": {"doctype": "Sample Order", "name_key": "order"},
			"correlation_options": [{"label": "This order", "value": "{{ doc.name }}"}],
		},
	}
]
```

| Schema key | Description |
|---|---|
| `label` | The label in the builder. Default: made from the name. |
| `description` | Help text in the builder. |
| `subject` | The record the event is about. Use `doctype` and `name_key`, or `doctype_key`, `name_key` and an optional `doctypes` list. The keys name keys of the payload. |
| `correlation_options` | Choices for the correlation key of a `WaitForEvent` step. Each item has `label` and `value` (a Jinja template). |

With a `subject`, the engine runs the flow on the subject record, not on the document that the code passed. The builder offers the event as a trigger on the subject DocTypes.

### Emit an event

```python
# sample_app/payments.py
from frappe.automation_engine import emit, is_enabled


def on_payment_submit(doc, method=None):
	if not is_enabled():
		return
	emit(
		"sample_app.order_paid",
		payload={"order": doc.order, "amount": doc.amount},
		correlation_key=doc.order,
	)
```

`emit(event, doc=None, payload=None, correlation_key=None) -> dict`

| Argument | Description |
|---|---|
| `event` | A registered event name. An unknown name raises `ValidationError`, unless `allow_unregistered_events` is set in `Automation Settings`. |
| `doc` | The document to match flows against, when the event has no `subject`. |
| `payload` | A dict. The engine adds `event_name`. Flows read it as `{{ payload.<key> }}`. Its JSON size must be under `event_payload_limit` (default 65536 bytes). |
| `correlation_key` | Resumes `WaitForEvent` steps that wait on this event with this rendered key. |

`emit` returns `{"queued": <flows queued>, "resumed": <waits resumed>}`. It only writes queue rows. The flows run in the drain job after the commit. Keep payloads to identifiers, because the engine stores them in run logs and subscriptions.

## Add actions

Subclass `AutomationAction` and list the class in the `automation_actions` hook.

```python
# sample_app/automation/actions.py
from typing import ClassVar

from frappe import _
from frappe.automation_engine.actions.base import AutomationAction, AutomationParamError, render_value


class AddOrderComment(AutomationAction):
	action_type = "SampleAddOrderComment"
	label = "Add Order Comment"
	description = "Add a comment to a Sample Order."
	applicable_doctypes: ClassVar[list] = ["Sample Order"]
	output_schema: ClassVar[dict] = {"comment": {"fieldtype": "Data"}}
	params_schema: ClassVar[list] = [
		{"fieldname": "text", "label": "Text", "fieldtype": "Small Text", "reqd": 1, "templatable": 1},
	]

	def validate(self, params, doctype):
		if not params.get("text"):
			raise AutomationParamError(_("Text is required"), fieldname="text")

	def execute(self, doc, params, context):
		comment = doc.add_comment("Comment", render_value(params["text"], doc, context))
		return {"detail": _("Added comment {0}").format(comment.name), "comment": comment.name}
```

```python
# sample_app/hooks.py
automation_actions = ["sample_app.automation.actions.AddOrderComment"]
```

### Class attributes

| Name | Type | Default | Description |
|---|---|---|---|
| `action_type` | `str` | `""` | The unique key stored in the step. Two classes with one key raise an error. |
| `label` | `str` | `""` | The label in the builder. |
| `description` | `str` | `""` | Help text in the builder. |
| `applicable_doctypes` | `list` or `None` | `None` | The DocTypes the action works on. `None` means all. |
| `requires_document` | `bool` | `True` | Set `False` when the action can run without a target document. |
| `transactional` | `bool` | `True` | Set `False` when the action reaches outside the database, such as an HTTP call. The run is then never replayed from the start. |
| `supported_trigger_types` | `list` or `None` | `None` | The trigger types that can use the action. `None` means all. |
| `params_schema` | `list` | `[]` | The parameters, as DocField-like dicts: `fieldname`, `label`, `fieldtype`, `options`, `reqd`. Editor hints: `options_source`, `control`, `link_filters`, `exclusive_with`, `templatable`. |
| `output_schema` | `dict` or `None` | `None` | The output keys that later steps can read. |

### Methods

| Method | Description |
|---|---|
| `validate(params, doctype)` | Check the params when the flow is saved and before each step runs. Raise `AutomationParamError(message, fieldname=...)`. |
| `execute(doc, params, context)` | Do the step on the target `doc`. Return a short string, or a dict with `detail` and output keys. |
| `output_doctype(params)` | The DocType of the record the action makes, when known. Later steps can then target `output_alias`. |

- Use `render_value(value, doc, context)` for a param that can hold Jinja. The template gets `doc`, `target`, `trigger`, `payload` and `context`.
- Before `execute`, the engine checks write permission on the target for the run user.
- Return `{"destination_reference": {"doctype": ..., "name": ...}}` to make the record available under the step's `output_alias`.
- Raise `StopAutomation(message, resume_after=60)` to pause the run. It resumes at the next step after `resume_after` seconds.
- A step output larger than `step_output_limit` (default 65536 bytes) is replaced with `{"truncated": true}`.

The built-in actions are `SetFieldValue`, `IncrementFieldValue`, `CreateDocument`, `SendNotification`, `AssignToUser`, `CallWebhook` and `RunScript`, in `frappe/automation_engine/actions/core.py`.

## Add relationships

The engine derives relationships from the schema: Link, child-table Link and Dynamic Link fields, in both directions. Add a provider only for paths that the schema does not know.

```python
# sample_app/automation/relationships.py
import frappe
from frappe.automation_engine.relationships import AutomationRelationshipProvider


class SampleRelationshipProvider(AutomationRelationshipProvider):
	def get_definitions(self, source_doctype):
		if source_doctype != "Sample Order":
			return []
		return [
			{
				"name": "owner_todos",
				"label": "Open ToDos of the Order Owner",
				"cardinality": "many",
				"target_doctype": "ToDo",
			}
		]

	def resolve(self, source_doc, relationship, params):
		names = frappe.get_all(
			"ToDo",
			filters={"allocated_to": source_doc.owner_user, "status": "Open"},
			pluck="name",
		)
		return [{"doctype": "ToDo", "name": name} for name in names]
```

```python
# sample_app/hooks.py
automation_relationships = ["sample_app.automation.relationships.SampleRelationshipProvider"]
automation_relationship_ignore = ["Sample Log"]
```

| Method | Description |
|---|---|
| `get_definitions(source_doctype)` | Return a list of definitions for the DocType. Each needs `name` and `cardinality` (`one` or `many`). Add `label` and `target_doctype` or `target_doctypes`. |
| `resolve(source_doc, relationship, params)` | Return a list of `{"doctype", "name"}` references. |
| `query(source_doc, relationship, filters, limit)` | Optional. The default resolves all references, then filters them with `frappe.get_list`. Override it for large `many` relationships. |

- A hook entry can name a class or an instance.
- App providers come before the schema provider. An app definition with the name of a derived one replaces it.
- Set `derived_from` to the name of a derived relationship to rename or relabel it.
- Every returned reference must exist, match the allowed target DocTypes, and be readable by the run user. Otherwise the step fails.
- `automation_relationship_ignore` is a list of DocTypes that the schema provider leaves out.

## Stop automations

`is_enabled()` returns `False` when:

- the code runs inside `skip_automations()`,
- `automation_disabled` is set in site config, or
- `disable_automations` is set in `Automation Settings`.

Use `skip_automations()` for bulk writes that must not start flows, such as data fixes and imports:

```python
from frappe.automation_engine import skip_automations

with skip_automations():
	for name in frappe.get_all("Sample Order", pluck="name"):
		frappe.get_doc("Sample Order", name).save()
```

The context manager sets `frappe.flags.skip_automations` and restores the old value after the block.

## Settings

`Automation Settings` fields and defaults:

| Field | Default | Description |
|---|---|---|
| `disable_automations` | 0 | Stop all dispatch. |
| `max_depth` | 3 | The depth limit of flows that trigger flows. |
| `failure_threshold` | 10 | Consecutive failed runs before the engine disables the flow. |
| `stale_running_minutes` | 30 | When a running queue row counts as stale. |
| `max_attempts` | 3 | The retries of a queue row. |
| `drain_seconds` | 0 | The time budget of one drain job. 0 means a part of the queue timeout. |
| `commit_every` | 50 | Rows per commit in the drainer. |
| `queue_retention_days` | 7 | Days to keep finished queue rows. |
| `step_output_limit` | 65536 | The size limit of a step output in bytes. |
| `event_payload_limit` | 65536 | The size limit of an event payload in bytes. |
| `allow_unregistered_events` | 0 | Accept `emit` of names that no hook declares. |

## Pitfalls

- `emit` and dispatch queue work only. The drain job runs after the commit. A rolled-back transaction runs nothing.
- Name events with your app prefix, such as `sample_app.order_paid`. All apps share one event namespace.
- `action_type` must be unique across all installed apps.
- A non-transactional action cannot be undone by a rollback. Set `transactional = False` for it, so a retry does not repeat it.

## Related

- [Automation Flow](../../user-guide/automation/automation-flow.md)
- [Hooks](../building-apps/hooks.md)
- [Hooks reference](../../reference/configuration/hooks.md)
- [Controllers](../data-model/controllers.md)
- [Background Jobs](../runtime/background-jobs.md)
- [Scheduler](../runtime/scheduler.md)
- [Configuration](../operations/configuration.md)
