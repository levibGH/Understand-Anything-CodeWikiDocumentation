# dashboard_state_and_ui-theme

## Introduction

The `dashboard_state_and_ui-theme` module owns the dashboardÃ¢â‚¬â„¢s visual theme state and the React context used to read and update it. It is responsible for:

- selecting the active theme preset
- tracking the active accent swatch and heading font
- resolving the initial theme from local storage, metadata, or defaults
- applying the theme to the UI through the theme engine
- exposing a safe hook for dashboard components to consume theme state

This module is intentionally small, but it sits at a critical boundary between persisted user preferences, asynchronously loaded project metadata, and the dashboardÃ¢â‚¬â„¢s runtime styling. For broader dashboard state concerns, see [dashboard_state_and_ui-store](dashboard_state_and_ui-store.md). For keyboard and localization concerns, see [dashboard_state_and_ui.md](dashboard_state_and_ui.md) and its child modules.

---

## Module responsibilities

### 1. Theme state ownership
The module stores the current `ThemeConfig` in React state and exposes mutation helpers:

- `setPreset(presetId)`
- `setAccent(accentId)`
- `setHeadingFont(font)`

### 2. Theme resolution
Initial theme selection follows this precedence:

1. persisted user preference from `localStorage`
2. `metaTheme` passed into `ThemeProvider`
3. `DEFAULT_THEME_CONFIG`

### 3. Theme application
Whenever the config changes, the module delegates to `applyTheme(config)` to update the dashboard styling layer.

### 4. Context access
The `useTheme()` hook provides typed access to the current theme context and enforces usage inside a `ThemeProvider`.

---

## Core types

The theme model is defined in [dashboard_state_and_ui-theme-types](dashboard_state_and_ui-theme-types.md) (see also the source module `themes/types.ts`).

### `ThemeConfig`
Represents the user-selected theme state:

- `presetId`: identifies the active preset
- `accentId`: identifies the active accent swatch within the preset
- `headingFont?`: optional heading font override

### `ThemePreset`
Describes a preset available to the dashboard:

- `id`, `name`, `isDark`
- `colors`: CSS token map used by the theme engine
- `accentSwatches`: available accent choices
- `defaultAccentId`: fallback accent when a preset is selected

### `AccentSwatch`
Represents a selectable accent variant with:

- `id`, `name`
- `accent`, `accentDim`, `accentBright`

---

## Architecture overview

```mermaid
flowchart TD
  A[Dashboard components] --> B[useTheme]
  B --> C[ThemeContext]
  C --> D[ThemeProvider]
  D --> E[React state: ThemeConfig]
  D --> F[applyTheme config]
  D --> G[localStorage ua-theme]
  D --> H[metaTheme prop]
  E --> I[getPreset presetId]
  I --> J[ThemePreset]
  J --> K[Accent swatches / default accent]
```

### Key architectural points

- `ThemeProvider` is the single source of truth for dashboard theme state.
- `ThemeContext` exposes both the raw config and the resolved preset.
- `getPreset()` converts a `presetId` into a full preset definition.
- `applyTheme()` is the side-effect boundary that writes theme values into the UI layer.
- `localStorage` persistence is used only for user preference retention, not as the canonical runtime state.

---

## Component relationships

```mermaid
classDiagram
  class ThemeProvider {
    +metaTheme?: ThemeConfig
    +children: ReactNode
  }

  class ThemeContextValue {
    +config: ThemeConfig
    +preset: ThemePreset
    +setPreset(presetId)
    +setAccent(accentId)
    +setHeadingFont(font)
  }

  class ThemeConfig {
    +presetId: PresetId
    +accentId: string
    +headingFont?: HeadingFont
  }

  class ThemePreset {
    +id: PresetId
    +name: string
    +isDark: boolean
    +colors: Record~string,string~
    +accentSwatches: AccentSwatch[]
    +defaultAccentId: string
  }

  class AccentSwatch {
    +id: string
    +name: string
    +accent: string
    +accentDim: string
    +accentBright: string
  }

  ThemeProvider --> ThemeContextValue
  ThemeContextValue --> ThemeConfig
  ThemeContextValue --> ThemePreset
  ThemePreset --> AccentSwatch
```

---

## Runtime behavior

### Initial theme resolution
`ThemeProvider` initializes state using `resolveInitialTheme(metaTheme)`.

```mermaid
flowchart TD
  A[ThemeProvider mount] --> B{localStorage has ua-theme?}
  B -- yes --> C[Use stored ThemeConfig]
  B -- no --> D{metaTheme provided?}
  D -- yes --> E[Use metaTheme]
  D -- no --> F[Use DEFAULT_THEME_CONFIG]
```

### Theme updates
When the config changes, the provider performs two actions:

1. calls `applyTheme(config)` to update the dashboard styling
2. persists the config to `localStorage` after the first render

```mermaid
sequenceDiagram
  participant UI as Dashboard UI
  participant P as ThemeProvider
  participant E as applyTheme()
  participant S as localStorage

  UI->>P: setPreset / setAccent / setHeadingFont
  P->>P: update React state
  P->>E: applyTheme(config)
  alt after initial mount
    P->>S: saveToLocalStorage(config)
  end
```

### Late-arriving metadata
If `metaTheme` is fetched asynchronously after mount, the provider will adopt it only when no stored preference exists.

```mermaid
flowchart TD
  A[metaTheme arrives later] --> B{localStorage has ua-theme?}
  B -- yes --> C[Keep user preference]
  B -- no --> D[setConfig metaTheme]
```

---

## Public API

### `ThemeProvider`
Wraps dashboard content and provides theme state to descendants.

**Props**
- `metaTheme?: ThemeConfig | null`
- `children: ReactNode`

**Behavior**
- resolves initial theme
- applies theme side effects
- persists user changes
- updates when metadata arrives and no preference exists

### `useTheme()`
Returns the current `ThemeContextValue`.

**Throws**
- `Error("useTheme must be used within ThemeProvider")` if called outside the provider tree.

---

## Data flow

```mermaid
flowchart LR
  A[Theme metadata / defaults] --> B[resolveInitialTheme]
  C[User actions] --> D[setPreset / setAccent / setHeadingFont]
  B --> E[ThemeConfig state]
  D --> E
  E --> F[getPreset]
  E --> G[applyTheme]
  E --> H[saveToLocalStorage]
  F --> I[Resolved ThemePreset]
  I --> J[ThemeContext value]
  G --> K[Dashboard styling]
```

---

## Dependencies

### Internal dependencies
- [dashboard_state_and_ui-theme-types](dashboard_state_and_ui-theme-types.md) for `ThemeConfig`, `ThemePreset`, `AccentSwatch`, `PresetId`, `HeadingFont`, and `DEFAULT_THEME_CONFIG`
- `presets.ts` for `getPreset(presetId)`
- `theme-engine.ts` for `applyTheme(config)`

### Related dashboard modules
- [dashboard_state_and_ui-store](dashboard_state_and_ui-store.md) for non-theme dashboard state
- [dashboard_state_and_ui-i18n](dashboard_state_and_ui-i18n.md) for localization context
- [dashboard_state_and_ui-keyboard-shortcuts](dashboard_state_and_ui-keyboard-shortcuts.md) for shortcut definitions
- [dashboard_state_and_ui-shortcuts-help](dashboard_state_and_ui-shortcuts-help.md) for shortcut UI
- [dashboard_graph_view](dashboard_graph_view.md) for graph rendering components that consume dashboard styling

### External runtime dependencies
- React context/state hooks: `createContext`, `useState`, `useEffect`, `useCallback`, `useContext`, `useRef`
- browser `localStorage`

---

## Implementation notes

### Persistence strategy
The module stores the full `ThemeConfig` under the `ua-theme` key. This keeps the persisted format simple and allows the provider to restore the exact user preference on reload.

### Accent selection behavior
Selecting a preset resets the accent to that presetÃ¢â‚¬â„¢s `defaultAccentId`. This prevents invalid accent combinations when switching between presets with different swatch sets.

### Heading font updates
`setHeadingFont()` only updates the font override and preserves the current preset and accent.

### Safety and resilience
- JSON parsing is wrapped in `try/catch`
- storage write failures are ignored gracefully
- `useTheme()` fails fast when used outside the provider

---

## Process flow summary

```mermaid
stateDiagram-v2
  [*] --> Uninitialized
  Uninitialized --> Resolved: resolveInitialTheme()
  Resolved --> Applied: applyTheme(config)
  Applied --> Persisted: saveToLocalStorage()
  Persisted --> Updated: user changes theme
  Updated --> Applied
  Applied --> Persisted
```

---

## Maintenance considerations

- Keep `ThemeConfig` backward compatible if persisted values need to survive upgrades.
- Update `resolveInitialTheme()` if the precedence rules change.
- Ensure new presets define a valid `defaultAccentId` and matching accent swatches.
- If theme application expands beyond CSS variables, keep `applyTheme()` as the single side-effect boundary.

---

## See also

- [dashboard_state_and_ui-store](dashboard_state_and_ui-store.md)
- [dashboard_state_and_ui-i18n](dashboard_state_and_ui-i18n.md)
- [dashboard_graph_view](dashboard_graph_view.md)
- [core_schema_and_types](core_schema_and_types.md) for shared graph and theme-related types
