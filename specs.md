# Solf Wakelock Detector - Specification

## Overview

An Android app that tracks which apps hold wakelocks during screen-off periods, with a focus on identifying apps that are the **sole reason** the phone isn't entering deep sleep. Targets root and Shizuku users initially; a non-root/non-Shizuku fallback tier may be added later.

## Core Concept

Periodically poll `dumpsys power` while the screen is off to capture point-in-time snapshots of all active wakelocks. By comparing snapshots over time, build per-app timelines showing:

- How long each app held wakelocks during screen-off
- When an app was the **sole holder** preventing deep sleep
- Wakelock tags (distinguishing e.g. music playback from stuck syncs)

## Data Source

### Primary: `dumpsys power` via privileged shell

Executed via root (`su`) or Shizuku (shell-level binder IPC). These two paths produce identical data and share the same parsing/collection code.

Example output (Wake Locks section):

```
Wake Locks: size=3
  PARTIAL_WAKE_LOCK  'NlpWakeLock' ACQ=-2s412ms (uid=10057 pid=4231)
  PARTIAL_WAKE_LOCK  '*job*/com.spotify.music/...' ACQ=-45s120ms (uid=10134 pid=8822 ws=WorkSource{10134})
  PARTIAL_WAKE_LOCK  'mqtt-wakelock' ACQ=-3m22s (uid=10089 pid=6104)
```

Fields of interest per wakelock entry:
- **Type**: PARTIAL_WAKE_LOCK (the only type that prevents CPU sleep)
- **Tag**: the string name (e.g. `NlpWakeLock`, `*job*/com.spotify.music/...`)
- **UID**: owning app's user ID (maps to package via PackageManager)
- **PID**: process ID
- **WorkSource**: if present, the UID of the app that actually caused this wakelock (important for system_server-mediated locks)
- **ACQ time**: how long the lock has been held

### Sleep Detection

- `SystemClock.elapsedRealtime()` vs `SystemClock.uptimeMillis()` delta gives cumulative deep sleep time
- Used to show overall sleep health stats (% of screen-off time spent in deep sleep)

## Polling Strategy: Zero-Wakelock Opportunistic Sampling

The app must not wake the device or hold wakelocks itself. Instead, it piggybacks on wake events caused by other apps.

### Mechanism

- A **foreground service** (no wakelock) stays alive to register runtime receivers and manage alarm scheduling. The foreground service keeps the process in memory but does NOT prevent CPU suspend.
- Listen for `ACTION_SCREEN_OFF` / `ACTION_SCREEN_ON` broadcasts (runtime-registered)
- On screen off: schedule `AlarmManager` repeating alarm using **`ELAPSED_REALTIME`** (NOT `_WAKEUP`). This alarm type only fires when the device is already awake for other reasons.
- When alarm fires: a BroadcastReceiver executes `dumpsys power`, stores the snapshot, and returns. The system holds a brief wakelock for `onReceive()` (~10s allowed), and `dumpsys` completes in ~50-100ms, so no app-held wakelock is needed.
- On screen on: cancel alarms, finalize the screen-off session

### Why This Works

- If the CPU is suspended (deep sleep), there are no wakelocks held by definition -- nothing to measure
- If the CPU is awake, some other app's wakelock is keeping it awake -- that's exactly when we need a snapshot
- The `ELAPSED_REALTIME` alarm silently waits during deep sleep and fires at the next wake event
- **Result: zero wakeups caused, zero wakelocks held, zero battery impact from monitoring**

### Configurable Interval

- Default: 60 seconds
- Shorter (30s): more temporal resolution, still zero battery impact since we don't wake the device
- Longer (120s): coarser data but fewer snapshots to store
- Since we don't wake the device, shorter intervals are less costly than in a traditional polling design

### Acceptable Tradeoffs

- Very short wakelocks (<interval) between sample points may be missed; these are generally benign
- Sampling is irregular (driven by when the device happens to be awake), so intervals between snapshots vary. Analysis uses actual timestamps, not fixed intervals.
- If no app holds wakelocks at all (phone is sleeping perfectly), we get no snapshots -- but there's nothing to report

## Privilege Tier

### Tier 1: Root

- Execute `su -c dumpsys power`
- Fully automatic, survives reboot, no user interaction after initial grant
- Detected at startup by checking for `su` binary

### Tier 2: Shizuku

- Use Shizuku API to execute `dumpsys power` with shell privileges
- Same data and code path as root
- Requires user to install Shizuku app and grant access
- **Reboot behavior**: Shizuku must be restarted after reboot
  - Android 11+: user can restart on-device via Wireless Debugging (few taps)
  - Android 10: requires ADB connection to a computer
- App detects Shizuku availability at startup and on reboot

### Degraded State (no root, no Shizuku)

- App cannot collect per-app wakelock data
- Shows a persistent notification explaining the situation
- Can still show aggregate sleep stats via clock-delta method
- Guides user toward setting up root or Shizuku

## Boot Behavior

- Register a `BOOT_COMPLETED` BroadcastReceiver
- On boot:
  1. Check for root access → if available, start monitoring service
  2. Check for Shizuku → if available and running, start monitoring service
  3. If Shizuku is installed but not yet started, register Shizuku's binder-received listener and wait
  4. If neither available → show notification: "Wakelock monitoring unavailable. Root or Shizuku required."
- Also register for Shizuku's lifecycle callbacks so monitoring can start/stop if Shizuku is started/stopped mid-session

## Data Model

### WakelockSnapshot

Captured each polling interval:

- `timestamp`: when the snapshot was taken
- `entries[]`: list of active wakelocks
  - `type`: wakelock type (PARTIAL, etc.)
  - `tag`: wakelock tag string
  - `uid`: owning UID
  - `pid`: owning PID
  - `workSourceUid`: attributed UID if WorkSource present (else same as uid)
  - `packageName`: resolved from UID via PackageManager
  - `acquiredDuration`: how long held at time of snapshot

### ScreenOffSession

A complete screen-off period:

- `startTime`: when screen turned off
- `endTime`: when screen turned on
- `deepSleepTime`: computed from clock-delta
- `snapshots[]`: ordered list of WakelockSnapshots
- `perAppSummary[]`: computed rollup
  - `packageName`
  - `totalWakelockTime`: estimated time this app held wakelocks
  - `soleHolderTime`: estimated time this app was the ONLY wakelock holder
  - `tags[]`: distinct wakelock tags seen

### Sole-Holder Calculation

For each interval between consecutive snapshots:

```
if snapshot has exactly 1 wakelock entry:
    sole_holder[entry.package] += interval_duration
else if snapshot has N > 1 entries:
    for each entry:
        shared_holder[entry.package] += interval_duration
```

Edge case: if a snapshot has 0 entries, the phone may have briefly entered deep sleep between polls. The clock-delta can confirm this.

## Storage

- Room database for persistence
- Retain detailed snapshots for a configurable period (default: 7 days)
- Retain per-session summaries for longer (default: 90 days)
- Old snapshots pruned on a schedule

## UI (Initial / MVP)

### Dashboard Screen

- Current status: monitoring active/inactive, privilege level (root/Shizuku/none)
- Today's summary: total screen-off time, deep sleep time, deep sleep %, awake time
- Top offenders: ranked list of apps by screen-off wakelock time (with sole-holder time highlighted)

### Session History

- List of screen-off sessions, most recent first
- Each shows: duration, deep sleep %, top wakelock holders
- Tap to drill into session detail

### Session Detail

- Timeline visualization of the screen-off period
- Show which apps held wakelocks at each point in time
- Highlight sole-holder periods
- Per-app breakdown with tag-level detail

### Settings

- Polling interval (30s / 60s / 120s)
- Data retention period
- Notification preferences
- Privilege mode info (root/Shizuku status)

## Target Compatibility

- **minSdk**: 21 (Android 5.0) -- but practical minimum for Shizuku path is API 23 (Android 6.0)
- **targetSdk**: latest stable (35 / Android 15)
- Root path works on all versions
- Shizuku path: full functionality on Android 11+ (on-device restart), limited on 10 (ADB restart)

## Technical Stack (Proposed)

- **Language**: Kotlin
- **UI**: Jetpack Compose (Material 3)
- **Architecture**: MVVM with clean-ish layers
- **Persistence**: Room
- **Background work**: Foreground Service + BroadcastReceivers
- **Dependency injection**: Hilt
- **Shizuku integration**: Shizuku API library

## Open Questions

- Should we also parse `dumpsys batterystats` for historical/cumulative data, or rely solely on our own snapshots?
- Notification channel strategy: separate channels for "monitoring active" (low priority) vs "degraded state" (high priority)?
- Export/share functionality for session data (CSV, share with developer, etc.)?
