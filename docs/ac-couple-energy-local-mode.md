# AC Couple Energy on EG4_OFFGRID — Local-Only Mode Limitation

**Status:** Known limitation in v3.2.1-rc.3+
**Affected:** `EG4_OFFGRID` family inverters (12000XP, 6000XP) configured in **local-only** mode (Modbus TCP or WiFi dongle, no cloud)
**Affected sensors:**

- `sensor.inverter_ac_couple_energy_today`
- `sensor.inverter_ac_couple_energy_lifetime` (a.k.a. `_total`)

## Summary

These two sensors are **not created** when the integration is configured in
local-only mode. The underlying inverter firmware does not expose AC couple
energy as a directly accumulating Modbus register, and the cloud-derived
fallback used in hybrid mode is unavailable. Users who need AC couple energy
on local-only setups can integrate the live `sensor.inverter_ac_couple_power`
via Home Assistant's built-in Riemann sum integration helper (see workaround
below).

In **hybrid mode** (cloud + local), both sensors are created and populated
correctly via the cloud-derivation path.

## Why

### What we tried

`coordinator_local._read_ac_couple_energy()` originally read input registers
124-126 on the assumption (carried from earlier sweep work) that those
registers held AC couple energy today / lifetime in raw watt-hours.

Field testing on a 12000XP (firmware `ceaa-0709`) showed both reads were
wrong:

| Register(s) | Expected | Observed |
|---|---|---|
| 124 ("today" Wh) | ~10+ kWh accumulating during a sunny day | Drifted only ~0.25 kWh in 17 hours; matched a March sweep snapshot value (2048) almost exactly |
| 125-126 ("lifetime" 32-bit Wh) | Smooth monotonic increase | Jumped 16,777 kWh between consecutive reads — exactly `2^24 / 1000`, a 24-bit overflow artifact |

### Delta-scan investigation

To find a register that *does* track AC couple energy, we ran a two-point
delta scan: capture all 240 input registers at T0, wait 10 min while AC
couple averaged ~800 W (so ~130 Wh of expected accumulation), capture again,
report deltas.

Method writeup: `docs/superpowers/specs/...` and the auto-memory entry
`feedback_register_scan_methodology.md` describe how to repeat this
on future firmware.

Result: **no register's delta matched ~130 Wh at any plausible scale**
(raw Wh, 0.01 kWh, 0.1 kWh, raw kWh, 32-bit pair little/big-endian).
Candidates considered:

| Reg | Pre-scan | Post-scan | Δ | Verdict |
|---|---|---|---|---|
| 80 | 1030 | 1030 | 0 | Constant (likely a config value, not an accumulator) |
| 174 | 33024 | 33024 | 0 | Constant |
| 69 | 44093 | 44703 | +610 | ~1/sec — operational seconds, not energy |
| 123 | 13142 | 13749 | +607 | Known generator/seconds counter |
| 199, 200 | varied | varied | +23 each | Per-leg power (W), fluctuates both directions over longer windows |
| 202 | 98 | 287 | +189 | Power (W), not an accumulator |

Conclusion: this firmware does not expose AC couple daily/lifetime energy as a
Modbus accumulator register.

### Why hybrid mode works

In hybrid mode, `coordinator_mixins._process_inverter_object()` derives the
value as:

```
ac_couple_energy_today  = max(0, cloud_today_usage  − energy_balance_today)
ac_couple_energy_total  = max(0, cloud_total_usage  − energy_balance_total)
```

The cloud's `todayUsage` / `totalUsage` are server-computed totals that
already include AC couple contribution. The energy balance
(`yield + discharging + grid_import − charging − grid_export`) covers all
*other* sources. The difference is the AC couple contribution, and it
matches the EG4 web UI within rounding error.

This derivation has no Modbus equivalent — `cloud_today_usage` is server-side.

## Workaround for local-only users

Add an HA Riemann sum integration over the live AC couple power sensor:

1. **Settings → Devices & Services → Helpers → Create Helper → Integration - Riemann sum**
2. **Source:** `sensor.inverter_ac_couple_power`
3. **Integration method:** Trapezoidal
4. **Precision:** 3
5. **Metric prefix:** kilo
6. **Time unit:** Hours
7. Pair with **Helpers → Utility Meter** (cycle: Daily) for a daily-resetting version.

The result is a true power-integrated kWh figure that survives HA restarts
and resets on schedule. Drawback: not aware of inverter-side resets; if the
inverter loses power or the dongle disconnects, the integration helper
keeps its last value.

## Proposed fixes for a future PR

Tracked separately rather than in this RC. Options in priority order:

### Option A — Built-in Riemann integration sensor

Add a derived sensor class in `sensor.py` that integrates `ac_couple_power`
inside the coordinator on each refresh, persists across restarts via HA's
restored state, and resets at local midnight (we already have `DSTSyncMixin`
for the timezone-aware reset).

- **Pros:** Auto-creates, no user setup, works in local-only mode. One
  consistent semantic regardless of connection mode.
- **Cons:** ~100 lines of code; needs persistence, midnight reset logic,
  drift handling for irregular polling intervals. Diverges from the
  cloud-derived value by the integration error.

### Option B — Document Helper recipe in README

Add a short "Local-only AC couple energy" section to the README pointing
users at the Riemann + Utility Meter recipe above.

- **Pros:** Zero code. Users in the affected configuration will find it.
- **Cons:** Easy to miss; not auto-created.

### Option C — Re-investigate registers on newer firmware

Re-run the delta scan methodology against newer 12000XP firmware versions
and against 6000XP. If a future firmware exposes the accumulator, wire it
up. Track in an issue with sweep data.

### Recommendation

Start with **Option B** (documentation only) for the next release after
3.2.1, then evaluate **Option A** if local-only adoption grows. **Option C**
is opportunistic — pursue when a new firmware arrives or when another
EG4_OFFGRID user reports a working register set.

## References

- `_read_ac_couple_energy()` — `custom_components/eg4_web_monitor/coordinator_local.py`
  (no-op stub in v3.2.1-rc.3+)
- Cloud-derivation path — `coordinator_mixins.py` `_process_inverter_object()`
  AC-couple-energy block (~line 645-677)
- Sensor gating — `sensor.py` `_should_create_sensor()` (`has_http_api` check)
- Field scan methodology — auto-memory `feedback_register_scan_methodology.md`
- Field finding — auto-memory `project_ac_couple_energy_registers.md`
