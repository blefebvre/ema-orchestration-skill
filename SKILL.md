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

Ask the user:
> "Does the EMA chat have any previous conversation context? If so, please clear it before we start — previous context can interfere with the migration instructions. Let me know when the chat is clean."

Wait for explicit confirmation before continuing.

### Check 3: EMA Login (automated)

Navigate to {ema-base} and confirm the user is signed in (avatar/user menu visible in top-right banner).

```bash
playwright-cli tab-list
# If a tab with {ema-base} exists, snapshot it.
# If no tab exists, open one:
playwright-cli open {ema-base}
```

If the page shows a login screen, ask the user to sign in and wait. Do not proceed until authenticated.

### Check 4: GitHub Connected (automated)

Navigate to {ema-base}settings and snapshot the page.

Look for the GitHub section under **Project**:
- Green indicator dot next to the GitHub username → connected ✓
- No username or red/missing indicator → not connected

If not connected, click the GitHub connect button in the UI to initiate the OAuth flow, then wait for the green indicator before proceeding.

### Check 5: Project Configured (automated)

From the settings snapshot, read:
- **GitHub username** (e.g., `blefebvre`)
- **Project** link (e.g., `blefebvre/kr-april-30`)
- **Preview URL** from the Support section (e.g., `https://main--kr-april-30--blefebvre.aem.page`)

Store these values — you'll need them for QA comparison later.

---

## Phase 1: Launch Migration

### Step 1.1: Navigate to the Migrate/Home view

```bash
playwright-cli goto --tab=<ema-tab> {ema-base}
# Or click the Home nav link if already on the app
```

Snapshot the page and identify the URL input field and the migrate/start button.

### Step 1.2: Enter the Source URL

Fill the URL input with the provided source URL:

```bash
playwright-cli fill --tab=<ema-tab> <url-input-ref> "<source-url>"
```

Re-snapshot to confirm the value is set correctly.

### Step 1.3: Start the Migration

Click the migrate/start button:

```bash
playwright-cli click --tab=<ema-tab> <start-button-ref>
```

Take a screenshot to confirm migration has begun. Note any job ID or task ID shown in the UI — this may be needed for later status checks.

---

## Phase 2: Monitor Migration Progress

The EMA runs several sub-tasks: parsing, transformation, block generation, styles, navigation, footer. Monitor the UI for progress indicators.

### Polling Loop

Every 15–30 seconds, snapshot the page and check:
- Progress bars, step indicators, or log entries
- Any error banners or warnings
- Whether the migration has completed (a "View result", "Preview", or "Done" state)

```bash
playwright-cli snapshot --tab=<ema-tab>
playwright-cli screenshot --tab=<ema-tab> --filename=/tmp/ema-progress-<ts>.png
```

Log each status transition so you can report the full timeline to the user.

### Error Handling

If the EMA reports an error:
1. Screenshot the error state
2. Read the error message from the snapshot
3. Attempt to resolve if straightforward (e.g., re-enter a field, click retry)
4. If unresolvable, report to the user with the error text and screenshot

---

## Phase 3: Initial QA — Visual Comparison

Once the EMA signals completion, capture both the source and the EDS preview for side-by-side comparison.

### Step 3.1: Screenshot the EDS Preview

The EMA typically shows a preview pane or provides a preview URL. Identify it from the snapshot:
- Inline preview iframe → screenshot the iframe element
- External preview URL → open in a new tab, full-page screenshot

```bash
# If opening a separate preview tab:
playwright-cli open <preview-url>
# Capture targetId, then:
playwright-cli screenshot --tab=<preview-tab> --fullPage --filename=/tmp/ema-eds-preview.png
```

### Step 3.2: Screenshot the Source Page

Open the original source URL in a new tab for a fresh baseline:

```bash
playwright-cli open <source-url>
playwright-cli screenshot --tab=<source-tab> --fullPage --filename=/tmp/ema-source.png
```

### Step 3.3: Visual Diff Analysis

View both screenshots:

```bash
open --view /tmp/ema-source.png
open --view /tmp/ema-eds-preview.png
```

Systematically compare each region top-to-bottom:
- **Navigation/Header** — logo position, links, layout, colors, fonts
- **Hero/Banner** — headline, subtext, CTA buttons, background, imagery
- **Content sections** — each block's layout, spacing, typography, colors
- **Footer** — links, columns, copyright, background

For each region, note:
- ✅ Match (no visible difference)
- ⚠️ Minor gap (spacing/color off slightly)
- ❌ Major gap (layout broken, content missing, wrong font)

---

## Phase 4: Iterative Refinement

For each identified gap, issue a targeted refinement prompt to the EMA chat interface and re-evaluate. Tackle issues in this priority order:

1. **Layout breaks** (columns collapsed, elements overlapping)
2. **Missing content** (blocks absent, text not migrated)
3. **Navigation/Header** issues
4. **Footer** issues
5. **Typography** (wrong font family, size, weight)
6. **Colors** (background, text, link, button colors off)
7. **Spacing** (padding/margin differences)
8. **Imagery** (wrong image, missing alt text, sizing off)

### Issuing a Refinement Prompt

Locate the EMA chat/prompt input in the UI:

```bash
playwright-cli snapshot --tab=<ema-tab>
# Find the chat input or refinement text area
playwright-cli fill --tab=<ema-tab> <chat-input-ref> "<refinement-prompt>"
playwright-cli press --tab=<ema-tab> Enter
# Or click the send button
```

### Refinement Prompt Templates

Use specific, actionable prompts. Examples:

**Parser / content extraction:**
> "The hero section is missing the subheading text. Update the parser to extract the `.hero-subtitle` element and include it in the hero block."

**Transformer / block structure:**
> "The cards block is rendering as a single column. The source uses a 3-column grid. Update the transformer to output 3 items per row."

**Styles:**
> "The heading font should be `{font-family}` at `{size}`. The current migration is using the wrong font. Update the styles to match."

**Navigation:**
> "The navigation is missing the dropdown menus. Update the nav parser to include the second-level link items."

**Footer:**
> "The footer background color should be `{hex}`. Update the footer styles."

**Specific block:**
> "The `{block-name}` block has incorrect padding — source shows `{n}px` top/bottom, the preview shows `{n}px`. Adjust the block CSS."

### Refinement Loop

After each prompt:
1. Wait for EMA to process (monitor progress as in Phase 2)
2. Re-screenshot the preview
3. Re-compare against source
4. If gap is resolved → mark ✅ and move to next issue
5. If gap persists → refine the prompt and try again (max 3 attempts per issue)

Maximum **5 refinement rounds** total before surfacing remaining gaps to the user for manual review.

---

## Phase 5: Final Report

After refinement rounds are exhausted or all gaps are resolved, produce a final report:

### Summary Format

```
## EMA Migration Report: <source-url>

**EDS Preview:** <preview-url>
**Project:** <owner/repo>

### Results by Region

| Region | Status | Notes |
|--------|--------|-------|
| Navigation | ✅ / ⚠️ / ❌ | ... |
| Hero | ✅ / ⚠️ / ❌ | ... |
| <Block> | ✅ / ⚠️ / ❌ | ... |
| Footer | ✅ / ⚠️ / ❌ | ... |

### Refinement History
- Round 1: <what was fixed>
- Round 2: <what was fixed>
...

### Remaining Gaps
- <description of any unresolved issues>

### Next Steps
- <recommended manual follow-up actions, if any>
```

Inline both the source and preview screenshots for the user to compare visually.

---

## Tab Management

- Always track tab IDs you open: EMA tab, source tab, preview tab
- Close source and preview tabs when the migration is complete
- Never close the EMA tab unless the user asks — they may want to continue refining manually

```bash
playwright-cli tab-close --tab=<source-tab>
playwright-cli tab-close --tab=<preview-tab>
# Leave the EMA tab open
```

---

## Notes for the Cone

- This skill is **cone-only** — do not delegate to a scoop. The cone drives the browser directly.
- EMA state is ephemeral in the browser — losing the tab loses the session. Keep the EMA tab open.
- If the EMA UI changes (new navigation structure, renamed buttons), re-snapshot and adapt. Don't hardcode refs.
- The EMA project is already configured (checked in pre-flight). Do not attempt to change the project mid-migration.
- Visual comparison is subjective — aim for ≥ 90% visual fidelity before declaring success.
