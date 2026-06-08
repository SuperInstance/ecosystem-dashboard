# Integration Guide: ecosystem-dashboard

## What This Dashboard Provides

A single-file, self-contained HTML dashboard for the SuperInstance ecosystem. No build step required — just open `index.html` in a browser or serve it statically. It connects directly to the Supabase REST API and renders real-time fleet telemetry.

### Dashboard Panels

- **Stats Row** — Live cards: Repositories, Languages, Capabilities, Agents, Total Budget (γ+η)
- **Language Breakdown** — CSS-only conic-gradient pie chart of repo languages with legend
- **Conservation Gauge** — Per-agent budget bars showing gamma (reasoning) and eta (execution) allocations with conservation law annotation
- **Fleet Repositories** — Searchable, sortable table of all repos with language badges and GitHub links
- **Capability Cloud** — Grouped tag cloud of capabilities by category (infrastructure, data, communication, intelligence, automation, security)
- **Fleet Events** — Scrolling timeline of recent spawn/complete/error/budget events

### Key Features

- **Auto-refresh** — Reloads all data every 60 seconds
- **Live indicator** — Pulsing green dot with last-refresh timestamp
- **Client-side search** — Instant filtering of repo table by name, description, or language
- **Column sorting** — Click table headers to sort repos
- **Responsive layout** — 2-column grid on desktop, single column on mobile
- **Zero dependencies** — Pure HTML/CSS/JS, no bundler or framework

### JavaScript API

- `apiFetch(table, query)` — Generic Supabase REST fetch with auth headers
- `renderLangPie(repos)` — CSS conic-gradient pie chart
- `renderRepoTable(filter)` — Sortable/searchable repo table
- `renderCapCloud(caps)` — Category-grouped capability tags
- `renderBudget(budgets)` — Gamma/eta stacked bar chart
- `renderEvents(events)` — Timeline with color-coded event types
- `loadAll()` — Orchestrates all panel refreshes
- `sortRepos(col)` — Toggle sort direction on column click

## How to Deploy

```bash
# Static hosting
python3 -m http.server 8080
# or
npx serve .
# or upload to any static host (GitHub Pages, Vercel, Netlify, S3)
```

## Cross-Repo Connections

### With `si-fleet-api`: Backend Data Source

The dashboard can read from `si-fleet-api` instead of direct Supabase:

```javascript
// Replace direct Supabase calls with API calls
async function apiFetch(endpoint) {
  const res = await fetch(`https://api.superinstance.dev/api/${endpoint}`);
  return res.json();
}

// Load repos via API
const repos = await apiFetch('repos');
renderRepoTable(repos);
```

### With `si-cli`: Supabase Sync

Repos discovered by `si scan` are synced to Supabase, which the dashboard renders:

```bash
# In CI or locally
si scan ./workspace
# → upserts repos to Supabase `repos` table
# → dashboard auto-refreshes and shows new repos
```

### With `conservation-law-rs`: Conservation Visualization

The dashboard visualizes `γ + η = total` conservation as stacked budget bars:

```javascript
// Rendered in renderBudget()
const maxTotal = Math.max(...budgets.map(b => (b.gamma || 0) + (b.eta || 0)), 1);
const gPct = (gamma / maxTotal) * 100;
const ePct = (eta / maxTotal) * 100;
// HTML: <div class="budget-gamma" style="width:${gPct}%">γ ${gamma}</div>
//       <div class="budget-eta" style="width:${ePct}%">η ${eta}</div>
```

### With Supabase: Direct REST Integration

Connects to Supabase using anon key for read-only access:

```javascript
const SUPABASE_URL = 'https://project.supabase.co';
const SUPABASE_KEY = 'eyJhbGciOiJIUzI1NiIs...'; // anon key

const headers = {
  'apikey': SUPABASE_KEY,
  'Authorization': `Bearer ${SUPABASE_KEY}`,
  'Content-Type': 'application/json'
};

async function apiFetch(table, query = '') {
  const url = `${SUPABASE_URL}/rest/v1/${table}${query ? '?' + query : ''}`;
  const res = await fetch(url, { headers });
  return res.json();
}

// Fetch all tables
const repos      = await apiFetch('repos',        'select=name,description,language,url&order=name');
const caps       = await apiFetch('capabilities', 'select=name,category,provides');
const budgets    = await apiFetch('fleet_budgets', 'select=*');
const events     = await apiFetch('fleet_events', 'select=*&order=created_at.desc&limit=20');
```

## Design Patterns

### Pattern: Embedded Static Dashboard

Host the dashboard as a GitHub Pages site that auto-updates:

```yaml
# .github/workflows/dashboard.yml
- name: Deploy to GitHub Pages
  uses: peaceiris/actions-gh-pages@v3
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    publish_dir: .
```

### Pattern: Kiosk Mode Display

Run the dashboard full-screen on a wall-mounted display:

```bash
# Raspberry Pi or any Linux box
chromium-browser --kiosk --app=http://localhost:8080/index.html
```

### Pattern: Custom Supabase Project

Point the dashboard at your own Supabase instance:

```javascript
// Edit these two lines in index.html
const SUPABASE_URL = 'https://your-project.supabase.co';
const SUPABASE_KEY = 'your-anon-key';
```

### With `fleet-warden-rs`: Disk Health Widget

Add a disk health panel to the dashboard by querying fleet-warden state:

```javascript
async function renderDiskHealth() {
  const events = await apiFetch('fleet_events', 'event_type=eq.cleanup&order=created_at.desc&limit=10');
  let html = '<div class="timeline">';
  events.forEach(e => {
    html += `<div class="event-item">
      <div class="event-time">${new Date(e.created_at).toLocaleString()}</div>
      <span class="event-type budget">cleanup</span>
      <div class="event-msg">${e.agent_id}: ${e.payload.category} — ${e.payload.recovered || '—'}</div>
    </div>`;
  });
  html += '</div>';
  document.getElementById('diskPanel').innerHTML = html;
}
```

### With `agent-homeostasis-rs`: Regulation Gauge

Visualize homeostatic parameters as real-time gauges:

```javascript
async function renderRegulation(agentId) {
  const readings = await apiFetch('sensor_readings', `agent_id=eq.${agentId}&order=timestamp.desc&limit=50`);
  // Render sparkline for each sensor
  const sensors = groupBy(readings, 'sensor_name');
  let html = '<div class="budget-bars">';
  for (const [name, values] of Object.entries(sensors)) {
    const latest = values[0].value;
    const target = values[0].target || 0;
    const deviation = Math.abs(latest - target);
    html += `<div class="budget-row">
      <div class="budget-label">${name}</div>
      <div class="budget-bar-track">
        <div class="budget-gamma" style="width:${Math.min(100, latest * 100)}%">${latest.toFixed(2)}</div>
      </div>
      <div class="budget-total">Δ${deviation.toFixed(2)}</div>
    </div>`;
  }
  html += '</div>';
  document.getElementById('regPanel').innerHTML = html;
}
```

### With Supabase: Row-Level Security

Secure dashboard data with RLS policies:

```sql
-- Allow anon read-only access to repos
CREATE POLICY "Allow anon read repos" ON repos
  FOR SELECT TO anon USING (true);

-- Allow anon read-only access to capabilities
CREATE POLICY "Allow anon read capabilities" ON capabilities
  FOR SELECT TO anon USING (true);

-- Restrict fleet_budgets to authenticated users
CREATE POLICY "Allow auth read budgets" ON fleet_budgets
  FOR SELECT TO authenticated USING (true);
```

## Design Patterns

### Pattern: Offline-First Dashboard

Cache dashboard data in localStorage for offline viewing:

```javascript
async function loadAll() {
  try {
    const repos = await apiFetch('repos', 'select=*');
    localStorage.setItem('dash_repos', JSON.stringify(repos));
    renderRepoTable(repos);
  } catch (err) {
    const cached = localStorage.getItem('dash_repos');
    if (cached) renderRepoTable(JSON.parse(cached));
    else showError('repoTableWrap', err);
  }
}
```

### Pattern: Custom Theme Injection

Allow theme customization via CSS variables:

```javascript
function setTheme(theme) {
  document.documentElement.style.setProperty('--bg', theme.bg);
  document.documentElement.style.setProperty('--purple', theme.primary);
  document.documentElement.style.setProperty('--green', theme.secondary);
}
```
