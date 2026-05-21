# Connectors

MCP connector reference implementations for the external systems the skills in this pack interact with. Each connector defines a typed action contract with explicit tier classification (autonomous, gated, forbidden) and a reference implementation suitable for adapting to your runtime.

## Planned connectors

| Connector | What it talks to | Primary skill consumer | Status |
|---|---|---|---|
| `siba-portal/` | SIBA reporting portal (Portuguese guest-registration system) | 03 SIBA submission | First to land |
| `email-imap/` | Generic IMAP for host's inbox | 01 Pre-arrival welcome, 04 In-stay support | Planned |
| `email-smtp/` | Generic SMTP for outbound from host's address | 01 Pre-arrival welcome | Planned |
| `pms-channel-sync/` | Booking-channel synchronization (Booking.com, Airbnb) | 04 In-stay support, 05 Accountant handoff | Later phase |
| `accounting-handoff/` | Accountant inbox + structured monthly export | 05 Accountant handoff | Later phase |

## Connector contract

Each connector ships with:

- A typed action manifest (JSON Schema) declaring every action it exposes, with `tier`, `input_schema`, `output_schema`, and `requires_extraction_agreement` where relevant.
- A reference implementation (Playwright for web UIs, MCP server for APIs and local systems).
- Tests with synthetic fixtures the connector framework can replay.
- Documentation of known drift modes and how the connector should halt + escalate when the external system changes.

## SIBA portal first

The SIBA portal connector is the first to land because the SIBA submission flow is the canonical regulator-touching workflow in this vertical. It will set the pattern other connectors follow.

## Status

Active development. Contributions welcome once the SIBA portal connector reaches a reviewable shape.
