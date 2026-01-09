# SwiftTerm Patches for iShell

This document describes the patches applied to SwiftTerm for use in iShell.

## Fork Details

- **Upstream**: [migueldeicaza/SwiftTerm](https://github.com/migueldeicaza/SwiftTerm)
- **Fork**: [cocl-pm/SwiftTerm](https://github.com/cocl-pm/SwiftTerm)
- **Branch**: `ishell-combined`
- **Base**: SwiftTerm 1.5.0 (commit `a30cc7e`)

## Applied Patches

### 1. Synchronized Output Mode 2026

**Commit**: `d879275`

Implements DEC private mode 2026 for synchronized output. This allows TUI applications to signal that a batch of updates should be rendered atomically, preventing partial frame artifacts.

**Files Modified**:
- `Sources/SwiftTerm/Terminal.swift` - Parse and track mode 2026 state
- `Sources/SwiftTerm/Apple/AppleTerminalView.swift` - Block display updates during sync
- `Sources/SwiftTerm/iOS/iOSTerminalView.swift` - Implement `synchronizedOutputDisabled` delegate

**Reference**: https://gist.github.com/christianparpart/d8a62cc1ab659194337d73e399004036

---

### 2. CADisplayLink Sync Mode Fix (iOS)

**Commit**: `1da1bac`

The mode 2026 patch blocked `queuePendingDisplay()` during sync mode, but iOS uses `CADisplayLink` which calls `updateDisplay()` directly via `step()`. This patch adds a check to the CADisplayLink callback to also respect synchronized output mode.

**Files Modified**:
- `Sources/SwiftTerm/iOS/iOSTerminalView.swift` - Check `synchronizedOutput` in `step()`

---

### 3. Prevent Partial Frames During feed()

**Commit**: `fbb5b99`

Previously, `feedPrepare()` would start the `CADisplayLink` which allowed display updates to occur mid-parse. This caused issues when apps switch to alternate buffer before enabling mode 2026 - the empty/partial buffer would be displayed.

Now display updates only happen after the entire feed batch is processed, via `queuePendingDisplay()` in `feedFinish()`.

**Files Modified**:
- `Sources/SwiftTerm/Apple/AppleTerminalView.swift` - Remove `startDisplayUpdates()` from `feedPrepare()`

---

### 4. Configurable Line Leading Multiplier

**Commit**: `c6bda08`

Adds `lineLeadingMultiplier` static property to `TerminalView` (default 1.0). Host apps can set to 0.0 to eliminate gaps between rows for block characters (▀ ▄ █) used by TUI applications.

**Files Modified**:
- `Sources/SwiftTerm/Apple/AppleTerminalView.swift` - Add property, apply in `computeFontDimensions()`

**Note**: This feature is currently **unused** by iShell. The multiplier is only applied in `computeFontDimensions()` (cell height calculation), not in `drawTerminalContents()` (rendering y-offset). A complete implementation would require patching both locations.

**Related**: SwiftTerm issue #231

---

## Known Issues

### OpenCode Horizontal Lines (Unresolved)

**Problem**: TUI apps (e.g., OpenCode) show horizontal white/gray lines on initial render. Lines disappear when toggling the sidebar (forcing SwiftUI relayout).

**Investigation Summary**:
- Root cause is a **rendering timing issue**, not font metrics
- The mode 2026 patches improved the situation but didn't fully resolve it
- The fix likely needs to happen at the **SwiftUI container level**, not SwiftTerm

**Attempted Fixes (None Worked)**:
1. `lineLeadingMultiplier` in `computeFontDimensions()` - No effect
2. `lineLeadingMultiplier` in `drawTerminalContents()` - No effect
3. Font reset after connection (`tv.font = tv.font`) - No effect
4. Multiple delayed font resets (50ms + 200ms) - No effect
5. `setNeedsLayout/setNeedsDisplay` after connection - No effect
6. `scheduleFullRedraw` in SwiftTerm - Crashed with tee output

**Unexplored Ideas**:
- Force UIView relayout at SwiftUI container level
- SwiftUI `.id()` modifier to force view recreation
- CALayer level: `tv.layer.setNeedsDisplay(); tv.layer.displayIfNeeded()`

**Investigation File**: `~/Downloads/opencode-horizontal-lines-investigation.md`

---

## Patch File

A combined diff of all patches is available at:
```
iShell/patches/swiftterm-ishell-combined.patch
```

To regenerate:
```bash
cd Packages/SwiftTerm
git diff a30cc7e..ishell-combined > ../../patches/swiftterm-ishell-combined.patch
```
