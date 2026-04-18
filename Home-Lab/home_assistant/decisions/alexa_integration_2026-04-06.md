# Decision: Alexa Integration Method — 2026-04-06

## Decision
Use **Nabu Casa HA Cloud** for Alexa integration.

## Rejected Option
`alexa_media` HACS integration — requires storing Amazon credentials in HA. Security risk, plus known session reauthentication issues over time.

## Chosen Approach
Nabu Casa subscription (already taken for remote HTTPS access). Enables native HA Cloud Alexa skill — no credentials stored in HA, no maintenance required.

`input_boolean` helpers exposed via Nabu Casa appear to Alexa as **contact sensors**, which can trigger Alexa routines on open/close (= on/off).

## Proactive State Reporting
Nabu Casa pushes state changes to Alexa in near real-time. No polling lag. Any observed delay is HA-side (trigger confirmation windows in automations).
