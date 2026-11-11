---
name: wivi-grafana-dashboards
description: Build or change Grafana dashboards from WiVi vehicle data (signals, DTCs, GPS) with the wivi-api, wivi-analytics and wivi-grafana MCP servers. Use when the user asks to create, draft, extend or fix a Grafana dashboard or panel for a WiVi vehicle or fleet.
---

# Building WiVi Grafana dashboards

Dashboards are built from tested recipes and saved as drafts in the **AI Drafts**
folder. Never write panel JSON or GraphQL queries by hand.

Show the plan and get the user's approval before building anything. After
approval, build and save in one go, without asking again.

Design each dashboard from what the data is, not from what a signal is called.
The user names signals or a question; you decide how each signal is best shown
and explain the choice.

## Steps

1. **Decide how the dashboard picks vehicles.**
   - Default: Organization, Fleet and Vehicle dropdowns at the top, and every
     panel follows the selection. Leave `vehicle_id` and `fleet_id` out of the
     draft. Several vehicles can be selected at once unless the dashboard has
     a GPS panel, which needs one vehicle.
   - Only when the user names a vehicle or fleet, or asks for no dropdowns:
     pass `vehicle_id` or `fleet_id` (not both). The dashboard is then fixed to
     it. With `fleet_id`, panels show every vehicle of the fleet.
2. **Find a vehicle to take the signal names from.** Use wivi-api
   (`organization_metadata`, `fleet_metadata`, `vehicle_metadata`) to get its
   numeric id. If the user named none, or more than one matches, ask.
3. **Find the signals.** Call wivi-analytics `signal_metadata` with the
   `vehicle_id` and a short filter. Signal names differ between loggers and the
   filter also matches units: engine speed may be `MasterSignal` with unit `RPM`.
   Never guess a name. If nothing fits, show the closest matches and ask.
4. **Profile the signals.** Call `signal_profile` with the signal names (up to
   10 per call) and a window that holds data.
   - If a profile's `kind` is `no_data`, call `signal_latest` to see when the
     signal last reported, tell the user, and profile a window that covers it
     (up to 2160 hours). Use that window for the dashboard too.
   - A signal can exist on several networks with different data. The profile
     has one entry per network: pick the one that fits the question, or ask,
     and pass its `network_name` with the panel.
5. **Design the panels.** Call `list_dashboard_templates` and choose for each
   signal the recipes that show it best. See "Choosing panels" below. Propose
   one to three panels per signal, plus panels that relate signals to each
   other when that answers the question. At most 12 panels.
6. **Pick the datasources.** Confirm the UIDs on this server with wivi-grafana
   `list_datasources`.
   - Panels: default `wildgraphql` (UID `wivi-graphql`). Use `infinity` only
     when the response needs reshaping. Pass `wildgraphql_uid` or
     `infinity_uid` if they differ.
   - Dropdowns and vehicle names: the WiVi MySQL datasource, usually named
     "WiVI Data" (UID `mysql001`). Pass `mysql_uid` if it differs. If the
     server has no MySQL datasource, say so and stop: signal and DTC panels
     need it for the vehicle names.
7. **Show the plan and ask for approval. Do not call `build_dashboard_draft`
   yet.** Show the title, the dropdowns (or the vehicle or fleet the dashboard
   is fixed to), the time range, and each panel with its signal, panel type and
   a one-line reason. Wait for an explicit yes. If the user asks for changes,
   change the plan and show it again.
8. **Build** with `build_dashboard_draft`. Pass with each signal panel what
   the profile reported: `kind`, `network_name`, `unit`, `value_labels` (the
   profile's `labels`) for state signals, and `min` and `max` for dials. A
   recipe that does not suit the stated kind is rejected with the recipes that
   do: correct the panel and build again, and tell the user afterwards what
   differs from the approved plan.
9. **Save straight away, without asking again.** Find the AI Drafts folder UID
   with wivi-grafana `search_folders`, then call `update_dashboard` with the
   returned dashboard, that folder UID and `overwrite: false`. Give the user
   the dashboard link; they can now look at it in Grafana and ask for changes.

## Choosing panels

`signal_profile` reports each signal's `kind` and `fitting_recipes`. Those are
starting points, not rules: pick what answers the user's question, and say why.

| Kind | What it is | Panels to consider |
| --- | --- | --- |
| `state` | A few named or whole-number states: gear, power mode, door, link status | `signal_state`: which state when, one row per vehicle. `state_count`: how often each state occurred. `signal_latest`: the current state. |
| `numeric` | A measurement: speed, temperature, voltage | `signal_trend`: change over time. `signal_gauge`: the latest value on a dial, when there is a natural range. `signal_histogram`: how the values are distributed. `signal_bar`: peaks or totals per hour or day. |
| `counter` | Only goes up: odometer, operating hours | `signal_latest`: the current reading. `signal_bar` with `max`: its level per day. `signal_trend`: its growth. |
| `text` | VIN, part number, software version | `signal_text`: the latest value. `signal_table`: the stored values with their times. |
| `constant` | One value in the whole window | `signal_latest`, or leave it out and tell the user it does not change. |

More guidance:

- **States:** pass `value_labels` so the panel shows "Park" and "Drive" instead
  of numbers. Never average a state signal; state recipes use the highest state
  in each interval.
- **Several vehicles or a fleet:** add `signal_compare` to put the vehicles side
  by side. Trends, bars and timelines already draw one series per vehicle.
- **Related signals:** signals with the same unit go in one panel: pass
  `signal_names` instead of `signal_name` on `signal_trend`, `signal_latest`
  or `signal_table`, for example the four wheel speeds in one trend. Signals
  of different kinds go in neighbouring panels over the same time range, for
  example speed above a gear timeline.
- **Units:** pass the profile's `unit` as a Grafana unit (`velocitykmh`,
  `celsius`, `volt`, `percent`, `lengthkm`, `rotrpm`). Most signals have no
  stored unit; leave it out then.
- **Little data:** when a profile's note says the kind is uncertain, say so and
  prefer `signal_table` or `signal_latest`.
- **DTCs:** `dtc_table` lists them. `dtc_count` counts them by one column
  (`group_by`): the DTC, its code, domain or failure type, the ECU, network,
  message, description or the vehicle. Use it for "top DTCs", "DTCs per ECU",
  "which vehicle has most". Both take the `dtc` filters: `status` (`active`
  by default, `inactive` or `all`), code, domain, failure, ECU, message and
  the status bits such as confirmed or pending. With `active` or `inactive`
  each DTC shows one occurrence; use `all` to show or count every occurrence
  (`count_of: occurrences` needs it).
- **GPS:** `gps_track` for the route, `gps_latest` for the last position.

**Layout.** Panels are placed left to right in the order you give them. Put
current values first (stats, dials, text), then trends and bars, then state
timelines (full width), then counts and tables.

## Rules

- Only AI Drafts is writable. Do not try to save anywhere else; other folders are
  read-only for this account.
- To change an existing draft, load it with `get_dashboard_by_uid` first and keep
  its `uid`, `id` and `version`. The plan names the dashboard and what changes
  in it; the approval covers saving over it.
- The builder supports "last N hours". For a fixed date range, set the
  dashboard's `time.from` and `time.to` in the returned JSON before saving, and
  say so.
- Never put credentials, tokens or sampled data values in a dashboard.
