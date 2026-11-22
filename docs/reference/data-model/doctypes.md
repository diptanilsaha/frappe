---
title: DocTypes Reference
description: Every DocType shipped with the frappe app, grouped by module, with flags and a one-line description.
version: v15
sources:
  - frappe/modules.txt
  - frappe/core/doctype
  - frappe/desk/doctype
  - frappe/custom/doctype
  - frappe/email/doctype
  - frappe/printing/doctype
  - frappe/website/doctype
  - frappe/workflow/doctype
  - frappe/automation/doctype
  - frappe/contacts/doctype
  - frappe/geo/doctype
  - frappe/integrations/doctype
  - frappe/social/doctype
---

# DocTypes Reference

This page lists every DocType that the `frappe` app ships, grouped by module. Use it to find the exact DocType name and its module. The list is generated from `frappe/*/doctype/*/*.json`. It has 275 DocTypes, the same as the count of non-custom DocTypes on a v15 site.

Flags come from the DocType JSON:

| Flag | JSON property | Meaning |
|---|---|---|
| single | `issingle` | One record only. Values are in `tabSingles`. |
| child | `istable` | Child table. Rows belong to a parent document. |
| submittable | `is_submittable` | Has Submit and Cancel (`docstatus`). |
| tree | `is_tree` | Nested set tree with `lft` and `rgt`. |
| virtual | `is_virtual` | No table. The controller supplies the data. |

For child tables, the description names the parent DocType and the Table field. See [DocTypes](../../developer-guide/data-model/doctypes.md) for how DocTypes work.


## Core (94)

| DocType | Flags | Description |
|---|---|---|
| `Access Log` |  | Log of exports, prints and report downloads by users. |
| `Activity Log` |  | Log of logins, logouts and other activity. |
| `Amended Document Naming Settings` | child | Child table of `Document Naming Settings` (`amend_naming_override`). |
| `Audit Trail` | single | Tool to compare versions of a document over a date range. |
| `Block Module` | child | Child table of `Module Profile` (`block_modules`), `User` (`block_modules`). |
| `Comment` |  | Comments, likes, assignments and other timeline entries of a document. |
| `Communication` |  | Emails, chats and other communication linked to documents. |
| `Communication Link` | child | Child table of `Communication` (`timeline_links`). |
| `Custom DocPerm` |  | Role permissions of a DocType set from Role Permissions Manager. |
| `Custom Role` |  | Extra roles for a Page or Report, set on the site. |
| `Data Export` | single | Tool to export records of a DocType to CSV or Excel. |
| `Data Import` |  | Imports or updates records from a CSV or Excel file. |
| `Data Import Log` |  | Result of each row of a Data Import. |
| `DefaultValue` | child | Child table of `User` (`defaults`). |
| `Deleted Document` |  | Copy of a deleted document. Use it to restore the document. |
| `DocField` | child | Child table of `DocType` (`fields`). |
| `DocPerm` | child | Child table of `DocType` (`permissions`). |
| `DocShare` |  | A document shared with a user, with read, write and share rights. |
| `DocType` |  | Defines a document type: fields, permissions and settings. Each DocType gets a table. |
| `DocType Action` | child | Child table of `DocType` (`actions`), `Customize Form` (`actions`). |
| `DocType Link` | child | Child table of `DocType` (`links`), `Customize Form` (`links`). |
| `DocType State` | child | Child table of `DocType` (`states`), `Customize Form` (`states`). |
| `Doctype To Sync` | child | Child table of `Report` (`doctype_to_sync`). |
| `Document Naming Rule` |  | Naming rule with prefix and counter, applied when conditions match. |
| `Document Naming Rule Condition` | child | Child table of `Document Naming Rule` (`conditions`). |
| `Document Naming Settings` | single | Settings for naming series, current counters and amended document names. |
| `Document Share Key` |  | Key that gives access to a document by link, with an expiry date. |
| `Domain` |  | A domain that apps can use to show or hide features. |
| `Domain Settings` | single | Active domains of the site. |
| `DuckDB Sync` | submittable | Copies tables to a DuckDB file for analytics. Submittable. |
| `DuckDB Sync Item` | child | Child table of `DuckDB Sync` (`db_tables`). |
| `Dynamic Link` | child | Child table of `Address` (`links`), `Contact` (`links`), `Event` (`links`), `Web Form Request` (`references`). |
| `Error Log` |  | Tracebacks of errors on the site. |
| `File` |  | A file uploaded to the site, a folder, or an attachment of a document. |
| `Has Domain` | child | Child table of `Domain Settings` (`active_domains`). |
| `Has Role` | child | Child table of `Custom Role` (`roles`), `Page` (`roles`), `Report` (`roles`), `Role Permission for Page and Report` (`roles`), `Role Profile` (`roles`), `SMS Settings` (`allowed_roles`), `User` (`roles`), `Custom HTML Block` (`roles`), `Dashboard Chart` (`roles`), `Workspace` (`roles`). |
| `Installed Application` | child | Child table of `Installed Applications` (`installed_applications`). |
| `Installed Applications` | single | Apps installed on the site, with versions, branches and setup status. |
| `Language` |  | A language with its code, name and date and number formats. |
| `Log Setting User` | child | Child table. No frappe DocType has a Table field for it. |
| `Log Settings` | single | How many days to keep each log DocType before it is cleared. |
| `Logs To Clear` | child | Child table of `Log Settings` (`logs_to_clear`). |
| `Module Def` |  | A module of an app. DocTypes, reports and pages belong to a module. |
| `Module Profile` |  | A set of blocked modules to apply to users. |
| `Navbar Item` | child | Child table of `Navbar Settings` (`settings_dropdown`), `Navbar Settings` (`help_dropdown`). |
| `Navbar Settings` | single | Logo, settings menu and help menu of the desk navbar. |
| `Package` |  | A package of DocTypes and other records that you can export without an app. |
| `Package Import` |  | Imports a package file into the site. |
| `Package Release` |  | A versioned release of a Package. |
| `Page` |  | A desk page, with JS and Python code in a module. |
| `Patch Log` |  | Patches that ran on the site. |
| `Permission Inspector` | single, virtual | Tool that explains why a user has or does not have a permission. |
| `Prepared Report` |  | Result of a report that ran in the background. |
| `Recorder` | virtual | Requests and jobs captured by the Recorder, with their SQL queries. |
| `Recorder Query` | child, virtual | Child table of `Recorder` (`sql_queries`). |
| `Recorder Suggested Index` | child, virtual | Child table of `Recorder` (`suggested_indexes`). |
| `Report` |  | A report: Report Builder, Query Report or Script Report. |
| `Report Column` | child | Child table of `Report` (`columns`). |
| `Report Filter` | child | Child table of `Report` (`filters`). |
| `Role` |  | A role. Permissions are given to roles, and roles to users. |
| `Role Permission for Page and Report` | single | Tool to set roles for a Page or Report. |
| `Role Profile` |  | A set of roles to apply to users. |
| `RQ Job` | virtual | Background jobs in Redis queues. |
| `RQ Worker` | virtual | Background workers and their status. |
| `Scheduled Job Log` |  | Log of each run of a scheduled job. |
| `Scheduled Job Type` |  | A scheduled job with its method, frequency or cron, and queue. |
| `Scheduler Event` |  | A method scheduled against a DocType. |
| `Security Settings` | single | Values for the site's `security.txt`: contacts, languages, policy and expiry. |
| `Security Settings Contact` | child | Child table of `Security Settings` (`public_contacts`). |
| `Security Settings Language` | child | Child table of `Security Settings` (`public_languages`). |
| `Server Script` |  | Python script that runs on a document event, as an API, on a schedule or as a permission query. |
| `Session Default` | child | Child table of `Session Default Settings` (`session_defaults`). |
| `Session Default Settings` | single | DocTypes for which users can set session defaults. |
| `SMS Parameter` | child | Child table of `SMS Settings` (`parameters`). |
| `SMS Settings` | single | SMS gateway URL and parameters. |
| `Submission Queue` |  | Documents queued for submit or cancel in the background. |
| `Success Action` |  | Message and next actions shown after a document of a DocType is saved or submitted. |
| `System Settings` | single | Site-wide settings: country, language, time zone, formats, security and more. |
| `Transaction Log` |  | Log of document data with chained SHA-256 hashes, to detect changes. Frappe does not write it. Apps such as ERPNext regional code do. |
| `Translation` |  | A custom translation of a source text for a language. |
| `User` |  | Someone who can log in to your site. |
| `User Document Type` | child | Child table of `User Type` (`user_doctypes`). |
| `User Email` | child | Child table of `User` (`user_emails`). |
| `User Group` |  | A named group of users. |
| `User Group Member` | child | Child table of `User Group` (`user_group_members`). |
| `User Invitation` |  | An invitation sent to an email address to join the site. |
| `User Permission` |  | Restricts a user to documents that match a value of a Link field. |
| `User Role` | child | Child table of `User Invitation` (`roles`). |
| `User Select Document Type` | child | Child table of `User Type` (`select_doctypes`), `User Type` (`custom_select_doctypes`). |
| `User Social Login` | child | Child table of `User` (`social_logins`). |
| `User Type` |  | A user type with its role and the DocTypes it can access. |
| `User Type Module` | child | Child table of `User Type` (`user_type_modules`). |
| `Version` |  | Change history of a document. |
| `View Log` |  | Log of document views. |

## Desk (54)

| DocType | Flags | Description |
|---|---|---|
| `Bulk Update` | single | Tool to set one field on many documents. |
| `Calendar View` |  | A calendar view of a DocType with subject and date fields. |
| `Changelog Feed` |  | Release notes of installed apps. A scheduled job fills it through the `get_changelog_feed` hook. Desk shows the latest 10. |
| `Console Log` |  | Scripts run in the System Console. |
| `Custom HTML Block` |  | HTML, CSS and JS block for workspaces. |
| `Dashboard` |  | A dashboard of charts and number cards. |
| `Dashboard Chart` |  | A chart from a DocType, report or custom source. |
| `Dashboard Chart Field` | child | Child table of `Dashboard Chart` (`y_axis`). |
| `Dashboard Chart Link` | child | Child table of `Dashboard` (`charts`). |
| `Dashboard Chart Source` |  | A custom data source for Dashboard Charts, with code in a module. |
| `Dashboard Settings` |  | Chart settings of a user. |
| `Desktop Icon` |  | Module icons per user from the older desk home screen. In v15 only its own controller uses it. |
| `Event` |  | A calendar event, with participants and reminders. |
| `Event Notifications` | child | Child table of `Event` (`notifications`). |
| `Event Participants` | child | Child table of `Event` (`event_participants`). |
| `Form Tour` |  | A guided tour of a form. |
| `Form Tour Step` | child | Child table of `Form Tour` (`steps`). |
| `Global Search DocType` | child | Child table of `Global Search Settings` (`allowed_in_global_search`). |
| `Global Search Settings` | single | DocTypes shown in global search, in priority order. |
| `Kanban Board` |  | A kanban board of a DocType grouped by a Select field. |
| `Kanban Board Column` | child | Child table of `Kanban Board` (`columns`). |
| `List Filter` |  | Saved filters of a list view. |
| `List View Settings` |  | List view settings of a DocType: count, sidebar stats, refresh and fields. |
| `Module Onboarding` |  | Onboarding steps for a module. |
| `Note` |  | A note. A public note can show to users on login. |
| `Note Seen By` | child | Child table of `Note` (`seen_by`). |
| `Notification Log` |  | Notifications shown in the desk notification panel. |
| `Notification Settings` |  | Notification preferences of a user. |
| `Notification Subscribed Document` | child | Child table of `Notification Settings` (`subscribed_documents`). |
| `Notification Type` |  | A type of notification that users can turn on or off. |
| `Notification Type Preference` | child | Child table of `Notification Settings` (`email_notification_types`). |
| `Number Card` |  | A card that shows one aggregated number. |
| `Number Card Link` | child | Child table of `Dashboard` (`cards`). |
| `Onboarding Permission` | child | Child table of `Module Onboarding` (`allow_roles`). |
| `Onboarding Step` |  | One step of a Module Onboarding. |
| `Onboarding Step Map` | child | Child table of `Module Onboarding` (`steps`). |
| `Route History` |  | Desk routes visited by users. |
| `System Console` | single | Tool to run Python or SQL on the site. |
| `System Health Report` | single, virtual | Report on workers, queues, scheduler, database and errors. |
| `System Health Report Errors` | child, virtual | Child table of `System Health Report` (`top_errors`). |
| `System Health Report Failing Jobs` | child, virtual | Child table of `System Health Report` (`failing_scheduled_jobs`). |
| `System Health Report Queue` | child, virtual | Child table of `System Health Report` (`queue_status`). |
| `System Health Report Tables` | child, virtual | Child table of `System Health Report` (`top_db_tables`). |
| `System Health Report Workers` | child, virtual | Child table of `System Health Report` (`background_workers`). |
| `Tag` |  | A tag that users can add to documents. |
| `Tag Link` |  | Links a Tag to a document. |
| `ToDo` |  | A task for a user. Assignments are ToDo records. |
| `Workspace` |  | A desk workspace page with shortcuts, links, charts and cards. |
| `Workspace Chart` | child | Child table of `Workspace` (`charts`). |
| `Workspace Custom Block` | child | Child table of `Workspace` (`custom_blocks`). |
| `Workspace Link` | child | Child table of `Workspace` (`links`). |
| `Workspace Number Card` | child | Child table of `Workspace` (`number_cards`). |
| `Workspace Quick List` | child | Child table of `Workspace` (`quick_lists`). |
| `Workspace Shortcut` | child | Child table of `Workspace` (`shortcuts`). |

## Custom (7)

| DocType | Flags | Description |
|---|---|---|
| `Client Script` |  | Adds a custom client script to a DocType. |
| `Custom Field` |  | Adds a custom field to a DocType. |
| `Customize Form` | single | Tool to change fields and properties of a DocType without changing its JSON. |
| `Customize Form Field` | child | Child table of `Customize Form` (`fields`). |
| `DocType Layout` |  | A custom form layout for a DocType. Loaded with desk boot. |
| `DocType Layout Field` | child | Child table of `DocType Layout` (`fields`). |
| `Property Setter` |  | Overrides a property of a standard DocType or field. |

## Email (19)

| DocType | Flags | Description |
|---|---|---|
| `Auto Email Report` |  | Sends a report by email on a schedule. |
| `Document Follow` |  | A user who follows a document and gets emails about changes. |
| `Email Account` |  | An incoming or outgoing email account. |
| `Email Domain` |  | Server settings shared by email accounts of a domain. |
| `Email Flag Queue` |  | Read or unread flags to sync back to the mail server. |
| `Email Group` |  | A mailing list. |
| `Email Group Member` |  | One email address in an Email Group. |
| `Email Queue` |  | Outgoing emails waiting to be sent, or sent. |
| `Email Queue Recipient` | child | Child table of `Email Queue` (`recipients`). |
| `Email Rule` |  | Marks an email address as spam. |
| `Email Template` |  | Subject and body template for emails. |
| `Email Unsubscribe` |  | An email address that unsubscribed, globally or from one document. |
| `IMAP Folder` | child | Child table of `Email Account` (`imap_folder`). |
| `Newsletter` |  | An email sent to the members of one or more Email Groups, now or on a schedule. Can also publish on the website. |
| `Newsletter Attachment` | child | Child table of `Newsletter` (`attachments`). |
| `Newsletter Email Group` | child | Child table of `Newsletter` (`email_group`). |
| `Notification` |  | Sends an email, Slack message, system notification or SMS on a document event. |
| `Notification Recipient` | child | Child table of `Notification` (`recipients`). |
| `Unhandled Email` |  | An incoming email that could not be processed. |

## Printing (7)

| DocType | Flags | Description |
|---|---|---|
| `Letter Head` |  | Header and footer for print formats. |
| `Network Printer Settings` |  | A network printer with server IP, port and printer name. |
| `Print Format` |  | A print layout for a DocType: Builder, Jinja HTML or raw commands. |
| `Print Format Field Template` |  | A template to render one field in print formats. |
| `Print Heading` |  | A heading that you can choose when you print. |
| `Print Settings` | single | Site-wide print and PDF settings. |
| `Print Style` |  | CSS styles for print formats. |

## Website (41)

| DocType | Flags | Description |
|---|---|---|
| `About Us Settings` | single | Content of the About Us page. |
| `About Us Team Member` | child | Child table of `About Us Settings` (`team_members`). |
| `Blog Category` |  | A category of blog posts, with its own website route. |
| `Blog Post` |  | A blog post on the website, written by a Blogger. |
| `Blog Settings` | single | Blog title, introduction, call to action, comments, likes and social sharing. |
| `Blogger` |  | An author of blog posts, linked to a User. |
| `Color` |  | A named color. |
| `Company History` | child | Child table of `About Us Settings` (`company_history`). |
| `Contact Us Settings` | single | Content and settings of the Contact Us page. |
| `Discussion Reply` |  | A reply in a discussion topic. |
| `Discussion Topic` |  | A discussion topic linked to a document. |
| `Help Article` |  | An article in the website help section. |
| `Help Category` |  | A category of help articles. |
| `Marketing Campaign` |  | A campaign name for UTM tracking links and Newsletters. |
| `Personal Data Deletion Request` |  | A request to delete a user's personal data. |
| `Personal Data Deletion Step` | child | Child table of `Personal Data Deletion Request` (`deletion_steps`). |
| `Personal Data Download Request` |  | A request to download a user's personal data. |
| `Portal Menu Item` | child | Child table of `Portal Settings` (`menu`), `Portal Settings` (`custom_menu`). |
| `Portal Settings` | single | Portal menu and default role at sign-up. |
| `Social Link Settings` | child | Child table. No frappe DocType has a Table field for it. |
| `Top Bar Item` | child | Child table of `Website Settings` (`top_bar_items`), `Website Settings` (`footer_items`). |
| `Web Form` |  | A form on the website that makes or edits documents of a DocType. |
| `Web Form Field` | child | Child table of `Web Form` (`web_form_fields`). |
| `Web Form List Column` | child | Child table of `Web Form` (`list_columns`). |
| `Web Form Request` |  | A key that lets a web form edit an existing record, with an expiry. |
| `Web Page` |  | A page to show on the website. |
| `Web Page Block` | child | Child table of `Web Page` (`page_blocks`). |
| `Web Page View` |  | Page views of the website. |
| `Web Template` |  | A reusable block for web pages. |
| `Web Template Field` | child | Child table of `Web Template` (`fields`). |
| `Website Meta Tag` | child | Child table of `Website Route Meta` (`meta_tags`). |
| `Website Route Meta` |  | Meta tags for a website route. |
| `Website Route Redirect` | child | Child table of `Website Settings` (`route_redirects`). |
| `Website Script` | single | JavaScript added to all website pages. |
| `Website Settings` | single | Website home page, theme, navbar, footer and other settings. |
| `Website Sidebar` |  | A sidebar for website pages. |
| `Website Sidebar Item` | child | Child table of `Website Sidebar` (`sidebar_items`). |
| `Website Slideshow` |  | A slideshow for website pages. |
| `Website Slideshow Item` | child | Child table of `Website Slideshow` (`slideshow_items`). |
| `Website Theme` |  | Theme of the website: colors, fonts, CSS and JS. |
| `Website Theme Ignore App` | child | Child table of `Website Theme` (`ignored_apps`). |

## Workflow (7)

| DocType | Flags | Description |
|---|---|---|
| `Workflow` |  | Defines workflow states and transitions for a DocType. |
| `Workflow Action` |  | A pending workflow action for a user on a document. |
| `Workflow Action Master` |  | A workflow action name, such as Approve or Reject. |
| `Workflow Action Permitted Role` | child | Child table of `Workflow Action` (`permitted_roles`). |
| `Workflow Document State` | child | Child table of `Workflow` (`states`). |
| `Workflow State` |  | A workflow state name with its icon and style. |
| `Workflow Transition` | child | Child table of `Workflow` (`transitions`). |

## Automation (8)

| DocType | Flags | Description |
|---|---|---|
| `Assignment Rule` |  | Assigns documents of a DocType to users automatically, by condition, round robin, load balancing or field value. |
| `Assignment Rule Day` | child | Child table of `Assignment Rule` (`assignment_days`). |
| `Assignment Rule User` | child | Child table of `Assignment Rule` (`users`), `Assignment Rule` (`weighted_users`). |
| `Auto Repeat` |  | Makes copies of a document on a schedule (daily, weekly, monthly and so on). |
| `Auto Repeat Day` | child | Child table of `Auto Repeat` (`repeat_on_days`). |
| `Milestone` |  | Records when a tracked field of a document changed to a value. |
| `Milestone Tracker` |  | Defines which DocType and field to track as milestones. |
| `Reminder` |  | A reminder for a user about a document at a date and time. |

## Contacts (7)

| DocType | Flags | Description |
|---|---|---|
| `Address` |  | A postal address of a customer, supplier, company or contact. |
| `Address Template` |  | Jinja template to format addresses for a country. |
| `Contact` |  | A person you deal with, with emails, phone numbers and links to other documents. |
| `Contact Email` | child | Child table of `Contact` (`email_ids`). |
| `Contact Phone` | child | Child table of `Contact` (`phone_nos`). |
| `Gender` |  | List of genders used in Contact and User. |
| `Salutation` |  | List of salutations used in Contact. |

## Geo (2)

| DocType | Flags | Description |
|---|---|---|
| `Country` |  | A country with its code, date and time formats and time zones. |
| `Currency` |  | A currency with its symbol, fraction and fraction units. |

## Integrations (25)

| DocType | Flags | Description |
|---|---|---|
| `Connected App` |  | An OAuth client connection from this site to another service. |
| `Dropbox Settings` | single | Scheduled backups of the database and files to Dropbox. |
| `Google Calendar` |  | Sync of a Google Calendar with Events. |
| `Google Contacts` |  | Sync of Google Contacts with Contacts. |
| `Google Drive` | single | Scheduled backups of the database and files to Google Drive. |
| `Google Settings` | single | Client ID, secret and API key for Google integrations. |
| `Integration Request` |  | Log of requests to and from external services. |
| `LDAP Group Mapping` | child | Child table of `LDAP Settings` (`ldap_groups`). |
| `LDAP Settings` | single | LDAP login settings and group to role mapping. |
| `OAuth Authorization Code` |  | An OAuth authorization code issued by this site. |
| `OAuth Bearer Token` |  | An OAuth access token issued by this site. |
| `OAuth Client` |  | An app allowed to use this site as an OAuth provider. |
| `OAuth Client Role` | child | Child table of `OAuth Client` (`allowed_roles`). |
| `OAuth Provider Settings` | single | OAuth provider settings, such as skip authorization. |
| `OAuth Scope` | child | Child table of `Connected App` (`scopes`), `Token Cache` (`scopes`). |
| `Push Notification Settings` | single | Registers the site on a relay server to send push notifications. |
| `Query Parameters` | child | Child table of `Connected App` (`query_parameters`). |
| `S3 Backup Settings` | single | Scheduled backups of the database and files to an S3 compatible bucket. |
| `Slack Webhook URL` |  | A Slack incoming webhook URL. Notifications with channel Slack send to it. |
| `Social Login Key` |  | Login with an external provider, such as Google or GitHub. |
| `Token Cache` |  | Tokens stored for a user and a Connected App. |
| `Webhook` |  | Sends an HTTP request on a document event. |
| `Webhook Data` | child | Child table of `Webhook` (`webhook_data`). |
| `Webhook Header` | child | Child table of `Webhook` (`webhook_headers`). |
| `Webhook Request Log` |  | Log of webhook requests and responses. |

## Social (4)

| DocType | Flags | Description |
|---|---|---|
| `Energy Point Log` |  | Energy points given to or taken from a user. Type is Auto (a rule), Appreciation, Criticism, Review or Revert. |
| `Energy Point Rule` |  | Gives energy points to a user when a document of a DocType matches a condition. |
| `Energy Point Settings` | single | Turns energy points on and sets review points per role and how often they are given. |
| `Review Level` | child | Child table of `Energy Point Settings` (`review_levels`). |

## Related

- [DocTypes](../../developer-guide/data-model/doctypes.md)
- [Field Types](field-types.md)
- [Python API](../api/python-api.md)
