# Visualization Engine Architecture (Terminal Grade)

This document defines the architectural standards for the NoCodeQuant Visualization Engine (Charts and Data Maps).

## 1. Intent-based Reducer Pattern
To ensure stability and auditability of the UI state, all navigation must follow a pure Reducer pattern.

- **Intents**: Discrete UI events (e.g., `SHIFT_TO`, `ZOOM`) must be dispatched as immutable Intent objects/strings.
- **Pure Reducer**: The `ViewportReducer` must remain a pure function: `(State, Intent, Value) -> NewState`. It must contain zero side-effects and zero external dependencies.
- **Invariants**:
    - `start_idx` must always be clamped within `[0, total_len - view_count]`.
    - `view_count` must be clamped within system-defined limits (e.g., 10-500).
    - **Anchor-Right Zoom**: During zoom, the rightmost visible index must remain fixed unless limited by data bounds.

## 2. Coordinate System Invariants
To maintain visual clarity and reduce cognitive load, chart areas must follow strict coordinate boundaries.

### 2.1 Price/Volume Split (80/20 Rule)
- **Primary Area (80%)**: Reserved for Price Action (Candles) and Technical Indicators (MA, BB).
- **Secondary Area (20%)**: Reserved for Volume bars and Oscillator-style indicators (RSI).
- **Absolute Boundary**: Coordinate mapping must use an absolute `price_bottom` limit to ensure 0% geometric overlap between areas.

## 3. High-Performance Rendering
- **Backing Store (QPixmap)**: Static data (candles, indicators) must be pre-rendered to a backing store.
- **Overlay Layer**: Dynamic elements (Crosshair, Tooltips) must be rendered on top of the backing store in a separate pass to ensure butter-smooth cursor interaction.
- **Update Guards**: Real-time updates (ticks) should use `patch_data()` paths to avoid recomputing historical geometry.

## 4. Navigation UX
- **Data Map Scaling**: Scrollbar handle size must be proportional to the visible data window (`view_count / total_len`).
- **Drift-Free Tracing**: Mouse drag must use a pixel accumulator (float-based) in future iterations to prevent integer rounding drift during sub-candle movements.

---
*Version: v1.0*
