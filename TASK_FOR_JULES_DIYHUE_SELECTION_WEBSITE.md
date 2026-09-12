# 🎯 TASK FOR JULES: Dedicated diyHue Device Selection Website & HA Integration

**Assigned Agent:** Jules (`jules.google.com`)  
**Scope:** `ha_config` (`custom_components/diyhue_sync/`, `dashboards/diyhue_dashboard.yaml`, `diyhue_customize.yaml`), `ha_extensions`  
**Authorization Mode:** Fast-Track Global Auto-Acceptance (Rule 1 & Rule 15: All Jules branches/PRs are automatically approved and merged).

---

## 1. Context & Empirical Audit Findings

An empirical diagnostic audit of the current diyHue environment revealed why the selection website / light management is currently broken:

1. **Add-on Web UI Isolation & Login Wall:**
   - The diyHue add-on (`c863ef44_diyhue`) runs on `http://192.168.200.20:80` (Flask / Werkzeug runtime, Python 3.13).
   - Ingress is disabled (`ingress: false`).
   - Any access to `http://192.168.200.20:80` redirects via HTTP 302 to `/login` requiring `admin@diyhue.org` credentials.
   - On the Home Assistant dashboard (`dashboards/diyhue_dashboard.yaml`), the action card opens `http://192.168.200.20:80` directly. When Home Assistant is accessed via HTTPS or remote URL, browsers block this as Mixed-Content or cross-origin iframe security violations.
2. **Missing Entity Selection GUI:**
   - The current integration `custom_components/diyhue_sync` only provides an awkward multi-select dropdown inside Home Assistant's OptionsFlow.
   - With hundreds of entities, this selector is unusable: no room grouping, no search filter, no instant visual feedback, and no bulk selection.
3. **Target Mechanism:**
   - diyHue consumes entities exclusively via Home Assistant customization (`diyhue_customize.yaml` loaded through `customize: !include diyhue_customize.yaml`).
   - A device only needs `diyhue: include`, `diyhue_room: <RoomName>`, and `diyhue_type: light|switch|scene` to be exposed to Hue apps, Alexa, and Google Assistant.

---

## 2. Work Packages for Jules

### Work Package 1: Interactive Web Selection Panel / Custom Lovelace Card

Jules shall build a dedicated, responsive web selection component for Home Assistant:

- **Option A (Preferred):** Custom Lovelace Webcomponent Card (`diyhue-selection-card.js` located in `custom_components/diyhue_sync/frontend/` or `www/community/diyhue-selection-card/`).
  - Embeddable directly on `/lovelace-diyhue` (`dashboards/diyhue_dashboard.yaml`).
  - Displays all Home Assistant devices and entities from domains `light`, `switch`, `scene` cleanly grouped by their **Home Assistant Area (Raum)**.
  - Interactive switches / checkboxes for each light with instant state feedback.
  - "Alle auswählen" / "Raum abwählen" per area.
  - Custom room name override input field per device.
- **Option B:** Native Custom Panel registered in the HA Sidebar via `hass.components.frontend.async_register_built_in_panel`.

### Work Package 2: Backend API in `custom_components/diyhue_sync`

Modernize `custom_components/diyhue_sync`:

1. **WebSocket API Handlers (`websocket_api.py`):**
   - `diyhue_sync/get_matrix`: Returns all lights/switches, their current assignment (included/excluded), current room name, and bridge connection status.
   - `diyhue_sync/toggle_entity`: Toggles include/exclude state for an entity with immediate persistence in `options.json` or config entry.
   - `diyhue_sync/bulk_set`: Bulk-enables or bulk-disables entire areas.
   - `diyhue_sync/apply_sync`: Generates `diyhue_customize.yaml`, calls `homeassistant.reload_core_config`, and calls diyHue's REST API (`POST http://127.0.0.1:80/api/<username>/lights`) to force a light rediscovery.
2. **Asynchronous File Writing:**
   - Ensure non-blocking file writes via `await hass.async_add_executor_job`.

### Work Package 3: Dashboard Overhaul (`dashboards/diyhue_dashboard.yaml`)

1. Remove the broken external redirect card to `http://192.168.200.20:80`.
2. Embed the new interactive selection card into the dashboard.
3. Add a live bridge health indicator card:
   - Ping / HTTP status of `http://192.168.200.20:80`
   - Total number of active Hue lights exposed.

---

## 3. Validation Criteria for Jules

- [ ] Interactive selection UI loads seamlessly inside Home Assistant (zero Mixed-Content errors, zero login prompts).
- [ ] Users can easily toggle lights by room.
- [ ] Clicking "Synchronisieren" updates `diyhue_customize.yaml` cleanly.
- [ ] `ha core check` passes with exit code 0.
- [ ] Code strictly complies with Home Assistant 2026 standards and async design patterns.
