# INTEGRATION.md — ecosystem-dashboard

> Static HTML dashboard for the SuperInstance ecosystem. Deployed to
> GitHub Pages (gh-pages branch), reads live data from Supabase via
> REST, and renders six dashboard panels for fleet monitoring.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Dashboard Panels](#dashboard-panels)
3. [Supabase Tables Read](#supabase-tables-read)
4. [Data Fetching](#data-fetching)
5. [Adding New Panels](#adding-new-panels)
6. [Panel Implementation Guide](#panel-implementation-guide)
7. [Deployment to gh-pages](#deployment-to-gh-pages)
8. [Styling & Theming](#styleing--theming)
9. [Auto-Refresh](#auto-refresh)
10. [Integration with si-cli](#integration-with-si-cli)
11. [Integration with si-fleet-api](#integration-with-si-fleet-api)
12. [Environment Configuration](#environment-configuration)
13. [Testing Locally](#testing-locally)
14. [Performance](#performance)

---

## Architecture Overview

The ecosystem-dashboard is a single `index.html` file with embedded CSS
and JavaScript. No build step, no frameworks — just vanilla HTML/CSS/JS
that reads directly from the Supabase REST API.

```
ecosystem-dashboard/
└── index.html    — Complete dashboard (552 lines)
    ├── <style>   — Dark theme CSS with CSS variables
    ├── <body>    — Panel layout (stats row + 2-column grid)
    └── <script>  — Supabase REST fetch + render functions
```

### Key Design Decisions

- **No build tools** — serves directly from gh-pages
- **CSS-only pie chart** — language distribution via `conic-gradient`
- **Client-side search/sort** — repo table filtering in JS
- **60-second auto-refresh** — `setInterval(loadAll, 60000)`
- **Responsive grid** — 2-column on desktop, 1-column on mobile

---

## Dashboard Panels

The dashboard consists of six panels in a 2-column grid layout:

### 1. Stats Row (Top)

Four stat cards showing aggregate metrics:

| Card         | Source                        | Field                |
|--------------|-------------------------------|----------------------|
| Repos        | `repos` table count           | `statRepos`          |
| Languages    | Unique languages in repos     | `statLangs`          |
| Capabilities | `capabilities` table count    | `statCaps`           |
| Agents       | `fleet_budgets` row count     | `statAgents`         |

### 2. Language Pie (Left)

- **Source**: `repos` table grouped by `language`
- **Rendering**: CSS `conic-gradient` pie chart + legend
- **Function**: `renderLangPie(repos)` — counts repos per language,
  generates gradient stops and color-coded legend

### 3. Repo Table (Right)

- **Source**: `repos` table with name, language, description, url
- **Features**: Client-side search, sortable columns (name, language, description)
- **Function**: `renderRepoTable(filter)` — filters and sorts `allRepos` array
- **Language badges**: Color-coded by language (Rust=#DEA584, TypeScript=#3178C6, etc.)

### 4. Capability Cloud (Left)

- **Source**: `capabilities` table with name, category, provides
- **Rendering**: Tag cloud grouped by category, size scaled by provides count
- **Categories**: infrastructure, data, communication, intelligence, automation, security, general
- **Function**: `renderCapCloud(caps)` — groups by category, renders styled tags

### 5. Conservation Gauge (Right)

- **Source**: `fleet_budgets` table with agent_id, gamma, eta, total_budget
- **Rendering**: Stacked bar chart (gamma in blue, eta in orange) per agent
- **Conservation note**: Explains γ + η = const invariant
- **Function**: `renderBudget(budgets)` — normalizes to max total, renders bars

### 6. Event Timeline (Full Width)

- **Source**: `fleet_events` table, last 20 events ordered by created_at desc
- **Rendering**: Timeline with timestamp, event type badge, message
- **Event types**: spawn, complete, error, info, budget
- **Function**: `renderEvents(events)` — formats timestamps, color-codes types

---

## Supabase Tables Read

The dashboard queries four Supabase tables via REST:

### repos

```sql
SELECT name, description, language, url FROM repos ORDER BY name;
```

Used for: stats row (count), language pie (group by language), repo table.

### capabilities

```sql
SELECT name, category, provides FROM capabilities;
```

Used for: stats row (count), capability cloud panel.

### fleet_budgets

```sql
SELECT * FROM fleet_budgets;
```

Used for: stats row (agent count, total budget), conservation gauge bars.

### fleet_events

```sql
SELECT * FROM fleet_events ORDER BY created_at DESC LIMIT 20;
```

Used for: event timeline panel.

---

## Data Fetching

All data is fetched via the Supabase REST API using the public anon key:

```javascript
const SUPABASE_URL = 'https://igogykhksgkaxcwzudwi.supabase.co';
const SUPABASE_ANON_KEY = '...'; // public key, read-only access

async function apiFetch(table, query) {
    const url = `${SUPABASE_URL}/rest/v1/${table}?${query}`;
    const res = await fetch(url, {
        headers: {
            'apikey': SUPABASE_ANON_KEY,
            'Authorization': `Bearer ${SUPABASE_ANON_KEY}`,
        },
    });
    if (!res.ok) throw new Error(`API error: ${res.status}`);
    return res.json();
}
```

### Query Patterns

```javascript
// Repos — all, ordered by name
const repos = await apiFetch('repos', 'select=name,description,language,url&order=name');

// Capabilities — name, category, provides
const caps = await apiFetch('capabilities', 'select=name,category,provides');

// Budgets — all fields
const budgets = await apiFetch('fleet_budgets', 'select=*');

// Events — last 20, newest first
const events = await apiFetch('fleet_events', 'select=*&order=created_at.desc&limit=20');
```

---

## Adding New Panels

To add a new panel to the dashboard:

### Step 1: Add Panel HTML

Insert a new `<div class="panel">` in the dashboard grid:

```html
<div class="panel">
    <div class="panel-header">
        <span class="icon">📊</span> My New Panel
    </div>
    <div class="panel-body" id="myNewPanel">
        Loading...
    </div>
</div>
```

### Step 2: Add Render Function

```javascript
function renderMyNewPanel(data) {
    if (!data.length) {
        document.getElementById('myNewPanel').innerHTML =
            '<div class="empty">No data</div>';
        return;
    }
    // Build HTML from data
    let html = '<ul>';
    data.forEach(item => {
        html += `<li>${esc(item.name)}: ${item.value}</li>`;
    });
    html += '</ul>';
    document.getElementById('myNewPanel').innerHTML = html;
}
```

### Step 3: Fetch Data in loadAll()

Add a new try/catch block to the `loadAll()` function:

```javascript
async function loadAll() {
    // ... existing panels ...

    // My new panel
    try {
        const data = await apiFetch('my_table', 'select=*&limit=50');
        renderMyNewPanel(data);
    } catch (err) { showError('myNewPanel', err); }
}
```

### Step 4: Full-Width Panels

For panels that span both columns:

```html
<div class="panel full-width">
    <div class="panel-header">
        <span class="icon">📈</span> Fleet Timeline
    </div>
    <div class="panel-body" id="timelinePanel">Loading...</div>
</div>
```

---

## Panel Implementation Guide

### Color-Coded Tags

Use the `data-cat` attribute for category-based coloring:

```html
<span class="cap-tag" data-cat="infrastructure">my-cap</span>
<span class="cap-tag" data-cat="data">my-data-cap</span>
```

Available categories and their colors:
- `infrastructure` — purple (#8b5cf6)
- `data` — blue (#3b82f6)
- `communication` — cyan (#06b6d4)
- `intelligence` — green (#22c55e)
- `automation` — orange (#f59e0b)
- `security` — red (#ef4444)
- `general` — slate (#94a3b8)

### Bar Charts

Follow the conservation gauge pattern:

```html
<div class="budget-bar-track">
    <div class="budget-gamma" style="width:35%">γ 0.35</div>
    <div class="budget-eta" style="width:65%">η 0.65</div>
</div>
```

### Error States

Use the `showError` helper:

```javascript
function showError(panelId, err) {
    document.getElementById(panelId).innerHTML =
        `<div class="error">⚠ ${esc(err.message)}</div>`;
}
```

---

## Deployment to gh-pages

The dashboard is deployed from the `gh-pages` branch of
`SuperInstance/ecosystem-dashboard`.

### Deployment Process

```bash
# Make changes to index.html on gh-pages branch
git checkout gh-pages
# Edit index.html...
git add index.html
git commit -m "dashboard: update panel layout"
git push origin gh-pages
```

The site is live at: `https://superinstance.github.io/ecosystem-dashboard/`

### No Build Step Required

Since the dashboard is a single HTML file with no dependencies, it
deploys as-is. GitHub Pages serves `index.html` directly from the
gh-pages branch.

---

## Styling & Theming

The dashboard uses CSS custom properties for theming:

```css
:root {
    --bg: #0d0d1a;           /* Deep navy background */
    --surface: #161628;      /* Panel background */
    --surface2: #1e1e38;     /* Hover/input background */
    --border: #2a2a4a;       /* Border color */
    --purple: #8b5cf6;       /* Primary accent */
    --green: #22c55e;        /* Success/healthy */
    --blue: #3b82f6;         /* Info/gamma */
    --orange: #f59e0b;       /* Warning/eta */
    --cyan: #06b6d4;         /* Links */
    --red: #ef4444;          /* Error/violation */
    --text: #e2e8f0;         /* Primary text */
    --text-dim: #94a3b8;     /* Secondary text */
    --text-muted: #64748b;   /* Tertiary text */
    --radius: 10px;          /* Border radius */
}
```

To change the theme, modify these variables in `:root`.

---

## Auto-Refresh

The dashboard automatically refreshes every 60 seconds:

```javascript
loadAll();                        // Initial load
setInterval(loadAll, 60000);      // Refresh every 60s
```

The "Last refreshed" timestamp updates on each cycle:

```javascript
document.getElementById('lastRefresh').textContent =
    new Date().toLocaleTimeString();
```

---

## Integration with si-cli

Data written by `si-cli` appears on the dashboard:

1. **`si scan .`** syncs repos → Supabase `repos` table → Dashboard repo table updates
2. **`si audit .`** logs results → Supabase `fleet_events` → Dashboard event timeline updates
3. **`si check --from-supabase`** reads same `fleet_budgets` as dashboard conservation gauge

The dashboard is a read-only view of the data that si-cli writes.

---

## Integration with si-fleet-api

The dashboard can alternatively fetch data through si-fleet-api instead
of direct Supabase REST. The endpoints map 1:1:

| Dashboard Panel   | Direct Supabase             | Via si-fleet-api              |
|-------------------|-----------------------------|-------------------------------|
| Language Pie      | `apiFetch('repos', ...)`    | `GET /api/repos`              |
| Repo Table        | `apiFetch('repos', ...)`    | `GET /api/repos`              |
| Capability Cloud  | `apiFetch('capabilities', ...)` | `GET /api/capabilities`   |
| Conservation      | `apiFetch('fleet_budgets', ...)` | `GET /api/fleet/budgets`  |
| Events            | `apiFetch('fleet_events', ...)`  | `GET /api/fleet/events`   |

To switch, replace `apiFetch()` calls with `fetch('http://localhost:3001/api/...')`.

---

## Environment Configuration

The Supabase URL and anon key are hardcoded in `index.html`:

```javascript
const SUPABASE_URL = 'https://igogykhksgkaxcwzudwi.supabase.co';
const SUPABASE_ANON_KEY = '...';
```

To use a different Supabase instance, update these constants.

---

## Testing Locally

```bash
# Clone and checkout gh-pages
gh repo clone SuperInstance/ecosystem-dashboard
cd ecosystem-dashboard
git checkout gh-pages

# Serve locally (any static server works)
python3 -m http.server 8080
# Or: npx serve .

# Open in browser
open http://localhost:8080
```

The dashboard requires network access to Supabase to load data.

---

## Performance

- **4 parallel requests** on each load (repos, capabilities, budgets, events)
- **Lightweight rendering** — no virtual DOM, direct innerHTML updates
- **60s refresh interval** — balances freshness with API load
- **Single file** — ~552 lines, <20KB gzipped, no external dependencies
