---
name: wivi-grafana-dashboards
description: Build or change Grafana dashboards from WiVi vehicle data (signals, DTCs, GPS) with the wivi-api, wivi-analytics and wivi-grafana MCP servers. Use when the user asks to create, draft, extend or fix a Grafana dashboard or panel for a WiVi vehicle or fleet.
---

# Building WiVi Grafana dashboards

Dashboards are built from tested recipes and saved as drafts in the **AI Drafts**
folder. Never write panel JSON or GraphQL queries by hand.

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
4. **Check when data exists.** Call `signal_latest` for each signal. If the
   latest reading is older than the requested window, tell the user and offer a
   window that covers the data (up to 2160 hours).
5. **Pick recipes** with `list_dashboard_templates`: `signal_trend` (time series),
   `signal_latest` (current value), `dtc_table`, `gps_track` (map),
   `gps_latest` (last position). At most 12 panels.
6. **Pick the datasources.** Confirm the UIDs on this server with wivi-grafana
   `list_datasources`.
   - Panels: default `wildgraphql` (UID `wivi-graphql`). Use `infinity` only
     when the response needs reshaping. Pass `wildgraphql_uid` or
     `infinity_uid` if they differ.
   - Dropdowns and vehicle names: the WiVi MySQL datasource, usually named
     "WiVI Data" (UID `mysql001`). Pass `mysql_uid` if it differs. If the
     server has no MySQL datasource, say so and stop: signal and DTC panels
     need it for the vehicle names.
7. **Build** with `build_dashboard_draft`. It does not publish anything.
8. **Ask for approval.** Show the title, the dropdowns (or the vehicle or fleet
   the dashboard is fixed to), each panel (signal and unit), the datasources
   and the time range. Wait for an explicit yes.
9. **Save.** Find the AI Drafts folder UID with wivi-grafana `search_folders`,
   then call `update_dashboard` with the returned dashboard, that folder UID and
   `overwrite: false`. Give the user the dashboard link.

## Rules

- Only AI Drafts is writable. Do not try to save anywhere else; other folders are
  read-only for this account.
- To change an existing draft, load it with `get_dashboard_by_uid` first and keep
  its `uid`, `id` and `version`.
- The builder supports "last N hours". For a fixed date range, set the
  dashboard's `time.from` and `time.to` in the returned JSON before saving, and
  say so.
- Never put credentials, tokens or sampled data values in a dashboard.
