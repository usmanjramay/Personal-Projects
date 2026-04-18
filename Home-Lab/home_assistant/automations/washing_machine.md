# Washing Machine Automation

> Built: 2026-04-04 | Status: Live

---

## How It Works

Power monitoring via Meross Smart Plug Mini (Matter, IP: 192.168.4.44).
A single `input_boolean.washing_machine_running` tracks cycle state and drives all notifications.

## Entities

| Entity | Purpose |
|--------|---------|
| `sensor.washing_machine_plug_meross_power` | Live watts |
| `switch.washing_machine_plug_meross` | On/off control |
| `input_boolean.washing_machine_running` | Cycle state tracker (exposed to Alexa) |

---

## Power Thresholds (tuned from 2026-04-04 real cycle)

| Event | Threshold | Confirmation | Rationale |
|-------|-----------|--------------|-----------|
| Cycle started | > 10W | 1 min | Clears 2.7W standby baseline |
| Cycle finished | < 3.9W | 1 min | Lowest mid-cycle reading was 4.656W — 0.75W margin |

Cycle stats: ~84 min active, peak 1,783W (heating), 456 Wh per cycle.

---

## Automations (automations.yaml)

- `washing_machine_cycle_started` — turns `washing_machine_running` ON when power > 10W for 1 min
- `washing_machine_cycle_finished` — sends notifications + turns `washing_machine_running` OFF when power < 3.9W for 1 min (condition: `washing_machine_running` is ON)

---

## Notifications on Finish

1. **iPhone push** — `notify.mobile_app_usmans_iphone`, title "🧺 Washing Machine Done"
2. **WhatsApp** — via n8n workflow `c7qgggehNayrs6Nn`
   - `rest_command.n8n_washing_machine_done` → `https://n8n.srv1016866.hstgr.cloud/webhook/washing-machine-done`
   - Message: "Washing Machine is done!"

---

## Alexa Announcements

`input_boolean.washing_machine_running` is exposed to Alexa via Nabu Casa.
It appears as a **contact sensor** in the Alexa routines menu.

Two routines configured in the Alexa app:

| Routine trigger | Alexa says |
|----------------|-----------|
| "Washing Machine Running" opens (turns ON) | "The washing machine has started" |
| "Washing Machine Running" closes (turns OFF) | "The washing machine is done, time to unload" |

**Note:** Any lag on the Alexa announcement is HA-side (the 1-min confirmation window), not Alexa polling. Nabu Casa proactive state reporting is near-instant.
