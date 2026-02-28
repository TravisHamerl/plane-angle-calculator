# Plane Angle & Z-Limit Calculator

## Overview

Single-file HTML/CSS/JS web application for 5-axis CNC plane feasibility analysis. Determines whether a given machining plane (defined by a 3x3 rotation matrix and origin) can be reached by a 5-axis machine tool, considering Z-axis travel limits, tool geometry, and part clearance.

Opens in any modern browser — no build step, no server, no dependencies beyond Three.js (loaded from CDN).

## File

- `index.html` — the entire application (~2,600 lines)

## Quick Start

Open `index.html` in a browser. Load a plane data file (Mastercam format or colon-separated), fill in machine/tool parameters, and the calculator auto-computes feasibility for all planes in the file.

## How It Works

### Input Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| Upper Z limit | Machine Z at upper travel limit (entered positive, stored negative) | 980 → -980 |
| Z axis travel | Total Z travel range; lower limit = upper + travel (e.g. -980+700=-280) | 700 mm (optional) |
| Tool length | Distance from center of rotation (CoR) to tool tip | 300 mm |
| Tool radius | Radius of the cutting tool | 50 mm (optional) |
| Machining ref | Whether to reference tool center or tool edge (radius) | center / radius |
| Z machining level | Height of the machining surface above the table | required |
| Part height | Height of the part/stone above the table | optional |
| Clearance height | Clearance plane above the part | optional |

### Plane Data Format

Supports two formats, auto-detected:

**Mastercam native:**
```
Name : Front Face
Origin (world)    X-100.0 Y-200.0 Z-50.0
Matrix            : X0.696 Y0.717 Z0.001
                  : X-0.605 Y0.586 Z0.537
                  : X0.385 Y-0.375 Z0.843
```

**Colon-separated:**
```
Name: Front Face
Origin (world) X: -100.0  Y: -200.0  Z: -50.0
Matrix row 1: X: 0.696  Y: 0.717  Z: 0.001
Matrix row 2: X: -0.605  Y: 0.586  Z: 0.537
Matrix row 3: X: 0.385  Y: -0.375  Z: 0.843
```

Multiple planes separated by `----` lines. Planes marked `*** NOT USED ***` are flagged.

### Core Math

- **C angle (tilt):** `C = acos(nz)` where `nz` is the Z component of the plane normal (matrix row 3). C=0 is straight down, C=90° is horizontal.
- **A angle (azimuth):** `A = atan2(ny, nx)`. Display convention: A=0 is front (operator/-Y), A=90 is +X (right), A=-90 is -X (left). Internally offset by +90°.
- **Z pivot (CoR position):** `zPivot = zFeature - toolLen * cos(C)`
- **C_min (minimum tilt for Z limit):** `C_min = acos((zFeature - zLimit) / toolLen)`
- **Radius mode offset:** When machining ref is "radius", the effective Z shifts by `toolRadius * sin(C)` to account for disk edge contact instead of center point.

### Machine Coordinate System

- Z=0 at table surface
- Z negative = above table (machine convention)
- User inputs are positive (converted internally)
- Display coordinates: Z positive = up (Three.js convention), converted via `machineZToDisplay(mz, fz) = -(mz - fz)`

### Feasibility Checks

| Check | Condition | Meaning |
|-------|-----------|---------|
| `featureOk` | `zPivot >= zLimit` (upper) | CoR doesn't exceed upper Z limit |
| `reachable` | `zPivot <= zLowerLimit` | CoR is within lower Z travel range |
| `stoneOk` | Stone pivot >= zLimit | Tool clears the stone at Z limit |
| `feasible` | featureOk AND reachable | Full feasibility |

### CoR Z Display

All Z values are displayed in machine coordinates (negative). For example:
- Upper Z limit: -980
- Lower Z limit: -280 (= -980 + 700 travel)
- CoR Z: -650.0 (green if between -980 and -280, red if outside)

## 3D Visualization

Interactive Three.js scene showing the machine setup from any angle.

### Scene Elements

| Element | Color | Description |
|---------|-------|-------------|
| Table | Dark grid | Machine table at Z=0 |
| Stone/Part | Translucent gray | Part geometry (if height specified) |
| Machining plane | Blue-green | Target machining surface |
| Clearance plane | Lighter plane | Clearance height above part |
| Z Upper Limit plane | Amber dashed | Upper Z travel boundary |
| Z Lower Limit plane | Dark gray dashed | Lower Z travel boundary |
| Main tool | Blue (ok) / Red (out of range) | Tool at commanded angle |
| Main disk | Green (feasible) / Red (not) | Tool tip disk, green only when fully feasible |
| Ghost tool | Green/Red translucent | Tool with CoR at upper Z limit |
| Reach tool | Dark gray translucent | Tool with CoR at lower Z limit (always gray, never red) |
| CoR cylinder | Green/Red | Center of rotation, perpendicular to tool shaft |
| Azimuth ring | White | A-axis rotation indicator |
| C arc | Blue | C-axis tilt arc from vertical |

### Tool Vector Direction

The tool visualization uses `dispDir = (-nx, -ny, -nz)` where `(nx, ny, nz)` is the plane normal. This:
- Negates XY to flip the tool so it points toward the surface being machined
- Negates Z to convert from machine coords (Z negative = up) to display coords (Z positive = up)
- The shaft extends from the tip toward the CoR/spindle
- The arrow continues past the CoR in the same direction

### View Buttons

Top, Front, Back, Right, Left preset camera positions. Camera state (position, target) persists across calculations.

### Table Labels

- FRONT (operator) — default dim color
- BACK, LEFT, RIGHT — positioned on table edges
- Part Top — labeled on left side to avoid overlap with clearance level

## Batch Mode

Load a file with multiple planes (separated by `----`). The calculator:
1. Parses all planes
2. Computes feasibility for each
3. Shows batch table with A°, C°, machining/clearance/C_min status
4. Filter buttons: All, Problems, Used
5. Click any row to view that plane in detail + 3D

### Batch Table Columns

| Column | Meaning |
|--------|---------|
| Name | Plane name |
| A° | Azimuth angle (display convention) |
| C° | Tilt angle |
| Mach | Machining level feasibility (green checkmark / red X) |
| Clr | Clearance level feasibility |
| C_min | Minimum tilt angle for Z limit |

### Adjusted Plane

When a plane is NOT feasible, the calculator auto-generates an adjusted plane with the minimum C angle that fits within the Z limit. The adjusted matrix is reconstructed via Gram-Schmidt orthogonalization. Can be toggled in 3D view and copied to clipboard.

## Settings Persistence

All input values and section collapse states are saved to `localStorage` and restored on page load. File handle is persisted via IndexedDB for the File System Access API.

## Auto-Calculate

All inputs trigger automatic recalculation on change — no manual Calculate button needed. When viewing a single plane from a batch, changes re-sync the full batch.

## Code Architecture

### Major Sections (in order)

1. **CSS styles** — Dark theme, responsive layout, scene labels
2. **HTML structure** — Left panel (inputs, batch, analysis, parsed, adjusted), right panel (3D canvas + view buttons)
3. **Globals** — Three.js scene objects, batch state
4. **Plane parser** — `parsePlaneText()`, `parseMultiplePlanes()`
5. **Core math** — `extractAngles()`, `calcZPivot()`, `calcCMin()`, `reconstructMatrix()`, `aDisplayDeg()`
6. **Main calculation** — `calculate()` — single plane analysis
7. **UI display** — `showParsed()`, `showAnalysis()`, `showAdjusted()`
8. **File loading** — File System Access API with IndexedDB handle persistence
9. **Batch calculation** — `calculateBatch()`, batch table rendering, filtering
10. **Settings persistence** — `saveSettings()`, `loadSettings()` via localStorage
11. **Three.js scene** — `initScene()`, lighting, controls
12. **Scene building** — `buildTable()`, `buildStone()`, `buildToolVector()`, `buildPivotPoint()`, `buildGhostTool()`, `buildReachTool()`, `buildZLimitPlane()`, `buildZLowerLimitPlane()`, `buildMachiningPlane()`, `buildAzimuthRing()`, `buildCArc()`
13. **Label system** — CSS2D-style labels positioned via 3D→screen projection
14. **Update scene** — `updateScene()` — orchestrates all 3D elements from calculation state
15. **Initialization** — Load settings, init scene, restore file handle

### Key Functions

| Function | Purpose |
|----------|---------|
| `parsePlaneText(text)` | Parse single plane block into {name, origin, matrix} |
| `parseMultiplePlanes(text)` | Split file into planes, parse each |
| `extractAngles(normal)` | Get A and C angles from plane normal |
| `calcZPivot(zFeature, toolLen, C)` | Compute CoR Z position |
| `calcCMin(zRef, zLimit, toolLen)` | Compute minimum C angle for Z limit |
| `reconstructMatrix(matrix, newNormal)` | Gram-Schmidt rebuild for adjusted plane |
| `aDisplayDeg(A)` | Convert internal A to display convention (+90°) |
| `machineZToDisplay(mz, fz)` | Convert machine Z to display coords |
| `calculate()` | Full single-plane calculation pipeline |
| `calculateBatch()` | Full batch calculation pipeline |
| `buildToolVector(...)` | Create tool shaft, disk, ring, arrow in 3D |
| `buildPivotPoint(...)` | Create CoR cylinder with wireframe crosshairs |
| `updateScene(skipReframe)` | Rebuild entire 3D scene from current state |

## Development Notes

- Three.js r0.162.0 loaded from CDN (jsdelivr)
- `depthTest: false` on tool elements so they render on top of other geometry
- `depthWrite: false` on transparent elements to avoid z-fighting
- Camera persists position/target across recalculations (only auto-frames on first calc)
- `<script type="module">` — functions exposed to HTML onclick via `window.functionName`
- Scene labels use absolute-positioned divs projected from 3D coordinates each frame
