# EG4_OFFGRID Register Fixes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix broken sensors, add missing sensors/controls, and correct register mappings for EG4_OFFGRID (12000XP/6000XP) inverters — without modifying pylxpweb.

**Architecture:** All fixes are integration-side workarounds in eg4_web_monitor. Generator power uses per-leg registers 188+189 (summed) instead of broken register 123. Bitfield switches (ECO, Green Mode, Export AC Couple) use raw register read-modify-write bypassing pylxpweb's named parameter system. New sensors map directly from pylxpweb runtime data properties or local Modbus reads.

**Tech Stack:** Python 3.13, Home Assistant custom component, pylxpweb>=0.9.26, pytest, pytest-homeassistant-custom-component

**Starting point:** `origin/main` (commit `42d780f`). Create a fresh branch.

**Reference docs:**
- `docs/offgrid-register-mode-matrix.md` — full register/sensor analysis with sweep evidence
- `docs/offgrid-register-110-analysis.md` — register 110 bit layout comparison
- `docs/findings-offgrid-register-fixes.md` — ECO/AC couple/transport findings from dev10-dev16
- `modbus_sweep/findings.md` — raw sweep data and cross-references

---

## File Map

| File | Responsibility | Tasks |
|------|---------------|-------|
| `custom_components/eg4_web_monitor/coordinator_mappings.py` | Sensor dict construction from pylxpweb runtime data | 1, 3, 4, 5, 6 |
| `custom_components/eg4_web_monitor/coordinator_mixins.py` | Transport overlay, OFFGRID-specific processing, feature extraction | 1, 3, 4, 7 |
| `custom_components/eg4_web_monitor/coordinator_local.py` | Local Modbus reads, parameter overrides, AC couple energy | 2, 7, 8, 9 |
| `custom_components/eg4_web_monitor/coordinator_http.py` | HTTP/cloud data path, OFFGRID suppression | 1 |
| `custom_components/eg4_web_monitor/switch.py` | Switch entities, raw register r/w methods | 7, 8, 9 |
| `custom_components/eg4_web_monitor/const/modbus.py` | Parameter name constants | 7, 8 |
| `custom_components/eg4_web_monitor/const/device_types.py` | Feature flag sets, sensor groups | 1, 3 |
| `custom_components/eg4_web_monitor/const/sensors/inverter.py` | Sensor metadata (units, device_class, icons) | 3, 4, 5, 6 |
| `custom_components/eg4_web_monitor/const/working_modes.py` | Working mode switch definitions | 7, 8, 9 |
| `custom_components/eg4_web_monitor/sensor.py` | Sensor entity creation, feature gating | 1, 3 |
| `tests/test_coordinator.py` | Coordinator processing tests | 1, 3, 4, 5, 6 |
| `tests/test_coordinator_local.py` | Local Modbus read tests | 2, 7, 8, 9 |
| `tests/test_sensor_entities.py` | Sensor creation/filtering tests | 1, 3 |

---

## Task 1: Fix generator_power — use I188+I189 sum for OFFGRID

Register 123 is a seconds counter on OFFGRID firmware. Use per-leg generator power registers 188 (S-phase/L1) and 189 (T-phase/L2) summed, and suppress register 123 for this family.

**Files:**
- Modify: `custom_components/eg4_web_monitor/coordinator_mappings.py` (~line 552)
- Modify: `custom_components/eg4_web_monitor/coordinator_mixins.py` (~line 650, transport overlay)
- Modify: `custom_components/eg4_web_monitor/coordinator_http.py` (OFFGRID suppression)
- Modify: `custom_components/eg4_web_monitor/sensor.py` (feature gating)
- Modify: `custom_components/eg4_web_monitor/const/device_types.py` (feature set)
- Test: `tests/test_coordinator.py`

**Evidence:** Sweep T5→T7: I123 delta=27590 over 27600s = 1.000/sec. I188=0, I189=0 (no gen connected, correct). LuxPower protocol: 188=GenPower_S, 189=GenPower_T. LuxPower-Advanced applies 125W noise floor to I123.

- [ ] **Step 1: Write failing test — generator_power suppressed for OFFGRID, replaced by I188+I189 sum**

In `tests/test_coordinator.py`, add a test that creates a mock OFFGRID inverter with `generator_power=39000` (the seconds counter value) and `generator_l1_power=500`, `generator_l2_power=300`. Verify that the processed sensor dict has `generator_power=800` (sum), `generator_power_l1=500`, `generator_power_l2=300`, and that the raw register 123 value (39000) is NOT used.

```python
async def test_offgrid_generator_power_uses_per_leg_sum(
    hass, mock_coordinator_offgrid
):
    """EG4_OFFGRID: generator_power = I188 + I189, not I123 (seconds counter)."""
    coordinator = mock_coordinator_offgrid
    # Mock inverter with broken I123 and valid I188/I189
    inverter = coordinator._station.all_inverters[0]
    inverter._runtime_properties.generator_power = 39000  # I123 seconds counter
    inverter._runtime_properties.generator_l1_power = 500  # I188
    inverter._runtime_properties.generator_l2_power = 300  # I189

    result = await coordinator._process_inverter_object(inverter, {})
    sensors = result["sensors"]

    # I123 value must NOT appear
    assert sensors.get("generator_power") != 39000
    # Sum of per-leg registers
    assert sensors["generator_power"] == 800
    assert sensors["generator_power_l1"] == 500
    assert sensors["generator_power_l2"] == 300
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/test_coordinator.py::test_offgrid_generator_power_uses_per_leg_sum -v`
Expected: FAIL

- [ ] **Step 3: Implement generator_power fix**

In `coordinator_mappings.py`, the `_build_inverter_sensor_dict()` function (around line 552) currently sets:
```python
"generator_power": runtime_data.generator_power,
```

After the sensor dict is built, add OFFGRID override logic. When `inverter_family == "EG4_OFFGRID"`:
1. Read `generator_l1_power` (I188) and `generator_l2_power` (I189) from `runtime_data`
2. Set `generator_power` = sum of non-None L1+L2 values (or None if both None)
3. This replaces the I123 value that was set by the property map

Also fix the `ac_couple_power` seeding on the same line (~590):
```python
"ac_couple_power": runtime_data.generator_power,
```
Change to:
```python
"ac_couple_power": None,
```
AC couple power comes from register 153 via `_read_ac_couple_registers()` in coordinator_local.py — seeding from I123 leaks the seconds counter into cloud-only mode.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest tests/test_coordinator.py::test_offgrid_generator_power_uses_per_leg_sum -v`
Expected: PASS

- [ ] **Step 5: Run full test suite**

Run: `uv run pytest tests/ -x --tb=short`
Expected: All existing tests still pass

- [ ] **Step 6: Commit**

```bash
git add custom_components/eg4_web_monitor/coordinator_mappings.py tests/test_coordinator.py
git commit -m "fix: OFFGRID generator_power uses I188+I189 sum instead of broken I123

Register 123 is a seconds counter on EG4_OFFGRID firmware, not power.
Use per-leg registers 188 (L1) + 189 (L2) summed instead.
Also fix ac_couple_power seeding to None (I123 leaked seconds counter)."
```

---

## Task 2: Fix AC couple energy scale (I124-126) for local reads

Registers 124-126 are repurposed on OFFGRID: raw watt-hours, not 0.1 kWh. pylxpweb applies DIV_10 which gives 100x wrong values. The integration needs its own local read with DIV_1000.

**Files:**
- Modify: `custom_components/eg4_web_monitor/coordinator_local.py`
- Test: `tests/test_coordinator_local.py`

**Evidence:** Sweep I124=2048. At DIV_10=204.8 kWh (impossible for 2kW array). At DIV_1000=2.048 kWh (correct).

- [ ] **Step 1: Write failing test — AC couple energy reads with DIV_1000**

```python
async def test_offgrid_ac_couple_energy_div_1000(hass, mock_transport):
    """AC couple energy regs 124-126 use raw Wh (DIV_1000) on OFFGRID."""
    mock_transport._read_input_registers = AsyncMock(return_value=[2304, 44225, 148])
    sensors = {"ac_input_type": "Grid"}

    await _read_ac_couple_energy(mock_transport, sensors)

    assert sensors["ac_couple_energy_today"] == pytest.approx(2.304, abs=0.001)
    total = (44225 + (148 << 16)) / 1000.0
    assert sensors["ac_couple_energy_total"] == pytest.approx(total, abs=0.001)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/test_coordinator_local.py::test_offgrid_ac_couple_energy_div_1000 -v`

- [ ] **Step 3: Implement `_read_ac_couple_energy()` in coordinator_local.py**

Add a function that reads registers 124-126 directly via `transport._read_input_registers(124, 3)`, divides by 1000 (not 10), and populates `sensors["ac_couple_energy_today"]` and `sensors["ac_couple_energy_total"]`. Only run when `sensors["ac_input_type"] == "Grid"` (when port is Generator mode, these are real generator energy and the default scale applies).

```python
async def _read_ac_couple_energy(
    transport: Any, sensors: dict[str, Any]
) -> None:
    """Read AC couple energy from regs 124-126 with correct scale for OFFGRID.

    On EG4_OFFGRID, registers 124-126 store AC couple energy in raw watt-hours.
    pylxpweb annotates these as DIV_10 (0.1 kWh), which is wrong for OFFGRID.
    Only populate when ac_input_type == "Grid" (AC couple active, not generator).
    """
    if sensors.get("ac_input_type") != "Grid":
        return

    read_fn = getattr(transport, "_read_input_registers", None)
    if read_fn is None:
        return

    try:
        regs = await read_fn(124, 3)
    except Exception as err:
        _LOGGER.warning("AC couple energy registers 124-126 read failed: %s", err)
        return

    if not regs or len(regs) < 3:
        return

    today = regs[0] / 1000.0
    total_raw = regs[1] | (regs[2] << 16)
    total = total_raw / 1000.0
    sensors["ac_couple_energy_today"] = round(today, 3)
    sensors["ac_couple_energy_total"] = round(total, 3)
```

- [ ] **Step 4: Run test to verify it passes**
- [ ] **Step 5: Commit**

```bash
git commit -m "fix: AC couple energy uses DIV_1000 on OFFGRID (raw Wh, not 0.1 kWh)"
```

---

## Task 3: Fix L1/L2 voltage entity creation (I193, I194, I127, I128)

Entities are never created because the keys aren't in the sensor dict at entity platform setup time.

**Files:**
- Modify: `custom_components/eg4_web_monitor/coordinator_mappings.py` (sensor dict)
- Test: `tests/test_coordinator.py`

**Evidence:** Sweep: I193=1236 (123.6V), I194=1229 (122.9V), I127=1223 (122.3V), I128=1230 (123.0V). Ivan PR#55 confirms register assignments.

- [ ] **Step 1: Write failing test — L1/L2 voltage keys present in sensor dict**

```python
async def test_offgrid_l1l2_voltage_keys_in_initial_dict(
    hass, mock_coordinator_offgrid
):
    """L1/L2 voltage keys must be in sensor dict for entity creation."""
    coordinator = mock_coordinator_offgrid
    inverter = coordinator._station.all_inverters[0]

    result = await coordinator._process_inverter_object(inverter, {})
    sensors = result["sensors"]

    # These keys must exist (even if None) for entity platform to create entities
    assert "grid_voltage_l1" in sensors
    assert "grid_voltage_l2" in sensors
    assert "eps_voltage_l1" in sensors
    assert "eps_voltage_l2" in sensors
```

- [ ] **Step 2: Run test to verify it fails**
- [ ] **Step 3: Implement — add L1/L2 voltage keys to sensor dict**

In `coordinator_mappings.py`, the sensor dict already includes `eps_voltage_l1` and `eps_voltage_l2` from `runtime_data.eps_l1_voltage` and `runtime_data.eps_l2_voltage`. These ARE in the property map on main.

Check if `grid_voltage_l1` and `grid_voltage_l2` are also present. If not, add them. The pylxpweb runtime data properties `grid_l1_voltage` and `grid_l2_voltage` map to registers 193 and 194.

The keys may already be in the dict from the property map but with None values (registers not read yet on first cycle). The key point is they must be present — None is acceptable for entity creation as long as the key exists.

If the keys are present but entities still don't get created, the issue is in `sensor.py`'s `_should_create_sensor()` — it may filter out None-valued keys. Check and fix the filter to allow None for voltage sensors.

- [ ] **Step 4: Run test to verify it passes**
- [ ] **Step 5: Commit**

```bash
git commit -m "fix: ensure L1/L2 voltage keys in sensor dict for entity creation"
```

---

## Task 4: Add per-phase EPS load power sensors (I129, I130, I129+I130)

New sensors for per-phase EPS load breakdown. Only non-zero in EPS/discharge mode.

**Files:**
- Modify: `custom_components/eg4_web_monitor/const/sensors/inverter.py` (sensor metadata)
- Modify: `custom_components/eg4_web_monitor/coordinator_mappings.py` (sensor dict)
- Test: `tests/test_coordinator.py`

**Evidence:** Sweep T8: I129=1031W (L1), I130=296W (L2), sum=1327W vs cloud epsLoadPower=1338W. Ivan PR#55: I_PEPS_L1N=129, I_PEPS_L2N=130.

- [ ] **Step 1: Write failing test — eps_load_power sensors in dict**

```python
async def test_offgrid_eps_load_power_per_phase(hass, mock_coordinator_offgrid):
    """EPS load power per-phase from I129/I130 with L1+L2 sum."""
    coordinator = mock_coordinator_offgrid
    inverter = coordinator._station.all_inverters[0]
    inverter._runtime_properties.eps_l1_power = 1031
    inverter._runtime_properties.eps_l2_power = 296

    result = await coordinator._process_inverter_object(inverter, {})
    sensors = result["sensors"]

    assert sensors["eps_load_power_l1"] == 1031
    assert sensors["eps_load_power_l2"] == 296
    assert sensors["eps_load_power"] == 1327
```

- [ ] **Step 2: Run test to verify it fails**
- [ ] **Step 3: Implement**

The `eps_power_l1` and `eps_power_l2` sensors already exist from `runtime_data.eps_l1_power` / `runtime_data.eps_l2_power` (regs 129/130). These may already be the same thing under a different name. Check if `eps_power_l1` already maps to register 129.

If `eps_power_l1` IS register 129, then `eps_load_power_l1` is redundant — we just need the sum sensor `eps_load_power`. Add it as a derived sensor: `eps_load_power = eps_power_l1 + eps_power_l2` (only when both are non-None).

Add sensor metadata in `const/sensors/inverter.py`:
```python
"eps_load_power": {
    "name": "EPS Load Power",
    "device_class": SensorDeviceClass.POWER,
    "native_unit_of_measurement": UnitOfPower.WATT,
    "state_class": SensorStateClass.MEASUREMENT,
    "icon": "mdi:home-lightning-bolt",
},
```

- [ ] **Step 4: Run test to verify it passes**
- [ ] **Step 5: Commit**

```bash
git commit -m "feat: add eps_load_power sum sensor (I129+I130) for OFFGRID"
```

---

## Task 5: Enable load_power sensor (I170) for OFFGRID

Register 170 (Pload) is valid but cloud zeroes it. Use local register value.

**Files:**
- Modify: `custom_components/eg4_web_monitor/coordinator_mappings.py`
- Test: `tests/test_coordinator.py`

**Evidence:** Sweep T1: I170=3788W (grid), T9: I170=1324W (EPS) vs cloud epsLoadPower=1338W.

- [ ] **Step 1: Write failing test**

```python
async def test_offgrid_load_power_from_register_170(hass, mock_coordinator_offgrid):
    """load_power uses I170 from local transport, not cloud's zeroed pLoad170."""
    coordinator = mock_coordinator_offgrid
    inverter = coordinator._station.all_inverters[0]
    # Simulate: cloud returns 0, but transport has real value
    inverter._runtime_properties.load_power = 0  # cloud path
    inverter._transport_runtime.load_power = 3788  # local Modbus I170

    result = await coordinator._process_inverter_object(inverter, {})
    assert result["sensors"]["load_power"] == 3788
```

- [ ] **Step 2: Run test to verify it fails**
- [ ] **Step 3: Implement — add load_power to transport overlay**

In `coordinator_mixins.py`, add `("load_power", "load_power")` to the `_TRANSPORT_OVERLAY` tuple. This ensures the local Modbus value from register 170 overrides the cloud's zeroed value.

Note: `load_power` is already in `SENSOR_TYPES` on main (line 36 of `const/sensors/inverter.py`). It was removed from the property map with a comment saying "register 27 (pToUser) is grid import, NOT consumption." But register 170 IS load power per the 6kXP PDF. The fix is to add `load_power` back to the property map from `runtime_data.load_power` (which reads I170 on local transport).

- [ ] **Step 4: Run test to verify it passes**
- [ ] **Step 5: Commit**

```bash
git commit -m "feat: enable load_power (I170) for OFFGRID via transport overlay"
```

---

## Task 6: Add consumption_power sensor (I234)

Register 234 matches cloud `consumptionPower` within 2W. Provides hardware measurement vs derived calculation.

**Files:**
- Modify: `custom_components/eg4_web_monitor/coordinator_mixins.py` (transport overlay)
- Test: `tests/test_coordinator.py`

**Evidence:** Sweep: I234=936W vs cloud consumptionPower=934W. Not in any protocol doc.

- [ ] **Step 1: Write failing test**

```python
async def test_consumption_power_from_transport(hass, mock_coordinator):
    """consumption_power from I234 via transport overlay."""
    coordinator = mock_coordinator
    inverter = coordinator._station.all_inverters[0]
    inverter._transport_runtime.consumption_power = 936

    result = await coordinator._process_inverter_object(inverter, {})
    assert result["sensors"]["consumption_power"] == 936
```

- [ ] **Step 2: Run test to verify it fails**
- [ ] **Step 3: Implement — add to transport overlay**

Check if pylxpweb's `InverterRuntimeData` has a `consumption_power` field mapped to register 234. If so, add `("consumption_power", "consumption_power")` to the `_TRANSPORT_OVERLAY`. If not, this needs a direct register read in coordinator_local.py — read register 234, populate `sensors["consumption_power"]`.

Note: `consumption_power` already has sensor metadata in `const/sensors/inverter.py` (line 51) and is currently computed from energy balance. The transport overlay value takes precedence when available.

- [ ] **Step 4: Run test to verify it passes**
- [ ] **Step 5: Commit**

```bash
git commit -m "feat: add consumption_power from I234 (direct hardware measurement)"
```

---

## Task 7: Add Battery ECO Mode switch with raw bit-15 r/w

pylxpweb maps ECO to bit 9 of register 110. Hardware uses bit 15 on OFFGRID. Bypass pylxpweb with raw register read-modify-write.

**Files:**
- Modify: `custom_components/eg4_web_monitor/switch.py`
- Modify: `custom_components/eg4_web_monitor/const/working_modes.py`
- Modify: `custom_components/eg4_web_monitor/const/modbus.py`
- Modify: `custom_components/eg4_web_monitor/coordinator_local.py` (read override)
- Modify: `custom_components/eg4_web_monitor/coordinator_mixins.py` (hybrid read override)
- Test: `tests/test_coordinator_local.py`

**Evidence:** Sweep: H110=0x8080 (ECO on), 0x0080 (ECO off) — only bit 15 changes. Live write: `write_parameters({110: val | (1<<15)})` toggles ECO. `write_named_parameters({"FUNC_BATTERY_ECO_EN": True})` does NOT. luxpower-ha: bit 15 confirmed.

- [ ] **Step 1: Write failing test — ECO read override extracts bit 15**

```python
async def test_eco_mode_reads_bit_15(hass, mock_transport):
    """ECO mode reads bit 15 of register 110, not pylxpweb's bit 9."""
    # H110 = 0x8080: bit 7 (buzzer) + bit 15 (ECO) set
    mock_transport.read_parameters = AsyncMock(return_value={110: 0x8080})
    params = {}

    await _override_eco_mode_from_raw_reg_110(mock_transport, params)

    assert params["FUNC_BATTERY_ECO_EN"] is True
    assert params["_raw_reg_110"] == 0x8080
```

- [ ] **Step 2: Run test to verify it fails**
- [ ] **Step 3: Implement read override**

In `coordinator_local.py`, after `read_named_parameters()`, read register 110 raw, extract bit 15, override `FUNC_BATTERY_ECO_EN` in the parameter dict, and cache raw value as `_raw_reg_110`.

```python
raw_110_data = await transport._read_holding_registers(110, 1)
if raw_110_data:
    raw_110 = raw_110_data[0] if isinstance(raw_110_data, list) else raw_110_data.get(110, 0)
    eco_on = bool(raw_110 & (1 << 15))
    params["FUNC_BATTERY_ECO_EN"] = eco_on
    params["_raw_reg_110"] = raw_110
```

- [ ] **Step 4: Implement ECO write in switch.py**

Add `battery_eco_mode` to `WORKING_MODES` in `const/working_modes.py`. In `switch.py`, detect `FUNC_BATTERY_ECO_EN` param in `_execute_working_mode()` and route to `_execute_eco_mode_raw()`:

```python
async def _execute_eco_mode_raw(self, turn_on: bool) -> None:
    self._optimistic_state = turn_on
    self.async_write_ha_state()
    try:
        params = self.coordinator.data.get("parameters", {}).get(self._serial, {})
        current_val = params.get("_raw_reg_110", 0)
        new_val = current_val | (1 << 15) if turn_on else current_val & ~(1 << 15)
        await self.coordinator.write_raw_register(110, new_val, serial=self._serial)
        params["FUNC_BATTERY_ECO_EN"] = turn_on
        params["_raw_reg_110"] = new_val
        await asyncio.sleep(0.5)
        await self.coordinator.async_refresh()
        self._optimistic_state = None
        self.async_write_ha_state()
    except Exception as err:
        self._optimistic_state = None
        self.async_write_ha_state()
        raise HomeAssistantError(f"Failed to set Battery ECO Mode: {err}") from err
```

- [ ] **Step 5: Run tests, commit**

```bash
git commit -m "feat: Battery ECO Mode switch using raw bit-15 of register 110

pylxpweb maps FUNC_BATTERY_ECO_EN to bit 9, but 12000XP hardware uses
bit 15. Confirmed via live Modbus sweep and write testing."
```

---

## Task 8: Fix Green Mode (Off-Grid Mode) — raw bit-14 r/w

Same pattern as ECO Mode. pylxpweb uses bit 8, hardware likely uses bit 14.

**Files:**
- Modify: `custom_components/eg4_web_monitor/switch.py` (EG4OffGridModeSwitch)
- Modify: `custom_components/eg4_web_monitor/coordinator_local.py` (read override)
- Test: `tests/test_coordinator_local.py`

**Evidence:** luxpower-ha: `GreenModeEn` = bit 14 of register 110. Same register, same mismatch pattern as ECO (bit 9→15). **Needs live verification** — test toggle in local-only mode before finalizing.

- [ ] **Step 1: Write failing test — Green Mode read override extracts bit 14**

```python
async def test_green_mode_reads_bit_14(hass, mock_transport):
    """Green Mode reads bit 14 of register 110, not pylxpweb's bit 8."""
    # H110 = 0xC080: bit 7 (buzzer) + bit 14 (green) + bit 15 (eco) set
    mock_transport.read_parameters = AsyncMock(return_value={110: 0xC080})
    params = {}

    await _override_green_mode_from_raw_reg_110(mock_transport, params)

    assert params["FUNC_GREEN_EN"] is True
    assert params["_raw_reg_110"] == 0xC080
```

- [ ] **Step 2: Run test to verify it fails**
- [ ] **Step 3: Implement read override for bit 14**

Same function as ECO read override — extend it to also extract bit 14 for `FUNC_GREEN_EN`:

```python
green_on = bool(raw_110 & (1 << 14))
params["FUNC_GREEN_EN"] = green_on
```

- [ ] **Step 4: Implement Green Mode write — raw bit-14**

Modify `EG4OffGridModeSwitch` to use `_execute_green_mode_raw()` instead of `_execute_local_with_fallback()`:

```python
async def _execute_green_mode_raw(self, turn_on: bool) -> None:
    """Same pattern as _execute_eco_mode_raw but for bit 14."""
    self._optimistic_state = turn_on
    self.async_write_ha_state()
    try:
        params = self.coordinator.data.get("parameters", {}).get(self._serial, {})
        current_val = params.get("_raw_reg_110", 0)
        new_val = current_val | (1 << 14) if turn_on else current_val & ~(1 << 14)
        await self.coordinator.write_raw_register(110, new_val, serial=self._serial)
        params["FUNC_GREEN_EN"] = turn_on
        params["_raw_reg_110"] = new_val
        await asyncio.sleep(0.5)
        await self.coordinator.async_refresh()
        self._optimistic_state = None
        self.async_write_ha_state()
    except Exception as err:
        self._optimistic_state = None
        self.async_write_ha_state()
        raise HomeAssistantError(f"Failed to set Off-Grid Mode: {err}") from err
```

- [ ] **Step 5: Run tests, commit**

```bash
git commit -m "fix: Off-Grid Mode (Green Mode) uses raw bit-14 of register 110

pylxpweb maps FUNC_GREEN_EN to bit 8, but luxpower-ha-integration
confirms bit 14. Same mismatch pattern as ECO Mode (bit 9 vs 15)."
```

- [ ] **Step 6: Live verification (manual)**

Test toggle in local-only mode on Jesse's 12000XP. If bit 14 doesn't work, check bit 8 and update accordingly.

---

## Task 9: Add Export AC Couple switch (H226 bit 14)

New switch entity for "Export AC couple" — controls power flow direction for AC coupled solar.

**Files:**
- Modify: `custom_components/eg4_web_monitor/switch.py`
- Modify: `custom_components/eg4_web_monitor/const/working_modes.py`
- Modify: `custom_components/eg4_web_monitor/coordinator_local.py` (read/cache H226)
- Test: `tests/test_coordinator_local.py`

**Evidence:** Live probe 2026-04-21: H226=0x4000 (bit 14 set) when Export enabled in EG4 web UI. Separate from AC Couple enable (H179 bit 11).

- [ ] **Step 1: Write failing test — Export AC Couple read**

```python
async def test_export_ac_couple_reads_bit_14_of_h226(hass, mock_transport):
    """Export AC Couple reads bit 14 of holding register 226."""
    mock_transport.read_parameters = AsyncMock(return_value={226: 0x4000})
    params = {}

    await _read_export_ac_couple(mock_transport, params)

    assert params["FUNC_EXPORT_AC_COUPLE"] is True
    assert params["_raw_reg_226"] == 0x4000
```

- [ ] **Step 2: Run test to verify it fails**
- [ ] **Step 3: Implement read path**

In `coordinator_local.py`, after reading register 110 and 179, also read register 226. Extract bit 14, store as `FUNC_EXPORT_AC_COUPLE` and cache raw value as `_raw_reg_226`.

- [ ] **Step 4: Implement switch entity and write path**

Add `export_ac_couple` to `WORKING_MODES`:
```python
"export_ac_couple": {
    "name": "Export AC Couple",
    "param": "FUNC_EXPORT_AC_COUPLE",
    "description": "Allow AC coupled solar to export power",
    "icon": "mdi:solar-power-variant-outline",
    "entity_category": EntityCategory.CONFIG,
},
```

In `switch.py`, detect `FUNC_EXPORT_AC_COUPLE` param and route to `_execute_export_ac_couple_raw()`:
```python
async def _execute_export_ac_couple_raw(self, turn_on: bool) -> None:
    """Toggle Export AC Couple via raw bit-14 write to register 226."""
    # Same pattern as ECO (H110 bit 15) and AC Couple (H179 bit 11)
    self._optimistic_state = turn_on
    self.async_write_ha_state()
    try:
        params = self.coordinator.data.get("parameters", {}).get(self._serial, {})
        current_val = params.get("_raw_reg_226", 0)
        new_val = current_val | (1 << 14) if turn_on else current_val & ~(1 << 14)
        await self.coordinator.write_raw_register(226, new_val, serial=self._serial)
        params["FUNC_EXPORT_AC_COUPLE"] = turn_on
        params["_raw_reg_226"] = new_val
        await asyncio.sleep(0.5)
        await self.coordinator.async_refresh()
        self._optimistic_state = None
        self.async_write_ha_state()
    except Exception as err:
        self._optimistic_state = None
        self.async_write_ha_state()
        raise HomeAssistantError(f"Failed to set Export AC Couple: {err}") from err
```

Gate entity creation on EG4_OFFGRID family (or presence of `has_ac_couple` feature flag).

- [ ] **Step 5: Run tests, commit**

```bash
git commit -m "feat: Export AC Couple switch (H226 bit 14)

Confirmed via live Modbus probe: H226=0x4000 when enabled in EG4 web UI.
Separate from AC Couple enable (H179 bit 11). OFFGRID only."
```

---

## Task 10: Gate peak shaving and forced discharge for OFFGRID

These modes require grid-tied operation. Suppress for EG4_OFFGRID.

**Files:**
- Modify: `custom_components/eg4_web_monitor/switch.py` (entity creation)
- Test: `tests/test_sensor_entities.py`

- [ ] **Step 1: Write failing test**

```python
async def test_offgrid_suppresses_grid_tied_modes(hass, mock_coordinator_offgrid):
    """Peak shaving and forced discharge suppressed for EG4_OFFGRID."""
    entities = await async_setup_switch_entities(hass, mock_coordinator_offgrid)
    entity_keys = [e.entity_key for e in entities]

    assert "peak_shaving_mode" not in entity_keys
    assert "forced_discharge_mode" not in entity_keys
    # AC charge and battery backup should still be present
    assert "ac_charge_mode" in entity_keys
```

- [ ] **Step 2: Run test to verify it fails**
- [ ] **Step 3: Implement — gate in switch entity creation**

In `switch.py`, where working mode entities are created, add family check:

```python
if family == INVERTER_FAMILY_EG4_OFFGRID and mode_key in (
    "peak_shaving_mode",
    "forced_discharge_mode",
):
    continue
```

- [ ] **Step 4: Run tests, commit**

```bash
git commit -m "fix: suppress peak shaving and forced discharge for EG4_OFFGRID"
```

---

## Task 11: Add AC Coupling Mode switch with raw bit-11 r/w (H179)

Same raw r/w pattern as ECO. pylxpweb may not map bit 11 correctly for all families.

**Files:**
- Modify: `custom_components/eg4_web_monitor/switch.py`
- Modify: `custom_components/eg4_web_monitor/coordinator_local.py` (read/cache H179)
- Test: `tests/test_coordinator_local.py`

**Evidence:** Sweep: H179=0x0800 (bit 11 set) when AC coupling enabled. Cloud: `_12KAcCoupleInverterData=true`.

- [ ] **Step 1: Write failing test — AC couple read override extracts bit 11**

```python
async def test_ac_couple_reads_bit_11_of_h179(hass, mock_transport):
    """AC Coupling reads bit 11 of register 179."""
    mock_transport.read_parameters = AsyncMock(return_value={179: 0x0800})
    params = {}

    await _override_ac_couple_from_raw_reg_179(mock_transport, params)

    assert params["FUNC_179_BIT11"] is True
    assert params["_raw_reg_179"] == 0x0800
```

- [ ] **Step 2: Run test to verify it fails**
- [ ] **Step 3: Implement read override and raw write**

Same pattern as ECO:
- Read: after `read_named_parameters()`, read H179 raw, extract bit 11, cache as `_raw_reg_179`
- Write: `_execute_ac_couple_raw()` — read-modify-write bit 11

AC Coupling Mode is already in `WORKING_MODES` on main (`"ac_couple_mode"` with param `"FUNC_179_BIT11"`). Just need to route it to the raw write path in `_execute_working_mode()`.

- [ ] **Step 4: Run tests, commit**

```bash
git commit -m "feat: AC Coupling Mode uses raw bit-11 of register 179

Bypass pylxpweb named parameter system for register 179 bit 11.
Same raw read-modify-write pattern as ECO Mode (H110 bit 15)."
```

---

## Task 12: Suppress internal temp and board temp for OFFGRID

Registers 64 and 108 always return 0 on 12000XP.

**Files:**
- Modify: `custom_components/eg4_web_monitor/coordinator_mappings.py` (feature flags)
- Modify: `custom_components/eg4_web_monitor/sensor.py` (gating)
- Modify: `custom_components/eg4_web_monitor/const/device_types.py` (feature set)
- Test: `tests/test_sensor_entities.py`

**Evidence:** Sweep: I64=0, I108=0 across all modes. Radiator temps (I65, I66) work correctly.

- [ ] **Step 1: Write failing test**

```python
async def test_offgrid_suppresses_board_temps(hass, mock_coordinator_offgrid):
    """internal_temperature and bt_temperature suppressed for OFFGRID."""
    coordinator = mock_coordinator_offgrid
    features = coordinator._extract_inverter_features(inverter)

    assert features["supports_inverter_board_temps"] is False
```

- [ ] **Step 2: Run test to verify it fails**
- [ ] **Step 3: Implement feature flag**

In `coordinator_mappings.py` `_extract_inverter_features()`, when family is EG4_OFFGRID:
```python
if family == "EG4_OFFGRID":
    features["supports_inverter_board_temps"] = False
```

In `sensor.py` `_should_create_sensor()`, check this flag for `INVERTER_BOARD_TEMP_SENSORS`.

- [ ] **Step 4: Run tests, commit**

```bash
git commit -m "fix: suppress internal/board temp sensors for OFFGRID (regs 64, 108 always 0)"
```

---

## Task 13: Suppress AC couple per-leg sensors for OFFGRID (I206, I207)

Registers 206-207 always return 0 on OFFGRID. GridBOSS only.

**Files:**
- Modify: `custom_components/eg4_web_monitor/coordinator_mappings.py` (feature flags)
- Modify: `custom_components/eg4_web_monitor/sensor.py` (gating)
- Test: `tests/test_sensor_entities.py`

**Evidence:** Sweep: I206=0, I207=0 across all modes. I153 (total) works correctly.

- [ ] **Step 1: Write failing test**

```python
async def test_offgrid_suppresses_ac_couple_per_leg(hass, mock_coordinator_offgrid):
    """ac_couple_power_l1/l2 suppressed for OFFGRID."""
    features = coordinator._extract_inverter_features(inverter)
    assert features["supports_ac_couple_per_leg"] is False
```

- [ ] **Step 2: Implement feature flag and gating**
- [ ] **Step 3: Run tests, commit**

```bash
git commit -m "fix: suppress AC couple per-leg sensors for OFFGRID (I206/207 always 0)"
```

---

## Execution Order

Tasks can be grouped into three phases:

**Phase 1 — Core sensor fixes (Tasks 1-6):** Fix generator_power, AC couple energy, L1/L2 voltages, add new sensors. No switch changes. Low risk.

**Phase 2 — Switch entities (Tasks 7-11):** ECO Mode, Green Mode, Export AC Couple, AC Coupling Mode, mode gating. Each follows the same raw r/w pattern. Medium risk (live verification needed for Green Mode).

**Phase 3 — Cleanup (Tasks 12-13):** Feature flag gating for sensors that return 0 on OFFGRID. Low risk.

Deploy and test after each phase before proceeding.
