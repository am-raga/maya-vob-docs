# Adding a New Agent to RagaAI Clinicops Usage Guides

This document is the complete reference for any Claude instance onboarding a new agent into this site. Read it fully before making any changes. The site is a **single static HTML file** — there is no build step, no framework, no backend. Everything lives in `index.html`.

---

## Repo Overview

```
TRAINING MANUAL/
├── index.html              ← The entire site (HTML + CSS + JS, all inline)
├── CLAUDE.md               ← Core project rules — read before touching anything
├── AGENT_ONBOARDING.md     ← This file
└── assets/
    ├── logo/
    │   └── maya-vob-logo.png
    └── images/
        ├── vob/            ← Screenshots for the VOB agent (h0-step1.png, etc.)
        │   ├── h0/
        │   ├── h1/
        │   └── ...
        └── prior-auth/     ← Placeholder — populated when agent is ready
```

**Deployed at:** https://ragaai-clinicalops-usageguides.vercel.app  
**GitHub:** https://github.com/am-raga/maya-vob-docs (branch: `main`)  
**Deploy command:** `cd "TRAINING MANUAL" && vercel --prod`

---

## Step 0 — Read CLAUDE.md First

Key rules that override everything:
- Never use the word "Maya" — brand is "RagaAI Clinical Ops"
- Commit messages: plain descriptive text only, no `Co-Authored-By` trailers
- Never expose `gid=2002806364` (internal tracking sheet)
- Login creds are intentionally hardcoded: `AdminRaga/Raga@123`, `AdminClinic/Allergy@123`

---

## Phase 1 — Gather Agent Info (Generate the CSVs)

Before touching any files, generate two CSVs from the user's description. Ask the user for:
1. The agent's name and ID (e.g. "Prior Auth" → `prior-auth`)
2. A one-sentence description of what the agent does
3. The list of sections and articles they want documented
4. Which articles have step-by-step screenshots and how many steps each has

### CSV 1: `articles.csv`

This defines every article in the new agent's guide.

**Schema:**

| Column | Type | Description |
|---|---|---|
| `agent_id` | string | Kebab-case agent identifier. e.g. `prior-auth` |
| `section_type` | enum | One of: `tutorial`, `howto`, `reference`, `explanation`, `appendix` |
| `article_id` | string | Unique short ID used in the URL hash. Prefix with agent initials to avoid collision. e.g. `pa-t1`, `pa-h0`, `pa-r1` |
| `title` | string | Display title shown in sidebar and article header |
| `lede` | string | One sentence shown under the title — what the user will learn |
| `read_min` | integer | Estimated read time in minutes |
| `version` | string | e.g. `1.0 · July 2026` |
| `has_steps` | boolean | `true` if the article has numbered screenshot steps |
| `step_count` | integer | Number of steps (0 if `has_steps` is false) |
| `sidebar_label` | string | Shorter label for the sidebar link (can match title) |

**Section type → sidebar dot colour mapping (CSS already defined):**
- `tutorial` → blue (`#3B82F6`)
- `howto` → green (`#10B981`)
- `reference` → purple (`#A855F7`)
- `explanation` → amber (`#F59E0B`)
- `appendix` → slate (`#64748B`)

**Example `articles.csv` (Prior Auth agent):**

```csv
agent_id,section_type,article_id,title,lede,read_min,version,has_steps,step_count,sidebar_label
prior-auth,tutorial,pa-t1,Get Started with Prior Auth,Your first walkthrough of the Prior Auth submission flow.,5,1.0 · July 2026,false,0,Get Started
prior-auth,howto,pa-h0,Submitting a Prior Auth Request,How to submit a new prior auth request from the patient queue.,3,1.0 · July 2026,true,4,Submitting a Request
prior-auth,howto,pa-h1,Checking Auth Status,How to look up the status of a submitted prior auth.,2,1.0 · July 2026,true,2,Checking Status
prior-auth,reference,pa-r1,Supported Payer List,All payers currently supported for automated prior auth submission.,2,1.0 · July 2026,false,0,Supported Payers
prior-auth,explanation,pa-e1,How the Auth Decision Works,Why requests are approved or pended and what the system does next.,4,1.0 · July 2026,false,0,How Auth Decisions Work
prior-auth,appendix,pa-a1,Prior Auth Terminology,Glossary of terms used throughout the Prior Auth guide.,3,1.0 · July 2026,false,0,Terminology
```

---

### CSV 2: `images.csv`

This defines every screenshot asset for the new agent. Fill this in once the user has provided the actual image files.

**Schema:**

| Column | Type | Description |
|---|---|---|
| `agent_id` | string | Same as articles.csv |
| `article_id` | string | Matches `article_id` in articles.csv |
| `step_num` | integer | Step number within the article (1-indexed) |
| `filename` | string | Just the filename. Convention: `{article_id}-step{N}.png` |
| `alt_text` | string | Screen reader / fallback text |
| `caption` | string | Shown below the image in the article (can be empty) |

**Example `images.csv`:**

```csv
agent_id,article_id,step_num,filename,alt_text,caption
prior-auth,pa-h0,1,pa-h0-step1.png,Click Prior Auth in the agents panel,Navigate to the Prior Auth section from the main portal
prior-auth,pa-h0,2,pa-h0-step2.png,The patient queue for prior auth requests,Select the patient you want to submit a request for
prior-auth,pa-h0,3,pa-h0-step3.png,The prior auth submission form,Fill in the diagnosis code and requested service
prior-auth,pa-h0,4,pa-h0-step4.png,Confirmation screen after submission,The system confirms submission and shows the reference number
prior-auth,pa-h1,1,pa-h1-step1.png,Auth status dashboard,Statuses shown as Pending Approved or Denied
prior-auth,pa-h1,2,pa-h1-step2.png,Detail view of a single auth record,Click any row to expand the full auth detail
```

---

## Phase 2 — Organize the Asset Folder

The user will supply a folder of raw image files. These are typically unorganized — arbitrary names, flat structure, maybe mixed formats. This phase turns them into the correct layout before any HTML is written.

### 2.1 — Audit the user's folder

Ask the user to share or describe the folder. List all image files:

```bash
find assets/images/raw-input -type f | sort
```

You will see something like:
```
prior-auth/raw-input/screenshot1.png
prior-auth/raw-input/Screenshot 2026-07-01 step2.png
prior-auth/raw-input/form-fill.png
...
```

### 2.2 — Map raw files to `images.csv`

Cross-reference every file in the user's folder against `images.csv`. For each row in `images.csv`, identify which raw file corresponds to it. Ask the user to clarify any ambiguous mapping — e.g. if two screenshots look like they could both be `pa-h0-step2`.

Build a rename plan:

| Raw filename | → Target path |
|---|---|
| `screenshot1.png` | `assets/images/prior-auth/pa-h0/pa-h0-step1.png` |
| `Screenshot 2026-07-01 step2.png` | `assets/images/prior-auth/pa-h0/pa-h0-step2.png` |
| `form-fill.png` | `assets/images/prior-auth/pa-h0/pa-h0-step3.png` |

Present this plan to the user and confirm before executing.

### 2.3 — Execute: create folders, rename, move

```bash
# Create folder structure
mkdir -p "assets/images/prior-auth/pa-h0"
mkdir -p "assets/images/prior-auth/pa-h1"
# one folder per article_id where has_steps = true in articles.csv

# Rename and move files per the confirmed plan
mv "path/to/screenshot1.png" "assets/images/prior-auth/pa-h0/pa-h0-step1.png"
mv "path/to/Screenshot 2026-07-01 step2.png" "assets/images/prior-auth/pa-h0/pa-h0-step2.png"
# ...
```

### 2.4 — Verify completeness

After moving, confirm every row in `images.csv` has a matching file:

```bash
# For each expected file from images.csv, check it exists
ls assets/images/prior-auth/pa-h0/
ls assets/images/prior-auth/pa-h1/
```

If any file is missing, flag it to the user before proceeding. Do not write HTML `<img>` tags for images that don't exist yet — the `onerror` handler will silently hide them on the live site, which is fine, but you should note the gap.

### 2.5 — Update `images.csv` if filenames changed

If any files were renamed during the process, update the `filename` column in `images.csv` to reflect the final filenames. The HTML in Phase 3 will be generated from this corrected CSV.

**Naming convention (canonical):**
- Pattern: `{article_id}-step{N}.png`
- Examples: `pa-h0-step1.png`, `pa-h1-step2.png`
- No spaces. No uppercase. No special characters except hyphens.
- Format: PNG preferred. JPG acceptable. No WebP (Vercel serves these fine but keep it simple).

**Target path:** `assets/images/{agent_id}/{article_id}/{filename}`  
**HTML src attribute:** `src="assets/images/{agent_id}/{article_id}/{filename}"`

---

## Phase 3 — Update `index.html`

Open `index.html`. It is one large file. Work through each section below in order. Each section is delimited by HTML comments — use these to navigate.

### 3.1 — Unlock the agent selector card

Find the **agent selector** section (search for `id="agent-selector"`). There is one card per agent. The new agent's card currently has class `locked`. Remove `locked`, add `onclick`, and update the icon colour.

**Locked card pattern (what exists for a not-yet-launched agent):**
```html
<div class="agent-card locked">
  <div class="agent-card-badge">
    <svg ...lock icon...></svg> Coming soon
  </div>
  <div class="agent-card-icon" style="background:var(--surface-soft);color:var(--text-muted);">
    ...icon svg...
  </div>
  <div class="agent-card-name">Prior Authorization</div>
  <div class="agent-card-desc">...</div>
</div>
```

**Active card pattern (after unlocking):**
```html
<div class="agent-card" onclick="selectAgent('prior-auth')">
  <div class="agent-card-icon" style="background:var(--brand-soft);color:var(--brand);">
    ...icon svg...
  </div>
  <div class="agent-card-name">Prior Authorization</div>
  <div class="agent-card-desc">Automated prior auth submission and status tracking.</div>
</div>
```

Remove the `<div class="agent-card-badge">` block entirely when unlocking.

---

### 3.2 — Update the agent switcher dropdown (topbar)

Find the `id="agent-switcher-dropdown"` div. It lists agents inline. Add an entry for the new agent and remove its `locked` class. The current active agent always keeps the `current` class at runtime — the HTML just needs the item to exist and be clickable.

**Add this item (not locked):**
```html
<div class="agent-switcher-item" onclick="closeAgentSwitcher();showAgentSelector();selectAgent('prior-auth')">
  <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><polyline points="20 6 9 17 4 12"/></svg>
  Prior Auth
</div>
```

The `current` class is managed dynamically by JS — do not hardcode it here.

> **Note:** The `current` class highlighting in the switcher is purely visual (CSS). It does not get toggled by JS in the current implementation. If you want it to reflect the active agent dynamically, add logic to `updateTopbar()` that iterates `.agent-switcher-item` elements and applies `.current` to the one matching `_currentAgent`.

---

### 3.3 — Add sidebar navigation

Find the comment block:
```
<!-- {AGENT_NAME} AGENT: Sidebar navigation -->
```

If this is a new agent being added fresh (no placeholder exists yet), add a new comment block **after** the existing agent's `END` comment, still inside `<aside class="sidebar">`:

```html
<!-- ==========================================
     PRIOR AUTH AGENT: Sidebar navigation
     ========================================== -->
<div id="sidebar-prior-auth" style="display:none;">

  <div class="sidebar-section">
    <div class="sidebar-section-label tutorial" onclick="toggleSection(this)"><span class="dot"></span> Tutorial <svg class="chevron" width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><polyline points="6 9 12 15 18 9"/></svg></div>
    <a class="sidebar-link" data-target="pa-t1" onclick="showArticle('pa-t1');return false;"><span class="article-num">T1</span>Get Started</a>
  </div>

  <div class="sidebar-section">
    <div class="sidebar-section-label howto" onclick="toggleSection(this)"><span class="dot"></span> How-To <svg class="chevron" width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><polyline points="6 9 12 15 18 9"/></svg></div>
    <a class="sidebar-link" data-target="pa-h0" onclick="showArticle('pa-h0');return false;"><span class="article-num">H0</span>Submitting a Request</a>
    <a class="sidebar-link" data-target="pa-h1" onclick="showArticle('pa-h1');return false;"><span class="article-num">H1</span>Checking Status</a>
  </div>

  <div class="sidebar-section">
    <div class="sidebar-section-label reference" onclick="toggleSection(this)"><span class="dot"></span> Reference <svg class="chevron" width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><polyline points="6 9 12 15 18 9"/></svg></div>
    <a class="sidebar-link" data-target="pa-r1" onclick="showArticle('pa-r1');return false;"><span class="article-num">R1</span>Supported Payers</a>
  </div>

</div>
<!-- ==========================================
     END PRIOR AUTH AGENT: Sidebar navigation
     ========================================== -->
```

**Important:** The sidebar for each agent is wrapped in a `<div id="sidebar-{agent_id}">`. The `showAgentSelector()` / `selectAgent()` JS functions must show/hide the correct sidebar wrapper when switching agents. See Section 3.6 for the JS changes required.

---

### 3.4 — Add article content

Find the comment block:
```
<!-- PRIOR AUTH AGENT: ... Article content ... -->
```

Add all article `<div>` blocks inside it. Each article follows this exact structure:

**Section type → kicker colours:**

| Section type | Background var | Text colour |
|---|---|---|
| `tutorial` | `var(--brand-soft)` | `var(--brand-text)` |
| `howto` | `var(--green-soft)` | `#047857` |
| `reference` | `#F3E8FF` | `#6B21A8` |
| `explanation` | `#FEF3C7` | `#92400E` |
| `appendix` | `var(--surface-soft)` | `var(--text-muted)` |

**Article shell (no steps):**
```html
<div class="article" data-id="pa-t1" style="display:none"><div class="main-inner">
  <div class="breadcrumb">
    <a href="#" onclick="return false">Help Center</a><span class="sep">/</span>
    <a href="#" onclick="return false">Tutorial · T1</a><span class="sep">/</span>
    <span class="current">Get Started with Prior Auth</span>
  </div>
  <div class="article-header">
    <div class="article-kicker" style="background:var(--brand-soft);color:var(--brand-text);"><span class="dot" style="background:var(--brand);width:5px;height:5px;border-radius:50%;"></span> Tutorial · T1</div>
    <h1 class="article-title">Get Started with Prior Auth</h1>
    <p class="article-lede">Your first walkthrough of the Prior Auth submission flow.</p>
    <div class="article-meta">
      <div class="article-meta-item"><svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="10"/><polyline points="12 6 12 12 16 14"/></svg>5 min read</div>
      <div class="article-meta-item"><svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/></svg>Version 1.0 · July 2026</div>
    </div>
  </div>
  <div class="section" style="margin-top:32px;">
    <h2 class="section-title">Section heading</h2>
    <p>Article body text goes here.</p>
  </div>
</div></div>
```

**Article with screenshot steps (add inside the article shell):**
```html
<div class="section" style="margin-top:32px;">
  <h2 class="section-title">Step heading</h2>
  <p>Instruction text for this step.</p>
  <div class="step-block">
    <div class="step-label">Step 1</div>
    <p>What the user should do in this step.</p>
    <div class="screenshot-wrap">
      <img src="assets/images/prior-auth/pa-h0/pa-h0-step1.png"
           alt="Alt text from images.csv"
           class="screenshot"
           loading="lazy"
           onerror="this.closest('.screenshot-wrap').style.display='none'">
      <p class="screenshot-caption">Caption from images.csv</p>
    </div>
  </div>
</div>
```

Wrap all new agent articles in the comment block:
```html
<!-- ==========================================
     PRIOR AUTH AGENT: Prior Authorization
     Article content — pa-t1, pa-h0, pa-h1, pa-r1, pa-e1, pa-a1
     ========================================== -->

... article divs ...

<!-- ==========================================
     END PRIOR AUTH AGENT: Prior Authorization
     ========================================== -->
```

---

### 3.5 — Update JS globals

Find the first `<script>` block. Update three variables:

**`_validArticleIds`** — append all new article IDs:
```js
// Before
var _validArticleIds = ['t1','h0',...,'a3'];

// After
var _validArticleIds = ['t1','h0',...,'a3','pa-t1','pa-h0','pa-h1','pa-r1','pa-e1','pa-a1'];
```

**`_agentLabels`** — add the new agent:
```js
var _agentLabels = {'vob': 'Insurance Verification', 'prior-auth': 'Prior Auth'};
```

**`selectAgent()`** — remove the hard guard and add sidebar switching logic:

Current (VOB-only guard):
```js
function selectAgent(agent) {
  if (agent !== 'vob') return;
  _currentAgent = 'vob';
  document.getElementById('agent-selector').style.display = 'none';
  document.getElementById('main-layout').style.display = '';
  updateTopbar('vob');
  showArticle('t1');
}
```

Updated (multi-agent):
```js
var _agentFirstArticle = {'vob': 't1', 'prior-auth': 'pa-t1'};

function selectAgent(agent) {
  if (!_agentLabels[agent]) return;
  _currentAgent = agent;
  document.getElementById('agent-selector').style.display = 'none';
  document.getElementById('main-layout').style.display = '';
  // Show only the sidebar for this agent
  Object.keys(_agentLabels).forEach(function(a) {
    var el = document.getElementById('sidebar-' + a);
    if (el) el.style.display = a === agent ? '' : 'none';
  });
  updateTopbar(agent);
  showArticle(_agentFirstArticle[agent]);
}
```

Also update the **`DOMContentLoaded`** and **`popstate`** handlers — they call `selectAgent`-equivalent logic inline. Make sure they also call the sidebar show/hide logic above when restoring from a URL hash. The easiest way is to extract the sidebar switching into a helper and call it from both places.

---

### 3.6 — Update the agent switcher `current` highlighting (optional but recommended)

In `updateTopbar(agent)`, after setting the agent name, add logic to mark the correct switcher item as active:

```js
function updateTopbar(agent) {
  // ... existing code ...
  // Highlight correct item in switcher dropdown
  document.querySelectorAll('.agent-switcher-item').forEach(function(el) {
    el.classList.remove('current');
  });
  if (agent) {
    var items = document.querySelectorAll('.agent-switcher-item');
    items.forEach(function(el) {
      if (el.textContent.trim().toLowerCase().includes((_agentLabels[agent]||'').toLowerCase())) {
        el.classList.add('current');
      }
    });
  }
}
```

---

## Special Article Types

These are available on-demand based on what the user asks for. Wire them up during Phase 3 when the user says things like "this article should pull live from a Google Sheet" or "I want a collapsible FAQ". Each is self-contained — add only what's needed for the specific article.

---

### Live Google Sheets sync

**When to use:** User says an article (typically a reference table like a payer list or code list) should stay live and auto-pull from a Google Sheet.

**What you need from the user:**
- The Google Sheet URL or Sheet ID
- The GID (tab ID — visible in the URL as `gid=XXXXXXX` when on that tab)
- The column layout (which column is which — name, status, category, etc.)
- Whether the sheet is set to "Anyone with the link can view" (required — no API key is used)

**Article HTML — add a container div where the data will render:**
```html
<div class="article" data-id="pa-r1" style="display:none"><div class="main-inner">
  <div class="breadcrumb">...</div>
  <div class="article-header">
    <h1 class="article-title">Supported Payers</h1>
    <p class="article-lede">All payers currently supported for automated prior auth submission.</p>
    ...
  </div>
  <div class="section" style="margin-top:32px;">
    <h2 class="section-title">Payer List</h2>
    <div id="pa-r1-live-data">
      <p style="color:var(--text-muted);font-size:13px;">Loading…</p>
    </div>
  </div>
</div></div>
```

The `id` on the container (`pa-r1-live-data`) is what the JS will target. Name it `{article_id}-live-data`.

**JS — add inside the existing Google Sheets `<script>` block** (the IIFE that starts with `var SHEET_ID`). The existing block already has `parseCSV`, `statusBadge`, `groupBy`, `syncNote`, and `loadSheet` helpers — reuse them.

Step 1: Add the new GID as a variable at the top of the IIFE:
```js
var PA_PAYERS_GID = '123456789'; // replace with actual GID
```

Step 2: Add the new key to the `loaded` cache object:
```js
var loaded = {r4: false, r5: false, 'pa-r1': false};
```

Step 3: Write a render function for the specific column layout:
```js
function renderPriorAuthPayers(rows) {
  var data = rows.slice(1).filter(function(r) { return r[0]; });
  // Adjust column indices to match your sheet layout
  // cols: Payer Name(0), Status(1)
  var html = syncNote();
  html += '<table class="status-table"><thead><tr><th>Payer</th><th>Status</th></tr></thead><tbody>';
  data.forEach(function(r) {
    html += '<tr><td>' + (r[0]||'') + '</td><td>' + statusBadge(r[1]) + '</td></tr>';
  });
  html += '</tbody></table>';
  return html;
}
```

Step 4: Hook into `showArticle` — add a new `if` line in the monkey-patch:
```js
window.showArticle = function(id, _noPush) {
  _origShowArticle(id, _noPush);
  if (id === 'r4') loadSheet(PAYERS_GID, 'r4-live-data', renderPayers, 'r4');
  if (id === 'r5') loadSheet(CPT_GID, 'r5-live-data', renderCPT, 'r5');
  if (id === 'pa-r1') loadSheet(PA_PAYERS_GID, 'pa-r1-live-data', renderPriorAuthPayers, 'pa-r1'); // ← add this
};
```

Step 5: Add to the DOMContentLoaded preload block inside the same IIFE:
```js
if (id === 'pa-r1') loadSheet(PA_PAYERS_GID, 'pa-r1-live-data', renderPriorAuthPayers, 'pa-r1');
```

**Status badge values:** The `statusBadge()` helper maps `'live'` → green finalized badge, anything else → grey pending badge. If the sheet uses different status words, update the badge logic or write a custom badge function.

**Important:** Never expose the internal tracking sheet GID `2002806364` — that sheet contains Stedi portal URLs with account IDs. Only wire public-facing sheets.

---

### Callout / info box

**When to use:** User wants a highlighted note, warning, or tip inside an article body.

```html
<!-- Info / tip -->
<div class="callout callout-info">
  <strong>Note:</strong> This step only applies if the payer requires pre-certification.
</div>

<!-- Warning -->
<div class="callout callout-warn">
  <strong>Important:</strong> Do not submit duplicate requests — the payer may flag the account.
</div>
```

**CSS to add** (once, in the `<style>` block near other component styles):
```css
.callout{padding:12px 16px;border-radius:var(--radius-sm);font-size:13.5px;line-height:1.55;margin:16px 0;}
.callout-info{background:#EFF6FF;border-left:3px solid var(--brand);color:#1e3a5f;}
.callout-warn{background:#FFFBEB;border-left:3px solid #F59E0B;color:#78350F;}
```

---

### Collapsible FAQ / accordion

**When to use:** User wants a list of Q&A pairs where each answer is hidden until clicked.

```html
<div class="faq-list">
  <div class="faq-item">
    <button class="faq-q" onclick="this.parentElement.classList.toggle('open')">
      What happens if the payer rejects the request?
      <svg class="faq-chevron" width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><polyline points="6 9 12 15 18 9"/></svg>
    </button>
    <div class="faq-a">
      <p>The system marks the request as Denied and surfaces a reason code. You can resubmit with corrections from the detail view.</p>
    </div>
  </div>
</div>
```

**CSS to add:**
```css
.faq-list{margin:16px 0;border:1px solid var(--border);border-radius:var(--radius-sm);overflow:hidden;}
.faq-item{border-bottom:1px solid var(--border);}
.faq-item:last-child{border-bottom:none;}
.faq-q{width:100%;display:flex;align-items:center;justify-content:space-between;padding:13px 16px;background:none;border:none;font-size:13.5px;font-weight:600;color:var(--text-primary);cursor:pointer;text-align:left;gap:12px;}
.faq-q:hover{background:var(--surface-soft);}
.faq-chevron{flex-shrink:0;transition:transform .2s;color:var(--text-muted);}
.faq-item.open .faq-chevron{transform:rotate(180deg);}
.faq-a{display:none;padding:0 16px 14px;font-size:13px;color:var(--text-secondary);line-height:1.6;}
.faq-item.open .faq-a{display:block;}
```

No JS function needed — the toggle is inline on the button.

---

### Embedded video

**When to use:** User wants to embed a Loom or YouTube walkthrough inside an article.

```html
<div class="video-wrap" style="margin:20px 0;">
  <iframe
    src="https://www.loom.com/embed/YOUR_LOOM_ID"
    frameborder="0"
    allowfullscreen
    style="width:100%;aspect-ratio:16/9;border-radius:var(--radius-sm);">
  </iframe>
</div>
```

No CSS or JS changes needed. Replace `YOUR_LOOM_ID` with the actual ID from the share URL.

---

## Phase 4 — Commit and Deploy

```bash
cd "/Users/ragaai_user/Desktop/VOBV/TRAINING MANUAL"
git add index.html assets/images/{agent_id}/
git commit -m "Add {Agent Name} agent guide"
git push origin main
vercel --prod
```

Do not commit `CLAUDE.md`, `maya-vob-docs-full.html`, or `assets/BAXTERIMAGE.jpg` unless explicitly asked.

---

## Checklist

Use this to verify completeness before deploying:

**CSVs generated:**
- [ ] `articles.csv` — all articles have unique IDs, valid section types, and non-empty ledes
- [ ] `images.csv` — step counts in articles.csv match rows in images.csv

**Asset folder:**
- [ ] `assets/images/{agent_id}/{article_id}/` folders created
- [ ] Image files placed at correct paths

**`index.html` — HTML:**
- [ ] Agent selector card: `locked` class removed, `onclick` added
- [ ] Agent switcher dropdown: new item added, `locked` class removed
- [ ] Sidebar: new `<div id="sidebar-{agent_id}">` block added with correct links
- [ ] Article comment block added with correct label
- [ ] All articles present as `<div class="article" data-id="{id}" style="display:none">`
- [ ] All screenshot `<img>` src paths match `assets/images/{agent_id}/{article_id}/{filename}`

**`index.html` — JS:**
- [ ] All new article IDs added to `_validArticleIds`
- [ ] New agent added to `_agentLabels`
- [ ] `_agentFirstArticle` map updated
- [ ] `selectAgent()` updated to handle multi-agent sidebar switching
- [ ] `DOMContentLoaded` and `popstate` handlers correctly restore sidebar state from URL

**Smoke test after deploy:**
- [ ] Agent selector screen shows new card, clicking it navigates to first article
- [ ] URL updates to `#{agent_id}/{article_id}` on navigation
- [ ] Direct URL `#{agent_id}/{article_id}` loads correctly after page refresh
- [ ] Browser back navigates correctly
- [ ] Agent switcher dropdown shows new agent, clicking it switches context
- [ ] Search indexes new agent's articles
- [ ] Clicking logo returns to agent selector
- [ ] All screenshots load (or fail silently with `onerror` hide)
- [ ] VOB agent is completely unaffected

---

## Section Type Quick Reference

| Type | Sidebar colour | Kicker style | When to use |
|---|---|---|---|
| `tutorial` | Blue | Blue pill | End-to-end walkthroughs |
| `howto` | Green | Green pill | Single-task step-by-step guides |
| `reference` | Purple | Purple pill | Tables, lists, lookup content |
| `explanation` | Amber | Amber pill | Conceptual background, "how it works" |
| `appendix` | Slate | Grey pill | Glossaries, settings, supplementary info |

## Article ID Conventions

- Prefix all IDs with a short agent code to prevent collision: `pa-` for Prior Auth, `vob-` for VOB (existing VOB IDs have no prefix — do not rename them)
- Tutorial articles: `{prefix}t1`, `{prefix}t2`, …
- How-to articles: `{prefix}h0`, `{prefix}h1`, …
- Reference articles: `{prefix}r1`, `{prefix}r2`, …
- Explanation articles: `{prefix}e1`, `{prefix}e2`, …
- Appendix articles: `{prefix}a1`, `{prefix}a2`, …
- Keep IDs lowercase, no spaces, no underscores — hyphens allowed after the prefix number is not needed
