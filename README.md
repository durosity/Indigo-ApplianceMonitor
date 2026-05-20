# Indigo-ApplianceMonitor
Monitor your household appliances for when they’ve finished a task

# ApplianceMonitor — Setup & User Guide

A guide to getting ApplianceMonitor configured and using the auto-learn feature to reliably detect appliance cycles.

---

## What it does

ApplianceMonitor watches the power draw of an appliance (washing machine, dishwasher, tumble dryer, oven, etc.) via a smart plug or energy monitor, and detects when a cycle starts, finishes, gets stuck, or hasn't been unloaded. It can fire Indigo triggers on any of those events.

The auto-learn system records the power signature of each programme your appliance runs and uses that to identify which programme is currently running — so a "Quick 30" finishing can fire a different trigger than a "Cottons 60" finishing.

---

## What you need

- An Indigo device that reports current power draw in watts as one of its states (a smart plug like a TP-Link Kasa, a Shelly, a Sonoff with energy monitoring, a UniFi smart outlet — anything that exposes wattage to Indigo).
- The appliance plugged into that smart device.
- Optionally: a door sensor on the appliance (handy for washing machines and dishwashers so the plugin knows when you've unloaded).

---

## Installing

1. Double-click the `.indigoplugin` bundle to install in Indigo.
2. Enable the plugin from **Plugins → ApplianceMonitor → Enable**.

---

## Creating your first appliance device

1. **Devices → New…**, type **ApplianceMonitor**, device type **Appliance Monitor**.
2. Give it a name (e.g. *Washing Machine*).
3. Click **Edit Device Settings…**.

### Device Type preset

Pick the closest preset from the dropdown (washing machine, dishwasher, tumble dryer, oven, etc.) and click **Load Preset**. This fills in sensible default thresholds. You can fine-tune these or — better — use auto-learn (see below) to replace them with values measured from your actual appliance.

### Power source

- **Power Monitoring Device**: pick the smart plug / energy device.
- **Power State Key**: the state on that device that reports current watts (often `curEnergyLevel` for Insteon energy devices, or `power` / `powerW` on others). The dropdown updates once you pick the device.
- **Poll Interval**: how often to read the power value. 10 seconds is a good default — lower uses more CPU, higher might miss short power dips.

### Thresholds (the fallback ones)

These are what the plugin uses when you have no learned profiles. Once you've learned at least one profile, the plugin builds its own envelope from your profiles and the fallback values become dormant. You can leave them at the preset defaults for now.

### Door sensor (optional)

Tick **Enable Door Sensor** if you have one. Pick the sensor device and the state that goes true when the door opens. This lets the plugin distinguish "cycle finished but not unloaded" from "cycle finished and unloaded".

### Alerts (optional)

- **Long Running Alert**: fires if a cycle runs longer than the threshold you set (handy for catching wash cycles that hang).
- **Stuck Detection**: marks the device as `stuck` if the cycle exceeds the max run time (a more serious version of long-running — escalate it differently).
- **Unload Reminder**: fires N minutes after cycle completion if the door hasn't been opened (your "go and empty the washing machine" nudge).

### High/Low sub-state (optional)

When enabled, the plugin tracks whether the appliance is in a high-power phase (heating, etc.) vs low-power (tumbling, rinsing). Useful if you want triggers based on the heating stage rather than just running/finished.

**Save the device.** It should now show as `idle` in Indigo.

---

## Using Auto-Learn

This is where the plugin gets clever. Instead of guessing thresholds, it measures them from your actual appliance.

### Learning a profile

1. Open the device config again.
2. Scroll to the **Auto-Learn Profiles** section.
3. **Profile Name**: name this programme — e.g. `Cottons 60`, `Quick 30`, `Eco 40`, `Heavy Wash`. Be specific; you'll want to tell them apart later.
4. **Cycle Duration (minutes)**: roughly how long this programme runs. Round up generously — if a cycle says 1h45m on the appliance, set 110 minutes. The plugin auto-stops at this time, so giving extra buffer is fine; cutting it short will truncate the profile.
5. Click **Start Learning**. The config closes.
6. **Immediately start the appliance programme** you're learning.
7. The device state in Indigo will show `learning` while it's recording. Progress is logged to the Indigo log every minute.
8. When the duration elapses, the plugin analyses the trace and saves the profile. You'll see something like this in the log:

   ```
   Washing Machine: learning complete — profile 'Cottons 60' saved.
     Duration 87.3 min, peak 2050W, avg 612W, 2 high-power bursts, 0.892 kWh.
   Washing Machine: suggested thresholds — start 8.0W, stop 3.0W, stop debounce 180s, high 1100W.
   ```

9. The device returns to `idle`.

Repeat for every programme you regularly use. **You don't need to learn every possible programme** — just the ones you actually run. The plugin's permissive envelope means even unfamiliar programmes will still get detected as "running" and "finished" — they just won't have a profile name attached.

### Tips for good profiles

- **One profile per programme**, not per appliance. If you do quick washes and cottons washes, that's two profiles.
- **Run a real load**, not an empty cycle. Empty cycles use less energy and shorter motor runs, which throws off the signature.
- **Pad the duration** a bit. If the cycle ends before the timer, you've recorded a few minutes of idle at the end, which is harmless. If the timer ends before the cycle, you've truncated the signature, which hurts matching.
- **One profile at a time**. Don't start learning a new profile while another appliance is using the same smart plug.

### Reviewing and deleting profiles

Open the device config and scroll to **Saved profiles for this device**. The dropdown lists each profile with its duration and peak power. Pick one and click **Delete Selected Profile** to remove it.

If you want to inspect or edit the raw data, profiles live in JSON files in:

```
~/Library/Application Support/Perceptive Automation/Indigo X/Preferences/Plugins/com.durosity.appliancemonitor/
```

One file per device, named `device_<deviceID>_profiles.json`. They contain the signature, suggested thresholds, and the full raw power trace.

---

## What happens during a cycle

Once you've learned at least one profile, here's what happens when you run an appliance:

1. **Idle** — power is below the start threshold. Plugin watches and waits.
2. **Power rises** above the start threshold and stays there for the start debounce period (default 30s, prevents transient spikes triggering false starts).
3. **Running** — `applianceState` becomes `running`, `cycleStartTime` is set, the trigger **Cycle Started** fires.
4. **Matching begins** — the plugin updates a live signature every poll and scores it against every learned profile every ~minute. The best match populates `activeProfile`, and `matchConfidence` rises as the cycle progresses.
5. **Matched profile can change** — early in a cycle, "Quick 30" and "Cottons 60" look identical (both heating). The plugin commits provisionally, then switches if the live trace diverges from the candidate. Each switch fires the **Profile Match Changed** trigger.
6. **Power drops** below the stop threshold and stays there for the stop debounce period (the long-pause-tolerance). For washing machines this can be 2-3 minutes — they pause between rinses.
7. **Finished** — `applianceState` becomes `finished`, cycle stats are saved, the trigger **Cycle Complete** fires. `lastCycleProfile` now holds the matched profile name.
8. **Unloaded** (if door sensor enabled) — opening the door moves the device to `unloaded` and fires **Door Opened (Unloaded)**.
9. **Back to idle** — closing the door (or after a delay if no door sensor) returns the device to `idle`, ready for the next cycle.

---

## Building automations

The most useful trigger events:

| Event | Fires when | Typical use |
|-------|-----------|-------------|
| **Cycle Started** | Cycle begins | Log the start time, switch on a status light |
| **Cycle Complete** | Cycle finishes | Send a push notification, speak via HomePod, flash a light |
| **Door Opened (Unloaded)** | Door opens after cycle complete | Cancel pending reminders, log unload time |
| **Unload Reminder** | N minutes after finish, door still closed | Send "go empty the machine" nudge |
| **Long Running Alert** | Cycle exceeds expected duration | Investigate (jammed cycle, didn't drain, etc.) |
| **Stuck / Fault Detected** | Cycle wildly exceeds max run time | Urgent alert — appliance may have faulted |
| **Profile Match Changed** | Matched profile switches mid-cycle | Adjust ETA notifications based on which programme is now matched |
| **New Profile Learned** | Learning session completes | Log that a new profile is available |

### Acting on which profile completed

Add an Indigo trigger conditional on the device's `lastCycleProfile` state to differentiate actions:

```
IF Cycle Complete on "Washing Machine"
   AND lastCycleProfile equals "Cottons 60"
THEN
   Notify: "Cottons wash done — hot iron required"
ELSE IF lastCycleProfile equals "Quick 30"
THEN
   Notify: "Quick wash done — straight on the airer"
```

### Gating on match confidence

Early-cycle matches are unreliable (10 min into any wash, everything looks like heating). For triggers that fire mid-cycle, gate on confidence:

```
IF activeProfile changes
   AND matchConfidence > 0.7
THEN
   ...act with confidence...
```

For end-of-cycle triggers, confidence is usually high (the full trace has been observed by then) so you can act on `lastCycleProfile` directly without gating.

---

## Manual override actions

The plugin exposes these actions in **Indigo → Action Group editor → ApplianceMonitor**:

- **Set State: Idle / Running / Finished / Unloaded / Stuck** — force a state change. Handy for testing automations without running a real cycle.
- **Reset Cycle Stats** — zero out the current cycle's energy/duration/peak values.
- **Reset Cycle Count** — zero out the total cycle counter.

---

## Troubleshooting

**The plugin never detects a cycle starting.**
The start threshold is set higher than the actual cycle's peak. Look at the smart plug's reported power during a wash — if it never goes above (say) 8W and your start threshold is 10W, the plugin will never trigger. Lower the threshold, or learn a profile and let the envelope do it for you.

**It detects a cycle ending halfway through.**
The stop debounce is too short. Washing machines especially have long quiet periods between rinses. Increase the stop debounce time, or learn a profile (which measures the longest mid-cycle quiet gap automatically and sizes the debounce accordingly).

**It's matching the wrong profile.**
This is most common in the first 10–15 minutes when programmes look identical. Wait — the matching keeps refining throughout the cycle. If it's *still* wrong at the end, the profiles may be too similar to distinguish, or one was learned on an empty cycle. Re-learn with a real load.

**Confidence is always low.**
Either you only have one profile (no comparison possible, confidence stays modest) or your profiles are very similar. The score gap between best and second-best match drives confidence — closely-matched profiles produce low confidence by design.

**`UI dynamic list function returned illegal ID string` in the log.**
You're running v1.0.0. Upgrade to v1.1.0.

**The device shows `learning` after a plugin restart but isn't actually learning.**
Restarts mid-learning are now handled cleanly in v1.1.0 (the device resets to `idle` automatically). If you see this on v1.0.x, just open the device config and toggle a setting to force a state refresh.

---

## A worked example: setting up a washing machine

1. Plug the washing machine into a smart plug that reports wattage to Indigo.
2. Create the device, name it *Washing Machine*, load the washing machine preset.
3. Set the power device and state key. Save.
4. Run a load of cottons on 60°C. Time it — say it's 2h10m.
5. Open device config, set profile name *Cottons 60*, duration 130 mins. Hit **Start Learning**, close the dialog, start the wash.
6. 2h10m later, check the log — profile saved with measured thresholds.
7. Next wash day, do a quick 30°C. Time it. Repeat the learn process with profile name *Quick 30*.
8. Build a trigger: when **Cycle Complete** fires AND `lastCycleProfile` is *Cottons 60*, send a notification "Cottons done — hang on the line".
9. Build another: when **Cycle Complete** fires AND `lastCycleProfile` is *Quick 30*, send "Quick wash done".
10. Add an **Unload Reminder** trigger set to 30 minutes for the gentle "the wet washing is going to start smelling" nudge.

You're done. Every future wash gets correctly identified and the right notification fires.
