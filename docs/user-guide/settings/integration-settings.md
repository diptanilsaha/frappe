---
title: Integration Settings
description: Field reference for OAuth Provider Settings, LDAP Settings, Push Notification Settings, Google Settings, SMS Settings and the Dropbox, Google Drive and S3 backup settings, with where Frappe reads each field.
version: v15
sources:
  - frappe/integrations/doctype/oauth_provider_settings/oauth_provider_settings.json
  - frappe/integrations/doctype/oauth_provider_settings/oauth_provider_settings.py
  - frappe/integrations/oauth2.py
  - frappe/integrations/doctype/ldap_settings/ldap_settings.json
  - frappe/integrations/doctype/ldap_settings/ldap_settings.py
  - frappe/integrations/doctype/ldap_group_mapping/ldap_group_mapping.json
  - frappe/www/login.py
  - frappe/www/login.html
  - frappe/templates/includes/login/login.js
  - frappe/core/doctype/user/user.js
  - frappe/core/doctype/system_settings/system_settings.py
  - frappe/integrations/doctype/push_notification_settings/push_notification_settings.json
  - frappe/integrations/doctype/push_notification_settings/push_notification_settings.py
  - frappe/push_notification.py
  - frappe/integrations/doctype/google_settings/google_settings.json
  - frappe/integrations/doctype/google_settings/google_settings.py
  - frappe/integrations/google_oauth.py
  - frappe/integrations/doctype/google_calendar/google_calendar.py
  - frappe/integrations/doctype/google_contacts/google_contacts.py
  - frappe/website/doctype/website_settings/website_settings.py
  - frappe/public/js/frappe/file_uploader/FileUploader.vue
  - frappe/public/js/integrations/google_drive_picker.js
  - frappe/core/doctype/sms_settings/sms_settings.json
  - frappe/core/doctype/sms_settings/sms_settings.py
  - frappe/core/doctype/sms_parameter/sms_parameter.json
  - frappe/patches/v16_0/seed_sms_settings_allowed_roles.py
  - frappe/twofactor.py
  - frappe/boot.py
  - frappe/email/doctype/notification/notification.py
  - frappe/integrations/doctype/dropbox_settings/dropbox_settings.json
  - frappe/integrations/doctype/dropbox_settings/dropbox_settings.py
  - frappe/integrations/doctype/dropbox_settings/dropbox_settings.js
  - frappe/integrations/doctype/google_drive/google_drive.json
  - frappe/integrations/doctype/google_drive/google_drive.py
  - frappe/integrations/doctype/google_drive/google_drive.js
  - frappe/integrations/doctype/s3_backup_settings/s3_backup_settings.json
  - frappe/integrations/doctype/s3_backup_settings/s3_backup_settings.py
  - frappe/integrations/doctype/s3_backup_settings/s3_backup_settings.js
  - frappe/integrations/offsite_backup_utils.py
  - frappe/hooks.py
---

# Integration Settings

This page is the field reference for the settings DocTypes that connect a site to outside systems. For each field it gives the default and the code that reads it. For step-by-step setup of OAuth clients, LDAP login, Google, push notifications and cloud backups, see [Integrations](../website-and-integrations/integrations.md) and [Backups](../system-administration/backups.md). For how Single DocTypes work in general, see [How Settings Work](how-settings-work.md).

All DocTypes on this page are Single DocTypes. Only the `System Manager` role can read or write them.

| DocType | Module | Route |
|---|---|---|
| `OAuth Provider Settings` | Integrations | `/app/oauth-provider-settings` |
| `LDAP Settings` | Integrations | `/app/ldap-settings` |
| `Push Notification Settings` | Integrations | `/app/push-notification-settings` |
| `Google Settings` | Integrations | `/app/google-settings` |
| `SMS Settings` | Core | `/app/sms-settings` |
| `Dropbox Settings` | Integrations | `/app/dropbox-settings` |
| `Google Drive` | Integrations | `/app/google-drive` |
| `S3 Backup Settings` | Integrations | `/app/s3-backup-settings` |

## OAuth Provider Settings

This DocType has one field. It sets if a user must approve an OAuth client again when the user already has an active token for it. Open it at `/app/oauth-provider-settings`.

| Field | Default | Effect |
|---|---|---|
| Skip Authorization (`skip_authorization`) | `Force` (first option) | `Force` or `Auto`. Read by `get_oauth_settings()` in `frappe/integrations/doctype/oauth_provider_settings/oauth_provider_settings.py`. With `Auto`, `authorize` in `frappe/integrations/oauth2.py` skips the approval page when the user has an `Active` `OAuth Bearer Token` for the client. With `Force`, the user sees the approval page each time. |

The per-client check **Skip Authorization** on `OAuth Client` skips the approval page for that client in all cases.

```python
from frappe.integrations.doctype.oauth_provider_settings.oauth_provider_settings import get_oauth_settings

get_oauth_settings().skip_authorization  # "Force" or "Auto"
```

## LDAP Settings

`LDAP Settings` lets users log in with an LDAP or Active Directory account. Open it at `/app/ldap-settings`. Fields marked mandatory in the JSON must be filled before the form saves, even when **Enabled** is off. When **Enabled** is on, `validate` binds to the server with the Base DN and its password, and searches both search paths. A bad connection, path or attribute stops the save.

| Field | Default | Effect |
|---|---|---|
| Enabled (`enabled`) | 0 | Read by `frappe/www/login.py`: the login page shows an LDAP login button (`frappe/www/login.html`) that posts to `frappe.integrations.doctype.ldap_settings.ldap_settings.login`. Also read by `validate_user_pass_login` in System Settings (LDAP counts as another login method when you turn on **Disable Username/Password Login**), and by `frappe/core/doctype/user/user.js` (a **Reset LDAP Password** button for System Managers, not shown for `Administrator`). |

### LDAP Server Settings

| Field | Default | Effect |
|---|---|---|
| Directory Server (`ldap_directory_server`) | empty, mandatory | `Active Directory`, `OpenLDAP` or `Custom`. Sets how `fetch_ldap_groups` finds a user's groups. `Active Directory`: object class `Group`, member attribute `member`, matched against the user DN. `OpenLDAP`: object class `posixgroup`, attribute `memberuid`, matched against the username. `Custom`: uses the fields in **LDAP Custom Settings**. |
| LDAP Server Url (`ldap_server_url`) | mandatory | Host passed to `ldap3.Server` in `connect_to_ldap`. |

### LDAP Auth

| Field | Default | Effect |
|---|---|---|
| Base Distinguished Name (DN) (`base_dn`) | mandatory | Bind user for the service connection. |
| Password for Base DN (`password`) | mandatory | Password field (stored encrypted). Read with `get_password` to bind. |

### LDAP Search and Paths

| Field | Default | Effect |
|---|---|---|
| LDAP search path for Users (`ldap_search_path_user`) | mandatory | Search base for users at login and password reset. |
| LDAP Search String (`ldap_search_string`) | mandatory | Filter for the login name. Must start with `(`, end with `)`, have balanced brackets and contain `{0}`, or `validate` throws when **Enabled** is on. `{0}` is replaced with the escaped login name. Example: `(&(objectclass=user)(uid={0}))`. |
| LDAP search path for Groups (`ldap_search_path_group`) | mandatory | Search base for group lookups. |

### LDAP User Creation and Mapping

Each field holds the name of an LDAP attribute. At login, `convert_ldap_entry_to_dict` copies the attribute values to the `User` fields shown. On each login the values overwrite the existing user's fields, except email.

| Field | Default | Effect |
|---|---|---|
| LDAP Email Field (`ldap_email_field`) | mandatory | Maps to `email`. Also used to find the LDAP entry in **Reset LDAP Password**. If the attribute has many values, Frappe picks the first one that matches an existing user. |
| LDAP Username Field (`ldap_username_field`) | mandatory | Maps to `username`. Also the group match value for `OpenLDAP` and `Custom`. |
| LDAP First Name Field (`ldap_first_name_field`) | mandatory | Maps to `first_name`. |
| Do Not Create New User (`do_not_create_new_user`) | 0 | When on, login fails for an LDAP user who has no `User` with that email. When off, Frappe inserts a `User` with **User Type** from **Default User Type** and no welcome email. |
| LDAP Middle Name Field (`ldap_middle_name_field`) | empty | Maps to `middle_name`. |
| LDAP Last Name Field (`ldap_last_name_field`) | empty | Maps to `last_name`. |
| LDAP Phone Field (`ldap_phone_field`) | empty | Maps to `phone`. |
| LDAP Mobile Field (`ldap_mobile_field`) | empty | Maps to `mobile_no`. |

### LDAP Security

| Field | Default | Effect |
|---|---|---|
| SSL/TLS Mode (`ssl_tls_mode`) | `Off` | `StartTLS` binds with `AUTO_BIND_TLS_BEFORE_BIND`. |
| Require Trusted Certificate (`require_trusted_certificate`) | `No` | `Yes` sets `ssl.CERT_REQUIRED` on the TLS configuration. `No` uses `ssl.CERT_NONE`. |
| Path to private Key File (`local_private_key_file`) | empty | File path on the server, passed to `ldap3.Tls`. |
| Path to Server Certificate (`local_server_certificate_file`) | empty | File path on the server, passed to `ldap3.Tls`. |
| Path to CA Certs File (`local_ca_certs_file`) | empty | File path on the server, passed to `ldap3.Tls`. |

### LDAP Custom Settings

Used only when **Directory Server** is `Custom`. With **Enabled** on, `validate` requires **Group Object Class** and **LDAP Group Member attribute** for `Custom`.

| Field | Default | Effect |
|---|---|---|
| Group Object Class (`ldap_group_objectclass`) | empty | Object class of group entries, for example `group`. |
| Custom Group Search (`ldap_custom_group_search`) | empty (`{0}`) | Value matched against the member attribute. `{0}` is the username. If filled, it must contain `{0}`. |
| LDAP Group Member attribute (`ldap_group_member_attribute`) | empty | Member attribute of group entries, for example `member`. |

### LDAP Group Mappings

| Field | Default | Effect |
|---|---|---|
| Default User Type (`default_user_type`) | mandatory, `validate` sets `Website User` when empty | **User Type** of new users. |
| Default User Role (`default_role`) | empty | Shows and is mandatory when **Default User Type** is `System User`. Added to the user at each login. For other user types, the role of the `User Type` is added. |
| LDAP Group Mappings (`ldap_groups`) | empty | Table of `LDAP Group Mapping` rows: **LDAP Group** (`ldap_group`) and **User Role** (`erpnext_role`). At each login, `sync_roles` adds the roles of groups the user is in, and removes mapped roles of groups the user is not in. Group names match without case. |
| LDAP Group Field (`ldap_group_field`) | empty | Deprecated. Used only when **Directory Server** has none of the three values: the groups are read from this attribute of the user entry. |

### For developers

- `LDAPSettings.get_ldap_client_settings()` returns `{"enabled": True, "method": "frappe.integrations.doctype.ldap_settings.ldap_settings.login"}` when enabled. The login page uses it.
- `login` is a guest method with no rate limit. It runs two-factor authentication when it applies.
- `reset_password(user, password, logout)` is whitelisted for `System Manager`. It writes a salted SHA hash to `userPassword` in LDAP.
- The library `ldap3` is a dependency of Frappe.

## Push Notification Settings

`Push Notification Settings` turns on the push notification relay. Frappe apps then send mobile and browser push notifications through a central relay server that uses Firebase Cloud Messaging. Open it at `/app/push-notification-settings`.

### Relay Settings

| Field | Default | Effect |
|---|---|---|
| Enable Push Notification Relay (`enable_push_notification_relay`) | 0 | Read by `PushNotification.is_enabled()` in `frappe/push_notification.py`. When off, every relay call raises `Push Notification Relay is not enabled`. `validate` throws when you turn it on and `push_relay_server_url` is not in the site config. |

### Authentication

| Field | Default | Effect |
|---|---|---|
| API Key (`api_key`) | empty | Key for the relay server. Filled by `_get_credential()` on the first authenticated relay call, not when you save the form. |
| API Secret (`api_secret`) | empty | Password field. Filled with the API Key. Delete both to register the site again. |

### For developers

```python
from frappe.push_notification import PushNotification

push = PushNotification("sample_app")
if push.is_enabled():
	push.send_notification_to_user("user1@example.com", "Order shipped", "Sample Order SO-0001 is on its way")
```

- The relay server URL comes from `push_relay_server_url` in `site_config.json`. See [Site Config](../../reference/configuration/site-config.md).
- `frappe.push_notification.subscribe` and `unsubscribe` (`GET`, logged-in user) add or remove a device token for the session user.
- During registration the relay server calls `frappe.push_notification.auth_webhook`. The token it checks is kept in Redis for 600 seconds.

## Google Settings

`Google Settings` holds the Google Cloud credentials for Google Calendar, Google Contacts, Google Drive backups, Google indexing and the Google Drive picker. Open it at `/app/google-settings`. The controller has no `validate`.

| Field | Default | Effect |
|---|---|---|
| Enable (`enable`) | 0 | Shows the **OAuth Client ID** and **API Key** sections. `GoogleOAuth` (`frappe/integrations/google_oauth.py`, used by Google Contacts, Google Drive and Google indexing) throws `Please enable Google Settings before continuing.` when it is off. `Google Calendar` and `Google Contacts` throw `Enable Google API in Google Settings.`. `Website Settings` throws the same message on save if **Enable Google Indexing** is on and this field is off. |

### OAuth Client ID

Shows when **Enable** is on.

| Field | Default | Effect |
|---|---|---|
| Client ID (`client_id`) | empty | OAuth client ID for the authorization and token calls in `frappe/integrations/google_oauth.py` and `google_calendar.py`. The form makes it mandatory when **Google Drive Picker Enabled** is on. Sent to the browser as `clientId` for the Drive picker. |
| Client Secret (`client_secret`) | empty | Password field. Used for the token and refresh calls. `GoogleOAuth` and `Google Calendar` throw when Client ID or Client Secret is empty. |

### API Key

Shows when **Enable** is on.

| Field | Default | Effect |
|---|---|---|
| API Key (`api_key`) | empty | Not read by Frappe code. |

### Google Drive Picker

| Field | Default | Effect |
|---|---|---|
| Google Drive Picker Enabled (`google_drive_picker_enabled`) | 0 | When this and **Enable** are on, `get_file_picker_settings()` returns `enabled`, `appId` and `clientId`. The file uploader (`frappe/public/js/frappe/file_uploader/FileUploader.vue`) then shows the Google Drive option. |
| App ID (`app_id`) | empty | Shows when the picker is on, and the form makes it mandatory. The Google Cloud project number. Passed to `setAppId` in `frappe/public/js/integrations/google_drive_picker.js`. |

!!! note
    The mandatory rules for **Client ID** and **App ID** use `mandatory_depends_on`. In v15 only the desk form checks them. A save from code skips the check.

## SMS Settings

`SMS Settings` configures an HTTP SMS gateway. Frappe uses it for SMS notifications and for two-factor authentication by SMS. Open it at `/app/sms-settings`.

| Field | Default | Effect |
|---|---|---|
| SMS Gateway URL (`sms_gateway_url`) | mandatory | URL Frappe calls for each receiver. When empty, `_send_sms` shows `Please Update SMS Settings` and sends nothing. System Settings refuses **Two Factor Authentication method** `SMS` while it is empty. Also sent to the browser as the boolean `frappe.boot.sms_gateway_enabled`, which no Frappe JS reads. |
| Message Parameter (`message_parameter`) | mandatory | Request parameter name for the message text. |
| Receiver Parameter (`receiver_parameter`) | mandatory | Request parameter name for the mobile number. |
| Static Parameters (`parameters`) | empty | Table of `SMS Parameter` rows: **Parameter**, **Value** and **Header**. A row with **Header** on is sent as an HTTP header. Other rows are added to the request parameters. A header `Content-Type: application/json` makes Frappe send the parameters as a JSON body. |
| Use POST (`use_post`) | 0 | Send a `POST` with form data. When off, Frappe sends a `GET` with query parameters. |
| Allowed Roles (`allowed_roles`) | empty | Table of roles. `check_sms_permission()` lets only `System Manager` and users with one of these roles call the whitelisted `send_sms`. The patch `frappe.patches.v16_0.seed_sms_settings_allowed_roles` adds `System Manager` on sites that upgrade. New sites start empty. |

### For developers

- `frappe.core.doctype.sms_settings.sms_settings.send_sms(receiver_list, msg, sender_name="", success_msg=True)` is whitelisted and checks **Allowed Roles**. Server code that must skip that check calls `_send_sms`, as `Notification` does for the `SMS` channel.
- v15 has no hook to replace the SMS gateway. All SMS go through the fields on this page.
- Two-factor OTP messages (`frappe/twofactor.py`) read the gateway fields directly and are sent from the `short` queue. They do not check **Allowed Roles**. They add every **Static Parameters** row to the request parameters, also rows with **Header** on.
- A failed gateway call logs the error type and status code only, because the URL can contain credentials.

## Backup settings

Three Single DocTypes copy site backups to cloud storage: `Dropbox Settings`, `Google Drive` and `S3 Backup Settings`. Scheduled jobs in `scheduler_events` of `frappe/hooks.py` run them. Each job runs only when the integration is enabled and the frequency matches.

| DocType | Daily job | Weekly job | Monthly job |
|---|---|---|---|
| `Dropbox Settings` | `daily_maintenance` | `weekly_long` | none |
| `Google Drive` | `daily_maintenance` | `weekly_long` | none |
| `S3 Backup Settings` | `daily_maintenance` | `weekly_long` | `monthly_long` |

Shared behavior in `frappe/integrations/offsite_backup_utils.py`:

- `send_email` emails the notification address after a failed upload. It emails after a successful upload only when **Send Email for Successful Backup** is on. With no address, it writes an Error Log.
- `validate_file_size` checks the latest local database backup. If it is larger than 1 GB, Frappe uploads the latest existing backup files instead of taking a new backup.

### Dropbox Settings

Open it at `/app/dropbox-settings`. `validate` throws when **Enabled** and **Limit Number of DB Backups** are on and **Number of DB Backups** is less than 1.

| Field | Default | Effect |
|---|---|---|
| Enabled (`enabled`) | 0 | `take_backup_to_dropbox` does nothing when off. The **Take Backup Now** button shows only when it is on and a Dropbox token is saved. |
| Send Notifications To (`send_notifications_to`) | mandatory | Email addresses for the backup status emails. |
| Send Email for Successful Backup (`send_email_for_successful_backup`) | 1 | Also send an email after a successful backup. |
| Backup Frequency (`backup_frequency`) | mandatory | `Daily` or `Weekly`. Read by `take_backups_if`. |
| Limit Number of DB Backups (`limit_no_of_backups`) | 0 | When on, `delete_older_backups` keeps only the newest database backups in the Dropbox `/database` folder. |
| Number of DB Backups (`no_of_backups`) | 5 | Shows when **Limit Number of DB Backups** is on. Number of database backups to keep. |
| File Backup (`file_backup`) | 1 | Also upload public files to `/files` and private files to `/private/files`. Each uploaded `File` gets `uploaded_to_dropbox` set, so it is not uploaded again. |
| App Access Key (`app_access_key`) | empty | Key of your Dropbox app. When empty, Frappe uses `dropbox_access_key` from the site config. If this field is empty and the site config has the key, the form hides both key fields. |
| App Secret Key (`app_secret_key`) | empty | Password field. When empty, Frappe uses `dropbox_secret_key` from the site config. |
| Allow Dropbox Access (`allow_dropbox_access`) | button | Calls `get_dropbox_authorize_url` and opens the Dropbox approval page. After approval, `dropbox_auth_finish` saves the tokens and redirects to `/app/dropbox-settings`. |
| Dropbox Refresh Token (`dropbox_refresh_token`) | empty, hidden | Password field. Set by `dropbox_auth_finish`. |
| Dropbox Access Token (`dropbox_access_token`) | empty, hidden | Password field. Set by `dropbox_auth_finish`, and again when the Dropbox client refreshes the token. Kept for legacy Dropbox apps. |

Register `<site>/api/method/frappe.integrations.doctype.dropbox_settings.dropbox_settings.dropbox_auth_finish` as a redirect URI in the Dropbox app.

### Google Drive

Open it at `/app/google-drive`. It needs an enabled [Google Settings](#google-settings) with a Client ID and Client Secret. `validate` clears **Backup Folder ID** when you change **Backup Folder Name**.

| Field | Default | Effect |
|---|---|---|
| Enable (`enable`) | 0 | `daily_backup` and `weekly_backup` run only when it is on. Shows the **Google Drive** section. |
| Backup Folder Name (`backup_folder_name`) | mandatory | Name of the folder in Google Drive. `check_for_folder_in_google_drive` finds the folder by name, or makes it. |
| Frequency (`frequency`) | mandatory | `Daily` or `Weekly`. |
| Send Notification To (`email`) | mandatory | Email addresses for the backup status emails. |
| Send Email for Successful backup (`send_email_for_successful_backup`) | 0 | Also send an email after a successful backup. |
| File Backup (`file_backup`) | 0 | Also upload the public and private files backups. |
| Authorize Google Drive Access (`authorize_google_drive_access`) | button | Shows after the first save. Calls `authorize_access` and opens the Google approval page. A second click (when an authorization code exists) authorizes again and clears **Backup Folder ID**. |
| Backup Folder ID (`backup_folder_id`) | read only | ID of the backup folder. Set by `check_for_folder_in_google_drive`. |
| Last Backup On (`last_backup_on`) | read only | Set after each upload. |
| Refresh Token (`refresh_token`) | hidden | Set by `authorize_access`. The **Take Backup** button shows only when it is set and **Enable** is on. |
| Authorization Code (`authorization_code`) | hidden | Set by `authorize_access`. With **Enable** on and both tokens set, the form shows the indicator **Authorized**. |

### S3 Backup Settings

Open it at `/app/s3-backup-settings`. When **Enable Automatic Backup** is on, `validate` sets an empty **Endpoint URL** to `https://s3.amazonaws.com`, adds a trailing `/` to **Backup Path**, and calls `head_bucket`. An S3 client error, for example a missing bucket (404) or no access (403), stops the save. All sections below show only when **Enable Automatic Backup** is on.

| Field | Default | Effect |
|---|---|---|
| Enable Automatic Backup (`enabled`) | 0 | `take_backups_if` runs scheduled backups only when it is on. The **Take Backup Now** button does not depend on it. |

#### API Access

| Field | Default | Effect |
|---|---|---|
| Access Key ID (`access_key_id`) | mandatory when enabled | Passed to `boto3.client` as `aws_access_key_id`. The **Take Backup Now** button shows when this and the secret are set. |
| Access Key Secret (`secret_access_key`) | mandatory when enabled | Password field. Passed as `aws_secret_access_key`. |

#### Notification

| Field | Default | Effect |
|---|---|---|
| Send Notifications To (`notify_email`) | mandatory when enabled | Email addresses for the backup status emails. |
| Send Email for Successful Backup (`send_email_for_successful_backup`) | 1 | Also send an email after a successful backup. |

#### S3 Bucket Details

| Field | Default | Effect |
|---|---|---|
| Bucket Name (`bucket`) | mandatory when enabled | Target bucket. |
| Endpoint URL (`endpoint_url`) | `https://s3.amazonaws.com` | Change it for another S3 compatible storage. |
| Backup Path (`backup_path`) | empty | Prefix in the bucket. Empty: backups go to the root of the bucket. Each backup goes to a folder named with the backup date and time. |

#### Backup Details

| Field | Default | Effect |
|---|---|---|
| Backup Frequency (`frequency`) | mandatory when enabled | `Daily`, `Weekly`, `Monthly` or `None`. `None` turns off scheduled backups. |
| Backup Files (`backup_files`) | 1 | Also upload the public and private files backups. |

### For developers

- The `take_backup` methods of `dropbox_settings`, `google_drive` and `s3_backup_settings` are whitelisted for `System Manager`. Each queues a backup job in the `long` queue with a 1500 second timeout. The S3 method returns the job ID.
- Dropbox and S3 jobs queue themselves again up to two times after a `JobTimeoutException`.

## Related

- [Integrations](../website-and-integrations/integrations.md)
- [Backups](../system-administration/backups.md)
- [How Settings Work](how-settings-work.md)
- [System Settings](system-settings.md)
- [Security Settings](security-settings.md)
- [Authentication](../../developer-guide/security/authentication.md)
- [Site Config](../../reference/configuration/site-config.md)
