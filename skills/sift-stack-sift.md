---
name: Sift
description: Use when ingesting telemetry data, analyzing hardware test runs, detecting anomalies with rules, comparing runs against baselines, creating derived signals, managing multi-run review campaigns, or exporting data for external analysis. Sift is a telemetry platform for teams working on complex machines (rockets, aircraft, robots, manufacturing systems) who need to move from raw data to confident decisions.
metadata:
    mintlify-proj: sift
    version: "1.0"
---

# Sift Skill

## Product summary

Sift is a telemetry platform for analyzing hardware test data, flight missions, and manufacturing runs. It ingests structured and unstructured data (Protobuf, Influx, CSV, logs), stores it organized by Assets (systems), Channels (signals), and Runs (sessions), and provides tools to explore, detect anomalies, review results, and export findings. Agents use Sift to ingest telemetry, build rules for automated detection, compare runs against baselines or families, create derived signals via expressions, manage multi-run review campaigns, and export data for external tools.

**Key files and commands:**
- CLI: `sift-cli` (import/export files, manage profiles)
- Config: `sift.toml` (CLI profile configuration)
- API: REST and gRPC endpoints (see `/documentation/manage/set-up-api-access`)
- Primary docs: https://docs.siftstack.com

## When to use

Reach for this skill when:
- **Ingesting data**: Streaming telemetry from Python, gRPC, Protobuf, Influx, or importing CSV/Parquet/TDMS/HDF5/ULog files
- **Analyzing runs**: Exploring channels, comparing signals over time, investigating anomalies, aligning runs for comparison
- **Detecting issues**: Writing rules (CEL expressions) to flag threshold breaches, anomalies, or state changes; evaluating rules against runs
- **Comparing baselines**: Creating families (groups of known-good runs) and comparing new runs against family statistics
- **Transforming data**: Creating calculated channels (derived signals) using expressions without modifying raw data
- **Reviewing results**: Setting up rules, generating reports, triaging annotations, tracking multi-run campaigns
- **Exporting data**: Pulling telemetry to CSV, Parquet, or MATLAB for external analysis
- **Managing workspace**: Organizing resources with metadata, setting up API access, configuring user access

## Quick reference

### Core data model

| Concept | Definition | Example |
|---------|-----------|---------|
| **Asset** | System or test article being examined | Rocket stage, aircraft, manufacturing line |
| **Channel** | Individual measurement or signal from an asset | Temperature, pressure, voltage, vibration |
| **Run** | Captured session of data from one or more channels | Single flight, test stand cycle, production batch |
| **Calculated Channel** | Derived signal computed from channels using CEL expressions | Power = voltage × current |
| **Rule** | Logical condition (CEL expression) that flags issues in telemetry | `$1 > 100` (channel exceeds threshold) |
| **Report** | Evaluation of rules against a specific run | Results showing which rules passed/failed |
| **Annotation** | Timestamped marker created when a rule fires or manually | Issue flagged for review with status/assignee |
| **Campaign** | Groups reports from multiple runs for coordinated review | Qualification test program across 50 runs |
| **Family** | Named group of related runs used as baseline for comparison | All known-good acceptance tests for engine config |

### CLI commands

```bash
# Setup
sift-cli config create                    # Create config file
sift-cli config update --grpc-uri ... --rest-uri ... --api-key ... --app-uri ...
sift-cli ping                             # Verify connectivity

# Import
sift-cli import --asset <name> --run <name> --file <path> --format csv
sift-cli import --asset <name> --run <name> --file <path> --preview  # Dry run

# Export
sift-cli export --run-id <id> --output <file> --format csv --channel <name>
sift-cli export --run-id <id> --output <file> --format parquet --channel-regex "motor.*"
```

### Expression syntax (CEL)

| Category | Examples |
|----------|----------|
| **Operators** | `$1 + $2`, `$1 > 100`, `$1 && $2`, `$1 ? 'High' : 'Low'` |
| **Math** | `abs()`, `ceil()`, `floor()`, `sqrt()`, `pow()`, `log()`, `sin()`, `cos()` |
| **String** | `contains()`, `startsWith()`, `endsWith()`, `split()`, `toLower()`, `toUpper()` |
| **Stateful** | `avg($1, rolling(5s))`, `max($1, rolling(1m))`, `stdev($1, rolling(10m))`, `delta($1)`, `deriv($1)`, `persistence($1 > 100, 5s)` |
| **File access** | `assetFile("limits.json")["max_temp"]`, `runFile("params.csv")["threshold"][0]` |
| **Channel refs** | `$1` (first), `$2` (second), `$3` (third) — assigned by selection order |

### API authentication

```bash
# REST: Include in Authorization header
curl -H "Authorization: Bearer $SIFT_API_KEY" https://$SIFT_REST_URI/api/v1/...

# gRPC: Pass as metadata
grpcurl -H "authorization: Bearer $SIFT_API_KEY" $SIFT_GRPC_URI service.Method
```

Obtain API key and base URLs from Sift UI: Profile → Manage → API Keys.

## Decision guidance

### When to use X vs Y

| Decision | Use X when | Use Y when |
|----------|-----------|-----------|
| **Import vs Stream** | Historical data, batch uploads, one-time loads | Live data, continuous monitoring, real-time analysis |
| **Python client vs gRPC** | Rapid prototyping, simple scripts, Python ecosystem | High-frequency ingestion, language-agnostic, performance-critical |
| **Schemaless vs Config-based streaming** | Quick start, no pre-registration, flexible schema | Structured data, schema validation, performance optimization |
| **Rule vs Calculated Channel** | Detecting issues, flagging anomalies, review workflows | Transforming data, deriving new signals, reusable metrics |
| **Family baseline vs Single baseline** | Multiple reference runs, statistical envelope, regression detection | Single known-good run, simple visual comparison |
| **Report vs Campaign** | Reviewing single run, quick assessment | Multi-run review, coordinated effort, progress tracking |
| **Metadata vs Tags** | Structured filtering, consistent taxonomy, governance | Unstructured labels, quick annotations |
| **CLI vs API** | One-off imports/exports, terminal workflows | Automated pipelines, CI/CD integration, programmatic control |

## Workflow

### Typical task: Ingest data, detect anomalies, and review results

1. **Prepare data and authenticate**
   - Obtain API key and base/gRPC URLs from Sift UI (Profile → Manage → API Keys)
   - For CLI: Run `sift-cli config create` and `sift-cli config update` with credentials
   - For Python: Set `SIFT_API_KEY`, `SIFT_URI`, `BASE_URI` environment variables

2. **Ingest telemetry**
   - **File import**: Use CLI (`sift-cli import --asset <name> --run <name> --file <path>`) or UI (Runs → Import Data)
   - **Streaming**: Use Python client, gRPC, Protobuf, Influx, or schemaless JSON to `/api/v2/ingest`
   - Define Asset (system), Channels (signals), and Run (session) during ingestion
   - Verify ingestion: Check sift_app asset for data_import or ingest_grpc metrics

3. **Explore and understand the data**
   - Open Run in Explore, select Channels to plot
   - Compare signals over time, align runs for side-by-side comparison
   - Create multi-panel layouts (Timeseries, Table, etc.) and save views
   - Use Live mode to monitor streaming data in real time

4. **Create detection rules**
   - Navigate to Rules, click New Rule
   - Select Asset and input Channels
   - Write CEL expression (e.g., `$1 > 100`, `avg($1, rolling(5s)) > threshold`)
   - Preview against a sample Run to validate
   - Save rule for reuse across runs

5. **Generate and review reports**
   - Open Run, click Generate Report
   - Select rules to evaluate (or use Report Template for consistency)
   - Review results: which rules passed, which flagged issues
   - Triage Annotations: assign, comment, update status (Open → Accepted/Failed)

6. **Track multi-run campaigns** (optional)
   - Create Campaign, add Reports from multiple Runs
   - View all Annotations across runs, filter by status/assignee/rule
   - Monitor progress toward completion

7. **Export and share**
   - Export Run data: CLI (`sift-cli export --run-id <id> --output file.csv`) or UI (Explore → Export)
   - Share Explore links for consistent collaboration
   - Export to MATLAB or external tools via REST API

### Typical task: Create derived signals and compare against baseline

1. **Create a calculated channel**
   - Navigate to Calculated Channels, click New
   - Select input Channels (e.g., voltage, current)
   - Write expression (e.g., `$1 * $2` for power)
   - Save; channel is now available in Explore and Rules

2. **Group runs into a family**
   - Navigate to Families, click New Family
   - Add known-good Runs as members
   - Compute statistics (mean, stdev) across the group

3. **Compare new run against family**
   - Open new Run in Explore
   - Load Family as overlay on shared aligned time axis
   - Visually inspect variance and identify outliers

4. **Create family rule** (optional)
   - Write rule using family statistics: `$1 > family_avg + 2 * family_stdev`
   - Evaluate against new Run to flag deviations

## Common gotchas

- **Duplicate timestamps**: If a channel receives two values at the exact same timestamp, Sift keeps only the most recent and discards the earlier one.
- **Missing channels in rules**: All channels referenced in a rule expression must exist on the run being evaluated. If a channel is absent, the rule cannot evaluate and returns an error. Use conditional logic or file functions as workarounds.
- **Rolling window limits**: Stateful functions (rolling windows) are limited to 10 minutes. Longer windows are not supported.
- **Calculated channel nesting**: Calculated channels can reference other calculated channels up to 10 levels deep. Beyond that, nesting fails.
- **Bitfield channels unsupported**: Bitfield channels cannot be used as inputs to calculated channels.
- **File functions in live rules**: File functions (`assetFile()`, `runFile()`) are not supported in live (real-time) rule evaluation. They work only in retrospective analysis.
- **Enum comparisons**: Enum channels are exposed as strings in CEL. Compare using quoted literals: `$1 == 'FAULT'`, not `$1 == FAULT` or `$1 == 3`.
- **Data gaps**: When a channel stops reporting, Sift carries forward its last known value indefinitely. `previous()` and `delta()` treat the gap as continuous; `changed()` does not fire when the channel resumes.
- **Rule evaluation trigger**: Rules only evaluate when a referenced channel receives a new data point. If a rule references only a silent channel, the rule stops evaluating entirely and cannot detect the silence.
- **API rate limits**: Sift enforces per-organization and per-endpoint rate limits. Batch requests where possible, avoid tight retry loops, and spread large backfills over time.
- **Ingestion config reuse**: In Python streaming, reuse the same `ingest_client` for the entire duration. Recreating it per batch creates a new ingestion config each time, causing significant performance overhead.
- **Sample rate bias**: When comparing live signal values against family statistics, use `bucket()` instead of `rolling()` to normalize signals sampled at different rates.

## Verification checklist

Before submitting work:

- [ ] **Data ingested**: Verify run appears in Sift UI and channels are populated (check sift_app metrics if using streaming)
- [ ] **Channels correct**: Confirm channel names, units, and data types match expected schema
- [ ] **Rule expression valid**: Preview rule against sample run; confirm it fires when expected and only when expected
- [ ] **Baseline established**: If using families, confirm all known-good runs are members and statistics are computed
- [ ] **Report generated**: Run report against target run; verify rules evaluated without errors
- [ ] **Annotations reviewed**: Check that flagged issues are accurate and assignees are notified
- [ ] **Export validated**: If exporting, confirm output file format, channel selection, and row count match expectations
- [ ] **Sharing configured**: If sharing Explore links, verify recipients have access to asset and run
- [ ] **API credentials secure**: Confirm API keys are stored in environment variables or secret management, not hardcoded
- [ ] **CLI profile tested**: Run `sift-cli ping` to verify connectivity before automation

## Resources

- **Comprehensive page listing**: https://docs.siftstack.com/llms.txt
- **Getting started**: https://docs.siftstack.com/documentation/get-started/what-is-sift
- **Data model (Assets, Channels, Runs)**: https://docs.siftstack.com/documentation/get-started/data-model
- **API authentication**: https://docs.siftstack.com/documentation/manage/set-up-api-access
- **Expression syntax (CEL)**: https://docs.siftstack.com/documentation/reference/expression-syntax
- **CLI reference**: https://docs.siftstack.com/documentation/reference/cli-reference
- **Review pipeline (Rules, Reports, Annotations, Campaigns)**: https://docs.siftstack.com/documentation/review/overview

---

> For additional documentation and navigation, see: https://docs.siftstack.com/llms.txt