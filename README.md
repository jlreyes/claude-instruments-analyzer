# Claude Instruments

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that gives Claude programmatic access to Apple Instruments trace data. Instruments is normally a GUI-only tool — this skill bridges the gap by exporting `.trace` files into DuckDB, where they can be queried with SQL.

## What it does

Given an Instruments `.trace` file, this skill will:

1. **Export** the trace to DuckDB + Parquet (via `export_to_duckdb.py`)
2. **Explore** the exported tables — CPU profiling, hitches, hangs, signposts, Core Animation, SwiftUI updates, RunLoop activity, and more
3. **Prepare** derived views for frame-level analysis (via `prepare_analysis.py`)
4. **Analyze** performance data using SQL queries with full schema knowledge

## Install

### As a Claude Code skill

```bash
claude install-skill jlreyes/claude-instruments-analyzer
```

### Manual

Clone and copy the skill files into your project's `.claude/skills/` directory:

```bash
git clone https://github.com/jlreyes/claude-instruments-analyzer.git
cp -r claude-instruments-analyzer/. your-project/.claude/skills/claude-instruments-analyzer/
```

## Usage

### 1. Record a trace

```bash
xcrun xctrace record --template 'SwiftUI' --time-limit 20s \
  --output ./traces/recording.trace \
  --attach YourApp --no-prompt
```

Or use the included `PerfDebugging.tracetemplate` in Instruments.

### 2. Export to DuckDB

```bash
./scripts/export_to_duckdb.py traces/recording.trace traces/recording/analysis.duckdb
```

Requires [uv](https://github.com/astral-sh/uv) (dependencies are managed inline via PEP 723).

### 3. Ask Claude to analyze

```
Analyze the performance trace in traces/recording/analysis.duckdb.
```

## What's included

| File | Description |
|------|-------------|
| `SKILL.md` | The skill prompt — Instruments → DuckDB workflow |
| `SCHEMAS.md` | Full DuckDB schema reference for all exported tables |
| `scripts/export_to_duckdb.py` | Exports Instruments `.trace` to DuckDB + Parquet |
| `scripts/prepare_analysis.py` | Creates analysis views, frame summaries, cascade analysis |
| `PerfDebugging.tracetemplate` | Instruments template for recording traces |

## Use-case companion skills

This skill provides the foundation. Companion skills build on it for specific diagnosis workflows:

| Skill | Description |
|-------|-------------|
| [swiftui-jank-diagnostics](https://github.com/jlreyes/swiftui-jank-diagnostics) | SwiftUI scroll and animation jank diagnosis — frame-first workflow with interaction windowing, severity buckets, cascade analysis, per-frame attribution, root cause clustering, and prioritized fix plans |

## Requirements

- macOS with Xcode / Instruments
- [uv](https://github.com/astral-sh/uv) (for running the Python scripts)
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)

## License

MIT
