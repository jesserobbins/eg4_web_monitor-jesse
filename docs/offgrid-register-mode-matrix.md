# EG4_OFFGRID Register & Sensor Mode Matrix

**Date:** 2026-04-21
**Device:** EG4 12000XP / 6000XP (firmware ceaa-0709, device type 54, family `EG4_OFFGRID`)
**Topology:** US split-phase 120/240V 60Hz
**Data sources:**
- Modbus register sweep 2026-03-19 (9 timestamps, grid-tied + EPS discharge modes)
- Cloud API snapshots (simultaneous with sweep)
- Live write tests 2026-04-20/21
- ant0nkr/luxpower-ha-integration (upstream, working ECO/Green on same hardware)
- ivanfmartinez/luxpower-ha-integration branches `ac_couple`, `ac_couple_v101`, PR #55 (split-phase US/BR), PR #66/#130 (AC couple entities)
- andrew867/LuxPower-Advanced (125W noise floor on I123, model code mapping)
- gonzalop/wombatt (6000XP register blocks)
- joyfulhouse/pylxpweb v0.9.26 (current register definitions)
- 6kXP Modbus Protocol PDF V58, OwlBawl/Luxpower-Modbus-RTU V23

---

## 1. Generator Power — Wrong Register on OFFGRID

### Problem

pylxpweb maps input register 123 as `generator_power` (W) for all inverter families. On EG4_OFFGRID (12000XP/6000XP), register 123 is a **seconds-of-operation counter**, not power.

### Evidence

**Sweep rate analysis (T5→T7):**
- T5 (11:43 PDT): I123 = 9585
- T7 (19:23 PDT): I123 = 37175
- Delta = 27590 over 27600 seconds = **1.000 units/sec** (wall-clock match within 0.04%)
- Completely independent of load, AC couple power, SOC, or operating mode
- Does NOT reset on mode transition (status 17→64, grid-tied to EPS)

**Cross-reference:**
- Cloud labels it `genPower` but `_12KUsingGenerator=false`, genVolt=0, genFreq=0
- Every known source (pylxpweb, luxpower-ha-integration, LuxPower-Advanced, wombatt, lxp-bridge, 6kXP PDF) maps register 123 as GenPower (W)
- LuxPower-Advanced (`LXPPacket.py:1287-1288`) applies a **125W noise floor** to register 123, zeroing values below 125W — acknowledging garbage on non-generator systems. However, the OFFGRID seconds counter produces values of ~39000+, far above any noise floor.
- No source documents register 123 as anything other than generator power. The OFFGRID firmware repurposes it.

### Fix

**pylxpweb (`inverter_input.py`):**
- Register 123: change `models=ALL` to `models=frozenset({"EG4_HYBRID", "LXP"})` — exclude EG4_OFFGRID

**For EG4_OFFGRID, use per-leg generator power registers instead:**

| Register | Sensor Key | Unit | Scale | Sweep Value | Source |
|----------|-----------|------|-------|-------------|--------|
| 188 | `generator_power_l1` | W | NONE | 0 (no gen) | LuxPower protocol: GenPower S-phase. On US split-phase, S→L1. |
| 189 | `generator_power_l2` | W | NONE | 0 (no gen) | LuxPower protocol: GenPower T-phase. On US split-phase, T→L2. |
| 188+189 | `generator_power` | W | SUM | 0 (no gen) | Derived L1+L2 sum. Same pattern as eps_load_power (I129+I130). |

Sweep confirms I188=0 and I189=0 — correct behavior with no generator connected. When a generator is attached, these registers carry per-leg power.

---

## 2. Generator Energy — Repurposed on OFFGRID

### Problem

Registers 124-126 are mapped as generator energy with DIV_10 scale (0.1 kWh). On OFFGRID they are repurposed as AC couple energy in raw watt-hours (need DIV_1000).

### Evidence

| Register | Raw Value | At DIV_10 | At DIV_1000 | Plausible? |
|----------|-----------|-----------|-------------|------------|
| 124 (today) | 2048 | 204.8 kWh | 2.048 kWh | Only DIV_1000 — max 2kW array |
| 125-126 (total) | 44225 + (148<<16) | 974,457 kWh | 9,745 kWh | Only DIV_1000 for lifetime |

Integration already reads these separately in `_read_ac_couple_energy()` with correct DIV_1000 (commit `cf85679`).

### Fix

**pylxpweb (`inverter_input.py`):**
- Registers 124, 125-126: change `models` to exclude EG4_OFFGRID
- Prevents pylxpweb's wrong DIV_10 values from leaking through `from_modbus_registers()`
- Integration's `_read_ac_couple_energy()` continues to handle these correctly

---

## 3. Green Mode (Off-Grid Mode) — Likely Wrong Bit Position

### Problem

`EG4OffGridModeSwitch` writes `FUNC_GREEN_EN` via `write_named_parameters`, which targets **bit 8** of register 110. The luxpower-ha-integration maps Green Mode to **bit 14**.

### Evidence

**luxpower-ha-integration (confirmed working on same hardware family):**
- `constants/hold_registers.py:154`: `# Bit 14: GreenModeEn`
- `entity_descriptions/switch_types.py:129`: `extract: lambda reg: get_bits(reg, 14, 1)`
- `entity_descriptions/switch_types.py:130`: `compose: lambda orig, value: set_bits(orig, 14, 1, value)`

**ECO Mode precedent (identical mismatch pattern):**
- pylxpweb mapped ECO to bit 9 — hardware uses bit 15 (confirmed via live write test 2026-04-20)
- `write_named_parameters({"FUNC_BATTERY_ECO_EN": False})` → 200 OK, **NO state change**
- `write_parameters({110: current & ~(1<<15)})` → 200 OK, **YES state change**
- Green Mode has NOT been independently verified via live Modbus test
- In hybrid mode, the cloud fallback (`enable_green_mode`) may mask the local write failure — same pattern that hid the ECO bug

### Full Register 110 Bit Layout Comparison

| Bit | pylxpweb | luxpower-ha-integration | Hardware confirmed? |
|-----|----------|------------------------|---------------------|
| 0 | PV Grid Off | ubPVGridOffEn | — |
| 1 | Run Without Grid | ubFastZeroExport | — |
| 2 | Micro Grid | ubMicroGridEn | — |
| 3 | Bat Shared | ubBatShared | — |
| 4 | Charge Last | ubChgLastEn | — |
| 5-6 | (single bits) | CTSampleRatio (2-bit) | — |
| 7 | Buzzer | BuzzerEn | Sweep: bit 7 always set (0x0080) |
| **8** | **Green Mode** | PVCTSampleType (2-bit) | **Needs test** |
| **9** | **ECO Mode** | PVCTSampleType (2-bit) | **WRONG** — confirmed bit 15 |
| 10 | Working Mode | TakeLoadTogether | — |
| 11 | PVCT Sample | OnGridWorkingMode | — |
| 12-13 | PVCT Ratio | PVCTSampleRatio (2-bit) | — |
| **14** | (unmapped) | **GreenModeEn** | **Needs test** |
| **15** | (unmapped) | **EcoModeEn** | **CONFIRMED** |

### Fix

1. **Verify first:** Test Green Mode toggle in local-only mode (no cloud fallback) on Jesse's 12000XP
2. If broken (expected): implement raw bit-14 read/write — identical pattern to ECO bit-15 workaround:
   - Read: extract bit 14 from `_raw_reg_110` after `read_named_parameters()`
   - Write: `_execute_green_mode_raw()` using cached `_raw_reg_110`, flip bit 14
   - Existing code reference: `switch.py:615-658` (`_execute_eco_mode_raw`)

---

## 4. L1/L2 Voltage Entities — Not Created

### Problem

Grid and EPS per-leg voltage entities don't exist despite registers reading valid data.

### Evidence

| Register | Sensor Key | Sweep Value | Source |
|----------|-----------|-------------|--------|
| 193 | `grid_voltage_l1` | 1236 = 123.6V | Ivan PR#55: `I_GRID_VOLT_L1N` |
| 194 | `grid_voltage_l2` | 1229 = 122.9V | Ivan PR#55: `I_GRID_VOLT_L2N` |
| 127 | `eps_voltage_l1` | 1223 = 122.3V | Ivan PR#55: `I_EPS_VOLT_L1N`, 6kXP PDF: EPSVoltL1N |
| 128 | `eps_voltage_l2` | 1230 = 123.0V | Ivan PR#55: `I_EPS_VOLT_L2N`, 6kXP PDF: EPSVoltL2N |

**Root cause:** Entity platforms enumerate sensors once during `async_setup_entry`. The `setdefault()` seeding at `coordinator_mixins.py:577-585` runs during data updates — after entity creation. If the key isn't in the initial sensor dict, no entity is created.

**Compounding factor:** Entity registry was disrupted by multiple HACS deploys. L1/L2 entities were deleted via WebSocket API and couldn't be recreated through the normal seeding path.

### Fix

Ensure these keys are in the sensor dict returned by `_map_device_properties` on the first coordinator update — before entity platform setup. The registers are already read by the local transport; the values just need to be in the initial dict.

---

## 5. Per-Phase EPS Load Power — New Sensors

### Evidence

| Register | Sensor Key | Grid-tied | EPS Discharge | Cloud Cross-ref |
|----------|-----------|-----------|---------------|-----------------|
| 129 | `eps_load_power_l1` | 0 | 1031W | — |
| 130 | `eps_load_power_l2` | 0 | 296W | — |
| 129+130 | `eps_load_power` | 0 | 1327W | `epsLoadPower=1338W` (11W timing diff) |

L1=1031W vs L2=296W — 3.5:1 load imbalance across phases. Useful for breaker panel balance diagnostics.

Source: Sweep T8 (discharge), Ivan PR#55 (`I_PEPS_L1N=129`, `I_PEPS_L2N=130`), 6kXP PDF.

---

## 6. Load Power — Valid but Suppressed

### Evidence

| Condition | I170 | I211 (duplicate) | Cloud epsLoadPower |
|-----------|------|-------------------|--------------------|
| Grid-tied (T1) | 3788W | 3807W (19W diff) | — |
| EPS discharge (T9) | 1324W | 1326W (2W diff) | 1338W (14W timing) |

6kXP PDF labels I170 as `Pload`. Cloud explicitly zeroes it (`pLoad170=0`, `pLoadPowerShow=False`). Register is valid in both modes. I211 is a hardware duplicate (within 2-19W at every timestamp) — only I170 needs to be exposed.

---

## 7. Consumption Power — New Sensor

### Evidence

| Register | Sweep Value | Cloud Field | Cloud Value | Diff |
|----------|-------------|-------------|-------------|------|
| 234 | 936W | `consumptionPower` | 934W | 2W |

Not in any protocol document. Not in Ivan's branches, LuxPower-Advanced, galets, or wombatt. Discovered via sweep cross-reference against cloud API. Cloud computes consumption differently from any single register — but I234 tracks it within 2W. Provides direct hardware measurement vs derived energy-balance calculation. Available in local-only mode.

---

## 8. Battery SOH from Register 5 High Byte

### Evidence

| Condition | I5 Raw | Hex | SOC (low byte) | SOH (high byte) | Cloud SOC |
|-----------|--------|-----|----------------|-----------------|-----------|
| Grid-tied (T1) | 25693 | 0x645D | 93% (0x5D) | 100% (0x64) | 93 |
| Discharge (T7) | 17476 | 0x4444 | 68% (0x44) | 68% (0x44) | 68 |

pylxpweb already unpacks this in `from_modbus_registers()` (`data.py:444-451`):
```python
soc = raw & 0xFF
soh = (raw >> 8) & 0xFF
```

SOH is extracted but may not propagate to a HA sensor entity. The T7 reading where SOH=68 matches SOC=68 is unexpected — SOH shouldn't drop 32% in one day. Possible firmware behavior during heavy discharge, or the high byte means something different in that state.

---

## 9. Export AC Couple — New Switch

### Evidence

Live probe 2026-04-21 (dev20): H226=16384 (0x4000) when "Export AC couple" enabled in EG4 web UI.

- Holding register 226, bit 14
- Separate from AC Couple enable (H179 bit 11)
- Both visible in EG4 web UI: Smart Load Port > AC coupling tab
- Pattern: read-modify-write register 226 bit 14 — identical to ECO (H110 bit 15) and AC Couple (H179 bit 11) raw r/w patterns

---

## 10. AC Couple Control Thresholds — New Number Entities

### Evidence

From Ivan's `ac_couple` branch and Modbus sweep:

| Register | Entity Key | Unit | Sweep Value | Description |
|----------|-----------|------|-------------|-------------|
| H220 | `ac_couple_start_soc` | % | 98 | SOC at which AC couple absorption begins |
| H221 | `ac_couple_end_soc` | % | 100 | SOC at which AC couple absorption ends |
| H222 | `ac_couple_start_voltage` | V (÷10) | 500 = 50.0V | Voltage threshold to begin absorption |
| H223 | `ac_couple_end_voltage` | V (÷10) | 540 = 54.0V | Voltage threshold to end absorption |

Ivan implements these as number entities with min/max validation. Controls when the inverter throttles AC couple input to prevent overcharging.

---

## 11. Generator Voltage Per-Leg — New Sensors

### Evidence

From Ivan PR#55:

| Register | Sensor Key | Unit | Scale | Sweep Value | Source |
|----------|-----------|------|-------|-------------|--------|
| 195 | `generator_voltage_l1` | V | ÷10 | 0 (no gen) | Ivan: `I_GEN_VOLT_L1N`. US split-phase. |
| 196 | `generator_voltage_l2` | V | ÷10 | 0 (no gen) | Ivan: `I_GEN_VOLT_L2N`. US split-phase. |

---

## 12. EPS Energy Per-Leg — New Sensors

### Evidence

From Ivan PR#55 and sweep:

| Register | Sensor Key | Unit | Scale | Sweep Value | Source |
|----------|-----------|------|-------|-------------|--------|
| 133 | `eps_energy_today_l1` | kWh | ÷10 | 36 = 3.6 kWh (T8) | Ivan: `I_EEPS_L1N_DAY` |
| 134 | `eps_energy_today_l2` | kWh | ÷10 | 13 = 1.3 kWh (T8) | Ivan: `I_EEPS_L2N_DAY` |
| 135-136 | `eps_energy_total_l1` | kWh | ÷10 | 4278 = 427.8 kWh | Ivan: `I_EEPS_L1N_ALL` |
| 137-138 | `eps_energy_total_l2` | kWh | ÷10 | 2695 = 269.5 kWh | Ivan: `I_EEPS_L2N_ALL` |

L1:L2 energy ratio = 427.8:269.5 = 1.6:1, consistent with the 3.5:1 instantaneous power ratio (heavy loads run intermittently on L1).

---

## 13. EPS Apparent Power Per-Leg — New Sensors

### Evidence

From Ivan PR#55 and sweep:

| Register | Sensor Key | Unit | Scale | Grid-tied | EPS Discharge | Relationship |
|----------|-----------|------|-------|-----------|---------------|-------------|
| 131 | `eps_apparent_power_l1` | VA | NONE | 0 | 1093 | vs I129=1031W real → PF=0.94 |
| 132 | `eps_apparent_power_l2` | VA | NONE | 0 | 406 | vs I130=296W real → PF=0.73 |

The apparent/real power ratio reveals reactive load on L2 (PF=0.73 suggests inductive loads like motors or transformers on that leg).

---

## Already Working / Done

| Item | Register | Status | Commit/Reference |
|------|----------|--------|------------------|
| ECO Mode raw r/w | H110 bit 15 | Working | dev10-dev16, `switch.py:615-658` |
| AC Coupling Mode raw r/w | H179 bit 11 | Working | `switch.py:660-698` |
| AC couple energy DIV_1000 | I124-126 | Working | dev11, commit `cf85679` |
| AC couple per-leg suppressed | I206-207 | Suppressed | `supports_ac_couple_per_leg=False` |
| Peak shaving gated | — | Suppressed | `switch.py:176-185` |
| Forced discharge gated | — | Suppressed | `switch.py:176-185` |
| Internal temp suppressed | I64, I108 | Suppressed | `supports_inverter_board_temps=False` |
| AC couple power (I153) | I153 | Working | Sweep: 1290-1409W day, 0W night |

---

## Needs Investigation

| Item | Question | Approach |
|------|----------|----------|
| `output_power` = 0 | What register/property feeds this? I16 (`Pinv`) was 0 in sweep (grid-tied). Is this expected for OFFGRID? | Trace `runtime_data.output_power` in pylxpweb → which register. Check discharge mode value. |
| I5 high byte in discharge | SOH=68 during discharge when SOH should be ~100. Does high byte mean something different under load? | Monitor I5 across charge/discharge cycles. May need more sweep data. |
| I114 on-grid load power | Ivan maps as "Load power when not off-grid (for 12k inverter)". Sweep: 0. | Test during grid-tied with load. May provide load power in grid mode where I129/130 are 0. |

---

## Feature Flags — EG4_OFFGRID

| Flag | Value | Detection | Effect |
|------|-------|-----------|--------|
| `supports_split_phase` | `True` | device_type=54 | Enable L1/L2 sensors, map S/T→L1/L2 |
| `supports_three_phase` | `False` | device_type=54 | Suppress R/S/T naming |
| `supports_ac_couple_per_leg` | `False` | I206/207=0 always | Suppress ac_couple_power_l1/l2 |
| `supports_inverter_board_temps` | `False` | I64/I108=0 always | Suppress internal_temp, bt_temp |
| `supports_discharge_recovery_hysteresis` | `True` | H125/H126 present | Enable discharge recovery numbers |
| `supports_ac_couple_energy_derived` | `True` | No direct registers | Derive AC couple energy from cloud |
| `has_generator` | runtime | I77 bit 0 (0=Grid, 1=Gen) | Feature detection for gen sensors |
| `has_ac_couple` | runtime | I77 bit 2 (0=off, 1=on) | Feature detection for AC couple sensors |

---

## Change Summary

| Category | Count | Items |
|----------|-------|-------|
| **EXCLUDE register from OFFGRID** | 4 | I123, I124, I125-126 (pylxpweb `models` field) |
| **NEW sensors** | 15 | I188, I189, I188+189, I129, I130, I129+130, I170, I234, I195, I196, I131, I132, I133, I134, I5 high byte |
| **NEW energy sensors** | 4 | I133, I134, I135-136, I137-138 |
| **NEW number controls** | 4 | H220, H221, H222, H223 |
| **NEW switch** | 1 | H226 bit 14 (Export AC Couple) |
| **FIX entity creation** | 4 | I193, I194, I127, I128 (L1/L2 voltages) |
| **FIX bit position** | 1 | H110 bit 14 (Green Mode �� needs live verify first) |
| **DONE** | 7 | ECO bit 15, AC couple bit 11, AC couple energy scale, per-leg suppression, peak shaving gate, forced discharge gate, internal temp suppression |
| **INVESTIGATE** | 3 | output_power, I5 high byte in discharge, I114 |
