# ⚡ SuperInstance Ecosystem Dashboard

> **Live fleet dashboard** — repo count, capability cloud, conservation gauge, and fleet events. All powered by Supabase.

![Dashboard Preview](https://img.shields.io/badge/status-live-22c55e?style=flat-square) ![Supabase](https://img.shields.io/badge/powered%20by-Supabase-3ecf8e?style=flat-square) ![Zero Dependencies](https://img.shields.io/badge/deps-zero-8b5cf6?style=flat-square)

---

## Table of Contents

- [What It Shows](#what-it-shows)
- [Supabase Schema](#supabase-schema)
- [Conservation Law Visualization](#conservation-law-visualization)
- [Quick Start](#quick-start)
- [Deploying to GitHub Pages](#deploying-to-github-pages)
- [Architecture](#architecture)
- [Adding New Panels](#adding-new-panels)
- [API Reference](#api-reference)
- [Configuration](#configuration)
- [Development](#development)
- [Contributing](#contributing)
- [License](#license)

---

## What It Shows

The dashboard is a single-page application that provides a real-time view into the SuperInstance ecosystem. It renders six major sections:

### 1. Fleet Stats Header
A row of stat cards at the top showing:
- **Total Repositories** — number of repos in the fleet
- **Languages** — count of distinct programming languages
- **Capabilities** — total registered capabilities across all agents
- **Agents** — number of active agents with budget allocations
- **Total Budget (γ+η)** — aggregate conservation budget across all agents

### 2. Language Breakdown Panel
A CSS-only conic-gradient pie chart showing the distribution of programming languages across all repos. Hover over the legend to see counts. The chart is generated entirely in CSS — no canvas or SVG required.

### 3. Fleet Repositories Table
A searchable, sortable table of all repositories:
- **Search** — filter by name, language, or description
- **Sort** — click any column header to sort ascending/descending
- **Links** — repo names link directly to their GitHub URLs
- **Language badges** — color-coded tags using standard GitHub language colors

### 4. Capability Cloud
A visual tag cloud of all registered capabilities, grouped by category:
- **Infrastructure** (purple)
- **Data** (blue)
- **Communication** (cyan)
- **Intelligence** (green)
- **Automation** (orange)
- **Security** (red)
- **General** (gray)

Tag size scales with the number of provides. Hover for details.

### 5. Conservation Gauge
The signature visualization of the SuperInstance conservation law. Each agent is displayed as a horizontal bar with two segments:
- **γ (gamma)** in blue — reasoning budget
- **η (eta)** in orange — execution budget

The key insight: for every agent, **γ + η = constant**. The total budget is conserved across spawn/halt cycles.

### 6. Fleet Events Timeline
A reverse-chronological timeline of recent fleet events:
- `spawn` — agent creation events
- `complete` — task completions
- `error` — failure events
- `budget` — budget reallocation events
- `info` — general information

Limited to the 20 most recent events, auto-refreshed every 60 seconds.

---

## Supabase Schema

The dashboard reads from four tables in a public Supabase project. All access uses the anon key with Row Level Security (RLS) enabled — the key is read-only and safe to embed in client-side code.

### `repos`
| Column       | Type     | Description                |
|-------------|----------|----------------------------|
| `name`      | text     | Repository name            |
| `description` | text   | Repository description     |
| `language`  | text     | Primary programming language |
| `url`       | text     | GitHub repository URL      |

**Query:** `GET /rest/v1/repos?select=name,description,language,url&order=name`

### `capabilities`
| Column      | Type     | Description                        |
|------------|----------|------------------------------------|
| `name`     | text     | Capability name                    |
| `category` | text     | Category (infrastructure, data, etc.) |
| `provides` | jsonb    | Array of things this capability provides |

**Query:** `GET /rest/v1/capabilities?select=name,category,provides`

### `fleet_budgets`
| Column       | Type     | Description                     |
|-------------|----------|---------------------------------|
| `id`        | uuid     | Budget record ID                |
| `agent_id`  | text     | Agent identifier                |
| `agent_name`| text     | Human-readable agent name       |
| `gamma`     | integer  | Reasoning budget (γ)            |
| `eta`       | integer  | Execution budget (η)            |

**Query:** `GET /rest/v1/fleet_budgets?select=*`

### `fleet_events`
| Column       | Type      | Description                     |
|-------------|-----------|---------------------------------|
| `id`        | uuid      | Event ID                        |
| `created_at`| timestamptz | When the event occurred       |
| `event_type`| text      | Type (spawn, complete, error, etc.) |
| `message`   | text      | Event description               |

**Query:** `GET /rest/v1/fleet_events?select=*&order=created_at.desc&limit=20`

---

## Conservation Law Visualization

The conservation gauge is the philosophical heart of the dashboard. Here's the theory:

### The Law

In the SuperInstance model, every agent operates under a **conservation law**:

```
γ + η = B_total
```

Where:
- **γ (gamma)** — the reasoning budget: tokens spent on thinking, planning, deliberating
- **η (eta)** — the execution budget: tokens spent on acting, writing code, running commands
- **B_total** — the total budget allocated to this agent

This is analogous to conservation laws in physics (like energy conservation). You can transform reasoning into execution and vice versa, but the total is invariant.

### Why It Matters

1. **Predictable resource usage** — Fleet operators know exactly how much budget each agent will consume
2. **Trade-off visibility** — You can see which agents are "thinkers" (high γ) vs "doers" (high η)
3. **Spawn/halt invariance** — The total fleet budget doesn't change when agents spawn or halt; it's merely redistributed

### How We Visualize It

Each agent is a horizontal stacked bar:
- The **blue** segment shows γ
- The **orange** segment shows η
- The total bar width is proportional to B_total relative to the max budget in the fleet

An agent that's 80% blue and 20% orange is spending most of its budget reasoning. An agent that's 30% blue and 70% orange is heavily execution-oriented.

---

## Quick Start

### Prerequisites
- A modern web browser
- A Supabase project with the required tables (see schema above)
- The anon key for your Supabase project

### Local Development

1. Clone the repo:
   ```bash
   git clone https://github.com/SuperInstance/ecosystem-dashboard.git
   cd ecosystem-dashboard
   git checkout gh-pages
   ```

2. Open `index.html` in your browser:
   ```bash
   # macOS
   open index.html
   
   # Linux
   xdg-open index.html
   
   # Or just serve it
   python3 -m http.server 8080
   ```

3. That's it. No build step, no dependencies, no npm install.

### Configuration

Edit the constants at the top of the `<script>` block in `index.html`:

```javascript
const SUPABASE_URL = 'https://your-project.supabase.co';
const SUPABASE_KEY = 'your-anon-key-here';
```

> ⚠️ **Important:** Only use the `anon` key, never the `service_role` key. The anon key is designed for client-side use and should be restricted by RLS policies.

---

## Deploying to GitHub Pages

### Automatic (from gh-pages branch)

This repo is designed to deploy from the `gh-pages` branch:

1. Push your changes to `gh-pages`:
   ```bash
   git add -A
   git commit -m "feat: your changes"
   git push origin gh-pages
   ```

2. Enable GitHub Pages (one-time):
   ```bash
   gh api repos/SuperInstance/ecosystem-dashboard/pages -X POST \
     -f source.branch=gh-pages \
     -f source.path=/
   ```

   Or via GitHub's web UI:
   - Settings → Pages → Source: Deploy from branch → `gh-pages` / `/ (root)`

3. Your dashboard will be live at:
   ```
   https://superinstance.github.io/ecosystem-dashboard/
   ```

### Custom Domain (optional)

1. Add a `CNAME` file:
   ```
   echo "dashboard.superinstance.dev" > CNAME
   ```
2. Configure your DNS provider to point to GitHub Pages
3. Enable HTTPS in the GitHub Pages settings

---

## Architecture

### Design Principles

1. **Zero dependencies** — No React, no D3, no Chart.js. Pure HTML, CSS, and vanilla JS.
2. **Single file** — Everything lives in `index.html`. No build tools, no bundlers.
3. **Progressive enhancement** — Works without JavaScript (shows loading states), works great with it.
4. **Responsive** — CSS Grid adapts from desktop to mobile.
5. **Dark theme** — SuperInstance purple/green aesthetic on a deep navy background.

### Tech Stack

| Layer    | Technology                      |
|----------|---------------------------------|
| Layout   | CSS Grid + Flexbox              |
| Charts   | CSS conic-gradient (pie), div bars (gauge) |
| Data     | Supabase REST API via Fetch     |
| Refresh  | `setInterval` (60s polling)     |
| Styling  | CSS custom properties (theming) |
| Icons    | Unicode emoji (no icon library) |

### Data Flow

```
┌──────────────┐     GET /rest/v1/*     ┌─────────────┐
│  index.html   │ ──────────────────────► │  Supabase   │
│  (browser)    │ ◄────────────────────── │  (PostgreSQL)│
└──────────────┘     JSON response       └─────────────┘
       │
       ▼
 ┌──────────┐
 │  Render   │ → Stats Cards, Pie Chart, Table, Cloud, Gauge, Timeline
 └──────────┘
       │
       ▼
  setInterval(60s) → Refresh all panels
```

### File Structure

```
ecosystem-dashboard/
├── index.html    # The entire dashboard (HTML + CSS + JS)
├── README.md     # This file
└── LICENSE       # MIT
```

That's it. One file does everything.

---

## Adding New Panels

The dashboard is designed to be extended. Here's how to add a new panel:

### Step 1: Add the HTML

Insert a new `<div class="panel">` inside `.dashboard-grid`:

```html
<div class="panel">
  <div class="panel-header"><span class="icon">🔮</span> Your Panel Title</div>
  <div class="panel-body" id="yourPanel"><div class="loading">Loading…</div></div>
</div>
```

For full-width panels, add the `full-width` class:
```html
<div class="panel full-width">...</div>
```

### Step 2: Add the Render Function

In the `<script>` block, add a render function:

```javascript
function renderYourPanel(data) {
  if (!data.length) {
    document.getElementById('yourPanel').innerHTML = '<div class="empty">No data</div>';
    return;
  }
  // Build your HTML
  document.getElementById('yourPanel').innerHTML = `<div>...</div>`;
}
```

### Step 3: Fetch and Render

Add a fetch block to the `loadAll()` function:

```javascript
try {
  const data = await apiFetch('your_table', 'select=*&order=id');
  renderYourPanel(data);
} catch (err) { showError('yourPanel', err); }
```

### Step 4: Style It

Add CSS in the `<style>` block following the existing patterns:
- Use CSS custom properties (`var(--purple)`, `var(--surface)`, etc.)
- Follow the dark theme aesthetic
- Use `var(--radius)` for border-radius consistency

### Panel Ideas

Here are some panels you could add:

- **Agent Topology** — Show agent relationships as a graph (would need a canvas/SVG library)
- **Token Velocity** — Chart of token usage over time
- **Health Checks** — Green/red status indicators for each service
- **Dependency Graph** — Visualize inter-repo dependencies
- **Commit Activity** — Recent commits across all repos (heatmap)
- **Cost Tracker** — Spending by agent over time

---

## API Reference

### `apiFetch(table, query)`

Internal helper that wraps Supabase REST API calls:

```javascript
async function apiFetch(table, query = '')
```

**Parameters:**
- `table` (string) — Supabase table name (e.g., `'repos'`, `'capabilities'`)
- `query` (string) — URL query string (e.g., `'select=*&order=id'`)

**Returns:** `Promise<Array>` — parsed JSON response

**Example:**
```javascript
const repos = await apiFetch('repos', 'select=name,language&order=name');
// → [{ name: "agent-core", language: "TypeScript" }, ...]
```

### Supabase REST API Endpoints

All endpoints are relative to `SUPABASE_URL/rest/v1/`:

| Endpoint           | Method | Description            | Auth     |
|-------------------|--------|------------------------|----------|
| `/repos`          | GET    | List repositories      | anon     |
| `/capabilities`   | GET    | List capabilities      | anon     |
| `/fleet_budgets`  | GET    | List agent budgets     | anon     |
| `/fleet_events`   | GET    | List fleet events      | anon     |

All endpoints support PostgREST query parameters:
- `select` — column selection
- `order` — sorting
- `limit` — result count
- `filter` — column filters (e.g., `language=eq.TypeScript`)

---

## Configuration

### Environment Variables

There are no server-side environment variables. Configuration is done by editing the JavaScript constants at the top of `index.html`:

```javascript
const SUPABASE_URL = 'https://igogykhksgkaxcwzudwi.supabase.co';
const SUPABASE_KEY = 'eyJ...'; // anon key only!
```

### Customization Options

| What to change        | Where to find it               |
|----------------------|--------------------------------|
| Color scheme         | `:root` CSS custom properties  |
| Refresh interval     | `setInterval(loadAll, 60000)`  |
| Event limit          | `limit=20` in the fetch query  |
| Language colors      | `LANG_COLORS` object           |
| Panel layout         | CSS Grid in `.dashboard-grid`  |

### Theme Colors

The dashboard uses these CSS custom properties:

| Variable        | Value     | Usage                |
|----------------|-----------|----------------------|
| `--bg`         | `#0d0d1a` | Page background      |
| `--surface`    | `#161628` | Card backgrounds     |
| `--purple`     | `#8b5cf6` | Primary accent       |
| `--green`      | `#22c55e` | Success / live       |
| `--blue`       | `#3b82f6` | Gamma budget bars    |
| `--orange`     | `#f59e0b` | Eta budget bars      |
| `--cyan`       | `#06b6d4` | Links / code         |
| `--red`        | `#ef4444` | Errors               |

---

## Development

### Local Server

While you can open `index.html` directly, some features work better with a local server:

```bash
# Python
python3 -m http.server 8080

# Node.js
npx serve .

# PHP
php -S localhost:8080
```

### Making Changes

1. Edit `index.html` — it's the only file
2. Refresh your browser to see changes
3. Test with different data by changing the Supabase URL/key

### Debugging

Open your browser's DevTools (F12) and check:
- **Console** — API errors, JavaScript errors
- **Network** — Supabase request/response details
- **Elements** — Inspect rendered HTML

Common issues:
- **CORS errors** — Make sure your Supabase project allows requests from your domain
- **401 Unauthorized** — Check that the anon key is correct
- **Empty tables** — The dashboard shows "No data" states when tables are empty

---

## Contributing

Contributions are welcome! Here's how:

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-panel`
3. Make your changes in `index.html`
4. Test locally
5. Commit: `git commit -m "feat: add X panel"`
6. Push: `git push origin feature/my-panel`
7. Open a Pull Request against the `gh-pages` branch

### Guidelines

- Keep it dependency-free — vanilla HTML/CSS/JS only
- Follow the existing dark theme and CSS variable conventions
- Test on both desktop and mobile viewports
- Add appropriate loading and error states
- Keep the single-file architecture

---

## Security Considerations

### Why the anon key is safe to embed

The Supabase anon key is designed for client-side use. It's protected by Row Level Security (RLS):

- The key alone cannot write data
- The key alone cannot access tables without RLS policies
- The key can only read from tables explicitly configured for public access

**Never** use the `service_role` key in client-side code. It bypasses all RLS policies.

### Best Practices

1. Enable RLS on all tables
2. Create read-only policies for the anon role
3. Never commit the `service_role` key
4. Review RLS policies regularly
5. Consider adding rate limiting via Supabase Edge Functions if traffic grows

---

## Performance

The dashboard is designed to be lightweight:

- **Zero build step** — no bundler overhead
- **Single file** — one HTTP request
- **No framework** — no virtual DOM diffing
- **CSS charts** — no canvas/SVG rendering overhead
- **Polling at 60s** — reasonable balance of freshness vs. API load
- **Typical page weight** — ~20KB uncompressed

---

## License

MIT © SuperInstance

---

> *The conservation law governs all. γ + η = B. The fleet persists.*
