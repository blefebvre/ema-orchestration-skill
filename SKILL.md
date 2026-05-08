---
name: ema-migrate
description: Orchestrate the Experience Modernization Agent (EMA) at aemcoder.adobe.io to migrate a URL to Edge Delivery Services. Drives the EMA UI via browser automation, QAs its output, and refines parsers, transformers, styles, navigation, footer, and blocks iteratively toward a pixel-perfect result.
allowed-tools: bash
---

# EMA-Migrate Skill

Migrate a single URL to Edge Delivery Services by orchestrating the Experience Modernization Agent (EMA). The cone drives the EMA UI, monitors progress, performs QA via visual comparison, and issues targeted refinement prompts until the migration is pixel-perfect.

## EMA Base URL

The skill supports two deployments:

| Environment | Base URL |
|-------------|----------|
| **prod** (default) | `https://aemcoder.adobe.io/` |
| **stage** | `https://excat-stage.adobe.io/` |

**Determine the environment from the user's request:**
- If the user says "stage", "on stage", "using stage", or similar → use `https://excat-stage.adobe.io/`
- Otherwise → use `https://aemcoder.adobe.io/` (prod)

Store the resolved base URL as `{ema-base}` and substitute it for **every** EMA URL reference throughout this skill (settings page, home page, chat input, etc.). Never hardcode `aemcoder.adobe.io` after this point — always use `{ema-base}`.

## Trigger

Phrases like:
- "Migrate the page `<URL>` to Edge Delivery using EMA"
- "Use EMA to migrate `<URL>`"
- "Run EMA migration on `<URL>`"
- "Migrate `<URL>` using EMA on stage" → stage deployment

The user provides a source URL. The target EDS project is read from the EMA settings (already configured).

---

## Pre-Flight Checks

Before starting a migration, **stop and confirm the following with the user**. Do not proceed until you have explicit confirmation on each point — these steps can cause data loss if skipped.

### Confirm 1: Fresh GitHub Repo

Ask the user:
> "Have you created a new GitHub repo for this migration and configured it as your project in EMA settings? If not, do that first — go to {ema-base}settings and update the Project field."

Wait for explicit confirmation before continuing.

### Confirm 2: Clear EMA Chat Context

**Do not clear the chat automatically** — previous context may contain work the user wants to keep. Instead, navigate to `{ema-base}chat/code/files` and take a screenshot. If the chat shows any previous conversation (i.e. it is not showing the "Welcome" state), ask the user:
> "The EMA chat has existing context. Should I clear it before we start? This cannot be undone."

Wait for explicit confirmation before clearing. If the chat is already in the "Welcome" state, proceed without asking.

### Check 3: EMA Login (automated)

Navigate to `{ema-base}` and confirm the user is signed in (avatar/user menu visible in top-right banner).

```bash
playwright-cli tab-list
# If a tab with {ema-base} exists, snapshot it.
# If no tab exists, open one:
playwright-cli open {ema-base}
```

If the page shows a login screen, ask the user to sign in and wait. Do not proceed until authenticated.

### Check 4: GitHub Connected (automated)

Navigate to `{ema-base}settings` and snapshot the page.

Look for the GitHub section under **Project**:
- Green indicator dot next to the GitHub username → connected ✓
- No username or red/missing indicator → not connected

If not connected, click the GitHub connect button in the UI to initiate the OAuth flow, then wait for the green indicator before proceeding.

### Check 5: Project Configured (automated)

From the settings snapshot, read:
- **GitHub username** (e.g., `blefebvre`)
- **Project** link (e.g., `blefebvre/kr-april-30`)
- **Preview URL** from the Support section (e.g., `https://main--kr-april-30--blefebvre.aem.page`)
- **Library URL** — note if this is empty (see note below)

Store these values — you'll need them for QA comparison later.

> **Library URL note:** If the Library URL field is empty, EMA will fall back to WKND block examples when generating parsers. This is acceptable but means parsers are generated without knowledge of the project's specific block conventions. Flag this to the user after the migration completes — setting a sidekick library URL will improve future parser quality.

---

## Phase 1: Launch Migration

### Step 1.1: Navigate to the chat view

The migration prompt is entered at `{ema-base}chat/code/files` — not the home page. Navigate there directly:

```bash
playwright-cli goto --tab=<ema-tab> {ema-base}chat/code/files
```

Snapshot the page. You should see:
- A chat input at the bottom ("Type a message...")
- A file browser panel on the right showing the project repo structure
- The "Welcome" message if no prior chat context exists

### Step 1.2: Send the migration prompt

Fill the chat input and send:

```bash
playwright-cli snapshot --tab=<ema-tab>
# Find the chat input ref (typically "Chat message input")
playwright-cli fill --tab=<ema-tab> <chat-input-ref> "Migrate this page <source-url> to Edge Delivery Services"
playwright-cli snapshot --tab=<ema-tab>
# Find the "Send message" button ref
playwright-cli click --tab=<ema-tab> <send-button-ref>
```

Always re-snapshot before clicking send — refs change after fill.

---

## Phase 2: Monitor Migration Progress

EMA runs a structured 7-task pipeline. Each task has multiple sub-tasks that expand dynamically as EMA discovers more work. The full pipeline typically takes 8–15 minutes end-to-end.

### Known Task Pipeline

Based on observed runs, the 7 top-level tasks are:

| # | Phase | What happens |
|---|-------|-------------|
| 1 | Project Setup | Checks `.migration` dir, `page-templates.json`, workspace config |
| 2 | Template Check | Reads schema, builds/validates homepage template |
| 3 | Site Analysis | Creates template skeleton, 1 template per site |
| 4 | Page Analysis | Scrapes the source URL, identifies sections, creates block variants, caches block context |
| 5 | Block Mapping | Maps source sections → EDS block variants (5 blocks → 5 sections typical) |
| 6 | Import Infrastructure | Generates parsers (1 per block) + site-wide transformers, bundles import script |
| 7 | Content Import | Runs the bundled import script, produces `content/index.plain.html` and an Excel report |

### Polling Strategy

Poll every **30–60 seconds** (not faster — EMA sub-tasks run for 1–3 minutes each). Use snapshots, not screenshots, to read progress text efficiently:

```bash
playwright-cli snapshot --tab=<ema-tab> | grep -E "completed|thinking|Waiting|complete|failed|error"
```

Take a screenshot only when the task count advances or something notable happens.

### Interactive Prompts ("Waiting for your input")

EMA may pause and present a radio-button choice. **Do not answer these automatically** — surface them to the user first. Known prompts:

- **"An existing migration plan for a different site was found. How should I proceed?"**
  - Options: "Start fresh" / "Keep existing artifacts"
  - Always ask the user before selecting. Default recommendation: "Start fresh" for a new migration.

When the user confirms, select the radio and click "Submit answer":

```bash
playwright-cli snapshot --tab=<ema-tab>
# Find the radio ref and submit button ref
playwright-cli click --tab=<ema-tab> <radio-ref>
playwright-cli snapshot --tab=<ema-tab>   # re-snapshot before clicking submit
playwright-cli click --tab=<ema-tab> <submit-ref>
```

### Key Status Signals to Report

Report to the user when these milestones appear in the snapshot:

- `Site Analysis complete — N template(s) created`
- `Page Analysis complete — N sections identified, N new variants created (variant-a, variant-b, ...) and variant-c reused`
- `Block Mapping complete — N blocks mapped to N sections`
- `Import Infrastructure complete (N parsers + N transformers)`
- `Import result: N/N succeeded, N failures`

### Warning: Block Library Fetch Failing

If you see: _"The block library fetch is failing — will create placeholder description files"_ — note this in your status report. It means EMA used WKND fallback examples instead of the project's sidekick library. Parser quality may be lower. Flag it in the final report.

### Error Handling

If EMA reports an error:
1. Screenshot the error state
2. Read the error message from the snapshot
3. Attempt to resolve if straightforward (e.g., click retry)
4. If unresolvable, report to the user with the error text and screenshot

---

## Phase 2.5: Verify Preview Pane

After the 7/7 task completion, navigate to `{ema-base}chat/content/preview` to check the preview pane. This is where you evaluate the imported content structure before doing visual QA.

```bash
playwright-cli goto --tab=<ema-tab> {ema-base}chat/content/preview
sleep 3
playwright-cli screenshot --tab=<ema-tab> --filename=/tmp/ema-preview-pane.png
open --view /tmp/ema-preview-pane.png
```

### Expected states

**Healthy:** A page list appears on the left (e.g. "index") and clicking it shows the rendered preview on the right.

**"No pages found" / "No HTML files available":** The preview pane UI has not synced with the repo. This does **not** necessarily mean the import failed — the file may exist in the repo but the preview pane hasn't picked it up. Distinguish the two cases:

1. **Check if the file actually exists** — look for `content/index.plain.html` in EMA's completion message. If EMA reported "1/1 succeeded" and listed `content/index.plain.html` as an artifact, the file exists.

2. **If the file exists but the preview pane is empty** — this is a UI sync issue. Send the following prompt to trigger a clean re-import (EMA will verify the file, delete it, and re-run to confirm the pipeline end-to-end):

   > "No pages were imported. Please re-import the page and ensure a `index.plain.html` file is produced by the import infrastructure."

   Monitor for completion (typically 1–2 minutes). After re-import, try reloading the page. If the preview pane still shows "No HTML files available" after a reload and re-import, note it as a known stage UI sync issue and proceed to inspect the file directly via the file browser tab (`{ema-base}chat/code/files`) or GitHub API instead.

3. **If EMA reported failures** — the file genuinely wasn't produced. Use the same re-import prompt above, then monitor more carefully for errors in the task log.

### Two preview pane messages — distinction

| Message | Meaning |
|---------|---------|
| "No pages found" | Page list loaded but empty — UI hasn't synced yet |
| "No HTML files available. Ask to migrate a webpage." | Preview pane hasn't loaded at all, or no `.plain.html` files registered |

Both are recovered by the re-import prompt above. The re-import is fast (< 2 min) and idempotent — EMA reuses existing parsers/transformers.

---

## Phase 3: Review Migration Output

When the import completes (7/7 tasks), EMA produces a structured summary. Read it carefully from the snapshot — it contains the block mapping decisions that drive all subsequent QA.

### Step 3.1: Read the Block Mapping Summary

EMA's completion message lists:
- Each source section and the block variant it was mapped to
- Whether each variant is new or reused (with similarity %)
- All generated artifacts and their paths

Extract and report this to the user before proceeding to QA. Example format:

```
**5 sections → 5 blocks:**
1. Hero carousel → `carousel-banner` (new)
2. News + promo tiles → `columns-news` (new)
3. Three feature tiles → `cards-link` (reused, 88% similarity)
4. Home page tabs → `tabs-home` (new)
5. E-News signup + Connect With Us → `columns-toolbar` (new)
```

### Step 3.2: Inspect the Generated Content

Open the imported content file in the EMA file browser, or read it via the GitHub API:

```bash
PAT=$(cat /workspace/secrets/github-pat.txt)
# Get the raw content/index.plain.html from the repo
curl -s -H "Authorization: Bearer $PAT" \
  "https://api.github.com/repos/<owner>/<repo>/contents/content/index.plain.html" \
  | jq -r '.content' | base64 -d | head -100
```

Check the block table structure for each block:
- Does the number of columns match the EDS convention for that block type?
- Is all expected content present (headings, body text, images, links)?
- Are there any obviously malformed rows?

### Step 3.3: Screenshot the Source Page

Open the original source URL for visual baseline:

```bash
playwright-cli open <source-url>
# Capture targetId, then:
playwright-cli screenshot --tab=<source-tab> --fullPage --filename=/tmp/ema-source.png
open --view /tmp/ema-source.png
```

### Step 3.4: Screenshot the EDS Preview

Construct the preview URL from the project settings:
`https://main--<repo>--<owner>.aem.page/content/index`

```bash
playwright-cli open <preview-url>
playwright-cli screenshot --tab=<preview-tab> --fullPage --filename=/tmp/ema-preview.png
open --view /tmp/ema-preview.png
```

### Step 3.5: Visual Diff Analysis

Compare both screenshots systematically top-to-bottom:

| Region | Check |
|--------|-------|
| Navigation/Header | Logo position, links, layout, colors, fonts |
| Hero/Banner | Headline, subtext, CTA buttons, background, imagery |
| Each content block | Layout, column structure, spacing, typography, colors |
| Footer | Links, columns, copyright, background |

For each region, note:
- ✅ Match
- ⚠️ Minor gap (spacing/color slightly off)
- ❌ Major gap (layout broken, content missing, wrong structure)

---

## Phase 4: Iterative Refinement

For each identified gap, issue a targeted refinement prompt to the EMA chat and re-evaluate. Tackle in this priority order:

1. **Wrong block model** (section mapped to wrong block type)
2. **Parser structure** (wrong rows/columns, missing content)
3. **Missing content** (blocks absent, text not extracted)
4. **Navigation/Header** issues
5. **Footer** issues
6. **Typography** (wrong font family, size, weight)
7. **Colors** (background, text, link, button colors off)
8. **Spacing** (padding/margin differences)
9. **Imagery** (wrong image, missing alt text, sizing off)

### Sending a Refinement Prompt

```bash
playwright-cli snapshot --tab=<ema-tab>
playwright-cli fill --tab=<ema-tab> <chat-input-ref> "<refinement-prompt>"
playwright-cli snapshot --tab=<ema-tab>   # re-snapshot to get fresh send-button ref
playwright-cli click --tab=<ema-tab> <send-button-ref>
```

### Refinement Prompt Templates

**Wrong block model:**
> "The `{section-name}` section was mapped to `{block-name}` but it should use `{correct-block}` instead. The content has {N} items each with an image, heading, and description — that's a collection pattern, not a {wrong-pattern}. Please remap and regenerate the parser."

**Parser structure — wrong columns:**
> "The `{block-name}` parser is producing a single-column table. The source has {N} items side-by-side. Update the parser to output one row per item with columns: [image | heading | description | link]."

**Parser structure — missing content:**
> "The `{block-name}` parser is not extracting the {element} text. The source selector is `{css-selector}`. Update the parser to include it as a new column in the block table."

**Transformer issue:**
> "The `{block-name}` transformer is wrapping images in an extra `<div>`. Remove the wrapper so images are direct children of their table cells."

**Navigation:**
> "The navigation is missing the dropdown menus. Update the nav parser to include the second-level link items under each top-level nav item."

**Footer:**
> "The footer background color should be `{hex}`. Update the footer styles."

**Styles:**
> "The heading font should be `{font-family}` at `{size}`. Update the styles to match the source."

### Refinement Loop

After each prompt:
1. Monitor EMA progress (as in Phase 2) — refinement runs typically complete in 2–5 minutes
2. Re-screenshot the preview
3. Re-compare against source
4. If gap resolved → mark ✅ and move to next issue
5. If gap persists → refine the prompt and retry (max 3 attempts per issue)

Maximum **5 refinement rounds** total before surfacing remaining gaps to the user for manual review.

---

## Phase 5: Final Report

```
## EMA Migration Report: <source-url>

**EDS Preview:** <preview-url>
**Project:** <owner/repo>
**Environment:** prod | stage

### Block Mapping
| Source Section | EDS Block | Status |
|---------------|-----------|--------|
| ... | ... | new / reused (N% similarity) |

### QA Results
| Region | Status | Notes |
|--------|--------|-------|
| Navigation | ✅ / ⚠️ / ❌ | ... |
| Hero | ✅ / ⚠️ / ❌ | ... |
| <Block> | ✅ / ⚠️ / ❌ | ... |
| Footer | ✅ / ⚠️ / ❌ | ... |

### Refinement History
- Round 1: <what was fixed>
- Round 2: <what was fixed>

### Warnings
- <e.g. block library URL not set — parser quality may be lower>

### Remaining Gaps
- <description of any unresolved issues>

### Next Steps
- <recommended manual follow-up actions>
```

---

## Tab Management

- Track tab IDs for: EMA tab, source tab, preview tab
- Never close the EMA tab unless the user asks
- Close source and preview tabs when done

```bash
playwright-cli tab-close --tab=<source-tab>
playwright-cli tab-close --tab=<preview-tab>
# Leave the EMA tab open
```

---

## Notes for the Cone

- This skill is **cone-only** — do not delegate to a scoop. The cone drives the browser directly.
- EMA state is ephemeral — losing the tab loses the session. Keep the EMA tab open throughout.
- **Always re-snapshot before clicking** — refs are invalidated after every fill or navigation.
- **Use `snapshot | grep`** for polling, not screenshots — faster and doesn't flood the conversation.
- If the EMA UI changes (new nav structure, renamed buttons), re-snapshot and adapt. Never hardcode refs.
- The EMA project is already configured (verified in pre-flight). Do not attempt to change it mid-migration.
- Visual comparison is subjective — aim for ≥ 90% visual fidelity before declaring success.
- The first phase focus is **content structure** — correct block model selection and parser row/column output. Visual styling comes after structure is confirmed correct.
