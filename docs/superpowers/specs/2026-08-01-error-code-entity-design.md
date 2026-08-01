# Error code entity display — design

## Problem

When a vacuum entity is in the `error` state, the card shows a generic localized
status string (`status.error` → "Error"). Many integrations (e.g. Tuya-based
local vacuum integrations) expose fault detail on a _separate_ sensor entity
rather than as an attribute on the vacuum entity itself — for example
`sensor.kitchen_dining_room_smart_home_local_tuya_robot_vacuum_fault_code`.
There is currently no way to surface that detail on the card.

## Goals

- Let a user optionally point the card at a sensor entity holding fault/error
  detail.
- Show that detail alongside the status text only when the vacuum is actually
  in the `error` state.
- Follow the existing `battery_entity` pattern: a single optional entity-id
  config field, no separate enable/disable boolean — presence of a valid,
  available entity is what turns the feature on.

## Non-goals

- Reading an `error`/`error_code` _attribute_ directly off the vacuum entity.
  Ruled out because the attribute name and its very existence vary per
  integration; a separate sensor entity is the more general and already
  precedented (`battery_entity`) mechanism.
- The broader `custom_errors` design (configurable `error_messages` mapping,
  `resolveStatusText()` 3-tier fallback). This is a standalone, narrower
  feature shipped independently.

## Config

`src/types.ts` — add to `VacuumCardConfig`:

```ts
error_code_entity: string;
```

`src/config.ts` — `buildConfig()`:

```ts
error_code_entity: config.error_code_entity ?? '',
```

Empty string means disabled, identical semantics to `battery_entity`.

## Card logic (`src/vacuum-card.ts`)

New getter, mirroring `batteryEntity` (existing code at lines 85-91):

```ts
get errorCodeEntity(): HassEntityBase | null {
  const entityId = this.config.error_code_entity;
  if (!this.hass || !entityId) {
    return null;
  }
  return this.hass.states[entityId] ?? null;
}
```

`renderStatus()` changes: after computing `localizedStatus`, determine whether
to append the fault code:

```ts
const errorCodeEntity = this.errorCodeEntity;
const showErrorCode =
  this.entity.state === 'error' &&
  errorCodeEntity != null &&
  !['unknown', 'unavailable', ''].includes(errorCodeEntity.state);

const displayText = showErrorCode
  ? `${localizedStatus} — ${errorCodeEntity.state}`
  : localizedStatus;
```

Render `displayText` in place of the current `localizedStatus` usage, in both
the visible text node and the `alt` attribute of `.status-text`.

The `error` check uses `this.entity.state` (the entity's top-level HA state),
consistent with how `renderToolbar()` already switches on that same value —
not the `status` attribute chain used elsewhere in `renderStatus()`, which can
carry richer non-`error` strings from some integrations.

## Editor (`src/editor.ts`)

Add an `ha-select` block for `error_code_entity`, structurally identical to
the existing `battery_entity` block (lines 80-97): sourced from
`this.getEntitiesByType('sensor')` (the same list already computed as
`batteryEntities`), bound via `.configValue=${'error_code_entity'}`.

## Translations

Add `editor.error_code_entity` label key to `src/translations/en.json` and
every other locale file, per the existing i18n convention enforced by
`node scripts/validate-i18n`. Non-English files may carry the English string
as a placeholder if no translation is available immediately — key parity is
what's enforced, not translated content.

## Error handling / edge cases

| Condition                                                | Behavior                                                            |
| -------------------------------------------------------- | ------------------------------------------------------------------- |
| `error_code_entity` unset (`''`)                         | `errorCodeEntity` is `null`; status text unchanged from today.      |
| Configured entity id doesn't exist in `hass.states`      | `errorCodeEntity` is `null` (same as unset); no dangling separator. |
| Entity exists but state is `unknown`/`unavailable`/empty | `showErrorCode` is `false`; status text unchanged.                  |
| Vacuum entity not in `error` state                       | Fault code never appended, regardless of the sensor's value.        |

## Testing / verification

This repo has no unit test runner; `npm test` is lint + build only. Plan:

1. `npm run build` succeeds (type-checks the new config/type fields).
2. Manual smoke test via the `run-vacuum-card` skill: mock `hass.states` with
   the vacuum entity in `error` state plus a companion sensor entity set to a
   fault string; confirm the appended `" — <value>"` text renders.
3. Same smoke test with the sensor `unavailable`, and separately with the
   vacuum not in `error` state; confirm the fault code does not appear in
   either case.
