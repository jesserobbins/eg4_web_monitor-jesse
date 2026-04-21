# Register 110 Bit Field Analysis: EG4_OFFGRID vs pylxpweb

**Date:** 2026-04-20
**Device:** EG4 12000XP (firmware ceaa-0709, device type 54)
**Connection:** WiFi dongle (hybrid mode)
**pylxpweb version:** 0.9.26
**Cross-references:** luxpower-ha-integration, 6kXP Modbus Protocol V58, OwlBawl/Luxpower-Modbus-RTU V23

---

## Executive Summary

Holding register 110 (`uFunctionEn3` / `SYSTEM FUNCTION BITFIELD`) has **different bit layouts** depending on inverter family. The pylxpweb library currently implements a single mapping that works for EG4_HYBRID (18kPV, FlexBOSS21) but is incorrect for EG4_OFFGRID (12000XP, 6000XP). Specifically:

- **ECO Mode Enable**: pylxpweb maps to bit 9. Hardware uses **bit 15**.
- **Green Mode Enable**: pylxpweb maps to bit 8. Luxpower-ha-integration maps to **bit 14**.

This was confirmed through direct Modbus register sweeps, cross-reference with the luxpower-ha-integration source (which has working ECO toggle on the same hardware), and live write testing.

---

## The Problem

### Symptom

Battery ECO Mode switch reads as `off` even when the inverter LCD shows ECO enabled. Toggling the switch via Home Assistant appears to succeed (no error) but the inverter state doesn't change.

### Root Cause

`write_named_parameters({"FUNC_BATTERY_ECO_EN": True})` sets bit 9 of register 110. On the 12000XP, bit 9 is NOT the ECO enable bit. The write succeeds at the protocol level (the dongle accepts it, the register is written) but the wrong bit is flipped, so ECO doesn't actually toggle.

The correct bit for ECO on the 12000XP is **bit 15**.

---

## Evidence

### Direct Modbus Observation

From a register sweep on 2026-03-19 (EG4 12000XP, hybrid mode):

| State | Register 110 Value | Binary | Interpretation |
|-------|-------------------|--------|----------------|
| ECO ON | `0x8080` | `1000 0000 1000 0000` | bit 7 + bit 15 set |
| ECO OFF | `0x0080` | `0000 0000 1000 0000` | bit 7 only |

Only bit 15 changes when ECO state changes. Bit 7 remains set at all times on this unit.

### Live Write Testing (2026-04-20)

| Method | HTTP Result | Inverter State Changed? |
|--------|-------------|------------------------|
| `write_named_parameters({"FUNC_BATTERY_ECO_EN": False})` | 200 OK | **NO** — ECO remained enabled |
| `write_parameters({110: current & ~(1<<15)})` | 200 OK | **YES** — ECO disabled |
| `write_parameters({110: current \| (1<<15)})` | 200 OK | **YES** — ECO enabled |

### Luxpower-HA-Integration (Working Reference)

File: `custom_components/lxp_modbus/entity_descriptions/switch_types.py`
```python
{
    "name": "Eco Mode",
    "register": H_FUNCTION_ENABLE_3,  # 110
    "register_type": "hold",
    "extract": lambda reg: get_bits(reg, 15, 1),   # bit 15
    "compose": lambda orig, value: set_bits(orig, 15, 1, value),  # bit 15
}
```

File: `custom_components/lxp_modbus/constants/hold_registers.py`
```python
H_FUNCTION_ENABLE_3 = 110
# Bit 14: GreenModeEn
# Bit 15: EcoModeEn
```

This integration directly controls the same hardware (LXP/EG4 inverters via WiFi dongle) and has ECO working at bit 15.

---

## Full Bit Layout Comparison

### pylxpweb (current — `src/pylxpweb/registers/inverter_holding.py`)

| Bit | Name | API Key | Notes |
|-----|------|---------|-------|
| 0 | pv_grid_off_enable | FUNC_PV_GRID_OFF_EN | |
| 1 | run_without_grid | FUNC_RUN_WITHOUT_GRID | |
| 2 | micro_grid_enable | FUNC_MICRO_GRID_EN | |
| 3 | battery_shared | FUNC_BAT_SHARED | |
| 4 | charge_last | FUNC_CHARGE_LAST | |
| 5 | take_load_together | FUNC_TAKE_LOAD_TOGETHER | |
| 6 | buzzer_enable | FUNC_BUZZER_EN | |
| 7 | go_to_offgrid | FUNC_GO_TO_OFFGRID | |
| 8 | **green_mode_enable** | **FUNC_GREEN_EN** | |
| 9 | **battery_eco_enable** | **FUNC_BATTERY_ECO_EN** | |
| 10 | working_mode | BIT_WORKING_MODE | |
| 11 | pvct_sample_type | BIT_PVCT_SAMPLE_TYPE | |
| 12 | pvct_sample_ratio | BIT_PVCT_SAMPLE_RATIO | |
| 13 | ct_sample_ratio | BIT_CT_SAMPLE_RATIO | |
| 14-15 | (unmapped) | | |

### Luxpower-HA-Integration (working on 12000XP)

| Bit | Name | Notes |
|-----|------|-------|
| 0 | ubPVGridOffEn | |
| 1 | ubFastZeroExport | |
| 2 | ubMicroGridEn | |
| 3 | ubBatShared | |
| 4 | ubChgLastEn | |
| 5-6 | CTSampleRatio | 2-bit field |
| 7 | BuzzerEn | |
| 8-9 | PVCTSampleType | 2-bit field |
| 10 | TakeLoadTogether | |
| 11 | OnGridWorkingMode | |
| 12-13 | PVCTSampleRatio | 2-bit field |
| 14 | **GreenModeEn** | |
| 15 | **EcoModeEn** | |

### Key Differences

| Function | pylxpweb bit | Luxpower bit | Confirmed via |
|----------|-------------|--------------|---------------|
| Green Mode | 8 | 14 | Needs verification |
| ECO Mode | 9 | 15 | **Confirmed** (sweep + live write) |
| CT Sample Ratio | 13 (1-bit) | 5-6 (2-bit) | Uncertain |
| Take Load Together | 5 | 10 | Uncertain |
| PVCT Sample Type | 11 | 8-9 (2-bit) | Uncertain |

**Important:** Only the ECO bit position is independently confirmed via live hardware testing. The other differences are inferred from comparing the two source files. They may reflect family-specific differences, or one mapping may be incorrect for all families.

---

## Open Question: Is Green Mode Actually Working?

The eg4_web_monitor integration uses `write_named_parameters({"FUNC_GREEN_EN": True})` which writes bit 8. If the 12000XP hardware expects Green at bit 14, this would fail silently (same way ECO did).

However, the Off-Grid Mode switch in eg4_web_monitor uses `_execute_local_with_fallback()` which tries the local write first and falls back to the cloud API. In hybrid mode, even if the local Modbus write targets the wrong bit, the cloud fallback may succeed — making it *appear* to work.

**Verification needed:** Test Green Mode toggle in LOCAL-only mode (no cloud fallback) to determine if bit 8 is correct for the 12000XP.

---

## Proposed Fix: pylxpweb

### Option A: Family-Specific Bit Mappings (Recommended)

Add an `inverter_family` discriminator to register 110 definitions so the library uses the correct bit positions per family:

```python
# For EG4_OFFGRID (12000XP, 6000XP):
HoldingRegisterDefinition(
    address=110,
    bit_position=15,  # was 9
    canonical_name="battery_eco_enable",
    api_param_key="FUNC_BATTERY_ECO_EN",
    category=HoldingCategory.FUNCTION,
    families=[InverterFamily.EG4_OFFGRID],
    description="Battery ECO mode — bypass when at EOD and not AC charging.",
)

# For EG4_HYBRID (retain current behavior):
HoldingRegisterDefinition(
    address=110,
    bit_position=9,
    canonical_name="battery_eco_enable",
    api_param_key="FUNC_BATTERY_ECO_EN",
    category=HoldingCategory.FUNCTION,
    families=[InverterFamily.EG4_HYBRID],
    description="Battery eco mode — reduce battery cycling for longevity.",
)
```

**Pros:**
- Correct for both families without breaking EG4_HYBRID
- Extensible to other per-family differences
- Matches the existing pattern (pylxpweb already has family-aware properties)

**Cons:**
- Requires plumbing family context through the register lookup system
- More complex than a simple bit change

### Option B: Correct the Mapping Globally

If the luxpower-ha-integration's layout is correct for ALL families (including EG4_HYBRID), change bit 9 → bit 15 globally. This is simpler but risks breaking EG4_HYBRID users.

**Would require:** Testing on an EG4_HYBRID unit (18kPV or FlexBOSS21) to confirm bit 15 is also correct there.

### Option C: Integration-Side Workaround (Current State)

Keep the eg4_web_monitor override that reads/writes bit 15 directly via raw register access, bypassing pylxpweb's named parameter system. This is what dev16 implements.

**Pros:** No pylxpweb changes needed
**Cons:** Every consumer of pylxpweb needs the same workaround; defeats the purpose of a named parameter abstraction

---

## Proposed Fix: eg4_web_monitor (Interim)

Until pylxpweb is corrected, the integration uses a raw register override:

**Read:** After `read_named_parameters()` returns, re-read register 110 raw and check bit 15 to override `FUNC_BATTERY_ECO_EN`. Cache the raw value as `_raw_reg_110`.

**Write:** Use `write_parameters({110: new_value})` with manual bit-15 set/clear, using the cached raw value as the base.

This is implemented in:
- `coordinator_local.py` — local mode read override
- `coordinator_mixins.py` — hybrid mode read override
- `switch.py` (`_execute_eco_mode_raw`) — write path

---

## AC Couple Per-Leg Power (I206/I207) Not Supported on Offgrid

### Problem

The integration creates `ac_couple_power_l1` and `ac_couple_power_l2` sensors mapped to input registers 206-207. On the 12000XP, these registers are **always zero** regardless of AC coupling state or power level.

### Evidence

From Modbus sweep (2026-03-19), registers 206-207 returned 0 in every operating mode (grid-tied, EPS, day, night) while register 153 (`ac_couple_power` total) correctly tracked 1290-1409W during daylight and 0W at night.

These registers appear to be GridBOSS/MID-device-only. The 12000XP provides only a single total AC couple power value at register 153.

### Fix

Suppress `ac_couple_power_l1` and `ac_couple_power_l2` sensors for EG4_OFFGRID. The integration already gates these via the `AC_COUPLE_PER_LEG_SENSORS` feature set.

---

## AC Couple Energy Scale Factor (Bonus Finding)

### Problem

Registers 124-126 (AC couple energy today / total) are annotated with `DIV_10` in pylxpweb (implying 0.1 kWh units). The 12000XP stores **raw watt-hours**.

### Evidence

| Register | Raw Value | DIV_10 (wrong) | DIV_1000 (correct) | Plausible? |
|----------|-----------|-----------------|---------------------|------------|
| 124 (today) | 2304 | 230.4 kWh | 2.304 kWh | Only ÷1000 makes sense for a 2kW array |
| 125-126 (total) | 9,744,577 | 974,457 kWh | 9,745 kWh | Only ÷1000 is plausible for lifetime |

### pylxpweb Fix Needed

In the scaling constants for registers 124-126, change from `ScaleFactor.DIV_10` to `ScaleFactor.DIV_1000` (or add a family-specific scale factor if EG4_HYBRID uses different units for these registers).

---

## Offgrid Mode Gating (Bonus Finding)

The following working modes should be suppressed for EG4_OFFGRID:

| Mode | Why Not Applicable |
|------|--------------------|
| Grid Peak Shaving | Requires grid-tied operation (shaves grid import) |
| Forced Discharge | Requires grid export capability |

These are currently exposed but non-functional on offgrid inverters. The integration gates them at entity creation time based on `inverter_family`.

---

## Summary of Changes Needed

| Component | Change | Priority |
|-----------|--------|----------|
| **pylxpweb** | Fix ECO bit position (9 → 15) for EG4_OFFGRID | HIGH |
| **pylxpweb** | Verify Green Mode bit position (8 vs 14) | MED |
| **pylxpweb** | Fix AC couple energy scale (DIV_10 → DIV_1000) for EG4_OFFGRID | MED |
| **pylxpweb** | Consider family-specific register layouts | DESIGN |
| **eg4_web_monitor** | ECO raw read/write override (interim until library fix) | DONE (dev16) |
| **eg4_web_monitor** | AC couple energy ÷1000 (interim until library fix) | DONE |
| **eg4_web_monitor** | Suppress AC couple L1/L2 sensors for offgrid (regs 206-207 always zero) | DONE |
| **eg4_web_monitor** | Gate Peak Shaving / Forced Discharge for offgrid | DONE |

---

## Appendix: How the dongle/transport Write Works

For reference, the successful write path:

```
switch.turn_off service call
  → EG4WorkingModeSwitch._execute_working_mode(turn_on=False)
    → _execute_eco_mode_raw(False)
      → read cached _raw_reg_110 from coordinator.data["parameters"]
      → compute new_val = current_val & ~(1 << 15)
      → coordinator.write_raw_register(110, new_val, serial=...)
        → transport = get_local_transport(serial)
        → transport.write_parameters({110: new_val})
          → pylxpweb dongle transport builds Modbus write packet
          → sends to dongle on port 8000
          → dongle forwards to inverter
          → inverter writes holding register 110
          → response confirms write
```

The `write_parameters({address: value})` method accepts raw register addresses and writes the full 16-bit value directly. This bypasses the named parameter bit-field logic entirely, which is necessary when the bit mapping is wrong.
