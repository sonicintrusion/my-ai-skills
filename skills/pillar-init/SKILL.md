---
name: pillar-init
description: Bootstrap a new pillar workspace from scratch. Use when the user says "pillar-init", "init pho pillar", "set up a new pillar", or "create pillar workspace for <pillar>". Creates the folder structure and config, then runs pillar-sync.
---

# Pillar Init

## Goal

Create the local workspace structure for a new pillar and perform an initial sync from Confluence.

## Inputs

Ask the user for (or extract from their message):

1. **Pillar code** — short lowercase identifier (e.g. `pho`)
2. **Pillar name** — display name (e.g. `Platform Health and Observability`)
3. **Confluence page URL** — the roadmap page URL (e.g. `https://confluence.sie.sony.com/pages/viewpage.action?pageId=3116384383`)
4. **Jira project key** — defaults to `DSOSYS` if not specified

## Step 1: Create folder structure

```
pillar/<pillar-code>/
pillar/<pillar-code>/_archived/
```

## Step 2: Create config file

Write `pillar/<pillar-code>/config.md`:

```markdown
# Pillar config: <pillar-code>

- pillar_code: <pillar-code>
- pillar_name: <Pillar Name>
- confluence_page_id: <extracted_page_id>
- confluence_url: <full_url>
- jira_project_key: <PROJECT_KEY>
```

Extract the numeric page ID from the `pageId=` query parameter in the Confluence URL.

## Step 3: Run pillar-sync

Invoke the `pillar-sync` skill for this pillar using the same inputs. The sync will populate ROADMAP.md, epic folders, and ticket files.

## Step 4: Report

Confirm the workspace was created and report the pillar-sync summary.
