# Feature Request: Enable confirmed EG4_OFFGRID registers

**Integration Version:** v3.2.0
**Connection Mode:** All (Local, Cloud, Hybrid)
**Device Model(s):** 12000XP, 6000XP (EG4_OFFGRID family)

---

## Summary

Several useful registers on EG4_OFFGRID inverters are confirmed working via Modbus sweep + cloud cross-reference but are currently suppressed or unmapped. This request enables them using the existing feature flag model, only surfacing data the hardware already provides.

Related: #196 (AC couple register bug)

## Proposed Sensors

### 1. Per-phase EPS load power (I129 / I130) -- NEW

**Registers:** Input 129 (L1), Input 130 (L2)
**Unit:** Watts, SCALE_NONE
**Behavior:** Zero when grid-tied, non-zero in EPS/discharge mode

| Mode | I129 (L1) | I130 (L2) | Sum | Cloud epsLoadPower | Diff |
|------|-----------|-----------|-----|-------------------|------|
| Grid-tied | 0 | 0 | 0 | -- | -- |
| EPS discharge | 1031 | 296 | 1327 | 1338 | 11W (timing) |

These provide per-phase load breakdown unavailable from any other source. L1=1031W vs L2=296W shows real-world load imbalance across phases -- useful for diagnosing breaker panel balance and sizing.

**Proposed sensors:**
- `eps_load_power_l1` -- I129, gated on EG4_OFFGRID
- `eps_load_power_l2` -- I130, gated on EG4_OFFGRID
- `eps_load_power` -- I129 + I130 combined, following the existing L1+L2 sum pattern used elsewhere in the integration

### 2. Load power (I170) -- ENABLE

**Register:** Input 170
**Unit:** Watts, SCALE_NONE
**Current state:** Intentionally suppressed for EG4_OFFGRID (cloud sets pLoad170=0; integration follows suit)

| Mode | I170 | I211 (alt) | Cloud epsLoadPower | Diff |
|------|------|------------|-------------------|------|
| Grid-tied (T1) | 3788 | 3807 | -- | 19W between regs |
| EPS discharge (T9) | 1324 | 1326 | 1338 | 14W (timing) |

The 6kXP Modbus PDF labels I170 as `Pload`. Cloud explicitly zeroes it for EG4_OFFGRID (`pLoad170=0`), but the register holds valid data in both grid-tied and EPS modes. Confirmed across 4 sweep timestamps and 1 discharge test.

I211 appears to be a hardware duplicate of I170 (within 2-19W at every timestamp). Only I170 needs to be exposed -- it matches the documented register.

**Proposed change:** Un-suppress/enable I170 for EG4_OFFGRID. Use the local register value directly; do not trust the cloud's zeroed `pLoad170` field.

### 3. Battery discharge power (I11) -- ENABLE LOCAL

**Register:** Input 11
**Unit:** Watts, SCALE_NONE
**Current state:** Mapped in cloud mode (`pDisCharge`), I don't think is mapped in local mode for EG4_OFFGRID

| Mode | I11 | Cloud pDisCharge | Diff |
|------|-----|-----------------|------|
| Grid-tied | 0 | 0 | -- |
| EPS discharge | 1415 | 1401 | 14W (timing, 38s apart) |

I think this should be available in all connection modes as it is really helpful. 

## Evidence

### Methodology

Full Modbus register sweep on EG4 12000XP (firmware ceaa-0709, device type 54) on 2026-03-19. Nine Modbus sweep timestamps across grid-tied and EPS/discharge modes, plus two simultaneous cloud API snapshots. Cross-referenced against:

- 6kXP Modbus Protocol PDF V58
- pylxpweb v0.9.26 library maps
- EG4 cloud API (`getInverterRuntime`)
- OwlBawl/Luxpower-Modbus-RTU V23
- galets/eg4-modbus-monitor YAML maps