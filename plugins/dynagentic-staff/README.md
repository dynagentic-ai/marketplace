# Dynagentic Staff Plugin

A one-click Cowork plugin that bundles the Dynagentic workshop staff/instructor connector and the dynagentic-staff skill together. Instructors previously needed two separate manual steps to get set up; this plugin reduces that to a single install.

## What this plugin includes

| Component | What it does |
|-----------|-------------|
| **MCP connector** (`dynagentic-staff`) | Connects Claude to the Dynagentic workshop backend at `https://d3vfrcd9gs7j0x.cloudfront.net/mcp-staff`. Exposes all 17 tools: the 7 shared tools (`get_exercise_bundle`, `get_course_bundle`, `report_progress`, `check_broadcast`, `read_team_file`, `write_team_file`, `get_team_bundle`) plus the 10 instructor-only tools (`create_token`, `disable_token`, `enable_token`, `list_tokens`, `reissue_token`, `list_students`, `create_student`, `edit_student`, `send_broadcast`, `get_all_progress`). |
| **Skill** (`dynagentic-staff`) | Covers all 17 tools with usage guidance and handles auth context. No per-call token argument needed for the instructor-only tools; auth is handled by Google sign-in. On the shared team-file tools, an instructor may additionally pass an explicit `team`/`cohort` to inspect any team, unlike a student sign-in, which is always pinned to its own roster team. |

## Access restriction

This plugin is restricted to Dynagentic staff and instructors listed in `staff_instructor_allowlist_emails`. The current allowlist contains: Alan Spencer, Monique Mensah, Rick Farnell, Michel Debiche.

A user not on the allowlist will install the plugin successfully and see all 17 tools listed, but every tool call will return a clean 403. This is expected behavior, not a broken connector. To gain access, the user's `@dynagentic.ai` instructor account email needs to be added to the `STAFF_ALLOWLIST_EMAILS` environment variable on the deployed Lambda by a Dynagentic admin.

## Install

Install the `dynagentic-staff.plugin` file through Cowork's plugin management. Both the MCP connector and the skill activate together.

## One remaining manual step after install

The staff connector uses Google OAuth (Cognito PKCE). The `.mcp.json` plugin schema does not support declaring the OAuth client ID inline, so one step remains manual after install:

When Claude prompts you to authenticate the connector, select **"Use your own OAuth client"** and enter:

- **Client ID:** `1508kdjq1t75jfc74ol3p4vdu7`
- **Client secret:** (leave blank - public PKCE client, no secret)
- **Authorization URL:** `https://d3vfrcd9gs7j0x.cloudfront.net/.well-known/oauth-authorization-server`

Claude will complete the PKCE flow and open a browser window for Google sign-in. Once you have signed in with your `@dynagentic.ai` account, the connector is ready.

This is a one-time step per device. The session token is managed by Claude's connector machinery; you will not be asked to enter it again on the same device unless the token expires.

## Per-device authentication

The OAuth token is scoped to the device and session. Instructors using Claude on multiple devices will need to complete the OAuth sign-in once on each device. This is by design.

## What is NOT in this plugin

- **Student tools only**: for the student-side plugin, see `dynagentic-student.plugin`.
- **Backend logic**: all server-side exercise management, progress tracking, token administration, and team file storage lives in the deployed backend (AWS Lambda functions). This plugin is the client-side install only.
- **Allowlist management**: adding or removing users from the allowlist is an infrastructure operation, not something this plugin controls.
- **Marketplace bundling**: a combined marketplace plugin bundling both student and staff plugins together is tracked separately (LAB-01220).

## Relationship to standalone components

This plugin bundles and packages two independently maintained sources:

- **MCP connector**: the live `/mcp-staff` endpoint at CloudFront (`https://d3vfrcd9gs7j0x.cloudfront.net/mcp-staff`), deployed separately in `infra/dynagentic-connector/`.
- **Skill**: `infra/dynagentic-connector/plugins/dynagentic-staff/skills/dynagentic-staff/SKILL.md` in this repo. This is both the canonical location and the in-plugin copy (unlike the student plugin, there is no separate standalone staff skill to keep in sync with).

## Auth background

The `/mcp-staff` connector uses Cognito-fronted Google OAuth with PKCE. The auth flow that makes this work end-to-end was delivered across LAB-01201 (OAuth discovery + callback fix), LAB-01208 (access-token verification in the Lambda), and LAB-01217 (`email_verified` attribute mapping). The connector was confirmed working live on 2026-09-21 with a real Claude Desktop sign-in completing successfully and all 12 tools listed. See `infra/build-environment/docs/cognito-oauth-resource-server-gotchas.md` for the detailed auth engineering notes.
