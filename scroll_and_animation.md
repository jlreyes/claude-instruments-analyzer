# SwiftUI Scroll & Animation Jank Diagnosis

Use this reference when investigating **scroll lag**, **animation stutter**, and **frame drops** in SwiftUI apps.

Most agents fail for two reasons:

- **Bad scope**: analyzing a big time range and reporting aggregates
- **Bad attribution**: saying "SwiftUI took time" without mapping specific lost frames to causes

The workflow below prevents both.

---

## Output Contract

Always produce a Markdown report with:

1. **Scope**: interaction, windows, mode, budget
2. **Dropped frame inventory**: counts + severity buckets + worst frames list
3. **Frame deep dives**: top N worst frames with frame-local evidence and a cause call
4. **Root-cause clusters**: deduped causes with estimated payoff
5. **Plan**: prioritized fixes and follow-up traces or instrumentation if needed

If attribution confidence is low, end with a concrete "next trace" recipe: what to instrument, what template to record, what question it answers.

---

## Workflow

Progress:

- [ ] Understand: choose mode, define interaction, get rough time window
- [ ] Instrument: ensure boundaries exist (signposts). If not, propose minimal signposts
- [ ] Collect: record trace under realistic conditions; export to DuckDB
- [ ] Prepare: create canonical derived views (budget, frames, buckets) via script or SQL
- [ ] Identify: isolate interaction sub-windows; rank dropped frames by missed frames
- [ ] Execute: deep dive worst frames; attribute; cluster causes; quantify payoff; plan

---

## Step 1: Understand

### Decide mode

Infer from the user's wording:

- **Hitch mode**: "big jumps", "freezes", "super laggy", "sometimes it locks"
  - Focus: worst frames (p95, p99), biggest hitches
- **Smoothness mode**: "not buttery", "slightly stuttery", "not consistently smooth"
  - Focus: near-budget and median frames, recurring micro-jank

If unclear, default to **Hitch mode first**, then add a Smoothness pass.

### Minimum questions to ask

- What interaction is it (scroll list, resize, transition animation)?
- Roughly when in the trace did it happen (seconds)?
- Do you have custom signposts for begin/end of the interaction?

---

## Step 2: Instrument

### Goal

Make the analysis windows machine-readable.

### Best case: custom `os_signpost` intervals

Add signpost intervals for boundaries:

- `interaction.scroll` begin/end
- `interaction.animation` begin/end

Optional signposts around suspect work:

- data fetch
- text/layout pass
- image decode
- model diff / transform

### Verification query

Note: `os_signpost_*` timestamps are strings. Cast to BIGINT when comparing.

```sql
SELECT
  name, category, subsystem,
  COUNT(*) AS n,
  MAX(CAST(duration_ns AS BIGINT))/1e6 AS max_ms
FROM os_signpost_intervals
WHERE name IS NOT NULL
GROUP BY 1,2,3
ORDER BY max_ms DESC
LIMIT 50;
```

If there are no relevant boundary signposts, you may proceed with heuristic segmentation, but label confidence lower.

---

## Step 3: Collect

### Record

Prefer Release build behavior.

```bash
pgrep -x AppName

xcrun xctrace record --template 'SwiftUI' --time-limit 20s \
  --output ./traces/$(date +%Y%m%d_%H%M%S).trace \
  --attach AppName --no-prompt
```

### Export to DuckDB

Use the export script (run it directly, not with `python3`):

```bash
./scripts/export_to_duckdb.py traces/recording.trace traces/recording/analysis.duckdb
```

The script automatically creates any missing parent directories for the output path.

If key tables are empty, call it out and recommend a different template or longer recording.

---

## Step 4: Identify (Isolate the Moments and the Dropped Frames)

### Goal

Convert a broad user window (e.g., 10–20s) into:

- one or more **interaction sub-windows**
- the list of **dropped frames** inside them
- **severity buckets** (by missed frames)

### 4.1 Determine the frame budget

Best: infer from `coreanimation_lifetime_intervals.acceptable_latency_ns` (if present).

```sql
SELECT
  quantile_cont(acceptable_latency_ns, 0.5)/1e6 AS p50_budget_ms,
  quantile_cont(acceptable_latency_ns, 0.9)/1e6 AS p90_budget_ms,
  COUNT(*) AS n
FROM coreanimation_lifetime_intervals
WHERE acceptable_latency_ns IS NOT NULL;
```

If unavailable, assume:

- 60 fps budget: **16.67ms**
- 120 fps budget: **8.33ms**

and clearly label it as an assumption.

### 4.2 Pick the interaction window(s)

#### Preferred: signpost-bounded windows

```sql
-- Find candidate interaction intervals (scroll/animation)
SELECT
  CAST(start_ns AS BIGINT)/1e9 AS start_s,
  (CAST(start_ns AS BIGINT)+CAST(duration_ns AS BIGINT))/1e9 AS end_s,
  CAST(duration_ns AS BIGINT)/1e6 AS ms,
  name, category, subsystem, start_message
FROM os_signpost_intervals
WHERE name ILIKE '%scroll%' OR name ILIKE '%animation%' OR category ILIKE '%interaction%'
ORDER BY CAST(duration_ns AS BIGINT) DESC
LIMIT 50;
```

#### Fallback: heuristic segmentation (dropped-frame clusters)

If the user gives a wide range and you have no signposts, segment into "bursts":

```sql
-- Replace {t0_ns}, {t1_ns} with the user's rough range
WITH h AS (
  SELECT start_ns, duration_ns, swap_id
  FROM hitches
  WHERE start_ns BETWEEN {t0_ns} AND {t1_ns}
  ORDER BY start_ns
),
gaps AS (
  SELECT *,
    start_ns - LAG(start_ns) OVER (ORDER BY start_ns) AS gap_ns
  FROM h
),
seg AS (
  SELECT *,
    SUM(CASE WHEN gap_ns IS NULL OR gap_ns > 200000000 THEN 1 ELSE 0 END)
      OVER (ORDER BY start_ns) AS segment_id
  FROM gaps
)
SELECT
  segment_id,
  MIN(start_ns)/1e9 AS seg_start_s,
  MAX(start_ns+duration_ns)/1e9 AS seg_end_s,
  COUNT(*) AS hitch_count,
  MAX(duration_ns)/1e6 AS worst_hitch_ms
FROM seg
GROUP BY segment_id
ORDER BY worst_hitch_ms DESC;
```

Pick the segment(s) that match the user's described action.

### 4.3 Build the dropped-frame table (script method — preferred)

Use the `prepare_analysis.py` script to create the `frames` view:

```bash
./scripts/prepare_analysis.py traces/recording/analysis.duckdb
```

This creates a `frames` view from `hitches_frame_lifetimes` with computed `missed_frames` and severity buckets. The script:

- Infers frame budget from `coreanimation_lifetime_intervals.acceptable_latency_ns`
- Falls back to 60fps (16.67ms) if unavailable
- Prints summary statistics

Override budget for 120fps displays:

```bash
./scripts/prepare_analysis.py traces/recording/analysis.duckdb --budget-ms 8.33
```

### 4.4 Severity buckets (by missed frames)

Use this default (tunable, but consistent):

- **Low**: 1 missed frame
- **Medium**: 2–3 missed frames
- **High**: 4–7 missed frames
- **Extreme**: 8+ missed frames

```sql
SELECT
  CASE
    WHEN missed_frames = 0 THEN 'OK'
    WHEN missed_frames = 1 THEN 'Low'
    WHEN missed_frames BETWEEN 2 AND 3 THEN 'Medium'
    WHEN missed_frames BETWEEN 4 AND 7 THEN 'High'
    ELSE 'Extreme'
  END AS bucket,
  COUNT(*) AS frames
FROM frames
GROUP BY 1
ORDER BY frames DESC;
```

### 4.5 Mode-specific selection

#### Hitch mode (worst-case)

Pick top N by `missed_frames DESC, hitch_ms DESC` inside the interaction window.

#### Smoothness mode (median / "not buttery")

Look at:

- frames near budget (e.g., `frame_ms > 0.85*budget_ms`)
- p50/p75 frame_ms inside the window
- recurring small misses (missed_frames=1) clustered patterns

### 4.6 Cascade Analysis

A hitch can cause subsequent frames to miss their deadlines (cascade effect). When analyzing a dropped frame, always check the **5 frames before** to find the root cause.

**Key insight**: If frame N takes too long, frame N+1's work starts late. Even if N+1's work is fast, it may miss its deadline. The damage propagates forward until the system recovers.

**Use the script** to analyze a specific frame with cascade context:

```bash
./scripts/prepare_analysis.py traces/recording/analysis.duckdb --swap-id 166871
./scripts/prepare_analysis.py traces/recording/analysis.duckdb --swap-id 166871 --context-frames 10
```

This outputs:

- Target frame details
- 5 preceding frames with budget status
- Root cause identification (first over-budget frame in sequence)
- os_signposts during the root cause frame (sorted by duration)
- os_logs during the root cause frame

**Interpretation**:

- If a preceding frame is over budget, the target is a **CASCADE VICTIM** — fix the root cause frame instead
- If no preceding frame is over budget, the target is the **ROOT CAUSE** — focus analysis there
- The root cause frame's signposts reveal what work caused the hitch

---

## Step 5: Execute (Attribute Causes + Produce a Fix Plan)

### Goal

For each selected worst frame:

- shrink scope to **that frame only**
- compute a breakdown across subsystems
- make a cause call with evidence

Then:

- cluster frames into distinct root causes
- estimate payoff per fix
- output a prioritized plan

### 5.1 Pick frames to deep dive (script-first approach)

**Always start with the script** for each frame you want to analyze:

```bash
./scripts/prepare_analysis.py traces/recording/analysis.duckdb --swap-id {swap_id}
```

This gives you:

1. Cascade analysis (is this frame the root cause or a victim?)
2. Signposts and logs during the actual root cause frame
3. Ready-to-paste markdown for your report

Only run manual SQL queries (below) if you need deeper attribution than the script provides.

Default:

- Hitch mode: worst **5** frames
- Smoothness mode: worst **3** + most common near-budget pattern

Always show the full top list, but only deep dive N.

```sql
-- Replace with the chosen interaction window bounds
SELECT
  swap_id,
  start_ns/1e9 AS time_s,
  frame_ms,
  hitch_ms,
  missed_frames,
  severity,
  type_label
FROM frames
WHERE start_ns BETWEEN {win_start_ns} AND {win_end_ns}
ORDER BY missed_frames DESC, hitch_ms DESC NULLS LAST, frame_ms DESC
LIMIT 25;
```

### 5.2 Per-frame deep dive (the attribution loop)

For each `swap_id`, define:

- `frame_start = frames.start_ns`
- `frame_end = frames.end_ns`
- optional pad: ±2ms (small, to catch overlaps)

#### A) SwiftUI work in the frame

```sql
SELECT
  update_type,
  severity,
  duration_ns/1e6 AS ms,
  description
FROM updates
WHERE start_ns < {frame_end_ns}
  AND (start_ns + duration_ns) > {frame_start_ns}
ORDER BY duration_ns DESC
LIMIT 30;
```

Totals:

```sql
SELECT
  SUM(duration_ns)/1e6 AS total_swiftui_ms,
  COUNT(*) AS update_count
FROM updates
WHERE start_ns < {frame_end_ns}
  AND (start_ns + duration_ns) > {frame_start_ns};
```

#### B) Transaction/update-group cascades

```sql
SELECT
  duration_ns/1e6 AS ms,
  label,
  backtrace_json,
  cause_backtrace_json
FROM update_groups
WHERE start_ns < {frame_end_ns}
  AND (start_ns + duration_ns) > {frame_start_ns}
ORDER BY duration_ns DESC
LIMIT 10;
```

#### C) GPU / render pipeline signals (same frame via swap_id)

```sql
SELECT 'updates' AS phase, duration_ns/1e6 AS ms, is_hitch, frame_color, containment_level
FROM hitches_updates WHERE swap_id = {swap_id}
UNION ALL
SELECT 'render', duration_ns/1e6, is_hitch, frame_color, containment_level
FROM hitches_renders WHERE swap_id = {swap_id}
UNION ALL
SELECT 'gpu', duration_ns/1e6, is_hitch, frame_color, containment_level
FROM hitches_gpu WHERE swap_id = {swap_id}
UNION ALL
SELECT 'framewait', duration_ns/1e6, NULL, frame_color, layout_qualifier
FROM hitches_framewait WHERE swap_id = {swap_id}
ORDER BY ms DESC NULLS LAST;
```

Interpretation heuristics:

- **Render/GPU dominates** + high offscreen passes → likely GPU/render bound
- **SwiftUI dominates** + many updates/groups → likely invalidation/update storm
- **Framewait dominates** → presentation/latency contention (often downstream of earlier work)

#### D) RunLoop (main-thread scheduling context)

First get a main thread TID:

```sql
SELECT thread_tid
FROM runloop_intervals
WHERE is_main = true AND thread_tid IS NOT NULL
LIMIT 1;
```

Then:

```sql
SELECT
  interval_type,
  mode,
  duration_ns/1e6 AS ms,
  label
FROM runloop_intervals
WHERE is_main = true
  AND start_ns < {frame_end_ns}
  AND (start_ns + duration_ns) > {frame_start_ns}
ORDER BY duration_ns DESC
LIMIT 20;
```

#### E) CPU samples (Time Profiler)

If `time_profile` exists, this is your strongest "what actually ran" evidence.

```sql
SELECT
  COUNT(*) AS samples,
  SUM(weight_ns)/1e6 AS approx_ms,
  backtrace_json
FROM time_profile
WHERE thread_tid = {main_thread_tid}
  AND time_ns BETWEEN {frame_start_ns} AND {frame_end_ns}
  AND backtrace_json IS NOT NULL
GROUP BY backtrace_json
ORDER BY approx_ms DESC
LIMIT 10;
```

#### F) SwiftUI "why did this update?" signals (when available)

Use time-window filtering; don't force brittle joins.

```sql
SELECT
  source_description,
  destination_description,
  label,
  COUNT(*) AS n
FROM swiftui_causes
WHERE time_ns BETWEEN {frame_start_ns} AND {frame_end_ns}
GROUP BY 1,2,3
ORDER BY n DESC
LIMIT 30;
```

```sql
SELECT
  description,
  backtrace_json
FROM swiftui_changes
WHERE time_ns BETWEEN {frame_start_ns} AND {frame_end_ns}
ORDER BY time_ns
LIMIT 30;
```

### 5.3 Make the cause call (required structure)

For each deep-dived frame, write:

- Lost: missed_frames, hitch_ms, frame_ms
- Primary bottleneck: SwiftUI vs CPU vs GPU/render vs RunLoop vs mixed
- Evidence: 2–5 most convincing measurements
- Root cause hypothesis: specific view, transaction, or backtrace signature
- Fix hypotheses: 1–3 with validation steps

### 5.4 Cluster frames into root causes

You must dedupe frames into causes, otherwise the plan becomes "fix everything".

A practical signature hierarchy:

1. SwiftUI dominant: top `updates.description` or update_group cause backtrace
2. CPU dominant: top `time_profile` backtrace signature
3. GPU/render dominant: render ms, offscreen passes, CA phase dominance
4. If ambiguous: mark as "Needs instrumentation" and state what would disambiguate

Then quantify payoff per cluster:

- frames impacted
- total missed frames
- worst hitch
- confidence

Example payoff query idea (pattern-based):

```sql
-- Example: count how many dropped frames overlap a specific expensive view update pattern
WITH target_frames AS (
  SELECT swap_id, start_ns, end_ns, missed_frames
  FROM frames
  WHERE start_ns BETWEEN {win_start_ns} AND {win_end_ns}
    AND missed_frames > 0
),
hits AS (
  SELECT f.swap_id, f.missed_frames
  FROM target_frames f
  WHERE EXISTS (
    SELECT 1
    FROM updates u
    WHERE u.start_ns < f.end_ns
      AND (u.start_ns + u.duration_ns) > f.start_ns
      AND u.description ILIKE '%YourViewOrAccessorPattern%'
  )
)
SELECT
  COUNT(*) AS frames_impacted,
  SUM(missed_frames) AS total_missed_frames
FROM hits;
```

### 5.5 Final report template (always follow)

Use this structure:

```markdown
# Scroll/Animation Performance Report

## Scope

- Interaction: …
- Mode: Smoothness | Hitch
- Trace: …
- Window selection: signposts | heuristic clusters
- Frame budget: …ms (inferred | assumed)

## Dropped frames summary

- Total frames analyzed: …
- Dropped frames: …
- Severity buckets: (table)
- Worst frames: (top 10 table)

## Worst-frame deep dives (N = …)

### Frame {swap_id} at {time_s}s

- Lost: … missed frames (…ms hitch, …ms frame)
- Primary bottleneck: …
- Breakdown:
  | Component | Evidence | Approx ms |
- Root cause hypothesis: …
- Validation steps: …
- Fix ideas: …

(repeat)

## Root cause clusters (deduped)

| Cluster | Signature | Frames | Total missed frames | Worst | Confidence | Fix idea |
| ------- | --------- | -----: | ------------------: | ----: | ---------- | -------- |

## Plan (prioritized)

1. Fix …
   - Expected payoff: …
   - Risk/complexity: …
   - Validation: …
2. …

## If attribution is unclear: next trace/instrumentation

- Add signposts: …
- Record with template: …
- Collect: …
- Question answered: …
```

---

## Guardrails (Common Failure Modes)

- **Never** diagnose from "total CPU during 10–20s" without first isolating interaction windows and dropped frames.
- If the user says "scroll 10–20s", you MUST:
  1. split into sub-windows (signposts or clusters) and
  2. attribute per frame inside those windows.
- If you can't name the root cause, don't invent one. Output the best hypothesis and the instrumentation needed to confirm.
- Always quantify impact: "fix X affects ~N dropped frames / ~M missed frames".

---

## Reference

- Full schemas and extra SQL patterns: see [SCHEMAS.md](SCHEMAS.md)
- Export trace to DuckDB: `scripts/export_to_duckdb.py`
- **Primary analysis tool**: `scripts/prepare_analysis.py`
  - Default mode: creates views and prints summary
  - Frame analysis mode: `--swap-id {id}` for cascade analysis with signpost/log context
