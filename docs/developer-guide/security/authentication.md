---
title: Authentication
description: Authenticate HTTP requests with a session cookie, API key and secret, or an OAuth 2 bearer token, and how CSRF, two-factor login, guests and auth hooks work.
version: develop
sources:
  - frappe/auth.py
  - frappe/app.py
  - frappe/sessions.py
  - frappe/handler.py
  - frappe/oauth.py
  - frappe/integrations/oauth2.py
  - frappe/twofactor.py
  - frappe/core/doctype/user/user.py
  - frappe/core/doctype/user/user.json
  - frappe/__init__.py
---

# Authentication

Each request to a Frappe site runs as one user. `frappe.session.user` is that user, or `Guest` if the request is not authenticated. Frappe sets the user from one of these, in `frappe/auth.py`:

| Method | Header or cookie | Use it for |
|---|---|---|
| Session | `sid` cookie | Browsers, the desk and portal pages |
| API key and secret | `Authorization: token <api_key>:<api_secret>` | Server-to-server scripts and integrations |
| OAuth 2 | `Authorization: Bearer <access_token>` | Third-party apps that act for a user |
| Auth hooks | anything | Custom schemes in your app |

All requests on this page were sent with curl to `http://frappe.localhost:8000`.

## Session login

```bash
curl -s -c cookies.txt -X POST "http://frappe.localhost:8000/api/method/login" \
  -H "Content-Type: application/json" \
  -d '{"usr": "user1@example.com", "pwd": "Sample-Pass-123!"}'
```

```json
{"message": "Logged In", "home_page": "desk", "full_name": "User 1"}
```

The response sets the `sid` cookie (`HttpOnly`, `SameSite=Lax`) and the cookies `system_user`, `full_name`, `user_id` and `user_lang`. Send the cookie with later requests:

```bash
curl -s -b cookies.txt "http://frappe.localhost:8000/api/method/frappe.auth.get_logged_user"
# {"message":"user1@example.com"}
```

`usr` is the email. It can also be the user name or the mobile number when System Settings `allow_login_using_user_name` or `allow_login_using_mobile_number` is set. Log out with `POST /api/method/logout`.

Failed logins return HTTP 401 and are written to the Activity Log. Repeated failures lock the user and the IP for a time (`LoginAttemptTracker` in `frappe/auth.py`). The login also checks the IP allowlist of the user (`restrict_ip`) and the login hours (`login_after`, `login_before`).

The login runs the `before_login` and `on_login` hooks. Logout runs `on_logout`. Each handler gets `login_manager`. See [Hooks](../building-apps/hooks.md).

### CSRF

For a cookie session, `POST`, `PUT`, `DELETE` and `PATCH` requests must send the CSRF token of the session in the `X-Frappe-CSRF-Token` header (or as `csrf_token` in the form). Otherwise the request fails with `frappe.CSRFTokenError` (HTTP 400):

```json
{"exception": "frappe.exceptions.CSRFTokenError: Invalid Request", "exc_type": "CSRFTokenError"}
```

- The desk gets the token in `frappe.csrf_token` and `frappe.call` sends it for you.
- A session gets a token the first time it is needed (`frappe.sessions.get_csrf_token`). A session that has no token yet is not checked.
- Requests with token or bearer authentication have no session token, so the check does not apply.
- Requests from an origin in the site config key `allowed_referrers` skip the check.

## API key and secret

### Generate keys

A System Manager generates the keys for a user:

1. Open the User in the desk.
2. Open the **API Access** section.
3. Click **Generate Keys**.
4. Copy the API Secret from the dialog. It is shown only once.

From code or the API, call `frappe.core.doctype.user.user.generate_keys`. It requires the `System Manager` role and `POST`:

```bash
curl -s -b admin_cookies.txt -X POST \
  "http://frappe.localhost:8000/api/method/frappe.core.doctype.user.user.generate_keys" \
  -H "Content-Type: application/json" -d '{"user": "user1@example.com"}'
```

```json
{"message": {"api_key": "1a2b3c4d5e6f7a8", "api_secret": "9f8e7d6c5b4a3f2"}}
```

The API key is created once and does not change. Each call makes a new secret and the old secret stops working. The secret is stored encrypted in the `api_secret` Password field of the User.

### Use the keys

```bash
curl -s "http://frappe.localhost:8000/api/method/frappe.auth.get_logged_user" \
  -H "Authorization: token 1a2b3c4d5e6f7a8:9f8e7d6c5b4a3f2"
# {"message":"user1@example.com"}
```

HTTP Basic with `base64(api_key:api_secret)` also works:

```bash
curl -s "http://frappe.localhost:8000/api/method/frappe.auth.get_logged_user" \
  -H "Authorization: Basic $(printf '1a2b3c4d5e6f7a8:9f8e7d6c5b4a3f2' | base64)"
# {"message":"user1@example.com"}
```

A wrong key or secret returns HTTP 401 with `"exc_type": "AuthenticationError"`. The failure is written to the Activity Log, at most once per interval for each user.

- The request runs with the roles and permissions of that user. There is no separate scope.
- The user must be enabled.
- The `restrict_ip` allowlist of the user also applies.
- No session is created and no cookie is set.
- Two-factor authentication does not apply to token requests.

The header `Frappe-Authorization-Source: <DocType>` makes Frappe look up the key in another DocType instead of User. That DocType needs the fields `api_key`, `api_secret`, `enabled` and `user`. The request then runs as the value of `user`.

## OAuth 2

Frappe is an OAuth 2 and OpenID Connect provider. A third-party app gets an access token for a user and sends it as a bearer token. Set up the provider and clients in the desk. See [Integrations](../../user-guide/website-and-integrations/integrations.md).

| Endpoint | Path |
|---|---|
| Authorization | `/api/method/frappe.integrations.oauth2.authorize` |
| Token | `/api/method/frappe.integrations.oauth2.get_token` |
| Revocation | `/api/method/frappe.integrations.oauth2.revoke_token` |
| Introspection | `/api/method/frappe.integrations.oauth2.introspect_token` |
| User info (OpenID) | `/api/method/frappe.integrations.oauth2.openid_profile` |

The paths are in `ENDPOINTS` in `frappe/integrations/oauth2.py`.

### Authorization code flow

The example uses an OAuth Client with `client_id` `<client_id>`, scopes `all openid`, redirect URI `http://localhost:9999/callback`, grant type `Authorization Code` and response type `Code`.

1. Send the user to the authorization endpoint. A guest is redirected to `/login` first.

    ```bash
    curl -s -i -b user_cookies.txt -G "http://frappe.localhost:8000/api/method/frappe.integrations.oauth2.authorize" \
      --data-urlencode client_id=<client_id> --data-urlencode response_type=code \
      --data-urlencode scope=all --data-urlencode redirect_uri=http://localhost:9999/callback \
      --data-urlencode state=xyz
    ```

    ```http
    HTTP/1.1 302 FOUND
    Location: http://localhost:9999/callback?code=<authorization_code>&state=xyz
    ```

    The user sees a Confirm Access page first, unless the client has **Skip Authorization** set.

2. Exchange the code for tokens. Authenticate the client with HTTP Basic (`client_id:client_secret`) for the method `Client Secret Basic`.

    ```bash
    curl -s -X POST "http://frappe.localhost:8000/api/method/frappe.integrations.oauth2.get_token" \
      -u "<client_id>:<client_secret>" \
      -d grant_type=authorization_code -d code=<authorization_code> \
      -d redirect_uri=http://localhost:9999/callback
    ```

    ```json
    {"access_token": "<access_token>", "expires_in": 3600, "token_type": "Bearer",
     "scope": "all", "refresh_token": "<refresh_token>"}
    ```

3. Call the API with the access token.

    ```bash
    curl -s "http://frappe.localhost:8000/api/method/frappe.auth.get_logged_user" \
      -H "Authorization: Bearer <access_token>"
    # {"message":"user1@example.com"}
    ```

4. When the token expires, get a new one with the refresh token. The old access token stops working.

    ```bash
    curl -s -X POST "http://frappe.localhost:8000/api/method/frappe.integrations.oauth2.get_token" \
      -u "<client_id>:<client_secret>" -d grant_type=refresh_token -d refresh_token=<refresh_token>
    ```

    ```json
    {"access_token": "<new_access_token>", "expires_in": 3600, "token_type": "Bearer",
     "scope": "all", "refresh_token": "<new_refresh_token>"}
    ```

5. To end access, revoke the token. The endpoint always returns HTTP 200 with `{}`.

    ```bash
    curl -s -X POST "http://frappe.localhost:8000/api/method/frappe.integrations.oauth2.revoke_token" \
      -u "<client_id>:<client_secret>" -d token=<new_access_token>
    ```

Tokens are stored as `OAuth Bearer Token` documents with a hash of the token. An unknown, expired or revoked token returns HTTP 401. A token of a disabled user raises `frappe.AuthenticationError`. The request runs with the roles of the user who approved the token.

## Two-factor authentication

When 2FA applies to a user (System Settings `enable_two_factor_auth` and a role with **Two Factor Authentication** set), `POST /api/method/login` does not log in at once:

1. The first login request checks the password. The response has `tmp_id` and a `verification` object with the `method` (`OTP App`, `Email` or `SMS`). The user is still `Guest`.
2. Send the code: `POST /api/method/login` with `tmp_id` and `otp`. On success the session is created.

The `tmp_id` expires after 300 seconds for SMS and Email, and after 180 seconds for an OTP App that is set up. The logic is in `frappe/twofactor.py`.

## Guests

A request without credentials runs as `Guest`. The only role of `Guest` is `Guest`.

- A guest can call only methods with `@frappe.whitelist(allow_guest=True)`. Other methods return HTTP 403 with "Function ... is not whitelisted".
- A guest can read documents only through permissions given to the `Guest` role.
- Guest string arguments are sanitized for HTML unless the method sets `xss_safe=True`.

See [Whitelisted methods](../api/whitelisted-methods.md).

## Custom authentication

The `auth_hooks` hook lists functions that run on every request after the built-in checks. A function can call `frappe.set_user(user)` to authenticate the request.

```python
# sample_app/hooks.py
auth_hooks = ["sample_app.auth.validate_sample_header"]
```

```python
# sample_app/auth.py
import frappe


def validate_sample_header():
    token = frappe.get_request_header("X-Sample-Token")
    if not token:
        return
    user = frappe.db.get_value("Sample Device", {"token": token, "enabled": 1}, "user")
    if user:
        form_dict = frappe.local.form_dict
        frappe.set_user(user)
        frappe.local.form_dict = form_dict
```

`frappe.set_user` resets `frappe.local.form_dict`. Save it before the call and put it back after, as the built-in token and OAuth checks in `frappe/auth.py` do. Without this, the request loses its arguments.

If a request has an `Authorization` header with two parts and no method sets a user, Frappe raises `frappe.AuthenticationError` (HTTP 401).

## Agent attribution

A request with a token or a bearer token can send `X-Frappe-Agent: <name>`. Frappe records the name as the agent that acts for the user, and shows it in the Version log of changed documents. Cookie sessions ignore the header.

## Pitfalls

- An API secret gives full access as that user. Create a separate user with only the roles the integration needs.
- `generate_keys` replaces the secret. Integrations that use the old secret stop working.
- Do not send API keys in URLs. Use the `Authorization` header.
- A cookie session that calls `POST` from a script must send `X-Frappe-CSRF-Token`.

## Related

- [REST API](../api/rest-api.md)
- [Whitelisted methods](../api/whitelisted-methods.md)
- [Permissions](permissions.md)
- [Users and roles](../../user-guide/users-and-permissions/users-and-roles.md)
- [Integrations](../../user-guide/website-and-integrations/integrations.md)
- [Security](security.md)
- [Configuration](../operations/configuration.md)
- [Request lifecycle](../api/request-lifecycle.md)
