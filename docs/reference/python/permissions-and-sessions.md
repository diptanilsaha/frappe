---
title: Permissions and Sessions API
description: Signatures and behavior of frappe.permissions, frappe.share, frappe.sessions, frappe.auth, User helpers, frappe.utils.user and frappe.defaults.
version: v15
sources:
  - frappe/permissions.py
  - frappe/share.py
  - frappe/sessions.py
  - frappe/auth.py
  - frappe/core/doctype/user/user.py
  - frappe/core/doctype/user_permission/user_permission.py
  - frappe/utils/user.py
  - frappe/defaults.py
  - frappe/__init__.py
---

# Permissions and Sessions API

This page lists the public functions and classes of the permission, sharing, session, login, user and defaults modules. Use it to find an exact signature, default or side effect. For the permission model (roles, permlevels, User Permissions, `has_permission` hooks), read [Permissions](../../developer-guide/security/permissions.md). For login, API keys and OAuth, read [Authentication](../../developer-guide/security/authentication.md).

The wrappers on the `frappe` namespace (`frappe.has_permission`, `frappe.get_roles`, `frappe.only_for`, `frappe.set_user`, `frappe.get_user`) are in [frappe namespace](frappe.md).

**Whitelisted** means the function is callable at `/api/method/<dotted.path>`. See [REST Endpoints](../http/rest-endpoints.md). "any" means the default set: `GET`, `POST`, `PUT`, `DELETE`.

## frappe.permissions: constants

| Name | Value |
|---|---|
| `rights` | `("select", "read", "write", "create", "delete", "submit", "cancel", "amend", "print", "email", "report", "import", "export", "share")` |
| `GUEST_ROLE` | `"Guest"` |
| `ALL_USER_ROLE` | `"All"`. Website users have it too. |
| `SYSTEM_USER_ROLE` | `"Desk User"` |
| `ADMIN_ROLE` | `"Administrator"` |
| `AUTOMATIC_ROLES` | `(GUEST_ROLE, ALL_USER_ROLE, SYSTEM_USER_ROLE, ADMIN_ROLE)`. Frappe gives these by user type. They are not stored in `Has Role`. |

## frappe.permissions: checks

### has_permission

```python
frappe.permissions.has_permission(
	doctype, ptype="read", doc=None, user=None, raise_exception=True, *,
	parent_doctype=None, debug=False, ignore_share_permissions=False,
) -> bool
```

Return `True` if `user` (default: session user) has `ptype` on the DocType, or on `doc` when given. This function does not raise, also with `raise_exception=True`. `frappe.has_permission(..., throw=True)` is the raising wrapper.

The checks run in this order:

1. `Administrator` gets `True`.
2. `ptype="share"` gets `False` when System Settings `disable_document_sharing` is set.
3. If `doctype` is a document, it is used as `doc`.
4. A child DocType is checked on its parent with `has_child_permission`.
5. With `doc` (a document, or a name that is loaded with `frappe.get_doc`), `get_doc_permissions` decides. Without `doc`, role permissions at permlevel 0 decide. `submit` on a non-submittable DocType and `import` on a DocType without `allow_import` get `False`.
6. If still denied and `ignore_share_permissions` is not set, a share (`DocShare`) can grant `read`, `write`, `share`, `submit`, `email` and `print`. `email` and `print` are checked as `read` shares. Without `doc`, one shared document of the DocType is enough.
7. `select` falls back to `read`.

`raise_exception` only controls the denial message. When it is true and the user checks their own access, the reasons for a denial are sent with `frappe.msgprint`. With `debug=True`, the steps are collected in `frappe.local.permission_debug_log`.

### Other checks

| Function | Description |
|---|---|
| `get_doc_permissions(doc, user=None, ptype=None, debug=False)` | Return the evaluated rights for a document, for example `{"read": 1, "write": 0, ...}`. Runs `has_controller_permissions` first. If a hook denies, returns `{ptype: 0}`. Applies `if_owner` rules and User Permissions. Sets `submit` and `import` to 0 when the DocType does not allow them. |
| `get_role_permissions(doctype_meta, user=None, is_owner=None, debug=False)` | Return rights from role rules at permlevel 0, with an `if_owner` sub-dict and `has_if_owner_enabled`. `doctype_meta` can be a DocType name. Administrator gets `allow_everything()`. Cached per request in `frappe.local.role_permissions`. |
| `has_user_permission(doc, user=None, debug=False, ptype=None)` | Return `False` if User Permissions block the document, or a Link field value in the document or its child rows. Applies `apply_strict_user_permissions` from System Settings, except for Single DocTypes. |
| `has_controller_permissions(doc, ptype, user=None, debug=False) -> bool` | Call the `has_permission` hooks for the DocType and `"*"`, last registered first. The first hook that returns a value other than `None` decides. Returns `True` when every hook returns `None`. |
| `has_child_permission(child_doctype, ptype="read", child_doc=None, user=None, raise_exception=True, parent_doctype=None, *, debug=False) -> bool` | Check a child DocType on its parent. Needs `parent_doctype` or a `child_doc` with `parenttype`. Checks that the table field exists and its permlevel is accessible. |
| `check_doctype_permission(doctype: str, ptype: str = "read") -> None` | Raise `frappe.PermissionError` if the user has no `ptype` on the DocType. Ignores shares. Sets `frappe.flags.disable_traceback`. |
| `handle_does_not_exist_error(fn)` | Decorator for exception handlers. Turns a `DoesNotExistError` into a `PermissionError` when the user cannot read the DocType, so a missing record does not leak. |
| `is_system_user(user: str \| None = None) -> bool` | Return `True` if `user_type` is `System User`. Reads the cached User value. |
| `push_perm_check_log(log, debug=False)` | Add a message to the denial log that `has_permission` prints. |
| `print_has_permission_check_logs(func)` | Decorator used on `has_permission`. Prints the denial log with `msgprint`. |

## frappe.permissions: roles and rules

| Function | Description |
|---|---|
| `get_roles(user=None, with_standard=True)` | Return the roles of a user. Guest gets `["Guest"]`. Administrator gets every Role. Others get their `Has Role` rows plus `All`, `Guest`, and `Desk User` for System Users. Cached in the Redis hash `roles`. `with_standard=False` removes `AUTOMATIC_ROLES`. |
| `get_doctype_roles(doctype, access_type="read")` | Return the roles that have `access_type` in the DocType meta permissions. |
| `get_valid_perms(doctype=None, user=None)` | Return the permlevel 0 rules for the roles of the user. Uses `Custom DocPerm` for DocTypes that have it, else `DocPerm`. |
| `get_all_perms(role)` | Return all rules of one role, with `Custom DocPerm` replacing `DocPerm` per DocType. |
| `get_perms_for(roles, perm_doctype="DocPerm")` | Return permlevel 0 rules of `perm_doctype` for the roles. |
| `get_doctypes_with_read(user=None)` | Return DocTypes with a `read` rule for the user. |
| `get_doctypes_with_custom_docperms()` | Return DocTypes that have `Custom DocPerm` rows. |
| `allow_everything()` | Return every right of `rights` set to 1. |
| `get_linked_doctypes(dt: str) -> list` | Return `dt` and the targets of its Link fields that do not ignore User Permissions. |
| `get_doc_name(doc)` | Return the name of a document, or the string itself. |

## frappe.permissions: change rules

These functions write `Custom DocPerm` rows. They do not commit.

| Function | Description |
|---|---|
| `add_permission(doctype, role, permlevel=0, ptype=None)` | Add a rule with `ptype` (default `read`) set. Copies `DocPerm` to `Custom DocPerm` first if needed. Shows an alert and returns `None` if the rule exists. Returns the new row name. |
| `update_permission_property(doctype, role, permlevel, ptype, value=None, validate=True, if_owner=0)` | Set one right on an existing rule. Returns `True` if custom rules were set up in this call. |
| `setup_custom_perms(parent)` | Copy `DocPerm` to `Custom DocPerm` if the DocType has no custom rules. Returns `True` when it copies. |
| `copy_perms(parent)` | Copy all `DocPerm` rows of a DocType to `Custom DocPerm`. |
| `reset_perms(doctype)` | Delete all `Custom DocPerm` rows of the DocType, so `DocPerm` applies again. |

## User Permissions

| Function | Description |
|---|---|
| `frappe.permissions.add_user_permission(doctype, name, user, ignore_permissions=False, applicable_for=None, is_default=0, hide_descendants=0)` | Insert a `User Permission` if it does not exist. Raises `DoesNotExistError` if the record `name` does not exist. |
| `frappe.permissions.remove_user_permission(doctype, name, user, ignore_permissions=False)` | Delete the matching `User Permission`. |
| `frappe.permissions.clear_user_permissions_for_doctype(doctype, user=None)` | Delete all User Permissions for `allow=doctype`, for one user or all users. |
| `frappe.permissions.get_user_permissions(user)` | Same as `frappe.core.doctype.user_permission.user_permission.get_user_permissions`. |
| `frappe.permissions.get_allowed_docs_for_doctype(user_permissions, doctype)` | From a list of rules, return the allowed values that apply to `doctype`. |
| `frappe.permissions.filter_allowed_docs_for_doctype(user_permissions, doctype, with_default_doc=True)` | Same, and return `(allowed, default)` when `with_default_doc` is true. |
| `frappe.permissions.can_import(doctype, raise_exception=False)` | `True` for System Manager or `import` right. |
| `frappe.permissions.can_export(doctype, raise_exception=False, is_owner=False)` | `True` for System Manager or `export` right (also through `if_owner`). |
| `frappe.core.doctype.user_permission.user_permission.get_user_permissions(user: str \| None = None)` | Whitelisted (any). Return `{allow_doctype: [{"doc", "applicable_for", "is_default", "hide_descendants"}, ...]}`. Adds descendants of tree records unless `hide_descendants`. Empty for Administrator and Guest. Cached in the Redis hash `user_permissions`. Over HTTP it also accepts `user` for another user (see the pitfalls). |

## frappe.share

Sharing gives one user, or everyone (`everyone=1`), rights on one document through a `DocShare` row. See [Permissions](../../developer-guide/security/permissions.md).

| Function | Description |
|---|---|
| `add(doctype: str, name: str \| int, user: str \| None = None, read=1, write=0, submit=0, share=0, everyone=0, notify=0, **kwargs)` | Whitelisted (any). Checks that User Permissions do not hide `user` from the caller, then calls `add_docshare`. Extra `kwargs` are ignored. |
| `add_docshare(doctype, name, user=None, read=1, write=0, submit=0, share=0, everyone=0, flags=None, notify=0)` | Make or update the share and return the `DocShare`. `read` is always set to 1. Checks `check_share_permission` unless `flags={"ignore_share_permission": True}`. Saves with `ignore_permissions`. With `notify=1`, sends a Notification Log. Makes the user follow the document if `follow_shared_documents` is set on the User. `user` defaults to the session user. |
| `remove(doctype, name, user, flags=None)` | Delete the share of `user` on the document, if any. |
| `set_permission(doctype: str, name: str \| int, user: str \| None, permission_to: str, value=1, everyone=0)` | Whitelisted (any). Checks User Permissions on `user` when `value` is set, then calls `set_docshare_permission`. |
| `set_docshare_permission(doctype, name, user, permission_to, value=1, everyone=0, flags=None)` | Set one right on a share. Makes the share if needed. Setting `read` to 0 clears all rights. A share with no rights left is deleted and `None` is returned. |
| `get_users(doctype: str, name: str) -> list` | Whitelisted (any). Return all `DocShare` rows of the document. Returns `[]` if the caller cannot read the document. |
| `get_shared(doctype, user=None, rights=None, *, filters=None, limit=None)` | Return names of documents shared with the user (or with everyone, except for Guest). `rights` defaults to `["read"]`. |
| `get_shared_doctypes(user=None)` | Return DocTypes that have a share for the user or for everyone. |
| `get_share_name(doctype, name, user, everyone)` | Return the `DocShare` name, or `None`. |
| `check_share_permission(doctype, name, permissions=None)` | Raise `PermissionError` if the caller has no `share` right, or tries to grant `read`, `write` or `submit` that the caller does not have. |
| `validate_user_access(user: str \| None, label: str)` | Raise `LinkValidationError` ("Could not find") if User Permissions on `User` hide `user`. |
| `notify_assignment(shared_by, doctype, doc_name, everyone, notify=0)` | Queue a "Share" Notification Log for the user. Does nothing for `everyone` or when `notify` is 0. |

```python
import frappe.share

frappe.share.add_docshare("Sample Order", "SO-00002", "user1@example.com", write=1, notify=1)
frappe.share.get_shared("Sample Order", "user1@example.com")  # ["SO-00002"]
frappe.share.remove("Sample Order", "SO-00002", "user1@example.com")
```

## frappe.sessions

A session is a row in `tabSessions` and an entry in the Redis hash `session`. Both use the raw sid as key. The client cookie `sid` holds the same value. All guests share the sid `"Guest"`.

| Function | Description |
|---|---|
| `clear()` | Whitelisted (any). Extend the session, commit and clear the user cache. Sets `message` to "Cache Cleared". |
| `clear_sessions(user=None, keep_current=False, force=False)` | Delete sessions of the user. Without `force`, keeps the newest `simultaneous_sessions` sessions. |
| `get_sessions_to_clear(user=None, keep_current=False, force=False)` | Return the sids that `clear_sessions` deletes (max 100). |
| `delete_session(sid=None, user=None, reason="Session Expired")` | Delete one session from the table and the cache. Adds a logout entry to the Activity Log. Commits. Does nothing in read-only mode. |
| `clear_all_sessions(reason=None)` | Log out all users. Only Administrator can call it. |
| `get_expired_sessions()` | Return sids with `lastupdate` older than the expiry. |
| `clear_expired_sessions()` | Delete expired sessions. Runs from the scheduler. |
| `get()` | Return the desk boot info (`bootinfo`). Uses the Redis hash `bootinfo` unless `disable_session_cache` is set. Runs `extend_bootinfo` hooks. |
| `get_boot_assets_json()` | Whitelisted (any). Return the assets map. |
| `get_csrf_token()` | Return the CSRF token of the session. Makes one if missing. |
| `generate_csrf_token()` | Make a new CSRF token and save the session (not in tests). |
| `get_expiry_period()` | Return the global default `session_expiry` as `HH:MM:SS`. Default `240:00:00`. |
| `get_expiry_in_seconds(expiry=None)` | Convert an expiry string to seconds. |
| `get_expired_threshold()` | Return the datetime before which sessions are expired. |
| `get_expiry_period_for_query()` | Return the expiry string on Postgres, seconds on other databases. |
| `get_geo_from_ip(ip_addr)` | Look up the IP with `geolite2`. Returns `None` if the package is not installed or the lookup fails. |
| `get_geo_ip_country(ip_addr)` | `country` of `get_geo_from_ip`, or `None`. |

### Session

```python
frappe.sessions.Session(
	user: str, resume: bool = False, full_name: str | None = None, user_type: str | None = None,
	session_end: str | None = None, audit_user: str | None = None,
)
```

Reads the sid from `form_dict.sid` or the `sid` cookie. With `resume=True`, loads the session. Else validates that the user is enabled and starts a new session. Sets `frappe.local.session`.

| Member | Description |
|---|---|
| `sid`, `user`, `user_type`, `full_name`, `data` | Session values. `data.data` holds the stored session data. |
| `start(session_end=None, audit_user=None)` | Make a sid, insert the `Sessions` row, update `last_login`, `last_ip`, `last_active`. Commits. |
| `resume()` | Load the session from the cache or the database. Checks the IP restriction. Falls back to Guest when the session is missing or expired. |
| `update(force=False)` | Write `last_updated`, `lang` and the session data to the database at most every `min(expiry/2, 600)` seconds, or always with `force`. Also sets `last_active` on the User. Commits. Returns `True` if it wrote. |
| `set_impersonated(original_user)` | Store `impersonated_by` and save the session. |
| `validate_user()`, `insert_session_record()`, `get_session_record()`, `get_session_data()`, `get_session_data_from_cache()`, `get_session_data_from_db()`, `start_as_guest()` | Steps used by `start` and `resume`. |

## frappe.auth

| Name | Description |
|---|---|
| `SAFE_HTTP_METHODS` | `frozenset({"GET", "HEAD", "OPTIONS"})`. No CSRF check. |
| `UNSAFE_HTTP_METHODS` | `frozenset({"POST", "PUT", "DELETE", "PATCH"})` |
| `MAX_PASSWORD_SIZE` | `512`. Longer passwords fail at login. |
| `API_AUTH_FAILURE_LOG_INTERVAL` | `300` seconds. |
| `get_logged_user()` | Whitelisted (any). Return `frappe.session.user`. |
| `clear_cookies()` | Clear `frappe.session.sid` and mark the cookies `full_name`, `user_id`, `sid`, `user_image`, `system_user` for deletion. |
| `validate_ip_address(user)` | Raise `AuthenticationError` if the request IP does not start with a value in the user's `restrict_ip`. Skipped for `/api/method/frappe.realtime.*` and when the 2FA bypass settings apply. |
| `validate_auth()` | Authenticate the request from the `Authorization` header: OAuth bearer token, then API key (`token` or `basic`), then `auth_hooks`. Raises `AuthenticationError` if a two-part header did not log a user in. Checks IP restrictions for a user logged in this way. |
| `validate_oauth(authorization_header)` | Set the user from a valid OAuth bearer token. |
| `validate_auth_via_api_keys(authorization_header)` | Set the user from `token <key>:<secret>` or `basic <base64>`. Raises `InvalidAuthorizationToken` for bad base64. |
| `validate_api_key_secret(api_key, api_secret, frappe_authorization_source=None)` | Check the key and secret on an enabled `User`, or on the DocType named in the `Frappe-Authorization-Source` header. |
| `reject_api_key_auth(reason: str, user: str \| None = None) -> NoReturn` | Write a failed Activity Log (at most one per user each 5 minutes) and raise `AuthenticationError`. |
| `validate_auth_via_hooks()` | Call every `auth_hooks` function. |
| `get_login_attempt_tracker(key: str, raise_locked_exception: bool = True)` | Return a tracker set from System Settings. Raises `SecurityException` if the key is locked. |

### LoginManager

`frappe.local.login_manager` is a `LoginManager`. When the request path is `/api/method/login`, or `form_dict.cmd` is `login`, its constructor calls `login()`. Other requests resume the session.

| Method | Description |
|---|---|
| `login()` | Reject if System Settings `disable_user_pass_login` is set. Authenticate `usr`/`pwd`, force a password reset when it is due, run 2FA, then `post_login()`. |
| `post_login(session_end=None, audit_user=None)` | Run `on_login` hooks, check IP and login hours, make the session, set cookies and `home_page`. A System User without a default workspace goes to `/app`. |
| `authenticate(user=None, pwd=None)` | Find the user by email, mobile or username (System Settings), check the password and track failures. Calls `fail()` on error. |
| `check_password(user, pwd)` | Return the user name in the stored case. Calls `fail()` on a wrong password. |
| `fail(message, user=None)` | Write a failed Activity Log, commit, raise `AuthenticationError`. |
| `login_as(user, session_end=None, audit_user=None)` | Start a session for `user` without a password. |
| `login_as_guest()` | `login_as("Guest")`. |
| `impersonate(user)` | Log in as `user` and mark the session as impersonated by the current user. |
| `logout(arg="", user=None)` | Run `on_logout` hooks. For the current user: delete the session, clear cookies. For another user: delete their sessions. |
| `run_trigger(event="on_login")` | Call the hook functions for `event` with `login_manager=self`. |
| `validate_hour()` | Raise `AuthenticationError` outside the user's `login_after` and `login_before` hours. |
| `make_session(resume=False, session_end=None, audit_user=None)` | Make a `Session`. Runs `on_session_creation` hooks for a new session. |
| `clear_active_sessions()` | Clear other sessions when `deny_multiple_sessions` is set in site config or System Settings. |
| `force_user_to_reset_password()` | Return `True` when `force_user_to_reset_password` days have passed since the last reset. |
| `get_user_info()`, `set_user_info(resume=False)`, `setup_boot_cache()`, `clear_preferred_language()`, `clear_cookies()` | Steps of the login flow. |

### Other classes

| Class | Description |
|---|---|
| `HTTPRequest()` | Made for each request. Sets `request_ip` (from `X-Forwarded-For` first, except under `bench start`), the cookie and login managers and the language. Checks the CSRF token (`X-Frappe-CSRF-Token` header or `csrf_token` form value) for `UNSAFE_HTTP_METHODS` unless `ignore_csrf` is set. |
| `CookieManager()` | `set_cookie(key, value, expires=None, secure=False, httponly=False, samesite="Lax", max_age=None)`, `delete_cookie(to_delete)`, `flush_cookies(response)`, `init_cookies()`. `secure` becomes true on HTTPS. |
| `LoginAttemptTracker(key: str, max_consecutive_login_attempts: int = 3, lock_interval: int = 300, *, user_name=None)` | Count failures in the Redis hashes `login_failed_count` and `login_failed_time`. `add_failure_attempt()`, `add_success_attempt()`, `is_user_allowed() -> bool`, properties `login_failed_count` and `login_failed_time`. `user_name` is deprecated. |

## User DocType

`frappe.core.doctype.user.user`. Controller hooks (`validate`, `on_update` and others) are left out.

### User methods

| Method | Description |
|---|---|
| `append_roles(*roles)` | Add `Has Role` rows that are missing. Does not save. |
| `add_roles(*roles)` | `append_roles`, then `save()`. |
| `remove_roles(*roles)` | Remove the roles, then `save()`. |
| `get_fullname()` | Return `first_name last_name`. |
| `has_desk_access()` | Return the count of the user's roles with `desk_access`. |
| `get_blocked_modules()` | Return the modules in `block_modules`. |
| `get_restricted_ip_list()` | Return the `restrict_ip` values as a list, or `None`. |
| `get_social_login_userid(provider: str)` | Return the `userid` for the provider. |
| `set_social_login_userid(provider, userid, username=None)` | Append a `social_logins` row. Does not save. |
| `find_by_credentials(user_name, password, validate_password=True)` | Class method. Find the user by email, and by mobile number or username when System Settings allow. |
| `send_welcome_mail_to_user()` | Send the welcome email with a password set link. |
| `password_reset_mail(link)` | Send the password reset email now. |
| `populate_role_profile_roles()` | Replace the roles with the roles of the Role Profile in `role_profile_name`. Not whitelisted. |

### Module functions

| Function | Description |
|---|---|
| `get_all_roles()` | Whitelisted (any). Return enabled roles for active domains, without `AUTOMATIC_ROLES`. No permission check. |
| `get_roles(arg=None)` | Whitelisted (any). Return the roles of `form_dict.uid`, or of the session user. Needs read on that User. |
| `get_current_user_roles() -> list[str]` | Whitelisted (any). Return `frappe.get_roles()`. |
| `get_perm_info(role: str)` | Whitelisted (any). Return `get_all_perms(role)`. No permission check. |
| `get_timezones()` | Whitelisted (any). Return `{"timezones": [...]}`. |
| `update_password(new_password: str, logout_all_sessions: int = 0, key=None, old_password=None)` | Whitelisted (POST, guest). Needs a reset `key` or the `old_password`. Returns HTTP 410 with a message for an expired or used key. Logs the user in and returns a redirect path (`/app` for System Users). |
| `test_password_strength(new_password: str, key=None, old_password=None, user_data=None)` | Whitelisted (any, guest). Return `{"score", "feedback", ...}` when the password policy is on, else `{}`. `key` and `old_password` are deprecated. |
| `reset_password(user: str) -> None` | Whitelisted (POST, guest). Rate limit from `get_password_reset_limit` each hour. Send a reset link. Shows the same message whether the user exists or not. |
| `verify_password(password: str)` | Whitelisted (POST). Check the password of the session user. Raises `AuthenticationError` if wrong. |
| `sign_up(email: str, full_name: str, redirect_to: str) -> tuple[int, str]` | Whitelisted (any, guest). Make a Website User. Limited by System Settings `max_signups_allowed_per_hour` (default 300). |
| `generate_keys(user: str)` | Whitelisted (POST). System Manager only. Make a new API secret (and key if missing). Returns both. |
| `impersonate(user: str, reason: str)` | Whitelisted (POST). Administrator only. Writes an Activity Log, notifies the user (also by email if an outgoing account exists), then logs in as the user. |
| `switch_theme(theme: str)` | Whitelisted (any). Set `desk_theme` to `Dark`, `Light` or `Automatic`. |
| `has_email_account(email: str)`, `get_email_awaiting(user: str)`, `get_role_profile(role_profile: str)`, `get_module_profile(module_profile: str)` | Whitelisted (any). Helpers for the User form. |
| `user_query(doctype, txt, searchfield, start, page_len, filters)` | Whitelisted (any). Link search for enabled users. |
| `get_system_users(exclude_users=None, limit=None)` | Enabled users that are not Website Users, without Administrator and Guest. |
| `get_enabled_users()` | Names of enabled users. Cached in Redis under `enabled_users`. |
| `get_total_users()`, `get_active_users()`, `get_website_users()`, `get_active_website_users()` | Counts. "Active" means `last_active` in the last 72 hours. `get_total_users` sums `simultaneous_sessions` of enabled System Users. |
| `create_contact(user, ignore_links=False, ignore_mandatory=False)` | Make or update the Contact of a user. |
| `reset_user_data(user)` | Clear `reset_password_key` and `redirect_url` and save. Returns `(user_doc, redirect_url)`. |
| `get_restricted_ip_list(user)` | Same as the User method. |

## frappe.utils.user

| Function | Description |
|---|---|
| `UserPermissions(name="")` | The object behind `frappe.get_user()`. Lists `can_read`, `can_write`, `can_create`, `can_submit`, `can_cancel`, `can_delete`, `can_select`, `can_search`, `can_get_report`, `can_import`, `can_export`, `can_print`, `can_email`, `allow_modules`, `in_create`, `all_read` after `build_permissions()`. Methods: `get_roles()`, `get_defaults()`, `get_can_read()`, `load_user()`, `get_all_reports()`. |
| `get_user_fullname(user: str) -> str` | `first_name last_name` from the database, or `""`. |
| `get_fullname_and_avatar(user: str) -> _dict` | `{"fullname", "avatar", "name"}`. Raises `TypeError` for a missing user. |
| `get_system_managers(only_name: bool = False) -> list[str]` | Enabled System Managers, newest first, without Administrator and Guest. Default format `"Full Name <email>"`. |
| `add_role(user: str, role: str) -> None` | `frappe.get_doc("User", user).add_roles(role)`. Saves. |
| `add_system_manager(email, first_name=None, last_name=None, send_welcome_email=False, password=None)` | Insert a System User with all roles except `AUTOMATIC_ROLES`. |
| `get_enabled_system_users() -> list[dict]` | `email`, `language`, `name` of enabled System Users. |
| `is_website_user(username=None)` | `True` if `user_type` is `Website User`. |
| `is_system_user(username=None)` | Return the name of an enabled System User whose **email** matches, else `None`. Different from `frappe.permissions.is_system_user`. |
| `get_users() -> list[dict]` | `full_name`, `email`, `is_system_manager` for system users. |
| `get_users_with_role(role: str) -> list[str]` | Enabled users with the role, without Administrator. |

## frappe.defaults

Defaults are `DefaultValue` rows keyed by `parent`: a user name, `__default` (global) or `__global`. A key can hold more than one value. Then the read value is a list. Reads are cached in the Redis hash `defaults`, one entry per parent.

| Function | Description |
|---|---|
| `get_defaults(user=None)` | Global defaults updated with the user's defaults, plus `user` and `owner`. Default user is the session user. |
| `get_defaults_for(parent="__default")` | All defaults of one parent. |
| `get_user_default(key, user=None)` | The first value for `key` from `get_defaults(user)`. For a DocType key (for example `"Company"`), User Permissions decide the value. Returns `None` if User Permissions do not allow the value. |
| `get_user_default_as_list(key, user=None)` | All values for `key`, filtered by User Permissions of the session user. |
| `set_user_default(key, value, user=None, parenttype=None)` | Replace the value for the user. |
| `add_user_default(key, value, user=None, parenttype=None)` | Add one more value for the user. |
| `clear_user_default(key, user=None)` | Delete the key for the user. |
| `get_global_default(key)` | The first value from `get_defaults()`. A session user's own value wins over the global one. |
| `set_global_default(key, value)`, `add_global_default(key, value)` | Same as the user forms, with parent `__default`. |
| `set_default(key, value, parent, parenttype="__default")` | Replace the value. `value=None` deletes it. Locks the row with `FOR UPDATE`. Clears the defaults cache. |
| `add_default(key, value, parent, parenttype=None)` | Insert a row. Clears the cache of the parent (`frappe.clear_cache`). |
| `clear_default(key=None, value=None, parent=None, name=None, parenttype=None)` | Delete rows that match. Raises `Exception` when no filter is given. |
| `get_user_permission_default(key, defaults)` | The User Permission of the session user marked `is_default`, or the one that matches the stored default. |
| `is_a_user_permission_key(key)` | `True` when `key` has no `:` and is not in snake case. |
| `not_in_user_permission(key, value, user=None)` | `True` when User Permissions exist for `key` and `value` is not one of them. |
| `get_user_permissions(user=None)` | Kept for old code. Same as the User Permission function. |

```python
import frappe.defaults

frappe.defaults.set_user_default("sample_key", "A", "user1@example.com")
frappe.defaults.add_user_default("sample_key", "B", "user1@example.com")
frappe.defaults.get_user_default("sample_key", "user1@example.com")          # "A"
frappe.defaults.get_user_default_as_list("sample_key", "user1@example.com")  # ["A", "B"]
```

## Pitfalls

- `frappe.permissions.has_permission` does not raise, also with `raise_exception=True`. Use `frappe.has_permission(..., throw=True)`.
- `has_permission` hooks cannot grant a right that roles do not give. A hook that returns `None` has no effect. The first hook (last registered) that returns `True` or `False` stops the other hooks.
- `frappe.utils.user.is_system_user` and `frappe.permissions.is_system_user` return different things.
- `delete_session`, `Session.start`, `Session.update` and `sessions.clear` commit the transaction.
- `get_roles` is cached in Redis. After you change `Has Role` rows with SQL, call `frappe.clear_cache(user=...)`.
- Over HTTP, `get_user_permissions` replaces `user` with the session user only when `form_dict.cmd` is `get_user_permissions`. At `/api/method/frappe.core.doctype.user_permission.user_permission.get_user_permissions?user=<other>`, a logged-in user gets the User Permissions of another user.
- `get_user_default(key, user)` with a DocType key reads the User Permission default of the session user, not of `user`.

## Related

- [Permissions](../../developer-guide/security/permissions.md)
- [Authentication](../../developer-guide/security/authentication.md)
- [Security](../../developer-guide/security/security.md)
- [frappe namespace](frappe.md)
- [Document](document.md)
