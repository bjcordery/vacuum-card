# Error Code Entity Display Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a user point the vacuum card at a sensor entity holding fault/error detail, and show that value appended to the status text only when the vacuum is in the `error` state.

**Architecture:** Follow the existing `battery_entity` pattern exactly — one optional entity-id config string (`error_code_entity`), a getter on `VacuumCard` that resolves it via `hass.states`, and a display-time check in `renderStatus()` that appends the resolved entity's `state` to the localized status text when the vacuum entity's own state is `error` and the resolved entity has a meaningful value.

**Tech Stack:** TypeScript, Lit, `custom-card-helpers` (`HomeAssistant`, `HassEntity` types), Rollup build. No unit test framework in this repo — verification is `npm run build` (type-check) plus a manual smoke test via the `run-vacuum-card` skill.

## Global Constraints

- `error_code_entity` config field: empty string (`''`) means disabled — identical semantics to the existing `battery_entity` field. No separate boolean flag.
- The "is the vacuum erroring" check uses `this.entity.state === 'error'` (the entity's top-level HA state) — the same value `renderToolbar()` already switches on — not the `status` attribute chain (`getAttributes().status`) used elsewhere in `renderStatus()`.
- Fault code is only appended when the resolved entity exists and its `state` is not one of `'unknown'`, `'unavailable'`, `''`.
- Display format when shown: `` `${localizedStatus} — ${errorCodeEntity.state}` `` — em dash with a space on each side.
- Per `validate-i18n` (`scripts/validate-i18n`), only `en.json` is required to have the new `editor.error_code_entity` key — the script only fails on keys present in a locale file but _absent_ from `en.json`, never on a locale file missing a key (see `scripts/validate-i18n:49-66`). Do not spend time adding this key to the other 26 locale files.
- Out of scope: reading an `error`/`error_code` _attribute_ off the vacuum entity itself, and the broader `custom_errors`/`resolveStatusText()` design — this is a standalone feature.
- Spec: `docs/superpowers/specs/2026-08-01-error-code-entity-design.md`.

---

### Task 1: Add `error_code_entity` to the config type and defaults

**Files:**

- Modify: `src/types.ts:63-77` (`VacuumCardConfig` interface)
- Modify: `src/config.ts:20-35` (`buildConfig()`)

**Interfaces:**

- Produces: `VacuumCardConfig.error_code_entity: string`, defaulted to `''` by `buildConfig()`. Later tasks read `this.config.error_code_entity`.

- [ ] **Step 1: Add the field to the `VacuumCardConfig` interface**

In `src/types.ts`, in the `VacuumCardConfig` interface, add a line directly after `battery_entity: string;`:

```ts
export interface VacuumCardConfig {
  entity: string;
  name: string;
  battery_entity: string;
  error_code_entity: string;
  map: string;
  map_refresh: number;
  image: string;
  show_name: boolean;
  show_status: boolean;
  show_toolbar: boolean;
  compact_view: boolean;
  stats: Record<string, VacuumCardStat[]>;
  actions: Record<string, VacuumCardAction>;
  shortcuts: VacuumCardShortcut[];
}
```

- [ ] **Step 2: Default it in `buildConfig()`**

In `src/config.ts`, add a line directly after `battery_entity: config.battery_entity ?? '',`:

```ts
return {
  entity: config.entity,
  name: config.name ?? '',
  battery_entity: config.battery_entity ?? '',
  error_code_entity: config.error_code_entity ?? '',
  map: config.map ?? '',
  map_refresh: config.map_refresh ?? 5,
  image: config.image ?? 'default',
  show_name: config.show_name ?? true,
  show_status: config.show_status ?? true,
  show_toolbar: config.show_toolbar ?? true,
  compact_view: config.compact_view ?? false,
  stats: config.stats ?? {},
  actions: config.actions ?? {},
  shortcuts: config.shortcuts ?? [],
};
```

- [ ] **Step 3: Type-check**

Run: `npm run build`
Expected: build succeeds (this only adds an optional-with-default field; nothing consumes it yet).

- [ ] **Step 4: Commit**

```bash
git add src/types.ts src/config.ts
git commit -m "feat: add error_code_entity config field"
```

---

### Task 2: Resolve the entity and append its value to the status text

**Files:**

- Modify: `src/vacuum-card.ts:85-91` (add getter after `batteryEntity`)
- Modify: `src/vacuum-card.ts:408-427` (`renderStatus()`)

**Interfaces:**

- Consumes: `this.config.error_code_entity: string` (Task 1). `HassEntity` type — already imported in this file at line 23 (`import { ..., HassEntity, ... } from './types';`).
- Produces: `VacuumCard.errorCodeEntity` getter returning `HassEntity | null`, used only inside `renderStatus()` in this task.

- [ ] **Step 1: Add the `errorCodeEntity` getter**

In `src/vacuum-card.ts`, directly after the existing `batteryEntity` getter (currently lines 85-91):

```ts
  get errorCodeEntity(): HassEntity | null {
    const errorCodeEntityId = this.config.error_code_entity;
    if (!this.hass || !errorCodeEntityId) {
      return null;
    }
    return this.hass.states[errorCodeEntityId] ?? null;
  }
```

- [ ] **Step 2: Update `renderStatus()` to append the fault code**

Replace the current `renderStatus()` body (lines 408-427):

```ts
  private renderStatus(): Template {
    const { status } = this.getAttributes(this.entity);
    const localizedStatus =
      localize(`status.${status.toLowerCase()}`) || status;

    if (!this.config.show_status) {
      return nothing;
    }

    const errorCodeEntity = this.errorCodeEntity;
    const showErrorCode =
      this.entity.state === 'error' &&
      errorCodeEntity !== null &&
      !['unknown', 'unavailable', ''].includes(errorCodeEntity.state);

    const displayStatus = showErrorCode
      ? `${localizedStatus} — ${errorCodeEntity!.state}`
      : localizedStatus;

    return html`
      <div class="status">
        ${this.requestInProgress
          ? html`<ha-spinner class="status-spinner" size="tiny"></ha-spinner>`
          : nothing}
        <span class="status-text" alt=${displayStatus}>
          ${displayStatus}
        </span>
      </div>
    `;
  }
```

- [ ] **Step 3: Type-check**

Run: `npm run build`
Expected: build succeeds.

- [ ] **Step 4: Commit**

```bash
git add src/vacuum-card.ts
git commit -m "feat: append error code entity value to status when vacuum errors"
```

---

### Task 3: Add the editor field and its label

**Files:**

- Modify: `src/editor.ts:80-97` (add new `ha-select` block, reusing the existing `batteryEntities` list)
- Modify: `src/translations/en.json:65-84` (`editor` section)

**Interfaces:**

- Consumes: `VacuumCardConfig.error_code_entity` (Task 1), `this.getEntitiesByType('sensor')` (existing method, `src/editor.ts:38-43`, already called once as `batteryEntities` at line 51).
- Produces: nothing consumed by later tasks — this is the last code task.

- [ ] **Step 1: Add the `ha-select` block**

In `src/editor.ts`, directly after the closing `</div>` of the `battery_entity` option block (currently ending at line 97) and before the `map` option block (currently starting at line 99):

```ts
        <div class="option">
          <ha-select
            .label=${localize('editor.error_code_entity')}
            @selected=${this.valueChanged}
            .configValue=${'error_code_entity'}
            .value=${this.config.error_code_entity}
            @closed=${(e: Event) => e.stopPropagation()}
            fixedMenuPosition
            naturalMenuWidth
          >
            ${batteryEntities.map(
              (entity) =>
                html` <mwc-list-item .value=${entity}
                  >${entity}</mwc-list-item
                >`,
            )}
          </ha-select>
        </div>
```

This reuses the `batteryEntities` array already computed at `src/editor.ts:51` (`this.getEntitiesByType('sensor')`) — no new variable needed, since the fault-code entity is also expected to be a `sensor.*` entity.

- [ ] **Step 2: Add the translation key**

In `src/translations/en.json`, in the `"editor"` object, add a line directly after `"battery_entity": "Battery Entity (Optional)",`:

```json
    "battery_entity": "Battery Entity (Optional)",
    "error_code_entity": "Error Code Entity (Optional)",
```

- [ ] **Step 3: Lint and type-check**

Run: `npm run lint && npm run build`
Expected: both succeed. `lint:translations` passes because `en.json` (the source locale) has the new key — other locale files are not required to have it (see Global Constraints).

- [ ] **Step 4: Commit**

```bash
git add src/editor.ts src/translations/en.json
git commit -m "feat: add error code entity picker to card editor"
```

---

### Task 4: Manual verification

**Files:** none (no code changes — verification only).

**Interfaces:** N/A.

- [ ] **Step 1: Run the smoke test via the `run-vacuum-card` skill**

Invoke the `run-vacuum-card` skill. Configure a mock `hass` with:

- A `vacuum.*` entity whose `state` is `'error'`.
- A `sensor.*` entity (e.g. `sensor.test_fault_code`) with `state` set to a fault string, e.g. `'5: Wheel stuck'`.
- Card config including `error_code_entity: 'sensor.test_fault_code'`.

Confirm the status text renders as `Error — 5: Wheel stuck`.

- [ ] **Step 2: Verify the unavailable-sensor fallback**

With the same vacuum `error` state, set the sensor's `state` to `'unavailable'`.
Confirm the status text renders as just `Error` (no dangling separator).

- [ ] **Step 3: Verify it's gated on vacuum error state**

Set the vacuum entity's `state` to `'docked'` (or any non-`error` state) while the sensor still holds a fault value.
Confirm the status text shows the normal localized status for that state, with no fault code appended.

- [ ] **Step 4: Verify the editor field**

Open the card editor (`vacuum-card-editor`) in the smoke test. Confirm an "Error Code Entity (Optional)" dropdown appears, listing available `sensor.*` entities, positioned between "Battery Entity" and "Map Camera".

- [ ] **Step 5: Final full check and commit**

Run: `npm test` (lint + build)
Expected: passes.

If any manual check in Steps 1-4 failed, fix the relevant task's code before proceeding — do not commit a fix here; amend the task that introduced the bug instead, per this repo's "no unrelated commits" convention. If all checks passed, no commit is needed for this task (verification only).
