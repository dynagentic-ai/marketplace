---
name: dynagentic-student
description: Student toolkit for Dynagentic workshops. Guides first-time token setup (Authorization header, ?token= query parameter, or a per-call token argument, in that priority order), then uses the workshop tools automatically once authenticated. Covers all 7 student tools, including get_team_bundle, and the pinned-team model (students never name their own team; the connector assigns it from the roster). Use this skill whenever a student wants to work with workshop exercises, report progress, check for broadcasts, read or write shared team files, or do anything with the Dynagentic connector tools. Handles first-time setup when a student says things like "connect me", "my code is ...", "set up my workshop access", "I have my token", or just provides a token-looking string. On a new device, walks the student through configuring their token once - that is expected and designed.
---

# Dynagentic Student Skill

This skill gives you a smooth workshop experience. Your roster token can be set up once per device, and after that every Dynagentic tool call in your chats is authenticated automatically -- you never have to provide the token again in normal use.

## What is installed with this plugin

This plugin bundles two components in a single install:

1. **The `/mcp-student` MCP connector** - the actual connection to the workshop backend, pointing at `https://d3vfrcd9gs7j0x.cloudfront.net/mcp-student`. Installing this plugin adds it automatically.

2. **This skill** - guides token setup and tells Claude how to use the connector correctly.

If you installed this plugin, both components are active. If you see a message that workshop tools (`get_exercise_bundle`, `report_progress`, etc.) are not available, check that the plugin is fully installed and enabled.

## How authentication works (LAB-01225, LAB-01340)

The connector accepts your roster token in **three ways**, tried in this priority order on every call, including the initial tool-discovery handshake:

1. **`Authorization: Bearer <token>` request header** on the connector itself. Preferred -- set once per device, then every call in every chat is authenticated automatically. This is checked first.
2. **`?token=<token>` query parameter** appended to the connector URL. Use this when the client UI does not expose a way to set a custom request header on a connector (e.g. some mobile or embedded surfaces). Checked only if no header was presented.
3. **A per-call `token` argument** passed directly on the tool call itself. This is the fallback of last resort -- use it only when neither of the above is available to you in the current client, since it means re-supplying the token on every call. Used only if steps 1 and 2 both come back empty.

Try them in that order: header first, then the query parameter, then the per-call argument. Whichever path is used, once a valid token is accepted the call proceeds identically -- there is no functional difference in what the student can do.

- An unauthenticated connector (no header, no query token, and no per-call token argument) cannot even list the available tools.
- Once one of the three paths is satisfied, all seven workshop tools are available.

## First-time setup

Run this flow when the student says something like "connect me", "my code is ...", "set up my workshop access", "I have my token", or similar phrases -- or when a tool call returns an auth error.

### Step 1 -- get the token

Ask the student: "To connect you to the Dynagentic workshop tools, I need your roster token - your trainer should have given you a short code. What is it?"

### Step 2 -- set up the token, preferring the header path

Once the student provides their token, try the paths in priority order:

**Preferred: configure the connector header (one-time, per device)**

**In Claude Desktop:**
1. Open **Settings** → **Integrations** → find **dynagentic-student** → click **Edit** (or the pencil icon).
2. Under **Request headers**, add:
   - Header name: `Authorization`
   - Header value: `Bearer <their-token>` (replace `<their-token>` with their actual token)
3. Save and close.

**In claude.ai:**
1. Open **Settings** → **Connectors** → find **dynagentic-student** → click **Edit**.
2. Under **Headers**, add:
   - Name: `Authorization`
   - Value: `Bearer <their-token>`
3. Save.

**Alternatively**, if the student's device has the `DYNAGENTIC_STUDENT_TOKEN` environment variable set (e.g. in their shell profile), the plugin's `.mcp.json` will pick it up automatically.

**Fallback 1: `?token=` query parameter**

If the student's client does not offer a way to set a custom request header on the connector, guide them to append the token as a query parameter on the connector URL instead, e.g. `https://d3vfrcd9gs7j0x.cloudfront.net/mcp-student?token=<their-token>`. This is set once on the connector URL itself, similar to the header path.

**Fallback 2: per-call `token` argument**

If neither a custom header nor editing the connector URL is available in the current client, pass `token` directly as an argument on each tool call. Do this only as a last resort -- it means supplying the token on every single call rather than once per device. Tell the student this may happen if their client is unusually restrictive.

### Step 3 -- confirm

Once one of the three paths above is set up, confirm the outcome plainly, matching whichever path was actually used. For the header or query-parameter paths: "Done - your token is now set on this device. You will not need to enter it again here." For the per-call argument fallback: "Your client doesn't support a custom header or connector URL here, so I'll include your token on each workshop tool call instead."

Do not proceed without the token. If the student is not sure what their token is, tell them to ask their trainer.

## Using the workshop tools

Once one of the three token paths above is satisfied, all seven student tools are available. In the normal (header or query-parameter) case you do NOT pass a token as a tool argument -- authentication is already handled. Only pass `token` as a tool argument when using the per-call fallback from Step 2.

| Tool | What it does | Required arguments |
|------|--------------|--------------------|
| `get_exercise_bundle` | Fetch exercise files for a given exercise | `exercise_id` |
| `get_course_bundle` | Fetch the whole course at once (all exercises + shared files) | _(none)_ |
| `report_progress` | Report a progress event and return the current broadcast | `exercise`, `principle`, `event`, `detail` |
| `check_broadcast` | Report progress and check for any broadcast or overwrite notices | `exercise`, `principle`, `event`, `detail` |
| `read_team_file` | Read a shared file from your team's folder | `file` |
| `write_team_file` | Write or update a shared file in your team's folder | `file`, `content` (base_version optional) |
| `get_team_bundle` | Fetch every file in your team's folder in one call | _(none)_ |

**No `team` argument on any tool, ever.** You are automatically pinned to the team on your own roster row (LAB-01339 / LAB-01364). `read_team_file`, `write_team_file`, and `get_team_bundle` never take a `team` parameter for a student -- do not pass one, and do not offer to. (Only an instructor sign-in can pass an explicit `team`, to inspect a team other than their own.)

## When a team-file tool says "No team assigned"

If `read_team_file`, `write_team_file`, or `get_team_bundle` returns an error like "No team assigned: your roster entry has no team yet, so team files are unavailable":

1. Tell the student plainly: "You haven't been assigned to a team yet, so team files aren't available to you. Ask your trainer to assign you to a team."
2. Do **not** guess or suggest a team name, and do not retry with a made-up `team` value -- students cannot set their own team, and there is no tool argument for a student to work around this.
3. Once the trainer assigns a team (via `edit_student`), retry the same call with no changes needed on the student's side -- the connector will pick up the new assignment automatically.

## Setting up the workshop materials locally (get_course_bundle)

`get_course_bundle` now returns a CloudFront **download URL** to a pre-built zip
containing the full course (LAB-01299 caching). The response looks like:

```json
{
  "download_url": "https://d3vfrcd9gs7j0x.cloudfront.net/marketplace/course-bundle/abw_v2.zip",
  "generated_at": "2026-09-24T14:00:00Z",
  "cache_hit": true,
  "force_regenerated": false
}
```

Use it at the start of a workshop run to download and extract everything to the student's local machine at once, so every exercise's Cowork Project can load its files directly from disk.

### Step 1 -- fetch the bundle URL

Call `get_course_bundle` (no arguments needed). The tool returns a `download_url` pointing at the cached zip.

### Step 2 -- download and extract the zip

Download the zip from `download_url` and extract to the canonical local destination:

- **macOS:** `~/Documents/Dynagentic-ABW/` (i.e. `/Users/<username>/Documents/Dynagentic-ABW/`)
- **Windows:** `%USERPROFILE%\Documents\Dynagentic-ABW\` (i.e. `C:\Users\<username>\Documents\Dynagentic-ABW\`)

To detect the OS, check `platform.system()` in Python: `"Darwin"` = macOS, `"Windows"` = Windows, `"Linux"` = Linux (use `~/Documents/Dynagentic-ABW/` as a reasonable default on Linux too). Do not hardcode a Unix-style path when running on Windows.

The zip layout mirrors the Drive folder structure:
- `exercises.html`, `PRE-WORK.md`, `Welcome-to-SEG-pre-read_v2.docx` at the root
- `Shared/cowork-project-setup.md`, `Shared/Environment_Setup_Guide.md`, etc.
- `P1/CLAUDE.md`, `P1/exercise.md`, ..., `P10/skills/review/SKILL.md`, ...

### Step 3 -- create a Cowork Project for each exercise

Once the bundle is on disk, the student creates a new Claude Cowork Project for whichever exercise they are starting. Each exercise is its own project, pointed at its own subfolder:

- In Cowork (Claude Desktop or claude.ai), create a new project.
- Set the project's **folder** to the exercise subfolder, e.g.:
  - macOS: `~/Documents/Dynagentic-ABW/P1/`
  - Windows: `%USERPROFILE%\Documents\Dynagentic-ABW\P1\`
- Cowork automatically loads `CLAUDE.md` from that folder as the project instructions.
- The student can now work through that exercise with all its files available in-context.

The recommended pattern is one parent folder (`Dynagentic-ABW/`) with each exercise (`P1`, `P2`, ...) as its own Cowork Project. The student only needs to fetch and extract the bundle once per course version; after that, they create a new project per exercise as they progress.

### IMPORTANT: re-running setup against an existing folder

**Never blindly re-extract over a non-empty destination.** If `~/Documents/Dynagentic-ABW/` (or the Windows equivalent) already exists and has content, extracting into it again will silently overwrite any file in the bundle -- including ones the student may have annotated or modified. This has been confirmed empirically: a local edit to `P1/CLAUDE.md` was silently destroyed with no warning when setup was re-run (LAB-01312, 2026-09-25).

**What to do instead:**
- Before extracting, check whether the destination folder already has content.
- If it does, extract the fresh bundle into a **new, distinctly-named folder** (e.g. `Dynagentic-ABW-2026-09-25`) instead of overwriting in place. This completely avoids any data loss.
- Tell the student clearly: "Your existing `Dynagentic-ABW/` folder was left untouched. The fresh bundle is now in `Dynagentic-ABW-2026-09-25/`. If you want to pick up any updated course files, compare the two folders and copy only what you need."
- If the destination is empty or brand-new, proceed normally with the original folder name.

The example below implements this check automatically.

### Example (Python, cross-platform)

```python
import os, platform, urllib.request, zipfile, datetime
from pathlib import Path

def setup_course_bundle(tool_response):
    """Download and extract the course bundle from a get_course_bundle tool response.

    If the canonical destination already has content, extracts to a fresh dated
    subfolder instead of overwriting in place -- protecting any local edits the
    student may have made (LAB-01312).
    """
    download_url = tool_response["download_url"]
    generated_at = tool_response.get("generated_at", "unknown")

    system = platform.system()
    if system == "Windows":
        base = Path(os.environ["USERPROFILE"]) / "Documents" / "Dynagentic-ABW"
    else:
        base = Path.home() / "Documents" / "Dynagentic-ABW"

    # Protect existing local edits: if the destination already has content, extract
    # to a fresh dated folder instead of overwriting in place.
    if base.exists() and any(base.iterdir()):
        date_tag = datetime.date.today().isoformat()
        dest = base.parent / f"Dynagentic-ABW-{date_tag}"
        # If the dated folder also already exists with content, add a counter suffix.
        counter = 1
        while dest.exists() and any(dest.iterdir()):
            dest = base.parent / f"Dynagentic-ABW-{date_tag}-{counter}"
            counter += 1
        fresh_dest = True
    else:
        dest = base
        fresh_dest = False

    dest.mkdir(parents=True, exist_ok=True)

    # Download the zip
    zip_path = dest / "abw_v2.zip"
    urllib.request.urlretrieve(download_url, zip_path)

    # Extract
    with zipfile.ZipFile(zip_path) as zf:
        zf.extractall(dest)

    zip_path.unlink()  # remove the zip after extraction

    if fresh_dest:
        print(
            f"NOTICE: Your existing {base} folder was left untouched to protect your local work.\n"
            f"Fresh bundle (generated {generated_at}) extracted to: {dest}\n"
            f"To pick up updated course files, compare {dest} with {base} and copy only what you need."
        )
    else:
        print(f"Course bundle (generated {generated_at}) extracted to: {dest}")

    return str(dest)
```

**Note on `generated_at`:** the `generated_at` timestamp tells you when the cached zip was last built. The zip is refreshed automatically whenever the course content changes -- you do not need to do anything special to get an up-to-date bundle; every call checks for changes. If you need a guaranteed-fresh regeneration (e.g., after a known content edit), call `get_course_bundle` with query parameter `force_regenerate=true`.

## When a tool call returns an authentication error

If the connector returns a 401 or an error message like "invalid or inactive token":

1. Tell the student: "Your workshop token was rejected. Your trainer may have issued a new one. Can you check with them and give me the updated code?"
2. Guide them through Step 2 above to update whichever of the three token paths they are using with the new token.
3. Retry the failed operation.

## New device - no token path configured yet

If a student is starting on a device where none of the three token paths has been set up yet, this is expected. Walk them through first-time setup above. There is no error to apologise for -- re-entering the token on a new device is the documented, designed path.
