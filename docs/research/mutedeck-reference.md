# MuteDeck as a reference: state model and local API

Research for issue #7. Sources are MuteDeck's own help site and changelog, plus
two public consumers of its API (Bitfocus Companion module, Raycast extension).
Claims not traceable to one of those are marked **unverified**.

Sources:

- [API] https://mutedeck.com/help/api
- [SD] https://mutedeck.com/help/features/stream-deck
- [SDP] https://mutedeck.com/help/features/stream-deck-plugins
- [OA] https://mutedeck.com/help/features/custom-actions
- [MC] https://mutedeck.com/help/features/meeting-controls
- [TS] https://mutedeck.com/help/features/microsoft-teams-setup
- [SA] https://mutedeck.com/help/getting-started/supported-apps/
- [TR] https://mutedeck.com/help/troubleshooting/app-not-detecting
- [WH] https://mutedeck.com/help/features/webhooks
- [CS] https://mutedeck.com/help/features/companion-setup
- [CL] https://mutedeck.com/changelog (entries v4.9.2 through v4.12.1, Jul–Oct 2026)
- [B40] https://mutedeck.com/blog/2025-12-12-mutedeck-v40/
- [CM] https://github.com/bitfocus/companion-module-mutedeck-mutedeck (`src/api.js`, `actions.js`, `state.js`, `feedbacks.js`, `presets.js`, `main.js`)
- [RC] https://github.com/raycast/extensions/tree/main/extensions/mutedeck (`src/mutedeck.ts`, `show-status.tsx`)

## 1. Transport

- REST base URL `http://localhost:3491`, no auth, only while the app runs; Swagger UI at the same address [API].
- WebSocket for push updates. The API page's example uses `ws://localhost:3491` and a Node example uses `ws://localhost:3491/ws` [API], but both consumers default to `ws://localhost:3492` [CM `api.js`, RC `mutedeck.ts`]. Treat 3492 as the real push port; the doc page is inconsistent (**unverified** which is canonical).
- Client must send `{"source":"plugin","action":"identify"}` "before MuteDeck sends updates" [API]; Companion adds `"identifier":"Bitfocus Companion"` [CM].
- Rate limit 100 req/min; max 10 WebSocket clients; recommended poll interval "1 second or more" [API].
- Webhooks: POST to a user URL "on every change" of call/mute/video/share/record, same shape as the API [WH], plus `timestamp` [API].

## 2. Status model

`GET /v1/status` [API]:

| Field | Values | Notes |
|---|---|---|
| `control` | `"system"`, `"zoom"`, `"teams"`, `"meet"`, "etc." | Which system is controlling. `"system"` = no app detected, system-mic fallback [API, MC]. |
| `call` | `"active"` | Only present in the in-meeting example; absent in the no-meeting example [API]. Companion also expects `inactive`/`disabled` [CS, CM `state.js`]; Raycast treats only `call === "active"` as in-call [RC]. |
| `mute` | `active` (muted), `inactive` (unmuted), `disabled` | |
| `video` | `active` (on), `inactive` (off), `disabled` | |
| `share` | `active`, `inactive`; example shows `disabled` | |
| `record` | `active`, `inactive`; example shows `disabled` | |
| `status` | HTTP code, 200 | |

No-meeting example: `control:"system", mute:"inactive", video/share/record:"disabled"` [API].

Push messages (`action: "update-status"`) carry the same fields at top level; Companion copies `call, mute, video, record, share, control` as-is with no normalisation and keeps the previous value for any field missing from a message [CM `main.js`, `state.js`]. Companion's help lists `control_status` as `active/inactive/disabled` [CS], which conflicts with REST's app-name values (**unverified** which is right over WebSocket).

There are no Teams- or Zoom-specific fields, and no hand-raise or reaction state anywhere [API, CM, RC].

## 3. Commands

REST [API]:

| Endpoint | Semantics |
|---|---|
| `POST /v1/mute`, `/video`, `/share`, `/record` | Toggle only. Response `{"status":200,"action":"mute_toggle","result":"active"}`. |
| `POST /v1/leave` | One-shot. |
| `POST /v1/custom/{action_name}` | Fires a user-configured "Other Action", e.g. `/v1/custom/raise-hand`. |

WebSocket (as used by Companion) [CM `actions.js`]: `toggle_mute`, `toggle_video`, `toggle_share`, `toggle_record` with `state: toggle | on | off` (so **set** semantics exist over WebSocket; the push-to-talk preset sends `off` on press, `on` on release, where `on` = muted [CM `presets.js`]); `leave_meeting`; `bring-to-front`; `custom-action {name}`; `list-custom-actions`.

"Other Actions" are keyboard shortcuts sent to the app (Teams raise hand = Cmd+Shift+K on macOS); MuteDeck reports no state for them [OA]. Reactions and hand-raise ship only as pre-configured Other Actions in the "MuteDeck for Teams" starter pack [SDP]. Reactions "open menus ... may still bring the meeting app forward" [CL v4.11].

Unsupported / unavailable control: reported as `"disabled"` in status, HTTP `503` "MuteDeck not ready/no meeting detected" on commands, `400` invalid params, `404` unknown endpoint [API]. Raycast hides a toggle when its state is `disabled` or empty [RC]. Unrecognised WebSocket actions are silently ignored by Companion; there is no documented error frame [CM `api.js`].

## 4. Polling vs push

- Stream Deck plugin: the setup page says Stream Deck gets "priority updates for faster response" with "a ~0.5 second delay" that is "intentional for system optimization" [SD]. It does not say WebSocket; given the identify-as-`"plugin"` handshake [API], push over WebSocket is the likely mechanism (**unverified**).
- Companion: WebSocket only, no polling or status request; reconnects every 3 s [CM `api.js`].
- Raycast: REST polling every 1000 ms; after a toggle it waits for the reported state to change "because the status API lags behind actions" [RC `show-status.tsx`].
- MuteDeck itself scans apps roughly once a second: v4.12 "on macOS no longer checks every running app each second" [CL].

## 5. "No call", "not running", "unavailable"

| Condition | MuteDeck representation |
|---|---|
| App not running | REST: connection refused [API]. Raycast → `MuteDeckOffline` "MuteDeck isn't running" [RC]. Companion → `connected=false`, all fields reset to `""`, labels "Unknown", buttons greyed [CM, CS]. Stream Deck key: gray [SD]. |
| No call | `control:"system"`, `call` absent, video/share/record `disabled`, mute still live via system mic ("No Call Detected", "Using system-level microphone control") [API, MC, TR]. Stream Deck key: gray [SD]. |
| Control unavailable | field `disabled`; command → 503 [API]. |
| Press failed | Stream Deck shows a yellow triangle [SD, TR]. |
| Permission missing | No API state. v4.9.3 notes a false "Accessibility permission broken" warning that silently stopped detection; only a real denial now warns [CL]. |

## 6. Teams specifics after the third-party API retirement

- Microsoft's notice MC1266901 scheduled removal of the Teams third-party meeting/call control APIs for 30 June 2026 (mirror: https://mc.merill.net/message/MC1266901; **unverified** against the primary admin center).
- MuteDeck v4.0 (Dec 2025) added a "hybrid approach": official API when available, "intelligent fallback controls that work through the Teams interface directly" [B40].
- v4.9.3 (Aug 2026): "Microsoft removed the third-party device API from Teams, so pairing had nothing to connect to"; pairing dropped from Settings; "Teams keeps working through interface control" [CL].
- Detection is "interface discovery"; the Teams UI language must match MuteDeck's language setting or "MuteDeck will not be able to detect the meeting state" [TS]. Desktop Teams supports EN, DE, ES, FR, NL, JA [SA].
- macOS 13+, requires Accessibility + Automation permissions; they "may reset after macOS or MuteDeck updates" and must be removed/re-added [SA, TR, B40].
- Fragility logged in the changelog [CL]: Teams Leave button redesign broke Leave (v4.9.2); presses ignored when the meeting sat on a second display (v4.9.3); redesigned meeting controls needed re-matching (v4.9.4); background Teams "accepted the mute press but didn't actually update" (v4.12); opening a Teams menu "can hide the meeting for a moment" (v4.12.1); scanning for Teams made Outlook sluggish (v4.9.4).

## 7. Comparison with our Teams-only tool

| MuteDeck | Ours | Decision |
|---|---|---|
| `control` app name | Teams only | Drop; replace with explicit `source` health state (below). |
| `call` present/absent | in call / no call | Keep, but as a required enum, never absent. |
| `mute`, `video` tri-state | muted, camera on | Keep the `active/inactive/disabled` vocabulary. |
| `share`, `record` | not needed | Drop. |
| hand raised | needed | Add `hand` tri-state; MuteDeck has none. |
| reactions | needed | Add one-shot `reaction(kind)` command, no state. |
| `leave`, `bring-to-front` | not needed | Drop. |
| not running / permission / stale | needed, distinct | Add; MuteDeck conflates all three into "gray". |
| toggle-only REST, set over WS | toggle | Toggle is enough; keep optional `state: on/off` like Companion for push-to-talk later. |
| REST 503 / `disabled` | needed | Keep: `disabled` field value plus a typed error on command. |

## 8. Recommended minimal model

```
status:
  source:  running | not_running | permission_missing | stale   # health, replaces `control`
  call:    active | inactive
  mute:    active | inactive | disabled
  video:   active | inactive | disabled
  hand:    active | inactive | disabled
  readAt:  ISO timestamp                                          # lets the plugin derive `stale`
commands:
  toggle_mute, toggle_video, toggle_hand   (optional state: toggle|on|off)
  reaction {kind: like|love|applause|laugh|surprised}
errors:  not_running | permission_missing | no_call | control_disabled
```

Push state changes to the plugin (Companion-style `update-status`) and scan the Teams UI at about 1 s as MuteDeck does; fall back to polling at 1 s or more only if push proves unreliable. Expect the same Teams fragility MuteDeck logs (language, background window, control redesigns) and surface it as `stale` rather than silently repeating the last known state.
