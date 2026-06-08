# ecosystem-dashboard

**Live dashboard for the SuperInstance ecosystem.** Real-time fleet monitoring with conservation gauges, capability clouds, event timelines, and repo analytics. Hosted at [superinstance.github.io/ecosystem-dashboard](https://superinstance.github.io/ecosystem-dashboard).

---

## Live Dashboard

The dashboard is a single-page application deployed on GitHub Pages from the `gh-pages` branch. It queries the Supabase backend to display:

- **5 stat cards** — Repos, Languages, Capabilities, Agents, Total Budget
- **Language Breakdown** — Donut chart showing repo distribution by language
- **Conservation Gauge** — Per-agent γ/η budget bars with invariant verification
- **Fleet Repositories** — Searchable, sortable table of all ecosystem repos
- **Capability Cloud** — Tag cloud of all capabilities, color-coded by category
- **Fleet Events** — Real-time event timeline from `fleet_events` table
- **Test Coverage** — Test count metrics across repos
- **Conservation Gauge (Enhanced)** — Fleet-wide conservation health with circular gauge
- **Ecosystem Timeline** — Chronological view of ecosystem changes
- **Language FFI Matrix** — Cross-language FFI compatibility grid

---

## Quick Start

```bash
git clone https://github.com/SuperInstance/ecosystem-dashboard.git
cd ecosystem-dashboard
git checkout gh-pages

# Open locally
open index.html
# or
python -m http.server 8000
# then visit http://localhost:8000
```

No build step required — it's a single `index.html` file with embedded CSS and JavaScript.

---

## The 10 Panels

### 1. Stat Cards (Top Row)

Five summary cards showing key metrics:

| Card | Source | Display |
|------|--------|---------|
| Repositories | `repos` table count | Total number of ecosystem repos |
| Languages | `repos.language` distinct count | Number of programming languages |
| Capabilities | `capabilities` table count | Total registered capabilities |
| Agents | `fleet_budgets` row count | Active fleet agents |
| Total Budget (γ+η) | `fleet_budgets` sum | Fleet-wide conservation budget |

**Supabase queries:**

```sql
SELECT count(*) FROM repos;
SELECT count(DISTINCT language) FROM repos;
SELECT count(*) FROM capabilities;
SELECT count(*) FROM fleet_budgets;
SELECT sum(total_budget) FROM fleet_budgets;
```

### 2. Language Breakdown (🎨)

A CSS-only donut chart showing repo distribution by language:

- **Rust** — Core ecosystem crates
- **Python** — Runtime bindings and analysis
- **TypeScript** — API and tooling
- **Go** — Runtime implementation
- **C** — Embedded and WASM targets

**Color scheme:**

| Language | Color |
|----------|-------|
| Rust | `#dea584` |
| Python | `#3572A5` |
| C | `#555555` |
| TypeScript | `#3178c6` |
| Go | `#00ADD8` |

**Supabase query:**

```sql
SELECT language, count(*) FROM repos GROUP BY language;
```

### 3. Conservation Gauge (⚡)

Per-agent budget visualization showing the γ/η split:

Each agent gets a horizontal bar:
- **Blue segment** — Gamma (productive budget)
- **Orange segment** — Eta (entropy/waste budget)
- **Total** — Right-aligned sum

Conservation status is verified client-side: `gamma + eta ≈ total_budget`.

**Supabase query:**

```sql
SELECT agent_id, total_budget, gamma, eta FROM fleet_budgets ORDER BY total_budget DESC;
```

### 4. Fleet Repositories (📦)

Full-width searchable table of all ecosystem repos with:

- **Search bar** — Filter by name or description (client-side)
- **Sortable columns** — Click headers to sort
- **Language badges** — Color-coded by language
- **GitHub links** — Direct links to each repo

Columns: Name, Language, Description, URL

**Supabase query:**

```sql
SELECT * FROM repos ORDER BY name;
```

### 5. Capability Cloud (🧩)

Tag cloud of all registered capabilities, color-coded by category:

| Category | Color | Examples |
|----------|-------|---------|
| infrastructure | Purple | `cli`, `fleet-management`, `rest-api` |
| data | Blue | `conservation`, `budget-enforcement` |
| communication | Cyan | `network`, `http`, `rpc` |
| intelligence | Green | `compute`, `analysis` |
| automation | Orange | `scheduling`, `ci` |
| security | Red | `crypto`, `auth` |
| general | Gray | Uncategorized |

Tags scale slightly on hover for interactivity.

**Supabase query:**

```sql
SELECT * FROM capabilities ORDER BY name;
```

### 6. Fleet Events (📡)

Real-time event timeline from the `fleet_events` table. Shows:

- **Event type** — `transfer_out`, `transfer_in`, `audit`, `task_complete`
- **Agent ID** — Which agent triggered the event
- **Timestamp** — When it occurred
- **Payload** — Event-specific data (JSON)

**Supabase query:**

```sql
SELECT * FROM fleet_events ORDER BY created_at DESC LIMIT 50;
```

### 7. Test Coverage (🧪)

Aggregate test metrics across repos:

- Total test count
- Per-repo test breakdown
- Test type distribution (unit, integration, etc.)

**Supabase query:**

```sql
SELECT name, test_count FROM repos WHERE test_count > 0 ORDER BY test_count DESC;
```

### 8. Conservation Gauge — Enhanced (⚛️)

Fleet-wide conservation health with a circular gauge:

- **Circular progress indicator** — Shows fleet conservation status
- **Fleet totals** — Sum of all γ, η, and total
- **Violation count** — Number of agents with broken invariants
- **Health status** — `healthy: true/false`

**Supabase query:**

```sql
SELECT * FROM fleet_budgets;
```

Client-side verification:

```javascript
const valid = Math.abs(gamma + eta - total_budget) < 0.01;
```

### 9. Ecosystem Timeline (📅)

Chronological view of significant ecosystem events:

- Repo additions
- Capability registrations
- Fleet reconfigurations
- Conservation audits

### 10. Language FFI Matrix (🔗)

Cross-language compatibility grid showing which languages can call which via FFI:

| | Rust | Python | Go | C | TypeScript |
|---|---|---|---|---|---|
| **Rust** | ✓ | PyO3 | CGO | FFI | WASM |
| **Python** | PyO3 | ✓ | — | ctypes | — |
| **Go** | CGO | — | ✓ | CGO | — |
| **C** | FFI | ctypes | CGO | ✓ | WASM |
| **TypeScript** | WASM | — | — | WASM | ✓ |

---

## Supabase Configuration

The dashboard connects to a Supabase project. Configuration is embedded in `index.html`:

```javascript
const SUPABASE_URL = 'https://your-project.supabase.co';
const SUPABASE_ANON_KEY = 'your-anon-key';
```

### Required Tables

```sql
-- Repos table
CREATE TABLE repos (
    name TEXT PRIMARY KEY,
    description TEXT,
    language TEXT,
    url TEXT
);

-- Capabilities table
CREATE TABLE capabilities (
    name TEXT PRIMARY KEY,
    repo_name TEXT,
    version TEXT,
    provides TEXT[],
    requires TEXT[],
    category TEXT
);

-- Fleet budgets table
CREATE TABLE fleet_budgets (
    agent_id TEXT PRIMARY KEY,
    total_budget FLOAT,
    gamma FLOAT,
    eta FLOAT
);

-- Fleet events table
CREATE TABLE fleet_events (
    id SERIAL PRIMARY KEY,
    agent_id TEXT,
    event_type TEXT,
    payload JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Adding New Panels

To add a new panel to the dashboard:

### 1. Add the HTML

Insert a new panel div inside the `.dashboard-grid`:

```html
<div class="panel">
  <div class="panel-header">
    <span class="icon">🔧</span> My New Panel
  </div>
  <div class="panel-body" id="myNewPanel">
    <div class="loading">Loading…</div>
  </div>
</div>
```

### 2. Add the CSS

Follow the existing design system using CSS custom properties:

```css
/* Your panel-specific styles */
#myNewPanel .my-element {
  color: var(--text);
  background: var(--surface2);
  border: 1px solid var(--border);
  border-radius: var(--radius);
}
```

### 3. Add the JavaScript

Create a function to fetch and render data:

```javascript
async function loadMyNewPanel() {
  const { data, error } = await supabase
    .from('repos')
    .select('*');

  if (error) {
    document.getElementById('myNewPanel').innerHTML =
      `<div class="error">Failed to load: ${error.message}</div>`;
    return;
  }

  // Render your data
  document.getElementById('myNewPanel').innerHTML =
    data.map(item => `<div>${item.name}</div>`).join('');
}
```

### 4. Call from init

```javascript
// In the init() function
loadMyNewPanel();
```

### Full-Width Panels

For panels that span both columns:

```html
<div class="panel full-width">
  ...
</div>
```

---

## Design System

### CSS Custom Properties

```css
:root {
    --bg: #0d0d1a;
    --surface: #161628;
    --surface2: #1e1e38;
    --border: #2a2a4a;
    --purple: #8b5cf6;
    --green: #22c55e;
    --blue: #3b82f6;
    --orange: #f59e0b;
    --cyan: #06b6d4;
    --red: #ef4444;
    --text: #e2e8f0;
    --text-dim: #94a3b8;
    --text-muted: #64748b;
    --radius: 10px;
}
```

### Responsive Layout

The dashboard uses CSS Grid with a responsive breakpoint:

```css
/* Desktop: 2 columns */
.dashboard-grid {
    grid-template-columns: 1fr 1fr;
}

/* Mobile (< 900px): 1 column */
@media (max-width: 900px) {
    .dashboard-grid { grid-template-columns: 1fr; }
}
```

### Animations

- **Live dot** — Pulsing green indicator in the header (`@keyframes pulse`)
- **Hover effects** — Panel borders highlight on hover (`transition: border-color 0.2s`)
- **Bar transitions** — Budget bars animate width changes (`transition: width 0.6s ease`)

---

## Deployment

The dashboard auto-deploys from the `gh-pages` branch:

```bash
# Make changes to index.html
git checkout gh-pages
# edit index.html
git add index.html
git commit -m "feat: update panel"
git push origin gh-pages
```

Live at: `https://superinstance.github.io/ecosystem-dashboard/`

---

## Architecture

```
ecosystem-dashboard/
├── index.html    # Single-file dashboard (HTML + CSS + JS)
└── (gh-pages branch only — no build artifacts)
```

**Technology:**

- **HTML5** — Semantic structure with panel-based layout
- **CSS3** — Custom properties, Grid, Flexbox, animations
- **Vanilla JavaScript** — Supabase JS client for data fetching
- **Supabase** — Backend-as-a-service for real-time data
- **GitHub Pages** — Static hosting from `gh-pages` branch

**No build tools, no frameworks, no npm.** Just HTML.

---

## Supabase Queries Reference

All queries used by the dashboard:

```sql
-- Stat cards
SELECT count(*) FROM repos;
SELECT count(DISTINCT language) FROM repos;
SELECT count(*) FROM capabilities;
SELECT count(*) FROM fleet_budgets;
SELECT sum(total_budget) FROM fleet_budgets;

-- Language breakdown
SELECT language, count(*) as cnt FROM repos GROUP BY language;

-- Fleet budgets (conservation gauge)
SELECT agent_id, total_budget, gamma, eta
FROM fleet_budgets ORDER BY total_budget DESC;

-- Fleet repos
SELECT * FROM repos ORDER BY name;

-- Capabilities
SELECT * FROM capabilities ORDER BY name;

-- Fleet events
SELECT * FROM fleet_events ORDER BY created_at DESC LIMIT 50;

-- Stats
SELECT * FROM repos;
SELECT count(*) FROM capabilities;
```

---

## Related Repos

| Repo | Language | Description |
|------|----------|-------------|
| [`si-fleet-api`](https://github.com/SuperInstance/si-fleet-api) | TypeScript | REST API that populates the Supabase tables |
| [`si-cli`](https://github.com/SuperInstance/si-cli) | Rust | CLI that syncs repos to Supabase |
| [`conservation-law`](https://github.com/SuperInstance/conservation-law) | Rust | Core conservation law crate |
| [`agent-operations`](https://github.com/SuperInstance/agent-operations) | Docs | Strategic operations hub |

---

## License

MIT
