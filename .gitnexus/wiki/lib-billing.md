# lib — billing

# lib/billing/team-plan-custom-messaging

Provides plan-gating logic for team-level features, particularly around dataroom branding, layout customization, and visitor-facing settings.

## Overview

This module contains a set of predicate functions that answer a single question: *does a given team plan allow access to this feature?* Callers use these checks to conditionally render UI, gate API actions, or evaluate feature flags.

## Plan String Format

Plan strings follow a compound format: `base-plan[+modifier]`.

| Component | Example | Description |
|-----------|---------|-------------|
| `base-plan` | `business`, `datarooms`, `datarooms-plus` | The primary subscription tier |
| `modifier` | `drtrial`, `old` | Optional add-ons that unlock features on lower tiers |

All functions in this module:

1. Handle `null`, `undefined`, and `"free"` as a single rejection case
2. Treat any plan containing `"drtrial"` as a trial state that unlocks Data Rooms features
3. Extract the base plan by splitting on `+` before comparing

## Feature Gates

### `teamPlanAllowsCustomWelcomeAndCta`

Determines whether a team can set a custom welcome message and call-to-action. This applies to both global team branding and per-dataroom branding.

**Grants access to:** Business, Data Rooms, Data Rooms Plus, Data Rooms Premium, Data Rooms Unlimited, or any `drtrial` team.

```typescript
function teamPlanAllowsCustomWelcomeAndCta(
  plan: string | null | undefined
): boolean
```

**Used by:**
- `teams/[teamId]/branding.ts` — gates the welcome/CTA form
- `datarooms/[id]/branding.ts` — gates per-dataroom welcome/CTA settings

### `teamPlanIsDataroomPlusTier`

Checks whether a team is on Data Rooms Plus or a higher tier (Premium, Unlimited). This is a narrower gate than `teamPlanAllowsLayoutCustomization` — it excludes the base `datarooms` tier.

**Grants access to:** Data Rooms Plus, Data Rooms Premium, Data Rooms Unlimited only. Trials do not count for this check.

```typescript
function teamPlanIsDataroomPlusTier(
  plan: string | null | undefined
): boolean
```

**Used by:**
- `datarooms/[id]/calculate-indexes.ts` — evaluates index computation eligibility
- `lib/featureFlags/dataroom-index-viewer.ts` — determines whether the index viewer feature flag is active for a viewer

### `teamPlanAllowsLayoutCustomization`

Controls whether a team can save dataroom layout settings (preset, folder tree, card layout, header style, etc.). The companion function `teamPlanShowsLayoutUi` controls whether the UI renders at all; this function controls whether the *save* action succeeds.

**Grants access to:** Data Rooms, Data Rooms Plus, Data Rooms Premium, Data Rooms Unlimited, or any `drtrial` team.

```typescript
function teamPlanAllowsLayoutCustomization(
  plan: string | null | undefined
): boolean
```

**Used by:**
- `teams/[teamId]/branding.ts` — gates the layout save action in global branding
- `datarooms/[id]/branding.ts` — gates the layout save action in per-dataroom branding

### `teamPlanAllowsVisitorLanguage`

Controls whether a team can configure a visitor language picker for dataroom branding. English is always available as the free default; this gate determines whether additional languages are offered.

**Grants access to:** Data Rooms Plus, Data Rooms Premium, Data Rooms Unlimited, or any `drtrial` team.

```typescript
function teamPlanAllowsVisitorLanguage(
  plan: string | null | undefined
): boolean
```

**Note:** Callers must separately enforce that English remains the default when the feature is disabled.

**Used by:**
- `datarooms/[id]/branding.ts` — gates the language picker control

### `teamPlanShowsLayoutUi`

Controls whether the Layouts UI is rendered at all in the branding screens. This is a looser gate than `teamPlanAllowsLayoutCustomization` — Business plan users see and interact with the controls, but attempting to save triggers an upgrade modal.

**Grants access to:** Business, Data Rooms, Data Rooms Plus, Data Rooms Premium, Data Rooms Unlimited, or any `drtrial` team.

```typescript
function teamPlanShowsLayoutUi(
  plan: string | null | undefined
): boolean
```

**Used by:**
- `teams/[teamId]/branding.ts` — conditionally renders the layout controls
- `datarooms/[id]/branding.ts` — conditionally renders the layout controls

## Plan Hierarchy

```
free
  └── business          → Layout UI visible, custom welcome/CTA allowed
        └── datarooms   → Layout customization saved, custom welcome/CTA allowed
              └── datarooms-plus
                    ├── datarooms-premium
                    └── datarooms-unlimited
                          → Visitor language picker allowed
                          → Dataroom Plus tier features enabled
```

**`drtrial` modifier** — When appended to any plan (e.g., `business+drtrial`), it activates Data Rooms feature gates. This allows trial users to exercise the full feature set without a paid subscription.

## Usage Pattern

Callers typically follow a two-step pattern for layout settings:

```typescript
// Step 1: Determine whether to render the UI
const showLayoutUi = teamPlanShowsLayoutUi(team.plan);

// Step 2: Determine whether the save action is permitted
const canSaveLayout = teamPlanAllowsLayoutCustomization(team.plan);
```

For welcome/CTA and visitor language, a single gate check is sufficient since there is no separate "visible but not savable" state.