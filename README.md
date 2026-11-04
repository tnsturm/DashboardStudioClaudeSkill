# Dashboard Studio skill for Claude (Homey)

A Claude skill that builds, changes and repairs importable JSON dashboards for the Homey app
**Dashboard Studio** by Satoer (`com.satoer.dashboardstudio`).
Claude reads your devices and insights through the **MCP AI Bridge** Homey app, plans the pages with you,
generates the dashboard JSON with a build script, tests it in your Dashboard Studio editor and saves it
after you confirm. It also covers data feeds (HomeyScript → `homey/event/…`), the rewritten graph widget,
design rules for readable dashboards and how to keep the number of browser approvals low.

**Skill version 2.3.2** – the skill version matches the Dashboard Studio version it was verified against.
Tested with MCP AI Bridge 5.2.8. See `skills/dashboard-studio-json/SKILL.md` for what changed.

## Requirements

- Homey Pro with the apps **Dashboard Studio** and **MCP AI Bridge** (`community.mcp-ai-bridge`), the bridge connected to Claude
- For testing in the editor: a browser Claude can drive on your home network (the built-in browser of the Claude desktop app, or Claude in Chrome). Without it Claude hands over the JSON for manual import.

## Install

### Claude Code – as a plugin (recommended, with updates)

```
claude plugin marketplace add tnsturm/DashboardStudioClaudeSkill
claude plugin install dashboard-studio-json@tnsturm-homey
```

Update later with `claude plugin update dashboard-studio-json@tnsturm-homey`.

### claude.ai (web, desktop app) – upload the ZIP

1. Download `dashboard-studio-json.zip` from the latest [release](https://github.com/tnsturm/DashboardStudioClaudeSkill/releases/latest).
   Do not use GitHub's "Code → Download ZIP": its folder is named after the repository, but the ZIP for claude.ai must contain one folder named like the skill.
2. Upload it as a custom skill in claude.ai (Customize → Skills) and enable it.
3. For a new version, upload the new ZIP and replace the old skill.

### Claude Code – without the plugin system

Unzip the release ZIP into `~/.claude/skills/` (result: `~/.claude/skills/dashboard-studio-json/SKILL.md`),
or copy the folder `skills/dashboard-studio-json` from a clone of this repository there.

## Repository layout

```
.claude-plugin/plugin.json        plugin manifest (name, version)
.claude-plugin/marketplace.json   marketplace "tnsturm-homey" listing this plugin
skills/dashboard-studio-json/     the skill: SKILL.md + references/
.github/workflows/release.yml     builds the release ZIP
```

## Releasing a new version

1. Update the skill and set the same version in `SKILL.md` ("Skill version x.y.z") and `.claude-plugin/plugin.json`.
2. Merge to `main`, then push a tag: `git tag v2.3.2 && git push origin v2.3.2`.
3. The workflow checks that tag and both version numbers agree, builds `dashboard-studio-json.zip` and attaches it to the release.

## Credits

Design rules in `references/design.md` adapt principles from
[rubenhassid1/grenade](https://github.com/rubenhassid1/grenade) by Ruben Hassid (MIT license).
Dashboard Studio is developed by Michel Satoer; this skill is an independent community project.
