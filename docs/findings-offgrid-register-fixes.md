# Findings: EG4_OFFGRID Register Fixes (v3.2.1-dev16)

**Date:** 2026-04-20
**Device:** EG4 12000XP (firmware ceaa-0709)
**Connection Mode:** Hybrid (WiFi dongle + cloud)
**pylxpweb Version:** 0.9.26
**Branch:** `claude/fix-offgrid-registers-hb02U`

---

## Summary

Four issues discovered and fixed through live testing on a 12000XP in hybrid mode. All fixes confirmed working on hardware.

---

## Finding 1: Battery ECO Mode — Register 110 Bit 15

### Problem

pylxpweb maps `FUNC_BATTERY_ECO_EN` to **bit 9** of holding register 110. The 12000XP hardware actually uses **bit 15**.

- `write_named_parameters({"FUNC_BATTERY_ECO_EN": True})` sets bit 9 — **does not toggle ECO**
- `write_named_parameters({"FUNC_BATTERY_ECO_EN": False})` clears bit 9 — **no effect on ECO**
- The inverter accepts the write without error, but the wrong bit is changed

### Evidence

From Modbus sweep (2026-03-19):
- Register 110 = `0x8080` when ECO enabled (bit 7 + bit 15 set)
- Register 110 = `0x0080` when ECO disabled (bit 7 set, bit 15 clear)
- Bit 15 is the only bit that changes with ECO state

Cross-reference with luxpower-ha-integration (upstream):
- `custom_components/lxp_modbus/entity_descriptions/switch_types.py` defines ECO as register 110, bit 15
- `custom_components/lxp_modbus/constants/hold_registers.py` documents bit 15 as `EcoModeEn`

### Fix

**Read path:** Override `FUNC_BATTERY_ECO_EN` after pylxpweb parameter read by checking bit 15 of raw register 110. Applied in both:
- `coordinator_local.py` (`_read_modbus_parameters`) — local/hybrid local poll
- `coordinator_mixins.py` (`_refresh_device_parameters`) — hybrid cloud param refresh

**Write path:** Use `transport.write_parameters({110: new_value})` with manual bit-15 manipulation instead of `write_named_parameters`. Cache the raw register value as `_raw_reg_110` in the parameter dict during reads to avoid a separate transport read before each write.

### Root Cause of 500 Error

The initial implementation called `self._set_optimistic_state()` and `self._clear_optimistic_state()` — methods that don't exist on `EG4BaseSwitch`. The base class uses `self._optimistic_state = value` directly. The resulting `AttributeError` was unhandled by HA's service dispatcher, producing HTTP 500.

---

## Finding 2: AC Couple Energy Scale Factor — Registers 124-126

### Problem

pylxpweb annotates registers 124-126 with `DIV_10` (divide by 10, implying 0.1 kWh units). The 12000XP stores **raw watt-hours**.

With DIV_10:
- `ac_couple_energy_today` = 230.4 kWh (impossible — max PV is 2 kW)
- `ac_couple_energy_total` = 3,071,507 kWh (impossible)

With DIV_1000 (raw Wh → kWh):
- `ac_couple_energy_today` = 2.304 kWh (plausible for a sunny morning)
- `ac_couple_energy_total` = 30,715 kWh (plausible for lifetime)

### Evidence

- Register 124 = 2304 → 2.304 kWh today (consistent with ~2kW AC coupled array over partial day)
- Registers 125-126 = 44225 | (148 << 16) = 9,744,577 → at ÷1000 = 9,745 kWh, but this was from the earlier sweep; the current fix uses the same ÷1000 for the 32-bit combined value
- Cloud does not expose these values directly for comparison (cloud derives AC couple energy differently)

### Fix

Changed `_read_ac_couple_energy()` in `coordinator_local.py`:
```python
today = regs[0] / 1000.0          # was: regs[0] / 10.0
total_raw = regs[1] | (regs[2] << 16)
total = total_raw / 1000.0        # was: total_raw / 10.0
```

---

## Finding 3: Peak Shaving and Forced Discharge — Not Applicable to Offgrid

### Problem

The EG4_OFFGRID (12000XP, 6000XP) does not support Grid Peak Shaving or Forced Discharge modes. These modes require grid-tied operation. Exposing them creates confusion and potentially unsafe conditions.

### Fix

Gate these working mode switches in `switch.py` entity creation:
```python
if family == INVERTER_FAMILY_EG4_OFFGRID and mode_key in (
    "peak_shaving_mode",
    "forced_discharge_mode",
):
    continue
```

---

## Finding 4: Transport Reconnection in Hybrid Mode

### Problem

Several write operations failed because:
1. `get_local_transport()` only searched `_inverter_cache` (populated in pure-local mode only)
2. In hybrid mode, inverter objects live in `station.all_inverters`
3. The `write_raw_register` method didn't reconnect before writing

### Fix

- `get_local_transport()` now falls back to `station.all_inverters` when `_inverter_cache` lookup fails
- `write_raw_register()` and `write_named_parameter()` both check `transport.is_connected` and reconnect before writing

---

## Key Takeaway: pylxpweb Bit Mappings Are Not Universal

The pylxpweb library's register/bit mappings come from the LXP hybrid inverter family. The EG4_OFFGRID family (12000XP, 6000XP) uses different bit positions for some functions in shared registers. When adding controls for offgrid inverters:

1. **Always verify bit positions via direct Modbus reads** — don't trust the library mapping
2. **Use raw register writes when the library mapping is wrong** — `write_parameters({reg: val})` bypasses the named parameter bit logic
3. **Cache raw register values during reads** to avoid separate transport reads before writes
4. **Test both read AND write** — a read can appear correct while the write targets the wrong bit

---

## Commits (dev10 through dev16)

| Version | Commit | Fix |
|---------|--------|-----|
| dev10 | `e1193c1` | ECO bit-15 override in hybrid parameter path |
| dev11 | `7a19a66` | ECO write uses coordinator.write_raw_register |
| dev12 | `ff0377f` | get_local_transport falls back to station.all_inverters |
| dev13 | `265bc16` | Transport reconnection before read |
| dev14 | `92c8cba` | (reverted) Tried write_named_parameters — wrong bit |
| dev15 | `c3312a4` | Raw bit-15 write with cached _raw_reg_110 |
| dev16 | `33ad447` | Fix AttributeError: use _optimistic_state directly |
