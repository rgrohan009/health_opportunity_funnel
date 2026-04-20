# Health Opportunities Funnel Simulator — Claude Context

## Project
Single-file HTML tool (`index.html`) served locally via `npx serve` on port 8080.
No build step, no dependencies except html2canvas (CDN) and Google Fonts.
All state is in-memory JS + localStorage (for snapshots).

## Launch
```bash
# Server config: .claude/launch.json → name: "funnel-simulator", port: 8080
# Start via: mcp__Claude_Preview__preview_start { name: "funnel-simulator" }
```

## Business Logic

### Funnel Model
- Two channels: **VRM** and **Non-VRM**, each with identical 4-step funnel structure.
- Conversion rates entered by user are **6-month cumulative** (not monthly).
- **Top of funnel is a monthly push** — every month a new cohort of DPs enters.
- Each push generates revenue over 6 months → `monthly_premium = total_6m_premium / 6`.

### Funnel Steps
1. Opportunities Created (= DPs × avg_opp_per_DP) — no conversion applied
2. Opportunities Seen (Opp conv % × DP conv %)
3. Leads / Quote Sent (Opp conv % × DP conv %) — optional DP Lock cap
4. Converted (Opp conv % × DP conv %)

**DP Lock**: If enabled, caps the number of DPs at step 3 and back-solves steps 1 & 2.

### Revenue Calculation
- `p6 = converted_opps × avg_premium` (6-month total per single push)
- `pm = p6 / 6` (monthly revenue per single push)

### 12-Month Recurring Revenue (cohort stacking)
Each month adds a new push. In month M, `min(M, 6)` cohorts are active simultaneously.
```
Month 1: 1 cohort  → 1 × pm
Month 2: 2 cohorts → 2 × pm
...
Month 6: 6 cohorts → 6 × pm  ← steady state begins
Month 7–12: 6 cohorts → 6 × pm  (new cohort in, oldest cohort out)

6-Month cumulative  = (1+2+3+4+5+6) × pm_combined = 21 × pm
12-Month cumulative = 21 × pm + 6×6 × pm          = 57 × pm
Steady-state monthly = 6 × pm_combined
```

## File Structure
```
index.html          ← entire app (HTML + CSS + JS, ~1150 lines)
.claude/
  launch.json       ← preview server config
CLAUDE.md           ← this file
```

## Key Element IDs

### Left Panel Controls
| ID | Purpose |
|----|---------|
| `v-dp` | VRM: No. of DPs slider |
| `v-opk` | VRM: Avg opp per DP |
| `v-prm` | VRM: Avg premium (₹) |
| `v-c1o/c1d` | VRM: Step 2 conv % (opp/dp) |
| `v-c2o/c2d` | VRM: Step 3 conv % (opp/dp) |
| `v-c3o/c3d` | VRM: Step 4 conv % (opp/dp) |
| `v-cap` | VRM: DP Lock cap value |
| `v-lb` | VRM: Lock/Unlock button |
| `n-*` | Same pattern for Non-VRM |

### Right Panel Outputs
| ID | Purpose |
|----|---------|
| `v-svg` / `n-svg` | Funnel SVG charts |
| `s-vp6/s-np6/s-tp6` | Summary: 6M premiums (VRM/NV/Total) |
| `s-vpm/s-npm/s-tpm` | Summary: Monthly premiums |
| `rev-svg` | 12-Month recurring revenue bar chart |
| `rt-6m` | 6-Month cumulative (Lacs) |
| `rt-ss` | Steady-state monthly (Lacs/month) |
| `rt-12m` | 12-Month cumulative (Lacs) |

### Modals & UI
| ID | Purpose |
|----|---------|
| `snap-btn` | Snapshot capture button |
| `snap-modal` | Save snapshot dialog |
| `snap-preview-img` | Preview thumbnail in dialog |
| `snap-name-input` | Snapshot name field |
| `snap-folder-sel` | Folder dropdown |
| `lib-modal` | Snapshot Library modal |
| `lib-folder-list` | Sidebar folder list |
| `lib-grid` | Snapshot card grid |
| `snap-viewer` | Full-size image viewer |
| `snap-toast` | Save confirmation toast |

## Key JS Functions
| Function | Purpose |
|----------|---------|
| `U()` | Master update — recalculates everything, updates DOM, redraws charts |
| `calcVRM()` | Runs VRM funnel math, returns `{o[], d[], p6, pm, lk, ...}` |
| `calcNV()` | Same for Non-VRM |
| `draw(sid, dat, mxO, mxD, lkAt)` | Renders funnel SVG |
| `drawRevChart()` | Renders 12-month stacked bar chart |
| `TL()` | Toggle DP Lock |
| `switchTab(ch, btn)` | Switch VRM/Non-VRM tab |
| `takeSnapshot()` | html2canvas capture → opens save dialog |
| `saveSnapshot()` | Persists to localStorage |
| `openLibrary()` | Opens snapshot library modal |
| `_snapStore()` | Read snapshots from localStorage (`hofs_snaps`) |
| `_foldStore()` | Read folders from localStorage (`hofs_folders`) |

## Global State
```js
LK      // boolean — DP Lock active?
RV      // {pm, p6, d} — last VRM calc result
RN      // {pm, p6, d} — last NV calc result
```

## Design System (CSS Variables)
```css
--bg: #F4F3EF          /* page background */
--surface: #FFFFFF
--surface2: #F8F7F4
--border: #E2E0DA
--text-primary: #1A1915
--text-secondary: #6B6860
--text-muted: #9B9890
--vrm-1: #3B2F8A  --vrm-2: #5C4EC4  --vrm-3: #8B7EE8  --vrm-4: #C5BEFD  --vrm-light: #EEF0FF
--nv-1: #0D5C41   --nv-2: #1A8A5F   --nv-3: #4DB88A   --nv-4: #A8E6CA   --nv-light: #E8F7F0
--accent: #E85D30
--radius: 10px   --radius-sm: 6px
```

## localStorage Keys
| Key | Contents |
|-----|---------|
| `hofs_snaps` | JSON array of snapshot objects `{id, name, folder, date, image(base64 JPEG)}` |
| `hofs_folders` | JSON array of folder name strings, default `["General"]` |

## Layout
```
header (fixed)
tab-bar (VRM / Non-VRM)
.main (CSS grid: 340px | 1fr)
  .left-panel (scrollable)
    #ctrl-vrm or #ctrl-nv (shown based on active tab)
      cards: Top of Funnel, Key Values, Conversion % at Every Step
  .right-panel (scrollable, flex-col gap-16px)
    .funnel-pair (grid 1fr 1fr) → VRM funnel | NV funnel
    .summary-card → Single-push 6-month output table
    .rev-card → 12-month recurring revenue chart + 3 summary tiles
```

## Snapshot Feature
- **Snapshot button** (top-right header): captures `.right-panel` via html2canvas at 2× scale, saves as JPEG 78% quality
- **Library button**: Chrome-bookmark-style modal — folder sidebar + snapshot grid
- Snapshots stored in localStorage; each ~200–500 KB. ~10–25 snapshots fit before storage limit.
- Folder names and snapshot names are user-editable at save time.
