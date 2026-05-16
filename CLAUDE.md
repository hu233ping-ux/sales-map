# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Single-file static web app (`index.html`) deployed on GitHub Pages. A team sales coverage map for Hunan and Hubei provinces — team members click counties to "light them up", with real-time sync across all devices.

**Live URL:** https://hu233ping-ux.github.io/sales-map/

## No Build System

There is no build step, package manager, or test suite. Edit `index.html` directly, then deploy:

```bash
git add index.html
git commit -m "描述改动"
git push
```

GitHub Pages auto-deploys within ~1 minute after push.

## Architecture

Everything lives in `index.html` as inline `<style>` + `<script>`. Dependencies are CDN-loaded:
- **ECharts 5.4.3** — map rendering and interaction
- **Supabase JS v2** — cloud persistence and real-time sync
- **canvas-confetti** — milestone fireworks

### Data flow

1. On load, `loadMap()` fetches county-level GeoJSON from Aliyun DataV API in parallel with `loadCovered()` (Supabase query). Both resolve before `initChart()` is called.
2. `initChart()` registers the merged GeoJSON, builds the ECharts map, then manually replays `dispatchAction({type:'select'})` for every covered county — ECharts does not auto-restore visual highlights from `data.selected: true`.
3. User clicks dispatch a `selectchanged` event. The handler diffs old vs. new `coveredSet`, updates `leaderboardData` and `adcodeToLitBy`, then calls `syncToSupabase()`.
4. `setupRealtime()` subscribes to Supabase Postgres Changes (INSERT/DELETE). Incoming events update state and re-render without triggering another write.

### Key state variables

| Variable | Purpose |
|---|---|
| `coveredSet` | Set of adcode strings currently lit |
| `adcodeToLitBy` | adcode → username mapping (for tooltip and leaderboard decrement on remove) |
| `leaderboardData` | username → count object |
| `applyingRemote` | Flag set during remote-driven `dispatchAction` calls to skip the `selectchanged` handler's write path |
| `pendingInserts` | Set of adcodes we just inserted locally; suppresses the echo from Supabase Realtime for 10 s |
| `triggeredMilestones` | Pre-filled on init with already-passed milestones to prevent replaying fireworks on reload |

## Critical Implementation Details

**ECharts map selection** — must use `selectedMode: 'multiple'` on the `map` series and style via `select.itemStyle`. Do not use `emphasis` for the lit state.

**City border lines** — rendered as a `lines` series with `polyline: true` and `coordinateSystem: 'geo'`, layered over the `geo` component. This is the only way to draw borders independently from the choropleth fill.

**Aliyun API on GitHub Pages** — requests must include `{ referrerPolicy: 'no-referrer' }` or the API returns 403.

**Supabase DELETE events** — the table has `REPLICA IDENTITY FULL` enabled; without it, `payload.old` is empty and you cannot identify which county was unlit.

**FLIP leaderboard animation** — uses double `requestAnimationFrame` to ensure the browser paints the initial displaced position before the CSS transition starts. Single rAF is not reliable.

## Supabase Configuration

- **Table:** `covered_counties` — columns: `adcode TEXT PK`, `lit_by TEXT`, `updated_at TIMESTAMPTZ`
- **Realtime:** table is in `supabase_realtime` publication
- The anon key in the code is intentionally public (row-level security is not configured; this is an internal team tool)

## Team Members

Fixed list in `TEAM` constant: `Mico Mi`, `Gary Huang`, `Aimee Qu`, `Lucas Luo`, `Will Ouyang`, `John Fan`, `Jason Liu`, `TBD`. To add/remove members, update this array.
