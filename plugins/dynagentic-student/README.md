# Dynagentic Student Plugin

A one-click Cowork plugin that bundles the Dynagentic workshop student connector and the dynagentic-student skill together. Students previously needed two separate manual steps to get set up; this plugin reduces that to a single install.

## What this plugin includes

| Component | What it does |
|-----------|-------------|
| **MCP connector** (`dynagentic-student`) | Connects Claude to the Dynagentic workshop backend at `https://d3vfrcd9gs7j0x.cloudfront.net/mcp-student`. Exposes 7 student tools: `get_exercise_bundle`, `get_course_bundle`, `report_progress`, `check_broadcast`, `read_team_file`, `write_team_file`, `get_team_bundle`. |
| **Skill** (`dynagentic-student`) | Guides the student through setting up their roster token (via the connector's `Authorization` header, a `?token=` query parameter, or a per-call token argument, in that priority order), then uses the workshop tools automatically once authenticated. |

## One remaining prerequisite

After installing this plugin, students still need their personal **roster token** from their trainer, and one manual step to authenticate: setting it as the connector's `Authorization: Bearer <token>` request header (Settings → Connectors/Integrations → dynagentic-student → Edit) is the preferred path. If the client doesn't support setting a custom header, the token can instead be appended as a `?token=` query parameter on the connector URL, or passed as a per-call `token` tool argument as a last resort. The skill walks the student through exactly where to enter it, but the student (or trainer, on their behalf) must do the entering.

## Install

Install the `dynagentic-student.plugin` file through Cowork's plugin management. Both the MCP connector and the skill activate together. The token step above still needs to happen separately, once per device.

## Per-device token setup

The roster token is set once per device, via whichever of the three paths above the client supports (header preferred). Students using Claude on multiple devices (e.g. laptop and phone) will need to set it up once on each device. This is by design, as there is no cross-device sync. Trainers should let their cohort know this is expected.

## Team pinning

Students are automatically scoped to the team on their own roster row for `read_team_file`, `write_team_file`, and `get_team_bundle` -- there is no `team` argument for a student to pass. A student with no team assigned yet gets a clear "no team assigned" error telling them to ask their trainer, who assigns a team via the staff `edit_student` tool.

## What is NOT in this plugin

- **Staff/trainer tools**: the trainer endpoint uses a separate auth flow and is bundled separately. See the `dynagentic-staff` plugin.
- **Backend logic**: all server-side exercise management, progress tracking, and team file storage lives in the deployed backend (AWS Lambda functions). This plugin is the client-side install only.
- **Token generation**: tokens are issued by trainers via the staff tools. Students cannot generate their own tokens.
- **Team assignment**: students cannot set or change their own team. Only a trainer can, via `create_student`/`edit_student` on the staff connector.

## Relationship to standalone components

This plugin bundles and reuses content from two independently maintained sources:

- **MCP connector**: the live `/mcp-student` endpoint at CloudFront (`https://d3vfrcd9gs7j0x.cloudfront.net/mcp-student`), deployed separately in `infra/dynagentic-connector/`.
- **Standalone skill**: `infra/dynagentic-connector/skills/dynagentic-student/SKILL.md` in this repo. The same content is mirrored into this plugin's `skills/dynagentic-student/SKILL.md`.

The standalone skill source is the canonical reference. This plugin does not replace it; it packages it for one-click delivery.
