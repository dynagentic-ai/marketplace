---
name: dynagentic-staff
description: Instructor/staff toolkit for Dynagentic workshops. Provides access to all 12 tools the /mcp-staff connector exposes (the 5 shared student tools plus the 7 instructor-only management tools for token administration, broadcast messaging, and progress monitoring). Auth is handled by Google sign-in via Cognito OAuth (PKCE); no per-call token argument is needed. Use this skill whenever an instructor wants to manage student tokens, send broadcasts, view all progress, work with exercise bundles, read or write shared team files, or do anything with the Dynagentic staff connector tools. Access is gated by an allowlist: users not on the staff_instructor_allowlist_emails will connect successfully but receive a 403 on any tool call, which is expected behavior for unauthorised accounts.
---

# Dynagentic Staff Skill

This skill covers all 12 tools the `/mcp-staff` connector exposes to Dynagentic workshop instructors and staff. Authentication is handled once via Google sign-in (Cognito OAuth/PKCE) when the connector is added. No per-call token argument is needed after that.

## Authentication

The `/mcp-staff` connector uses Google sign-in via AWS Cognito (PKCE, public client). When you (or a user) first connect the connector, Claude triggers the OAuth flow automatically. There is no token argument to pass on individual tool calls; the access token is managed by Claude's connector machinery per session.

**Allowlist gate:** Access is restricted to `staff_instructor_allowlist_emails` (currently: Alan Spencer, Monique Mensah, Rick Farnell, Michel Debiche). A user not on that allowlist will connect successfully and see all 12 tools listed, but every tool call will return a clean 403. This is expected and by design. It is not a broken connector; the user simply needs to be added to the allowlist by an admin.

## What is installed with this plugin

This plugin bundles two components in a single install:

1. **The `/mcp-staff` MCP connector** - the actual connection to the workshop backend, pointing at `https://d3vfrcd9gs7j0x.cloudfront.net/mcp-staff`. Installing this plugin adds it automatically.

2. **This skill** - tells Claude how to use the connector's 12 tools correctly and handles auth context.

If you installed this plugin, both components are active. If you see a message that workshop tools are not available, check that the plugin is fully installed and enabled, and that your Google account has been added to the staff allowlist.

## One remaining manual step after install

After installing the plugin, the connector needs to be authorised through Claude's "Add custom connector" OAuth flow using the Dynagentic Cognito app client. The `.mcp.json` schema does not support declaring the OAuth client ID inline, so this one step remains manual:

When prompted to authenticate, select "Use your own OAuth client" and enter:
- **Client ID:** `1508kdjq1t75jfc74ol3p4vdu7`
- **Client secret:** (leave blank - this is a public PKCE client with no secret)
- **Authorization URL:** `https://d3vfrcd9gs7j0x.cloudfront.net/.well-known/oauth-authorization-server` (discovery endpoint)

Claude will complete the PKCE flow and store the resulting token for the session. You will be prompted to sign in with your Google account.

## All 12 tools

Staff mode advertises the full manifest unfiltered. Tool descriptions are taken directly from `_TOOL_DESCRIPTIONS` in `infra/dynagentic-connector/lambda-src/mcp_adapter/index.py`.

### Shared tools (available to students and staff)

| Tool | What it does | Required arguments | Optional arguments |
|------|--------------|--------------------|--------------------|
| `get_exercise_bundle` | Fetch the exercise bundle files for a given exercise | `exercise_id` | (none) |
| `report_progress` | Report a progress event for the current exercise and return the current broadcast | `exercise`, `principle`, `event`, `detail` | (none) |
| `check_broadcast` | Report a progress event and check for any broadcast messages or overwrite notices | `exercise`, `principle`, `event`, `detail` | (none) |
| `read_team_file` | Read a shared team file | `team`, `file` | (none) |
| `write_team_file` | Write or update a shared team file | `team`, `file`, `content` | `base_version` |

No `token` argument is needed on any of these calls in staff mode. The Bearer access token on the HTTP request already identifies the caller (verified server-side).

### Instructor-only tools

| Tool | What it does | Required arguments | Optional arguments |
|------|--------------|--------------------|--------------------|
| `create_token` | Create a new token for a roster entry (instructor only) | `row_id` | `capabilities` (array of strings) |
| `disable_token` | Disable an existing token (instructor only) | `row_id` | (none) |
| `list_tokens` | List all roster tokens and their status (instructor only) | (none) | (none) |
| `reissue_token` | Reissue a token for a roster entry (instructor only) | `row_id` | (none) |
| `list_students` | List all students in the roster (instructor only) | (none) | (none) |
| `send_broadcast` | Send a broadcast message to all students (instructor only) | `cohort`, `message` | (none) |
| `get_all_progress` | Get all students' progress summaries (instructor only) | `cohort` | (none) |

## Argument details

### `write_team_file`
- `base_version` (optional string): supply the version string returned by the last `read_team_file` call for conflict detection. Omit when creating a new file.

### `create_token`
- `capabilities` (optional array of strings): capability tags to assign to the roster entry (e.g. `["exercise-1", "exercise-2"]`). Omit to create a token with default capabilities.

### `send_broadcast` and `get_all_progress`
- `cohort`: the cohort directory name as configured in the backend (e.g. `"cohort-2026-09"`). Ask the Dynagentic admin if you are unsure of the correct identifier.

## When a tool call returns 403

A 403 from a staff tool call means the signed-in Google account is not in `staff_instructor_allowlist_emails`. This is not a broken connector or a wrong OAuth config; it is the expected response for an unauthorised account. To gain access, the account email needs to be added to the allowlist by a Dynagentic admin (currently managed via the `STAFF_ALLOWLIST_EMAILS` environment variable on the deployed Lambda).

## When a tool call returns 401

A 401 means the OAuth access token is missing, expired, or was rejected. Claude should trigger a re-authentication flow. If the issue persists after re-auth, check:
- The app client ID entered during setup is `1508kdjq1t75jfc74ol3p4vdu7` (no trailing characters)
- The authorised Google account matches the email in the allowlist
