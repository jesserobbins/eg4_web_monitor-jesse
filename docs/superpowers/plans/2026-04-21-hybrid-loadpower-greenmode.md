# Hybrid Path Fix, load_power, Green Mode Suppression Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix 6 of 13 OFFGRID sensors that show `unknown` in hybrid mode by wiring direct register reads into the hybrid path, fix load_power mapping, and suppress Green Mode switch for OFFGRID.

**Architecture:** The hybrid path in `coordinator_http.py` already calls some direct-read functions from `coordinator_local.py` after the cloud API returns data. We add the missing calls (`_read_generator_power_per_leg`, `_read_load_power`) and expand the transport overlay in `coordinator_mixins.py` with L1/L2 voltage and EPS power entries so the existing `eps_load_power` sum derivation triggers in hybrid mode. Green Mode suppression follows the existing peak-shaving gating pattern in `switch.py`.

**Tech Stack:** Python 3.13, Home Assistant custom component, pylxpweb>=0.9.26, pytest, pytest-homeassistant-custom-component

**Starting point:** Branch `claude/offgrid-register-fixes` at commit `14dfefd` (867 tests passing)

**Spec:** `docs/superpowers/specs/2026-04-21-hybrid-loadpower-greenmode-design.md`

---

## File Map

| File | Responsibility | Tasks |
|------|---------------|-------|
| `custom_components/eg4_web_monitor/coordinator_http.py` | HTTP/hybrid data path, direct register reads in hybrid mode | 1, 3 |
| `custom_components/eg4_web_monitor/coordinator_local.py` | Local Modbus read functions | 3 |
| `custom_components/eg4_web_monitor/coordinator_mixins.py` | Transport overlay tuple in `_process_inverter_object()` | 2, 3 |
| `custom_components/eg4_web_monitor/switch.py` | Switch entity creation, family gating | 4 |
| `tests/test_coordinator.py` | Hybrid transport overlay tests | 2 |
| `tests/test_coordinator_local.py` | Direct register read function tests | 1, 3 |
| `tests/test_switch_entities.py` | Switch entity creation/suppression tests | 4 |

---

## Task 1: Wire `_read_generator_power_per_leg` into hybrid path

The hybrid path in `coordinator_http.py:245-249` calls four direct-read functions but is missing `_read_generator_power_per_leg`. This causes `generator_power`, `generator_power_l1`, and `generator_power_l2` to show `unknown` in hybrid mode.

**Files:**
- Modify: `custom_components/eg4_web_monitor/coordinator_http.py:38-43` (imports) and `:245-249` (hybrid path block)
- Test: `tests/test_coordinator_local.py` (new test class)

- [ ] **Step 1: Write failing test — hybrid path calls `_read_generator_power_per_leg`**

In `tests/test_coordinator_local.py`, add a test that verifies the function is importable from coordinator_local (it already is) and that calling it in the hybrid-path pattern populates the expected keys. This validates the function works when called from the hybrid context (transport attached to a device fetched via HTTP).

```python
class TestReadGeneratorPowerPerLegHybridPath:
    """Verify _read_generator_power_per_leg works for hybrid-path invocation."""

    async def test_populates_generator_power_keys(self):
        """Generator power per-leg populates all three keys from regs 188-189."""
        from custom_components.eg4_web_monitor.coordinator_local import (
            _read_generator_power_per_leg,
        )

        transport = MagicMock()
        transport._read_input_registers = AsyncMock(return_value=[500, 300])
        sensors: dict = {}

        await _read_generator_power_per_leg(transport, sensors)

        assert sensors["generator_power_l1"] == 500
        assert sensors["generator_power_l2"] == 300
        assert sensors["generator_power"] == 800
```

- [ ] **Step 2: Run test to verify it passes (function already exists)**

Run: `uv run pytest tests/test_coordinator_local.py::TestReadGeneratorPowerPerLegHybridPath -v`
Expected: PASS (the function exists at `coordinator_local.py:224`)

Note: This test validates the function contract. The real fix is adding the import and call to coordinator_http.py.

- [ ] **Step 3: Add import and call to hybrid path**

In `coordinator_http.py`, add `_read_generator_power_per_leg` to the import block at line 38:

```python
from .coordinator_local import (
    _read_ac_couple_energy,
    _read_ac_couple_registers,
    _read_ac_input_type,
    _read_consumption_power,
    _read_generator_power_per_leg,
)
```

Then add the call at line 249, after `_read_consumption_power`:

```python
                # Read AC couple registers from local transport in hybrid mode
                if device_data.get("type") == "inverter":
                    await _read_ac_couple_registers(transport, device_data["sensors"])
                    await _read_ac_input_type(transport, device_data["sensors"])
                    await _read_ac_couple_energy(transport, device_data["sensors"])
                    await _read_consumption_power(transport, device_data["sensors"])
                    await _read_generator_power_per_leg(transport, device_data["sensors"])
```

- [ ] **Step 4: Run full test suite**

Run: `uv run pytest tests/ -x --tb=short`
Expected: All 867+ tests pass

- [ ] **Step 5: Commit**

```bash
git add custom_components/eg4_web_monitor/coordinator_http.py tests/test_coordinator_local.py
git commit -m "fix: wire _read_generator_power_per_leg into hybrid path

Hybrid mode was missing the direct register read for per-leg generator
power (I188+I189), causing generator_power to show unknown."
```

---

## Task 2: Expand transport overlay with L1/L2 voltages and EPS power

The transport overlay in `coordinator_mixins.py:664-673` maps Modbus-only sensor values into hybrid mode. It's missing L1/L2 grid voltages, EPS voltages, and EPS per-leg power. Without the EPS power entries, the existing `eps_load_power` sum derivation in `coordinator_mappings.py:611-617` never triggers in hybrid mode because `eps_power_l1` and `eps_power_l2` stay at their cloud values (which may be None/0).

**Files:**
- Modify: `custom_components/eg4_web_monitor/coordinator_mixins.py:664-673` (overlay tuple)
- Test: `tests/test_coordinator.py` (new tests in `TestHybridTransportExclusiveSensors`)

- [ ] **Step 1: Write failing tests — L1/L2 voltages and EPS power in overlay**

Add to `tests/test_coordinator.py` class `TestHybridTransportExclusiveSensors` (after line 4735):

```python
    async def test_transport_runtime_populates_l1l2_voltages(
        self, hass, mock_config_entry
    ):
        """L1/L2 grid and EPS voltages overlay from transport_runtime."""
        mock_config_entry.add_to_hass(hass)
        coordinator = EG4DataUpdateCoordinator(hass, mock_config_entry)

        mock_inverter = MagicMock()
        mock_inverter.serial_number = "1111111111"
        mock_inverter.model = "12000XP"
        mock_inverter.firmware_version = "2.0.0"
        mock_inverter.refresh = AsyncMock()
        mock_inverter.detect_features = AsyncMock()
        mock_inverter._transport = MagicMock()

        mock_runtime = MagicMock()
        mock_runtime.grid_l1_voltage = 123.6
        mock_runtime.grid_l2_voltage = 122.9
        mock_runtime.eps_l1_voltage = 122.3
        mock_runtime.eps_l2_voltage = 123.0
        mock_inverter._transport_runtime = mock_runtime

        result = await coordinator._process_inverter_object(mock_inverter)
        sensors = result["sensors"]

        assert sensors["grid_voltage_l1"] == 123.6
        assert sensors["grid_voltage_l2"] == 122.9
        assert sensors["eps_voltage_l1"] == 122.3
        assert sensors["eps_voltage_l2"] == 123.0

    async def test_transport_runtime_populates_eps_power_per_leg(
        self, hass, mock_config_entry
    ):
        """EPS per-leg power overlay enables eps_load_power sum derivation."""
        mock_config_entry.add_to_hass(hass)
        coordinator = EG4DataUpdateCoordinator(hass, mock_config_entry)

        mock_inverter = MagicMock()
        mock_inverter.serial_number = "1111111111"
        mock_inverter.model = "12000XP"
        mock_inverter.firmware_version = "2.0.0"
        mock_inverter.refresh = AsyncMock()
        mock_inverter.detect_features = AsyncMock()
        mock_inverter._transport = MagicMock()

        mock_runtime = MagicMock()
        mock_runtime.eps_l1_power = 1031
        mock_runtime.eps_l2_power = 296
        mock_inverter._transport_runtime = mock_runtime

        result = await coordinator._process_inverter_object(mock_inverter)
        sensors = result["sensors"]

        assert sensors["eps_power_l1"] == 1031
        assert sensors["eps_power_l2"] == 296
        # eps_load_power sum must be recomputed from overlaid values
        assert sensors["eps_load_power"] == 1327
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_coordinator.py::TestHybridTransportExclusiveSensors::test_transport_runtime_populates_l1l2_voltages tests/test_coordinator.py::TestHybridTransportExclusiveSensors::test_transport_runtime_populates_eps_power_per_leg -v`
Expected: FAIL — the overlay doesn't include these keys yet, and eps_load_power sum isn't recomputed after overlay

- [ ] **Step 3: Add entries to `_TRANSPORT_OVERLAY` and post-overlay sum derivation**

In `coordinator_mixins.py`, expand the `_TRANSPORT_OVERLAY` tuple at line 664. The runtime attribute names come from `coordinator_mappings.py:524-543` where the sensor dict maps them:

```python
            _TRANSPORT_OVERLAY = (
                ("bt_temperature", "temperature_t1"),
                ("grid_current_l1", "inverter_rms_current_r"),
                ("grid_current_l2", "inverter_rms_current_s"),
                ("grid_current_l3", "inverter_rms_current_t"),
                ("battery_current", "battery_current"),
                # load_power (I170, Pload) — cloud API zeroes this for OFFGRID,
                # but Modbus register 170 has the real value.
                ("load_power", "load_power"),
                # Split-phase voltages (I193/I194 grid, I127/I128 EPS)
                ("grid_voltage_l1", "grid_l1_voltage"),
                ("grid_voltage_l2", "grid_l2_voltage"),
                ("eps_voltage_l1", "eps_l1_voltage"),
                ("eps_voltage_l2", "eps_l2_voltage"),
                # Split-phase EPS power (I129/I130)
                ("eps_power_l1", "eps_l1_power"),
                ("eps_power_l2", "eps_l2_power"),
            )
```

**Critical:** The `eps_load_power` sum derivation in `coordinator_mappings.py:611-617` only runs inside `_build_runtime_sensor_mapping()`, which is local-path only. `_process_inverter_object()` never calls it. After the overlay loop, add a post-overlay recomputation of the sum. Insert after line 677 (after the `for sensor_key, runtime_attr` loop):

```python
            # Recompute eps_load_power from overlaid per-leg values.
            # The sum derivation in _build_runtime_sensor_mapping() only runs
            # in the local path — hybrid mode needs its own recomputation.
            l1 = sensors.get("eps_power_l1")
            l2 = sensors.get("eps_power_l2")
            if l1 is not None or l2 is not None:
                sensors["eps_load_power"] = (l1 or 0) + (l2 or 0)
```

This mirrors the exact logic from `coordinator_mappings.py:611-617`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_coordinator.py::TestHybridTransportExclusiveSensors -v`
Expected: PASS (all tests in class, including existing ones)

- [ ] **Step 5: Run full test suite**

Run: `uv run pytest tests/ -x --tb=short`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add custom_components/eg4_web_monitor/coordinator_mixins.py tests/test_coordinator.py
git commit -m "fix: add L1/L2 voltages and EPS power to transport overlay

Adds grid_voltage_l1/l2, eps_voltage_l1/l2, and eps_power_l1/l2 to
_TRANSPORT_OVERLAY. The EPS power entries enable the existing
eps_load_power sum derivation to work in hybrid mode."
```

---

## Task 3: Direct I170 read for load_power

The spec identifies that the transport overlay maps `load_power` to `transport_runtime.load_power`, which resolves to pylxpweb register 27 (`power_to_user` / `pToUser`) — that's grid import power, NOT total load. Register 170 (`Pload`) is the correct register. We need a direct register read function (same pattern as `_read_consumption_power`) and must remove `load_power` from `_TRANSPORT_OVERLAY` since it maps to the wrong register.

**However:** The overlay entry was added in Task 5 of the original plan (commit `851b7b0`) with a comment that it provides I170 via Modbus. We need to verify whether pylxpweb's `transport_runtime.load_power` actually reads register 27 or register 170 on the local transport path.

**Files:**
- Create function in: `custom_components/eg4_web_monitor/coordinator_local.py` (after `_read_consumption_power` at line 221)
- Modify: `custom_components/eg4_web_monitor/coordinator_mixins.py:664-673` (remove load_power from overlay)
- Modify: `custom_components/eg4_web_monitor/coordinator_http.py:245-249` (add call in hybrid path)
- Modify: `custom_components/eg4_web_monitor/coordinator_local.py:641-650` and `:1314-1321` (add call in both local paths)
- Test: `tests/test_coordinator_local.py`

- [ ] **Step 1: Verify what register pylxpweb maps load_power to**

Before changing anything, confirm the spec's claim. Check pylxpweb source:

Run: `uv run python -c "from pylxpweb.inverter_input import InverterInputRegisters; r = [x for x in InverterInputRegisters if 'load' in x.name.lower() or 'pload' in x.name.lower() or 'power_to_user' in x.name.lower()]; print([(x.name, x.address) for x in r])"`

If `load_power` resolves to register 27 (not 170), proceed with the fix. If it resolves to 170, the overlay is correct and this task reduces to just adding the call to the two local paths (skip Steps 3-4).

- [ ] **Step 2: Write failing tests for `_read_load_power`**

```python
class TestReadLoadPower:
    """Test direct register read for load power (register 170, Pload)."""

    async def test_reads_load_power(self):
        """Reads register 170 and populates load_power."""
        from custom_components.eg4_web_monitor.coordinator_local import (
            _read_load_power,
        )

        transport = MagicMock()
        transport._read_input_registers = AsyncMock(return_value=[3788])
        sensors: dict = {}

        await _read_load_power(transport, sensors)

        assert sensors["load_power"] == 3788

    async def test_skips_when_no_read_fn(self):
        """Gracefully skips when transport lacks _read_input_registers."""
        from custom_components.eg4_web_monitor.coordinator_local import (
            _read_load_power,
        )

        transport = MagicMock(spec=[])
        sensors: dict = {}

        await _read_load_power(transport, sensors)

        assert "load_power" not in sensors

    async def test_handles_read_failure(self):
        """Transport read failure is logged but doesn't raise."""
        from custom_components.eg4_web_monitor.coordinator_local import (
            _read_load_power,
        )

        transport = MagicMock()
        transport._read_input_registers = AsyncMock(
            side_effect=RuntimeError("modbus timeout")
        )
        sensors: dict = {}

        await _read_load_power(transport, sensors)

        assert "load_power" not in sensors
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `uv run pytest tests/test_coordinator_local.py::TestReadLoadPower -v`
Expected: FAIL — `_read_load_power` doesn't exist yet

- [ ] **Step 4: Implement `_read_load_power` in coordinator_local.py**

Add after `_read_consumption_power()` (after line 221), following the exact same pattern:

```python
async def _read_load_power(transport: Any, sensors: dict[str, Any]) -> None:
    """Read load power from input register 170 (Pload).

    pylxpweb maps load_power to register 27 (power_to_user / pToUser),
    which is grid import power — NOT total consumption load.
    Register 170 is the actual Pload measurement.
    """
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
    _LOGGER.debug("Load power: reg170=%dW", regs[0])
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `uv run pytest tests/test_coordinator_local.py::TestReadLoadPower -v`
Expected: PASS

- [ ] **Step 6: Remove `load_power` from `_TRANSPORT_OVERLAY` and add direct read calls**

In `coordinator_mixins.py`, remove the `("load_power", "load_power")` entry and its comment from `_TRANSPORT_OVERLAY` (lines 670-672).

In `coordinator_local.py`, add the call at both local path sites:

At line 647 (after `_read_consumption_power`):
```python
            await _read_consumption_power(static_transport, device_data["sensors"])
            await _read_load_power(static_transport, device_data["sensors"])
            await _read_generator_power_per_leg(
```

At line 1320 (after `_read_consumption_power`):
```python
                    await _read_consumption_power(transport_obj, sensors)
                    await _read_load_power(transport_obj, sensors)
                    await _read_generator_power_per_leg(transport_obj, sensors)
```

In `coordinator_http.py`, add the call in the hybrid path (after `_read_consumption_power` at line 249, and after `_read_generator_power_per_leg` from Task 1):
```python
                    await _read_consumption_power(transport, device_data["sensors"])
                    await _read_load_power(transport, device_data["sensors"])
                    await _read_generator_power_per_leg(transport, device_data["sensors"])
```

Also add `_read_load_power` to the import block:
```python
from .coordinator_local import (
    _read_ac_couple_energy,
    _read_ac_couple_registers,
    _read_ac_input_type,
    _read_consumption_power,
    _read_generator_power_per_leg,
    _read_load_power,
)
```

- [ ] **Step 7: Run full test suite**

Run: `uv run pytest tests/ -x --tb=short`
Expected: All tests pass. The existing `test_transport_runtime_populates_load_power` test in `test_coordinator.py:4661` will now fail since load_power was removed from the overlay. Update it:

In `tests/test_coordinator.py`, modify `test_transport_runtime_populates_load_power` (line 4661) — since load_power no longer comes from the transport overlay, this test should verify that load_power is NOT set by the overlay. Update the assertion:

```python
    async def test_transport_runtime_does_not_overlay_load_power(
        self, hass, mock_config_entry
    ):
        """load_power no longer in overlay (uses direct I170 read instead)."""
        mock_config_entry.add_to_hass(hass)
        coordinator = EG4DataUpdateCoordinator(hass, mock_config_entry)

        mock_inverter = MagicMock()
        mock_inverter.serial_number = "1111111111"
        mock_inverter.model = "12000XP"
        mock_inverter.firmware_version = "2.0.0"
        mock_inverter.refresh = AsyncMock()
        mock_inverter.detect_features = AsyncMock()
        mock_inverter._transport = MagicMock()

        # Set transport_runtime.load_power to a distinctive value
        mock_runtime = MagicMock()
        mock_runtime.load_power = 3788
        # Ensure other overlay attrs return None so they don't mask failures
        mock_runtime.grid_l1_voltage = None
        mock_runtime.grid_l2_voltage = None
        mock_runtime.eps_l1_voltage = None
        mock_runtime.eps_l2_voltage = None
        mock_runtime.eps_l1_power = None
        mock_runtime.eps_l2_power = None
        mock_inverter._transport_runtime = mock_runtime

        result = await coordinator._process_inverter_object(mock_inverter)
        sensors = result["sensors"]

        # load_power must NOT be 3788 — overlay should no longer touch it.
        # The real value comes from _read_load_power() direct register read.
        assert sensors.get("load_power") != 3788
```

- [ ] **Step 8: Commit**

```bash
git add custom_components/eg4_web_monitor/coordinator_local.py \
       custom_components/eg4_web_monitor/coordinator_mixins.py \
       custom_components/eg4_web_monitor/coordinator_http.py \
       tests/test_coordinator_local.py \
       tests/test_coordinator.py
git commit -m "fix: load_power uses direct I170 read instead of transport overlay

The transport overlay mapped load_power to pylxpweb's register 27
(power_to_user / grid import), not register 170 (Pload / total load).
New _read_load_power() reads I170 directly at all three call sites:
local legacy, local main loop, and hybrid path."
```

---

## Task 4: Suppress Green Mode switch for OFFGRID

`EG4OffGridModeSwitch` is appended unconditionally at `switch.py:153`. Jesse confirmed Green Mode is not configurable on the 12000XP OFFGRID. Must be suppressed using the same family-check pattern as peak shaving/forced discharge.

**Ordering issue:** The `EG4OffGridModeSwitch` append (line 153) runs BEFORE the `family` variable is extracted (line 156). We need to move the family extraction earlier or use a separate check.

**Files:**
- Modify: `custom_components/eg4_web_monitor/switch.py:152-156`
- Test: `tests/test_switch_entities.py` (new test)

- [ ] **Step 1: Write failing test — Green Mode suppressed for OFFGRID**

Add to `tests/test_switch_entities.py`, in the same test class as `test_offgrid_suppresses_grid_tied_modes` (near line 203):

```python
    @pytest.mark.asyncio
    async def test_offgrid_suppresses_green_mode(self, hass):
        """Off-Grid Mode (Green Mode) switch suppressed for EG4_OFFGRID."""
        coordinator = _mock_coordinator(
            model="12000XP",
            device_data={
                "features": {"inverter_family": "EG4_OFFGRID"},
            },
        )
        entry = MagicMock()
        entry.runtime_data = coordinator

        entities = []
        await async_setup_entry(hass, entry, lambda e, **kw: entities.extend(e))

        type_names = [type(e).__name__ for e in entities]
        assert "EG4OffGridModeSwitch" not in type_names
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/test_switch_entities.py::TestSwitchEntitySetup::test_offgrid_suppresses_green_mode -v`
Expected: FAIL — `EG4OffGridModeSwitch` is currently appended unconditionally

- [ ] **Step 3: Move family extraction before Green Mode append and add gate**

In `switch.py`, move the `family` extraction from line 156 to before the Green Mode append at line 152. Then gate the append:

Replace lines 152-156:
```python
                # Add off-grid mode switch (Green Mode)
                entities.append(EG4OffGridModeSwitch(coordinator, serial))

                # Add working mode switches
                family = (device_data.get("features") or {}).get("inverter_family")
```

With:
```python
                # Extract family early — needed for Green Mode and working mode gating
                family = (device_data.get("features") or {}).get("inverter_family")

                # Add off-grid mode switch (Green Mode) — not configurable on OFFGRID
                if family != INVERTER_FAMILY_EG4_OFFGRID:
                    entities.append(EG4OffGridModeSwitch(coordinator, serial))

                # Add working mode switches
```

This preserves the existing `family` variable for the working mode suppression below. The `INVERTER_FAMILY_EG4_OFFGRID` constant is already imported (used at line 160).

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest tests/test_switch_entities.py::TestSwitchEntitySetup::test_offgrid_suppresses_green_mode -v`
Expected: PASS

- [ ] **Step 5: Run full test suite**

Run: `uv run pytest tests/ -x --tb=short`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add custom_components/eg4_web_monitor/switch.py tests/test_switch_entities.py
git commit -m "fix: suppress Green Mode switch for EG4_OFFGRID

Green Mode (Off-Grid Mode) is not configurable on the 12000XP OFFGRID.
Moved family extraction before the switch append and gated on
INVERTER_FAMILY_EG4_OFFGRID, same pattern as peak shaving suppression."
```

---

## Execution Order

All four tasks are sequential — each builds on prior changes:

1. **Task 1** adds `_read_generator_power_per_leg` to hybrid path (new import + call)
2. **Task 2** expands `_TRANSPORT_OVERLAY` with L1/L2 entries (same file, different section)
3. **Task 3** replaces overlay `load_power` with direct I170 read (touches overlay again, modifies all three call sites)
4. **Task 4** gates Green Mode switch (independent file, but logically last)

Run full test suite after each commit. All 867+ existing tests must continue to pass.

---

## Out of Scope

- **Task 5 from spec (Export AC Couple re-probe):** Manual test protocol, no code changes. Jesse will test during next deploy.
- No pylxpweb changes.
- No new schedule/config entities.
