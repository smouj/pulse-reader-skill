---
name: pulse-reader
version: 1.2.0
description: Monitors and interprets real-time system patterns, metrics, and event rhythms. Provides anomaly detection, trend analysis, and predictive insights for system health and performance.
author: OpenClaw Team
license: MIT
min_openclaw_version: "2.1.0"
tags: [pulse, systems, patterns, insights, monitoring, analysis]
category: observability
dependencies:
  - name: htop
    optional: true
  - name: iostat
    optional: true
  - name: netstat
    optional: true
  - python:
    version: "3.8+"
    packages: [numpy,pandas,scipy]
  - name: prometheus-node-exporter
    optional: true
---

# Pulse Reader Skill

## Purpose

Pulse Reader establishes a continuous heartbeat monitoring system that captures, correlates, and interprets system-level rhythms across CPU, memory, disk I/O, network traffic, and application logs. Unlike basic monitoring that reports static metrics, Pulse Reader identifies emergent patterns, cyclical behaviors, and anomalies in real-time streams.

### Real Use Cases

1. **Detect Memory Leak Progression**: Monitor heap growth patterns across 24-72 hour cycles to identify gradual memory accumulation that standard thresholds miss.

2. **Network Traffic Rhythm Analysis**: Identify daily patterns in API response times correlated with request volume spikes, distinguishing normal load from degradation.

3. **Storage I/O Anomaly Detection**: Recognize when disk write latency diverges from its established 5-minute baseline by 3+ standard deviations, indicating emerging hardware issues.

4. **Application Health Correlation**: Correlate weave patterns between error logs, CPU spikes, and GC pauses to pinpoint root causes of intermittent issues.

5. **Capacity Forecasting**: Analyze historical resource utilization trends to predict when current infrastructure will exceed 80% sustained usage.

## Scope

### Commands

`pulse start [--duration <seconds>] [--interval <ms>] [--channels <list>]`

Starts continuous pulse monitoring with specified parameters. Default: 60s duration, 1000ms interval, all channels.

`pulse status`

Shows active monitoring sessions, baseline establishment status, and current pulse health.

`pulse baseline [--calc-from <minutes>] [--set-manual <json>]`

Establishes or updates the baseline patterns. Auto-calculates from last 60 minutes by default.

`pulse anomalies [--since <duration>] [--severity <level>] [--channel <name>]`

Queries detected anomalies with filtering options. Severity levels: low, medium, high, critical.

`pulse streams [--format {json,table,chart}] [--live]`

Displays raw metric streams or live visualization. Requires optional dependencies for chart format.

`pulse correlate [--with <skill_name>] [--over <duration>]`

Correlates pulse data with external skill outputs. Requires integration config.

`pulse export [--type {insights,raw,patterns}] [--output <path>]`

Exports analysis results in specified format. Raw exports include full metric histories.

### Environment Variables

- `PULSE_INTERVAL_MS`: Default sampling interval (default: 1000)
- `PULSE_STORAGE_BACKEND`: Where to store pulse data (sqlite, memory, prometheus)
- `PULSE_ANOMALY_SENSITIVITY`: Sigma threshold for anomaly detection (default: 2.5)
- `PULSE_MAX_HISTORY_DAYS`: Retention period for historical data (default: 30)

## Work Process

### 1. Initialization Phase

- Skill verifies system metric availability (procfs, sysfs, or node-exporter endpoint)
- Loads previous baseline if exists; otherwise enters baseline establishment mode
- Initializes rolling windows: 5-minute (short-term), 1-hour (medium-term), 24-hour (long-term)

### 2. Baseline Establishment (First 60 minutes)

- Collects metrics at configured interval without anomaly detection
- Calculates mean, median, standard deviation for each metric per time-of-day bucket
- Identifies cyclical patterns using FFT (Fast Fourier Transform) on 5-minute aggregates
- Stores baseline in local SQLite database at `~/.openclaw/pulse_baseline.db`

### 3. Continuous Monitoring Phase

- Each sampling cycle:
  - Collect current metrics from configured channels
  - Compare against time-appropriate baseline bucket
  - Apply statistical process control: if value > mean + (σ × sensitivity), flag as anomaly
  - Update rolling z-score arrays for trend detection
  - Detect pattern shifts using Mann-Kendall test on recent 30-minute windows
  - Generate insights only when anomaly persists for >3 consecutive samples

### 4. Insight Generation

- **Anomaly**: Single metric deviates from baseline (e.g., CPU 5-min avg > 95th percentile)
- **Pattern Shift**: Correlation between metrics changes significantly (e.g., memory usage no longer correlates with CPU)
- **Rhythm Disruption**: Established cycle amplitude changes >40% (e.g., nightly cleanup pattern weakens)
- **Cross-Channel**: Two or more channels show synchronized anomalies indicating systemic issue

### 5. Output Formatting

- Human-readable: colored table with severity indicators
- JSON: complete data including raw values, baselines, z-scores
- Summary: one-line insight per detected issue with suggested action

## Golden Rules

1. **Never trigger on one-time spikes**: Require 3+ consecutive anomaly samples before reporting. Brief spikes are noise.

2. **Time-of-day baselines are sacred**: Separate baselines for weekday/weekend, business/non-business hours. Do not mix.

3. **Correlation ≠ causation**: When multiple channels show anomalies, report correlation strength but avoid claiming root cause without additional evidence.

4. **Respect system load**: If host CPU >90%, automatically reduce sampling frequency by 2x until load normalizes.

5. **Historical context overrides**: If current pattern matches a previously whitelisted anomaly (e.g., scheduled backup), suppress alert.

6. **No state loss on restart**: Always persist pulse state to disk. Never start fresh baseline on each invocation.

7. **Signal-to-noise discipline**: If anomaly rate exceeds 20% of total samples during any hour, increase sensitivity threshold automatically and log warning.

8. ** graceful dependency degradation**: If optional chart libraries missing, fall back to text/table output without error.

## Examples

### Example 1: Start Continuous Pulse with Custom Channels

```bash
pulse start --duration 1800 --interval 500 --channels cpu,memory,disk
```

Output:
```
Starting Pulse Reader...
Duration: 1800s, Interval: 500ms, Channels: cpu,memory,disk
Baseline: Established (52 days)
Session ID: p-20260305-143022
```

### Example 2: Query Critical Anomalies from Last 24 Hours

```bash
pulse anomalies --since 24h --severity high
```

Output (table format):
```
TIME                 CHANNEL   METRIC        VALUE  BASELINE  Z-SCORE  INSIGHT
2026-03-04 08:15:22  memory   used_percent  94.2%   67.1%     3.8      Memory usage rhythm disrupted: sustained high during off-hours
2026-03-04 12:43:11  cpu      load_5min     4.8     1.9       2.9      CPU pattern shift: increased volatility detected
2026-03-05 02:00:05  disk     await_ms      42.3    15.2      3.2      I/O latency anomaly: 3x baseline deviation
```

### Example 3: Establish New Baseline After Major Deployment

```bash
pulse baseline --calc-from 120
```

Output:
```
Recalculating baseline from last 120 minutes of data...
Processing 14,400 samples across 4 channels...
Baseline updated: 2026-03-05 14:22:18 UTC
FFT detected cycles: daily (1440 min), weekly (10080 min)
Stored to ~/.openclaw/pulse_baseline.db
```

### Example 4: Correlate Pulse with Application Logs

```bash
pulse correlate --with log-analyzer --over 6h
```

Output:
```
Correlation window: last 6 hours
pattern-error-rate: r=0.87 (strong positive)
pattern-response-time vs memory-pressure: r=0.72 (moderate)
Detected: Memory pressure increases precede error rate spikes by ~45 seconds
```

### Example 5: Export Raw Stream for Offline Analysis

```bash
pulse export --type raw --output /tmp/pulse_dump.json
```

Output:
```
Exported 1,284,000 raw samples (CPU, Memory, Disk, Network)
 compressed size: 47.3 MB
Format: JSON Lines (one sample per line)
File: /tmp/pulse_dump.json
```

## Rollback Commands

```bash
# Restore baseline from backup (if corrupted)
pulse baseline --set-manual "$(cat ~/.openclaw/backups/pulse_baseline_20260304.json)"

# Disable anomaly detection temporarily (maintenance window)
pulse config set anomaly_detection false

# Clear current session state (start fresh)
rm -f ~/.openclaw/pulse_session_*.db && pulse reset

# Revert to previous baseline version (if auto-update caused false positives)
pulse baseline --restore previous

# Disable specific noisy channel
pulse config set channels_exclude [disk]

# Rollback correlation settings
pulse correlate --disconnect --all
```

## Troubleshooting

**Error: "No metric sources available"**
- Ensure `/proc` is mounted and readable
- Verify node-exporter running if configured: `systemctl status prometheus-node-exporter`
- Check `PULSE_STORAGE_BACKEND` points to writable location

**Baseline never stabilizes (anomaly rate >30%)**
- System may be inherently inconsistent. Run `pulse baseline --calc-from 1440` using full 24h cycle.
- Increase `PULSE_ANOMALY_SENSITIVITY` to 3.5 or 4.0 temporarily.
- Check for recurring cron jobs causing predictable spikes.

**High CPU usage from pulse itself**
- Reduce sampling interval to >=2000ms
- Limit channels to only essential metrics
- Switch storage backend to memory-only (ephemeral): `PULSE_STORAGE_BACKEND=memory`

**False positives during known maintenance windows**
- Add maintenance periods via `pulse whitelist add --window "Sat 02:00-04:00"`
- Or temporarily set `PULSE_ANOMALY_SENSITIVITY` higher during maintenance.

**Cross-correlation not working**
- Ensure dependent skill is actively generating output
- Verify time synchronization between skill logs and pulse timestamps
- Check `pulse correlate --validate` for connection status.

**Database locked errors**
- Another pulse session is running. Stop all: `pulse stop --all`
- Remove stale lock: `rm ~/.openclaw/pulse_*.lock`
```