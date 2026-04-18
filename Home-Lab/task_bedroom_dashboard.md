# Task: Master Bedroom E-Ink Dashboard

## Goal
Build a clean, minimal dashboard for the master bedroom Android tablet (e-ink screen) that shows:
- Clock, date, weather summary
- Bedroom temperature & humidity
- Roller blind controls (4 blinds)
- Scene shortcuts (Night, Morning, Privacy)
- Conditional camera cards (only visible when person detected / doorbell rings)

Also set up Fully Kiosk Browser for always-on display with auto-kiosk-on-charge.

---

## Prerequisites (confirm before executing)

- [ ] Identify the existing bedroom dashboard — open it in HA and note its name/URL path
- [ ] Export its current YAML: Dashboard → Edit → Raw Configuration Editor → copy all
- [ ] Confirm the entity IDs for:
  - Bedroom temperature sensor (e.g. `sensor.bedroom_temperature`)
  - Bedroom humidity sensor
  - 4 roller blind cover entities (e.g. `cover.bedroom_blind_1`)
  - Scenes: `scene.night`, `scene.morning`, `scene.privacy`
  - Frigate person detection binary sensors (e.g. `binary_sensor.frigate_doorbell_person`)
  - Doorbell binary sensor (if separate)
  - Weather entity (e.g. `weather.home`)
- [ ] Install **Fully Kiosk Browser** app on the Android tablet
- [ ] Install **Fully Kiosk Browser** HA integration via HACS (for automation control)

---

## Dashboard Architecture

**Two dashboards** — one for tablet (e-ink optimized), one for laptop use. No CSS tricks.

### Main View Layout

```
┌──────────────────────────────────┐
│  [Clock]        [Date]           │
│  [Weather icon + temp + summary] │
├──────────────────────────────────┤
│  Bedroom: 21°C  Humidity: 55%   │
├──────────────────────────────────┤
│  Blinds:  [▲][■][▼]  x 4       │
├──────────────────────────────────┤
│  [Night]  [Morning]  [Privacy]   │
├──────────────────────────────────┤
│  [CONDITIONAL: Camera card]      │
│  Only shows when person detected │
│  or doorbell rung                │
└──────────────────────────────────┘
```

### Secondary Views (reachable via nav tabs)
- Weather detail
- All bedroom lights/devices
- Frigate camera feeds (always visible on this view)

---

## E-Ink Design Rules

| Rule | Reason |
|------|--------|
| Black text on white background | E-ink renders gray poorly |
| No animations or transitions | E-ink has ghosting / slow refresh |
| Large tap targets (min 48px) | Touch interaction on tablet |
| Avoid live-updating cards where possible | Reduces flicker from screen refreshes |
| Simple card types: `entities`, `tile`, `button` | Custom cards can break rendering |

---

## Key Card Patterns

### Scenes (one button each)
```yaml
type: button
name: Night
icon: mdi:weather-night
tap_action:
  action: call-service
  service: scene.turn_on
  service_data:
    entity_id: scene.night
```
Repeat for `scene.morning` and `scene.privacy`.

### Conditional Camera (person detected)
```yaml
type: conditional
conditions:
  - condition: state
    entity: binary_sensor.frigate_doorbell_person
    state: "on"
card:
  type: picture-glance
  entity: camera.doorbell
  title: "Person at Door"
  entities:
    - entity: lock.front_door  # add door unlock control if available
```

**Note:** Frigate auto-creates `binary_sensor.<camera_name>_person` entities. Confirm exact entity names in HA Developer Tools → States.

### Roller Blinds (horizontal stack)
```yaml
type: horizontal-stack
cards:
  - type: cover
    entity: cover.bedroom_blind_1
    name: Blind 1
  - type: cover
    entity: cover.bedroom_blind_2
    name: Blind 2
  # ... repeat for 4 blinds
```

---

## Dashboard Editing Workflow (AI-assisted)

The most efficient workflow — avoids SSH YAML editing entirely:
1. Open dashboard in HA → Edit → **Raw Configuration Editor** → copy YAML
2. Paste into conversation with Claude — Claude edits it precisely
3. Copy revised YAML back into Raw Configuration Editor → Apply
4. Preview in desktop browser (exit edit mode to test conditional cards)
5. Check on tablet

This is the pattern to follow for all future dashboard iterations.

---

## Fully Kiosk Browser Setup

1. Install app on tablet
2. Settings in app:
   - Start URL: `http://192.168.4.150:8123/lovelace/bedroom?kiosk` (add `?kiosk` to hide HA header)
   - Enable "Kiosk Mode"
   - Set admin PIN for exit
3. Install **Fully Kiosk Browser** integration in HA (HACS)
4. Create automation: charging → kiosk on; unplugged → kiosk off

### Kiosk Automation YAML (after integration is set up)
```yaml
alias: Tablet kiosk mode when charging
trigger:
  - platform: state
    entity_id: sensor.tablet_plugged_in  # entity from Fully Kiosk integration
    to: "true"
action:
  - service: fully_kiosk.start_kiosk
    target:
      device_id: <tablet_device_id>
---
alias: Exit kiosk when unplugged
trigger:
  - platform: state
    entity_id: sensor.tablet_plugged_in
    to: "false"
action:
  - service: fully_kiosk.stop_kiosk
    target:
      device_id: <tablet_device_id>
```

---

## Status: Ready to execute — start by exporting current dashboard YAML
