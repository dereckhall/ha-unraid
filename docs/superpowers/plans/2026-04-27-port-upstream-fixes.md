# Port Upstream Fixes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Port 6 upstream fixes/improvements from ruaan-deysel/ha-unraid that prevent disk wake, remove dead code, improve switch reliability, and add a missing sensor — all without bumping the unraid-api dependency.

**Architecture:** Changes span coordinator (metrics query), websocket (remove array subscription), config flow (version checking), sensor (remove FlashUsageSensor, add Docker total memory), and switch (graceful 304 handling). Each task is independent and can be done in any order. Our existing `_try_with_compat_fallback` pattern handles API version differences.

**Tech Stack:** Python 3.13+, Home Assistant 2026.3.0+, unraid-api >=1.7.0, pytest, AwesomeVersion

---

## Task 1: Disk Wake Fix — Safe Metrics Query

The system coordinator polls `get_system_metrics()` every 30 seconds. The library's query includes `metrics.temperature.sensors` which triggers smartctl disk reads, waking sleeping disks. We need a compat query that omits the temperature sensors block.

**Files:**
- Modify: `custom_components/unraid/coordinator.py:60-75` (add safe metrics compat query)
- Modify: `custom_components/unraid/coordinator.py:367-371` (use safe query as primary)
- Test: `tests/test_coordinator.py`

- [ ] **Step 1: Update the compat metrics query to omit temperature sensors**

In `coordinator.py`, rename `_METRICS_COMPAT_QUERY` to `_METRICS_SAFE_COMPAT_QUERY` and ensure it does NOT include `metrics { temperature { sensors { ... } } }`. The existing query already omits it — it only queries `cpu`, `memory`, and `info.cpu.packages.temp`/`info.os.uptime`. So the existing compat query IS the safe query. We just need to use it as the primary path instead of a fallback.

Replace the system coordinator's metrics fetch (lines 367-371) so it uses the compat query directly instead of trying the library method first:

```python
# In _async_update_data, replace the metrics fetch:
metrics = await self._get_system_metrics_compat()
```

This means we always use our safe compat query that omits `temperature.sensors`, matching what upstream's `get_system_metrics_safe()` does. The library's `get_system_metrics()` is never called, so no smartctl reads happen.

- [ ] **Step 2: Update the test to expect the compat method**

In `tests/test_coordinator.py`, the mock client sets up `client.get_system_metrics`. Since we're no longer calling the library method, update the test assertions:

At line 169, the mock setup `client.get_system_metrics = AsyncMock(...)` is no longer needed for the primary path. Instead, the coordinator calls `self._get_system_metrics_compat()` which calls `self.api_client.query(...)`. Update the mock:

```python
# In mock_api_client fixture, add:
client.query = AsyncMock(return_value={
    "metrics": {
        "cpu": {"percentTotal": 25.5},
        "memory": {
            "total": 16000000000,
            "used": 8000000000,
            "free": 8000000000,
            "available": 8000000000,
            "percentTotal": 50.0,
            "swapTotal": 4000000000,
            "swapUsed": 1000000000,
            "percentSwapTotal": 25.0,
        },
    },
    "info": {
        "os": {"uptime": 86400},
        "cpu": {"packages": [{"temp": 55.0, "totalPower": None}]},
    },
})
```

Remove assertions that check `mock_api_client.get_system_metrics.assert_called_once()` at lines 316 and 331. The coordinator now calls `self.api_client.query()` instead.

At line 1084, update `mock_api_client.get_system_metrics.return_value = ...` to instead update the `query` mock return value.

- [ ] **Step 3: Run tests**

Run: `./script/test tests/test_coordinator.py -v`
Expected: All tests pass

- [ ] **Step 4: Commit**

```bash
git add custom_components/unraid/coordinator.py tests/test_coordinator.py
git commit -m "fix: use safe metrics query to prevent disk wake from standby"
```

---

## Task 2: Remove subscribe_array_updates WebSocket Subscription

The `subscribe_array_updates` subscription triggers `storage_coordinator.async_request_refresh()` on every array event, which queries disk data and wakes sleeping disks. Storage data is already polled every 5 minutes — the WS subscription adds no value and causes disk spin-ups.

**Files:**
- Modify: `custom_components/unraid/websocket.py` (remove array updates handler and storage_coordinator dependency)
- Modify: `custom_components/unraid/__init__.py:207` (remove storage_coordinator from WebSocket constructor)
- Test: `tests/test_websocket.py` (remove array update tests, update lifecycle tests)

- [ ] **Step 1: Remove array updates from websocket.py**

In `websocket.py`:

1. Remove `UnraidStorageCoordinator` from the TYPE_CHECKING import (line 23). Keep `UnraidSystemCoordinator`.
2. Remove `storage_coordinator` parameter from `__init__` (line 48) and the `self._storage_coordinator` assignment (line 54).
3. Remove the array_updates task from `async_start` (lines 72-75):
```python
# DELETE these lines:
            asyncio.create_task(
                self._run_subscription("array_updates", self._handle_array_updates),
                name=f"unraid_ws_array_updates_{self._server_name}",
            ),
```
4. Remove the entire `_handle_array_updates` method (lines 171-182).

- [ ] **Step 2: Remove storage_coordinator from WebSocket construction in __init__.py**

In `__init__.py` line 207, remove `storage_coordinator=storage_coordinator,`:

```python
    websocket_manager = UnraidWebSocketManager(
        api_client=api_client,
        system_coordinator=system_coordinator,
        server_name=server_name,
    )
```

- [ ] **Step 3: Update websocket tests**

In `tests/test_websocket.py`:

1. Remove `storage_coordinator` parameter from `_make_manager` helper (lines 30, 38-39, 43).
2. Remove all `api.subscribe_array_updates = ...` lines from lifecycle tests (lines 112, 127, 141).
3. Update task count assertions from `3` to `2` (lines 117, 132).
4. Delete the entire `TestArrayUpdatesSubscription` class (lines 258-286).

- [ ] **Step 4: Run tests**

Run: `./script/test tests/test_websocket.py tests/test_init.py -v`
Expected: All tests pass

- [ ] **Step 5: Commit**

```bash
git add custom_components/unraid/websocket.py custom_components/unraid/__init__.py tests/test_websocket.py
git commit -m "fix: remove array updates WS subscription that wakes sleeping disks"
```

---

## Task 3: Remove FlashUsageSensor

The Unraid GraphQL API never populates `fsSize`, `fsUsed`, `fsFree` for the boot/flash device, so this sensor is permanently unavailable with `restored: true` state. Remove it.

**Files:**
- Modify: `custom_components/unraid/sensor.py` (remove FlashUsageSensor class and setup entry reference)
- Modify: `custom_components/unraid/strings.json` (remove flash_usage translation)
- Modify: `custom_components/unraid/translations/en.json` (remove flash_usage translation)
- Modify: `custom_components/unraid/icons.json` (remove flash_usage icon)
- Modify: `custom_components/unraid/const.py` (remove SENSOR_FLASH_USAGE constant)
- Test: `tests/test_sensor.py` (remove FlashUsageSensor tests)

- [ ] **Step 1: Remove FlashUsageSensor class from sensor.py**

Delete the class at lines 1909-1953 and its comment header at lines 1906-1907.

- [ ] **Step 2: Remove FlashUsageSensor from async_setup_entry in sensor.py**

Delete lines 2542-2544:
```python
    # Flash device sensor (if boot device exists)
    if data.boot:
        entities.append(FlashUsageSensor(storage_coordinator, server_uuid, server_name))
```

- [ ] **Step 3: Remove flash_usage from strings.json and translations/en.json**

In both files, remove:
```json
      "flash_usage": {
        "name": "Flash usage"
      },
```

- [ ] **Step 4: Remove flash_usage from icons.json**

Remove:
```json
      "flash_usage": {
        "default": "mdi:usb-flash-drive"
      },
```

- [ ] **Step 5: Remove SENSOR_FLASH_USAGE from const.py**

Delete line 126: `SENSOR_FLASH_USAGE: Final = "flash_usage"`

- [ ] **Step 6: Remove FlashUsageSensor tests from test_sensor.py**

1. Remove the import `FlashUsageSensor` from line 45.
2. Delete all FlashUsageSensor test functions (lines ~3525-3617): `test_flashusagesensor_creation`, `test_flashusagesensor_state`, `test_flashusagesensor_none_data`, `test_flashusagesensor_none_boot`, `test_flashusagesensor_attributes`.
3. In `test_asyncsetupentry_creates_flash_sensor` (around line 3862), change the assertion from `assert "FlashUsageSensor" in entity_types` to `assert "FlashUsageSensor" not in entity_types` — or better, delete the entire test since it's specifically for testing FlashUsageSensor creation.

- [ ] **Step 7: Run tests**

Run: `./script/test tests/test_sensor.py -v`
Expected: All tests pass

- [ ] **Step 8: Commit**

```bash
git add custom_components/unraid/sensor.py custom_components/unraid/strings.json custom_components/unraid/translations/en.json custom_components/unraid/icons.json custom_components/unraid/const.py tests/test_sensor.py
git commit -m "fix: remove FlashUsageSensor — API never exposes flash usage data"
```

---

## Task 4: Switch Reliability — Handle 304 and Refresh After Actions

Container/VM switches don't refresh after start/stop, causing the UI to bounce back to the previous state until the next poll. Also, 304 "already started/stopped" errors should be handled gracefully instead of raising.

**Files:**
- Modify: `custom_components/unraid/switch.py` (add coordinator refresh + 304 handling)
- Test: `tests/test_switch.py`

- [ ] **Step 1: Add coordinator refresh and 304 handling to DockerContainerSwitch**

In `switch.py`, update `async_turn_on` and `async_turn_off` in `DockerContainerSwitch`:

```python
    async def async_turn_on(self, **kwargs: Any) -> None:
        """Start container."""
        try:
            await self.coordinator.async_start_container(self._container_id)
            _LOGGER.debug("Started Docker container: %s", self._container_id)
        except UnraidAPIError as err:
            if "304" in str(err):
                _LOGGER.debug("Container %s already running", self._container_name)
            else:
                _LOGGER.error("Failed to start Docker container: %s", err)
                raise HomeAssistantError(
                    translation_domain=DOMAIN,
                    translation_key="container_start_failed",
                    translation_placeholders={
                        "name": self._container_name,
                        "error": str(err),
                    },
                ) from err
        await self.coordinator.async_request_refresh()

    async def async_turn_off(self, **kwargs: Any) -> None:
        """Stop container."""
        try:
            await self.coordinator.async_stop_container(self._container_id)
            _LOGGER.debug("Stopped Docker container: %s", self._container_id)
        except UnraidAPIError as err:
            if "304" in str(err):
                _LOGGER.debug("Container %s already stopped", self._container_name)
            else:
                _LOGGER.error("Failed to stop Docker container: %s", err)
                raise HomeAssistantError(
                    translation_domain=DOMAIN,
                    translation_key="container_stop_failed",
                    translation_placeholders={
                        "name": self._container_name,
                        "error": str(err),
                    },
                ) from err
        await self.coordinator.async_request_refresh()
```

- [ ] **Step 2: Add coordinator refresh and 304 handling to VirtualMachineSwitch**

Same pattern for `async_turn_on` and `async_turn_off` in `VirtualMachineSwitch`:

```python
    async def async_turn_on(self, **kwargs: Any) -> None:
        """Start VM."""
        try:
            await self.coordinator.async_start_vm(self._vm_id)
            _LOGGER.debug("Started VM: %s", self._vm_id)
        except UnraidAPIError as err:
            if "304" in str(err):
                _LOGGER.debug("VM %s already running", self._vm_name)
            else:
                _LOGGER.error("Failed to start VM: %s", err)
                raise HomeAssistantError(
                    translation_domain=DOMAIN,
                    translation_key="vm_start_failed",
                    translation_placeholders={"name": self._vm_name, "error": str(err)},
                ) from err
        await self.coordinator.async_request_refresh()

    async def async_turn_off(self, **kwargs: Any) -> None:
        """Stop VM."""
        try:
            await self.coordinator.async_stop_vm(self._vm_id)
            _LOGGER.debug("Stopped VM: %s", self._vm_id)
        except UnraidAPIError as err:
            if "304" in str(err):
                _LOGGER.debug("VM %s already stopped", self._vm_name)
            else:
                _LOGGER.error("Failed to stop VM: %s", err)
                raise HomeAssistantError(
                    translation_domain=DOMAIN,
                    translation_key="vm_stop_failed",
                    translation_placeholders={"name": self._vm_name, "error": str(err)},
                ) from err
        await self.coordinator.async_request_refresh()
```

- [ ] **Step 3: Add tests for 304 handling**

In `tests/test_switch.py`, add tests:

```python
@pytest.mark.asyncio
async def test_container_switch_304_already_running() -> None:
    """Test that 304 error on start is handled gracefully."""
    coordinator = MagicMock(spec=UnraidSystemCoordinator)
    coordinator.data = make_system_data()
    coordinator.async_start_container = AsyncMock(
        side_effect=UnraidAPIError("304: Container already started")
    )
    coordinator.async_request_refresh = AsyncMock()

    container = DockerContainer(id="c1", name="nginx", state="running", is_running=True)
    switch = DockerContainerSwitch(coordinator, "uuid", "server", container)

    # Should not raise
    await switch.async_turn_on()
    coordinator.async_request_refresh.assert_called_once()


@pytest.mark.asyncio
async def test_container_switch_real_error_raises() -> None:
    """Test that non-304 errors still raise."""
    coordinator = MagicMock(spec=UnraidSystemCoordinator)
    coordinator.data = make_system_data()
    coordinator.async_start_container = AsyncMock(
        side_effect=UnraidAPIError("500: Internal Server Error")
    )
    coordinator.async_request_refresh = AsyncMock()

    container = DockerContainer(id="c1", name="nginx", state="exited", is_running=False)
    switch = DockerContainerSwitch(coordinator, "uuid", "server", container)

    with pytest.raises(HomeAssistantError):
        await switch.async_turn_on()
```

- [ ] **Step 4: Run tests**

Run: `./script/test tests/test_switch.py -v`
Expected: All tests pass

- [ ] **Step 5: Commit**

```bash
git add custom_components/unraid/switch.py tests/test_switch.py
git commit -m "fix: handle 304 errors gracefully and refresh after switch actions"
```

---

## Task 5: Add RuntimeError Handling in Coordinators

When the aiohttp session is closed mid-request (e.g., during HA shutdown), a `RuntimeError("Session is closed")` is raised. This should be caught and treated as a connection error instead of crashing.

**Files:**
- Modify: `custom_components/unraid/coordinator.py` (add RuntimeError catch in all three coordinators)
- Test: `tests/test_coordinator.py`

- [ ] **Step 1: Add RuntimeError handling to UnraidSystemCoordinator**

In `coordinator.py`, in `UnraidSystemCoordinator._async_update_data`, add a RuntimeError catch after the existing exception handlers (after line 418):

```python
        except RuntimeError as err:
            self._previously_unavailable = True
            msg = f"Connection error: {err}"
            _LOGGER.debug("System data update failed (session closed): %s", msg)
            raise UpdateFailed(msg) from err
```

- [ ] **Step 2: Add RuntimeError handling to UnraidStorageCoordinator**

Same pattern in `UnraidStorageCoordinator._async_update_data` (after line 618):

```python
        except RuntimeError as err:
            self._previously_unavailable = True
            raise UpdateFailed(f"Connection error: {err}") from err
```

- [ ] **Step 3: Add RuntimeError handling to UnraidInfraCoordinator**

Same pattern in `UnraidInfraCoordinator._async_update_data` (after line 796):

```python
        except RuntimeError as err:
            self._previously_unavailable = True
            raise UpdateFailed(f"Connection error: {err}") from err
```

- [ ] **Step 4: Add test for RuntimeError handling**

In `tests/test_coordinator.py`, add:

```python
@pytest.mark.asyncio
async def test_system_coordinator_handles_runtime_error(
    hass, mock_api_client, mock_config_entry
):
    """Test that RuntimeError (closed session) is treated as connection error."""
    mock_api_client.get_server_info = AsyncMock(
        side_effect=RuntimeError("Session is closed")
    )

    coordinator = UnraidSystemCoordinator(
        hass=hass,
        api_client=mock_api_client,
        server_name="test",
        config_entry=mock_config_entry,
    )

    with pytest.raises(UpdateFailed, match="Connection error"):
        await coordinator._async_update_data()
```

- [ ] **Step 5: Run tests**

Run: `./script/test tests/test_coordinator.py -v`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add custom_components/unraid/coordinator.py tests/test_coordinator.py
git commit -m "fix: catch RuntimeError (closed session) in coordinators"
```

---

## Task 6: Version Check — Replace check_compatibility() with Direct API Version Check

The library's `check_compatibility()` checks both Unraid OS version AND API version. But users can update the API independently via the Connect plugin, so the OS version check is misleading. Replace with a direct check of `get_server_info().api_version` against a hardcoded minimum.

**Files:**
- Modify: `custom_components/unraid/config_flow.py` (replace check_compatibility with direct check)
- Modify: `custom_components/unraid/strings.json` (update error message)
- Modify: `custom_components/unraid/translations/en.json` (same)
- Test: `tests/test_config_flow.py` (update version error tests)

- [ ] **Step 1: Update config_flow.py to use direct API version check**

1. Add `AwesomeVersion` import:
```python
from awesomeversion import AwesomeVersion
```

2. Add a constant at module level (after `MAX_PORT`):
```python
MIN_API_VERSION = "4.21.0"
```

3. Remove `UnraidVersionError` from the imports (line 22).

4. In `_validate_connection` (line 240), remove the `check_compatibility()` call and `UnraidVersionError` handler. Replace with the version check in `_fetch_server_info`:

Replace lines 246-248:
```python
            # Check version compatibility using library method
            # Raises UnraidVersionError if server version is below minimum
            await api_client.check_compatibility()
```
with nothing (just delete those lines).

Remove lines 255-257:
```python
        except UnraidVersionError as err:
            msg = str(err)
            raise UnsupportedVersionError(msg) from err
```

5. In `_fetch_server_info`, add the version check after getting server_info (after line 290):

```python
        api_version = server_info.api_version
        if not api_version or AwesomeVersion(api_version) < AwesomeVersion(
            MIN_API_VERSION
        ):
            msg = (
                f"API version {api_version or 'unknown'} is below minimum "
                f"required version {MIN_API_VERSION}"
            )
            raise UnsupportedVersionError(msg)
```

- [ ] **Step 2: Update error message in strings.json and translations/en.json**

In both files, change line 47:
```json
      "unsupported_version": "Unraid API version not supported. Minimum required: GraphQL API v4.21.0. Update via the Unraid Connect plugin.",
```

- [ ] **Step 3: Update tests**

In `tests/test_config_flow.py`:

1. Remove `UnraidVersionError` from imports (line 22).
2. Remove `mock_api.check_compatibility = AsyncMock(...)` from the `mock_api_client` fixture (lines 56-58).
3. Remove `mock_api.check_compatibility = AsyncMock(...)` from the SSL retry test factory (line 668-669).

4. Update `test_unsupported_version_error` (line 338): Instead of mocking `check_compatibility` to raise, mock `get_server_info` to return a low API version:
```python
async def test_unsupported_version_error(hass: HomeAssistant) -> None:
    """Test old API version shows version error."""
    mock_api = AsyncMock()
    mock_api.test_connection = AsyncMock(return_value=True)
    mock_api.get_server_info = AsyncMock(
        return_value=ServerInfo(
            uuid="test-uuid",
            hostname="tower",
            sw_version="6.9.0",
            api_version="4.10.0",
        )
    )
    mock_api.close = AsyncMock()

    with patch(
        "custom_components.unraid.config_flow.UnraidClient", return_value=mock_api
    ):
        result = await hass.config_entries.flow.async_init(
            DOMAIN,
            context={"source": config_entries.SOURCE_USER},
            data={"host": "unraid.local", "api_key": "valid-key"},
        )

    assert result["type"] is FlowResultType.FORM
    assert result["errors"]["base"] == "unsupported_version"
```

5. Update `test_reauth_flow_unsupported_version_error` (line 1044): Same pattern — mock `get_server_info` with low API version instead of `check_compatibility` raising.

6. Update `test_reconfigure_flow_unsupported_version_error` (line 1528): Same pattern.

- [ ] **Step 4: Run tests**

Run: `./script/test tests/test_config_flow.py -v`
Expected: All tests pass

- [ ] **Step 5: Commit**

```bash
git add custom_components/unraid/config_flow.py custom_components/unraid/strings.json custom_components/unraid/translations/en.json tests/test_config_flow.py
git commit -m "fix: replace check_compatibility() with direct API version check"
```

---

## Final Verification

- [ ] **Run full test suite**

Run: `./script/test -v`
Expected: All tests pass

- [ ] **Run lint**

Run: `./script/lint`
Expected: Clean

- [ ] **Run full check**

Run: `./script/check`
Expected: Clean
