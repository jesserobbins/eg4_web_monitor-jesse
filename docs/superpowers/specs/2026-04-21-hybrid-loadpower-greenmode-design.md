# Hybrid Path Fix, load_power, Green Mode Suppression, Export AC Couple Re-probe

**Date:** 2026-04-21
**Branch:** `claude/offgrid-register-fixes`
**Predecessor:** 3.2.1-offgrid-beta-1 (commit `e9b94f1`, 867 tests)

## Problem

Beta-1 testing revealed:

1. **Hybrid mode data gap**: 6 of 13 OFFGRID sensors show `unknown` in hybrid mode. The direct register reads (`_read_generator_power_per_leg`, `_read_consumption_power`, `_read_ac_couple_energy`) are only called in the local-only paths, not the hybrid path in `coordinator_http.py`. L1/L2 voltages and eps_load_power sum are also missing from the hybrid path.

2. **load_power unavailable in all local modes**: The transport overlay maps `load_power` to `transport_runtime.load_power`, which resolves to pylxpweb register 27 (`power_to_user` / `pToUser`). Confirmed via pylxpweb source: `TransportRuntimeData.from_cloud_runtime()` sets `load_power=float(runtime.pToUser or 0)` (data.py:388), and `from_canonical_registers()` reads from the `power_to_user` register def which is address 27 (inverter_input.py:342). Register 27 is grid import power, NOT total load. Register 170 (`Pload`) isn't mapped in pylxpweb at all. Needs a direct register read.

3. **Green Mode switch shown on OFFGRID**: `EG4OffGridModeSwitch` is appended unconditionally in `switch.py:async_setup_entry` (~line 152). Jesse confirmed Green Mode is not configurable on the 12000XP OFFGRID. Should be suppressed.

4. **Export AC Couple unconfirmed**: H226 bit 14 toggle showed no visible change in EG4 web UI during beta-1. The switch exists and the register mapping was confirmed via live probe (H226=16384 when enabled). Needs a controlled re-probe watching power flow sensors, not the web UI.

## Solution

### Task 1: Wire direct reads into hybrid path (`coordinator_http.py`)

**File:** `coordinator_http.py` (~line 248)

The hybrid path already calls `_read_ac_couple_energy()` and `_read_consumption_power()` after the transport overlay. Add `_read_generator_power_per_leg()` to the same block.

**Note:** `_read_generator_power_per_leg` is NOT currently imported in `coordinator_http.py` (only `_read_ac_couple_energy`, `_read_ac_couple_registers`, `_read_ac_input_type`, and `_read_consumption_power` are imported at lines 38-43). Must add the import alongside the call.

Guard with OFFGRID feature check (same pattern as existing calls).

### Task 2: Expand transport overlay + eps_load_power sum (`coordinator_mixins.py`)

**File:** `coordinator_mixins.py`, `_process_inverter_object()` (~line 664)

Add to `_TRANSPORT_OVERLAY` tuple (attr names confirmed from `coordinator_mappings.py:_build_runtime_sensor_mapping()` lines 524-537):
- `("grid_voltage_l1", "grid_l1_voltage")`
- `("grid_voltage_l2", "grid_l2_voltage")`
- `("eps_voltage_l1", "eps_l1_voltage")`
- `("eps_voltage_l2", "eps_l2_voltage")`
- `("eps_power_l1", "eps_l1_power")`
- `("eps_power_l2", "eps_l2_power")`

The `eps_power_l1/l2` entries are needed because `eps_load_power` sum derivation already exists in `coordinator_mappings.py:611-617` but only runs in the local-only path. Adding the per-leg values to the overlay makes the existing derivation work for hybrid mode too. No new sum code needed in `_process_inverter_object()`.

### Task 3: Direct I170 read for load_power (`coordinator_local.py`)

**File:** `coordinator_local.py`

New function following exact pattern of `_read_consumption_power()`:
```python
async def _read_load_power(transport: Any, sensors: dict[str, Any]) -> None:
    """Read load power from input register 170 (Pload)."""
    read_fn = getattr(transport, "_read_input_registers", None)
    if read_fn is None:
        return
    try:
        regs = await read_fn(170, 1)
    except Exception as err:
        _LOGGER.warning("Load power register 170 read failed: %s", err)
        return
    if not regs or len(regs) < 1:
        return
    sensors["load_power"] = regs[0]
```

Call sites (3 total):
- `coordinator_local.py` local legacy path (~line 648)
- `coordinator_local.py` local new loop (~line 1320)
- `coordinator_http.py` hybrid path (~line 248)

Remove `("load_power", "load_power")` from `_TRANSPORT_OVERLAY` since it maps to the wrong register (27 instead of 170).

**Ordering note:** `_read_load_power` must run after `_build_runtime_sensor_mapping()` in the local path to avoid being overwritten. Verify call ordering at each site.

### Task 4: Suppress Green Mode for OFFGRID (`switch.py`)

**File:** `switch.py`, `async_setup_entry()` (~line 152)

Gate `EG4OffGridModeSwitch` creation: skip when device features include `EG4_OFFGRID`. Same approach used for peak shaving and forced discharge suppression.

**Ordering note:** The `EG4OffGridModeSwitch` append (~line 152) runs before the `family` variable is extracted (~line 156). The implementer must either move the feature extraction earlier or use a different mechanism to check the OFFGRID flag at line 152. Check the existing suppression pattern carefully before implementing.

### Task 5: Export AC Couple re-probe protocol (no code change)

Test plan for next deploy:
1. Confirm AC Coupling Mode (H179 bit 11) is ON
2. Watch `ac_couple_power` sensor value (not the web UI)
3. Toggle Export AC Couple OFF via HA switch
4. Observe if `ac_couple_power` drops to 0
5. Toggle back ON, confirm power returns
6. If no effect on power flow: investigate whether bit 14 is wrong or feature requires grid-tied state. Open a separate spec for register re-investigation if unresolved.

## Testing

- All existing 867 tests must continue to pass
- New tests for each task:
  - Task 1: Test hybrid path calls `_read_generator_power_per_leg`
  - Task 2: Test L1/L2 voltages appear in hybrid mode sensors; test eps_power_l1/l2 overlay; test eps_load_power sum derivation triggers from overlay values (separate assertion from voltages)
  - Task 3: Test `_read_load_power` function; test it's called in all three paths; test transport overlay no longer contains load_power
  - Task 4: Test `EG4OffGridModeSwitch` not created when OFFGRID features present
- Run full suite: `uv run pytest tests/ -x --tb=short`

## Scope boundaries

- No new schedule/config entities (Priority 3 from beta-1 findings -- separate effort)
- No changes to Export AC Couple switch code
- No pylxpweb changes
