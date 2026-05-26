# docs/calibration_guide.md — Calibration Guide

**LSCI Platform v2.1 · TEBI Lab · IIT Bombay**

This document covers both calibration systems in the platform: spatial scale
calibration (pixels per millimetre) and galvo mirror calibration (image pixel
coordinates to galvo motor angles). Every step, option, and edge case is
documented here from the source code.

---

## Table of Contents

1. [Two Independent Calibration Systems](#1-two-independent-calibration-systems)
2. [Spatial Scale Calibration (px/mm)](#2-spatial-scale-calibration-pxmm)
   - 2.1 [When to Calibrate Scale](#21-when-to-calibrate-scale)
   - 2.2 [Step-by-Step Procedure](#22-step-by-step-procedure)
   - 2.3 [How the Calculation Works](#23-how-the-calculation-works)
   - 2.4 [Reference Values](#24-reference-values)
   - 2.5 [Limitations](#25-limitations)
3. [Galvo Calibration — Overview](#3-galvo-calibration--overview)
   - 3.1 [What It Does](#31-what-it-does)
   - 3.2 [The Regression Model](#32-the-regression-model)
   - 3.3 [Grid Size Selection](#33-grid-size-selection)
   - 3.4 [Prerequisites Before Any Calibration](#34-prerequisites-before-any-calibration)
4. [Mode 1: Dial-Based Calibration (Pure)](#4-mode-1-dial-based-calibration-pure)
5. [Mode 2: Manual Calibration (Pure)](#5-mode-2-manual-calibration-pure)
   - 5.1 [Opening the Manual Angle Input Window](#51-opening-the-manual-angle-input-window)
   - 5.2 [Filling in the Grid Table](#52-filling-in-the-grid-table)
   - 5.3 [Running Pure Manual Calibration](#53-running-pure-manual-calibration)
6. [Mode 3: Hybrid Calibration (Mixed)](#6-mode-3-hybrid-calibration-mixed)
   - 6.1 [When to Use Hybrid](#61-when-to-use-hybrid)
   - 6.2 [Step-by-Step Hybrid Procedure](#62-step-by-step-hybrid-procedure)
7. [Mode 4: Auto Calibrate (Click-Only)](#7-mode-4-auto-calibrate-click-only)
8. [The Calibration Click Loop — Detail](#8-the-calibration-click-loop--detail)
9. [What Happens After the Last Click](#9-what-happens-after-the-last-click)
10. [ROI and Calibration Interaction](#10-roi-and-calibration-interaction)
11. [Calibration Persistence](#11-calibration-persistence)
12. [Calibration Backup and Restore](#12-calibration-backup-and-restore)
13. [Viewing Calibration Status](#13-viewing-calibration-status)
14. [Home Galvo (Return to Start)](#14-home-galvo-return-to-start)
15. [Validating a Calibration](#15-validating-a-calibration)
16. [Troubleshooting Calibration Issues](#16-troubleshooting-calibration-issues)

---

## 1. Two Independent Calibration Systems

| System | Purpose | Stored in | Persists after restart |
|---|---|---|---|
| Scale calibration | Converts pixels to millimetres | calibration_factor (DoubleVar) | No — re-enter each session |
| Galvo calibration | Maps image pixel → galvo motor angles | galvo_calibration.pkl + .json | Yes — auto-loaded on startup |

These two systems are entirely independent. Scale calibration affects the ruler
overlay and coordinate display only. Galvo calibration controls where the laser
beam points during grid scanning and multi-capture.

---

## 2. Spatial Scale Calibration (px/mm)

### 2.1 When to Calibrate Scale

Recalibrate whenever:
- The camera objective or zoom level is changed
- The camera is repositioned vertically (working distance changed)
- A different lens or tube is installed
- The first time the platform is used with a new optical configuration

The calibration factor is not saved to disk. It must be re-entered at the
start of every session where mm-accurate measurements are needed.

---

### 2.2 Step-by-Step Procedure

**Step 1 — Prepare a reference object.**
Place a physical object of known length in the field of view at the same
focal plane as the specimen. Suitable references include:
- A stage micrometer (most accurate)
- A ruler placed on the specimen stage
- The stereotaxic frame itself if it has calibrated markings
- A machined tube or post of known diameter

**Step 2 — Start the live preview.**
Click Start Preview in the Camera Controls panel. Confirm the reference object
is visible and in focus.

**Step 3 — Open scale calibration.**
In the left sidebar, click Calibrate Scale (or the equivalent button in the
Camera Controls panel). The status bar will show:
```
Calibration: Draw an imaginary line between two 'mm'/scale marks
```
The cursor changes to a crosshair to indicate the tool is active.

**Step 4 — Draw the reference line.**
On the live image canvas:
- Click and hold at one end of the known distance
- Drag to the other end — a green line appears live as you drag
- Release the mouse button

The line can be drawn at any angle; the tool computes Euclidean distance.

**Step 5 — Enter the real-world length.**
A dialog appears: "Enter the real-world distance (mm):"
Type the known length of your reference in millimetres (e.g., 25.0).
Click OK.

**Step 6 — Confirm the result.**
A confirmation dialog shows:
```
Pixel distance: 3720.00 px
Real distance:  25.00 mm
Calibration factor set to: 148.80 px/mm
```
Click OK. The ruler overlay on the live image immediately updates with
the new mm tick marks, and the coordinate display in the status bar
begins showing mm values.

The CalibrationTool automatically deactivates after a successful save.

---

### 2.3 How the Calculation Works

The tool corrects for the display scaling applied to the live canvas before
computing the pixel distance in the original full-resolution image:

```python
# Convert display canvas coordinates to full-resolution image coordinates
pad_left, pad_top, _, _ = self.gui.display_padding
scale = self.gui.display_scale

x1 = (self.start_point[0] - pad_left) / scale    # remove letterbox padding
y1 = (self.start_point[1] - pad_top)  / scale    # divide by canvas-to-image scale
x2 = (self.end_point[0]   - pad_left) / scale
y2 = (self.end_point[1]   - pad_top)  / scale

pixel_distance = sqrt((x2 - x1)**2 + (y2 - y1)**2)
factor = pixel_distance / mm_value    # pixels per millimetre
```

The result is stored in self.gui.calibration_factor (a tk.DoubleVar).

**Effect of ROI:** The ruler drawing function (draw_mm_ruler) and the status bar
coordinate display both apply the calibration factor in display space, not raw
sensor space. When ROI is active, the cropped image is rescaled to fill the same
canvas, so the effective scale changes. Always calibrate with the same ROI state
(active or inactive) that will be used during imaging.

---

### 2.4 Reference Values

These values were measured in the TEBI Lab setup and are provided for quick
verification. Do not use these as substitutes for measuring your own setup:

| Reference object | Measured factor |
|---|---|
| 10 mm tube | 59.7 – 105.6 px/mm (varies with zoom) |
| 25 mm tube | 148.8 px/mm |
| 40 mm tube | 232.7 px/mm |
| 50 mm tube | 295.5 px/mm |

If your measured factor differs significantly from these ranges, check:
- Correct objective/lens is installed
- Camera is at the expected working distance
- You drew the line along the correct axis (not at an unexpected angle)

---

### 2.5 Limitations

- Not persisted to disk — must be re-entered every session
- Only a single scalar factor is stored — assumes isotropic pixels (equal
  horizontal and vertical resolution), which holds for the scTDC camera
- The factor applies uniformly across the entire field of view — for high
  magnification with significant lens distortion, a single factor will be
  less accurate near the edges than at the centre

---

## 3. Galvo Calibration — Overview

### 3.1 What It Does

The galvo calibration establishes a mathematical mapping from image pixel
coordinates (x, y) to galvo motor angles (angle_X, angle_Y). This mapping
is used during multi-capture grid scanning to steer the laser beam to
predetermined positions relative to the imaged area.

Without calibration, the galvo grid scan cannot run. The multi-capture
thread checks coord_selector.matrix is not None before starting.

---

### 3.2 The Regression Model

Two independent linear regression models are fitted simultaneously:

```
Galvo_X = β₀  +  β₁ × Image_X  +  β₂ × Image_Y
Galvo_Y = γ₀  +  γ₁ × Image_X  +  γ₂ × Image_Y
```

Where (Image_X, Image_Y) are pixel coordinates in the full-frame (or
ROI-absolute) image space, and (Galvo_X, Galvo_Y) are the angles sent
to the galvo motors in degrees.

**Why linear regression?**
The galvo mirror system has a linear angular response over the range
used in imaging (-40° to +40°). The mapping from image position to
required angle is also linear over the typical scan area (small-angle
approximation for the imaging geometry). Two parameters per axis (slope
in x, slope in y, and intercept) are sufficient to characterise this
mapping at imaging scale.

**Grid generation from the regression:**
After fitting, the software generates a uniformly spaced grid of target
angles by sampling the fitted models at evenly spaced image coordinate
positions:

```python
x_vals = np.linspace(x_min, x_max, cols)  # cols evenly spaced in image x
y_vals = np.linspace(y_min, y_max, rows)  # rows evenly spaced in image y

for i, y in enumerate(y_vals):          # rows (y) outer loop
    for j, x in enumerate(x_vals):      # cols (x) inner loop
        galvo_matrix[i, j, 0] = model_X.predict([[x, y]])[0]
        galvo_matrix[i, j, 1] = model_Y.predict([[x, y]])[0]
```

The resulting galvo_matrix has shape (rows, cols, 2) and is stored in
coord_selector.matrix.

---

### 3.3 Grid Size Selection

The platform supports rectangular grids from 2×2 to 3×3. The grid size
is set by cal_grid_rows and cal_grid_cols in the calibration controls section
of the Multi-Capture window.

| Grid | Points required | Minimum for regression |
|---|---|---|
| 2×2 | 4 | Yes (4 ≥ 3 required for 2D linear fit) |
| 2×3 | 6 | Yes |
| 3×2 | 6 | Yes |
| 3×3 | 9 | Yes — recommended default |

A 3×3 grid gives 9 calibration points, which over-determines the 3-parameter
linear model (by 6 degrees of freedom) and produces a more robust fit. Use
2×2 only when the imaging field is small and calibration time is constrained.

**Set the grid size before starting any calibration mode.** Changing the grid
size after partial calibration requires starting over (image_points and
galvo_points lists are reset at the beginning of every calibration run).

---

### 3.4 Prerequisites Before Any Calibration

Satisfy all of the following before starting galvo calibration:

```
[ ] Camera initialized (Initialize Camera clicked and confirmed)
[ ] Live preview running (Start Preview clicked, image visible)
[ ] Galvo enabled (Enable Galvo button green in Multi-Capture window)
[ ] Laser is on and beam is visible on the specimen or target surface
[ ] Display scale is stable (no exposure changes during calibration)
[ ] Grid size selected (cal_grid_rows and cal_grid_cols set as desired)
[ ] ROI state decided — calibrate with the same ROI state you will use for imaging
```

---

## 4. Mode 1: Dial-Based Calibration (Pure)

**Use when:** You have no prior knowledge of the angle-to-position mapping
and will determine all angles empirically by positioning the beam with the
dial controls.

**Number of points required:** rows × cols (e.g., 9 for 3×3)

**Step-by-step:**

```
1. Open the Multi-Capture and Galvo Settings window.

2. Set the grid size (e.g., 3×3).

3. Leave the Manual Angle Input table completely empty 
   (do not open it, or open it and leave all fields blank).

4. Click Auto Calibrate in the calibration section.
   The software shows a hybrid calibration info dialog:
     "Manual points pre-set: 0
      Dial-based points: 9
      For EACH point: Adjust dials to position beam, then click."
   Click OK to begin.

5. For Point 1 of 9 (top-left of the grid):
   a. The status bar shows: "Waiting for click 1/9 — Row:1 Col:1"
   b. Turn Galvo Motor 1 (X) dial until the laser beam is at the
      top-left position you want as grid position (1,1).
   c. Turn Galvo Motor 2 (Y) dial until the beam is at the correct 
      vertical position for (1,1).
   d. Click that position in the live image canvas.
   e. A confirmation dialog appears:
        "Confirm galvo angles?
         X: <read from dial>, Y: <read from dial>"
      Click YES to accept this point.
   f. The point is recorded: image coordinates → dial angles.

6. Repeat Step 5 for all remaining points in row-major order:
   (1,1) (1,2) (1,3) → (2,1) (2,2) (2,3) → (3,1) (3,2) (3,3)

7. After the last click, regression runs automatically and the matrix
   is generated and saved. See Section 9 for what happens next.
```

**Tips for dial-based calibration:**
- Move the beam to clearly separated positions across the field — do not
  cluster all points in the centre
- Ideal: corners and edges of the imaging region plus one centre point
- Settle the beam before clicking — wait 1–2 seconds after releasing the
  dial before clicking the beam position
- If you misclick, the confirmation dialog lets you cancel that point
  and redo it

---

## 5. Mode 2: Manual Calibration (Pure)

**Use when:** You have a calibration table from a previous session, a
manufacturer's angle map, or computed angles from the optics geometry.
All angles are known in advance and no physical dial adjustment is needed.

**Number of entries required:** All rows × cols cells filled in the table.

### 5.1 Opening the Manual Angle Input Window

In the calibration section of the Multi-Capture window:

```
Click: "Manual Angles" or "Open Hybrid Input"
→ ManualAngleInput window opens, titled:
  "Hybrid Angle Input (3×3)"
```

The window shows a grid of rows × cols entry fields, one per grid position.

---

### 5.2 Filling in the Grid Table

**Entry format:** `X_angle,Y_angle`
For example: `10.5,-5.2` means Galvo X = 10.5° and Galvo Y = -5.2°.

**Grid positions:**
```
         Col 1      Col 2      Col 3
Row 1   (1,1)      (1,2)      (1,3)
Row 2   (2,1)      (2,2)      (2,3)
Row 3   (3,1)      (3,2)      (3,3)
```

**Controls per cell:**

| Button | Action |
|---|---|
| Set | Reads the entered angle, sends it to the galvo motors, and moves the dials to match |
| X (Clear) | Erases the entry for that cell |
| Set from Current Dials (global) | Prompts for a (row,col) position and fills that cell with the current dial readings |
| Preview Combined | Shows a summary of how many cells are filled vs. empty |

**Example — filling a 3×3 grid:**
```
Row 1: (-12.0,10.0)  |  (0.0,10.0)   |  (12.0,10.0)
Row 2: (-12.0,0.0)   |  (0.0,0.0)    |  (12.0,0.0)
Row 3: (-12.0,-10.0) |  (0.0,-10.0)  |  (12.0,-10.0)
```
This describes a symmetric 3×3 grid spanning ±12° in X and ±10° in Y.

**After filling all cells:**
Click Save Manual Points. The status bar in the window shows:
```
Saved 9 manual point(s) out of 9 total
```
The gui.manual_angle_points dict is populated:
```python
{(0,0): (-12.0, 10.0), (0,1): (0.0, 10.0), ..., (2,2): (12.0, -10.0)}
```
Close the Manual Angle Input window.

---

### 5.3 Running Pure Manual Calibration

```
1. After saving all 9 manual points, click Auto Calibrate.

2. The info dialog shows:
     "Manual points pre-set: 9
      Dial-based points: 0"
   Click OK.

3. For each point in row-major order, the software:
   a. Automatically moves the galvo to the pre-set angle for that point
      (calls send_galvo_motor1_direct and send_galvo_motor2_direct)
   b. Updates the dials to show the angle visually
   c. The status bar shows the current angle being used
   d. Waits for you to click the beam position in the live image

4. Click the beam spot in the live image for each of the 9 points.
   No confirmation dialog appears for manual-angle points (the angle 
   is already confirmed by virtue of being pre-typed).

5. After 9 clicks, regression runs and the matrix is saved.
```

**Advantage over dial-based:** The beam moves to the exact pre-defined
angles automatically, so you only need to observe and click — no physical
dial adjustment is needed mid-sequence. This is faster and more repeatable.

---

## 6. Mode 3: Hybrid Calibration (Mixed)

### 6.1 When to Use Hybrid

Hybrid calibration is useful when:
- You know the angles for some "anchor" points (e.g., corner positions
  measured in a previous session) but need to determine the intermediate
  positions empirically
- Some positions are difficult to reach with the dials alone (dead zones
  or high-friction regions of the dial)
- You want to reuse angles from a previous calibration for boundary points
  and re-measure only the interior

---

### 6.2 Step-by-Step Hybrid Procedure

**Example: 3×3 grid, corners pre-set manually, centre row and column
determined by dials.**

```
1. Open the Manual Angle Input window.

2. Fill ONLY the corner cells (1,1), (1,3), (3,1), (3,3):
     (1,1): -12.0,10.0
     (1,3):  12.0,10.0
     (3,1): -12.0,-10.0
     (3,3):  12.0,-10.0
   Leave all other cells empty.

3. Click Save Manual Points.
   Status shows: "Saved 4 manual point(s) out of 9 total".

4. Click Preview Combined. The preview dialog shows:
     "Manual points: 4 — at: (1,1), (1,3), (3,1), (3,3)
      Dial-based points: 5
      Image clicks required for ALL points"

5. Click Auto Calibrate.
   The info dialog shows:
     "Manual points pre-set: 4
      Dial-based points: 5"
   Click OK.

6. Walk through all 9 points in row-major order:

   Point (1,1) — MANUAL:
     Beam moves automatically to (-12.0, 10.0)
     Status: "Point 1: Manual angle used X:-12.00, Y:10.00"
     Click the beam in the image.

   Point (1,2) — DIAL-BASED:
     Status: "Point 2 of 9 — Adjust dials and click beam position"
     Adjust dials to position beam at middle-top of your scan region
     Click beam position
     Confirmation dialog appears — click YES

   Point (1,3) — MANUAL:
     Beam moves automatically to (12.0, 10.0)
     Click the beam.

   ... continue through all 9 points ...

7. After point 9, regression runs automatically.
```

**Key behaviour:** For manual-angle points, the galvo moves automatically
and no confirmation dialog appears. For dial-based points, the confirmation
dialog always appears so you can verify or cancel a misclick.

---

## 7. Mode 4: Auto Calibrate (Click-Only)

This is effectively the same code path as Modes 1–3. The "Auto Calibrate"
button always calls auto_calibrate_from_clicks(), which checks the
manual_angle_points dict and handles any mix automatically.

The difference between "modes" is only in what you put (or do not put) in
the Manual Angle Input table before clicking Auto Calibrate:

| Table state when you click Auto Calibrate | Effective mode |
|---|---|
| All cells empty | Pure dial-based (Mode 1) |
| All cells filled | Pure manual (Mode 2) |
| Some cells filled, some empty | Hybrid (Mode 3) |

There is no separate button for "Mode 1" vs "Mode 2" — they all use the
same Auto Calibrate button. The mode is determined entirely by the table.

---

## 8. The Calibration Click Loop — Detail

This section documents exactly what happens at each click during calibration,
extracted from the on_click() handler in auto_calibrate_from_clicks().

**Per-click processing:**

```
User clicks live_image_canvas at (event.x, event.y)
    │
    ├─ [1] Check click is within the image content area (not the letterbox padding)
    │       If outside: print "[Calibration] Click outside image area, ignored."
    │       Wait for a valid click inside the image.
    │
    ├─ [2] Convert display coordinates → full-resolution image coordinates
    │       display_x = int((event.x - pad_left) / scale)
    │       display_y = int((event.y - pad_top)  / scale)
    │
    ├─ [3] Apply ROI offset if ROI is active
    │       if roi_active:
    │           original_x = display_x + roi_x1
    │           original_y = display_y + roi_y1
    │       else:
    │           original_x = display_x
    │           original_y = display_y
    │       image_points.append((original_x, original_y))
    │
    ├─ [4] Determine angle source for this grid position (row, col)
    │       if (row, col) in calibration_manual_points:
    │           → use pre-set angle: (angle_x, angle_y)
    │           → update dials to show the angle
    │           → no confirmation dialog
    │       else:
    │           → read current dial values
    │           → show confirmation dialog: "Confirm galvo angles? X:... Y:..."
    │           → if user clicks NO: pop the image_point, return (wait for redo)
    │
    ├─ [5] Record the angle pair
    │       galvo_points.append((angle_x, angle_y))
    │
    ├─ [6] Increment calibration_index
    │
    ├─ [7] If more points remain:
    │       if next point is manual:
    │           move galvo automatically: send_galvo_motor1/2_direct()
    │           update status: "Next point is manual (row+1, col+1): X:... Y:..."
    │       else:
    │           update status: "Adjust dials for point N/total, then click"
    │
    └─ [8] If this was the last point:
            Unbind click handler from canvas
            Call generate_dynamic_calibration()
```

**Coordinate precision note:** All image_points are stored in the coordinate
space of the full 1920×1440 frame, even when ROI is active. The ROI offsets
(roi_x1, roi_y1) are added in step 3. This ensures the regression model
operates in a consistent, camera-absolute coordinate space regardless of
whether ROI is active during calibration.

---

## 9. What Happens After the Last Click

After the final point is collected, the calibration pipeline runs automatically:

```
generate_dynamic_calibration() is called
    │
    ├─ [1] Validate point counts
    │       len(image_points) == len(galvo_points) == rows × cols
    │       If mismatch: error dialog, abort
    │
    ├─ [2] Build regression input matrix
    │       img_A, img_B  = zip(*image_points)   # image x, image y
    │       galvo_C, galvo_D = zip(*galvo_points) # galvo x-angles, y-angles
    │       image_coords = np.column_stack((img_A, img_B))  # shape (N, 2)
    │
    ├─ [3] Fit two linear models
    │       model_X = LinearRegression().fit(image_coords, galvo_C)
    │       model_Y = LinearRegression().fit(image_coords, galvo_D)
    │
    ├─ [4] Determine grid boundary in image space
    │       if roi_active:
    │           x_min, x_max = roi_x1, roi_x2
    │           y_min, y_max = roi_y1, roi_y2
    │       else:
    │           x_min, x_max = min(img_A), max(img_A)
    │           y_min, y_max = min(img_B), max(img_B)
    │
    ├─ [5] Generate the evenly spaced galvo angle grid
    │       x_vals = np.linspace(x_min, x_max, cols)
    │       y_vals = np.linspace(y_min, y_max, rows)
    │       for each (i,j): galvo_matrix[i,j] = (model_X.predict, model_Y.predict)
    │
    ├─ [6] Load matrix into coord_selector
    │       coord_selector.load_matrix(galvo_matrix)
    │       coord_selector.matrix_type = "regression"
    │       coord_selector.grid_rows = rows
    │       coord_selector.grid_cols = cols
    │
    ├─ [7] Auto-save calibration to disk
    │       coord_selector.save_calibration_to_file()
    │       → saves galvo_calibration.pkl (pickle, binary)
    │       → saves galvo_calibration_config.json (JSON, human-readable)
    │
    └─ [8] Show completion dialogs
            Status bar: "Calibration completed successfully ✅ (3×3)"
            MessageBox 1: "Galvo calibration completed and 3×3 matrix loaded."
            MessageBox 2: "Galvo calibration completed and saved to file.
                           Grid: 3×3
                           Calibration will persist after restart."
```

The console prints the full matrix for inspection:
```
[Auto Calibration] Galvo matrix (3x3):
 [[[-12.34  9.87] [ -0.12  9.91] [ 12.10  9.89]]
  [[-12.38  0.03] [ -0.08 -0.01] [ 12.06  0.00]]
  [[-12.41 -9.81] [ -0.09 -9.93] [ 12.03 -9.86]]]
```

---

## 10. ROI and Calibration Interaction

The ROI system and galvo calibration interact in two important ways:

**During calibration:**
- If ROI is active when you click during calibration, the ROI offsets
  (roi_x1, roi_y1) are added to each click coordinate before storing in
  image_points. This converts ROI-relative pixel positions to full-frame
  absolute pixel positions.
- The regression therefore operates in full-frame pixel space regardless
  of whether ROI was active.

**During scanning (multi-capture):**
- At the start of each multi-capture session,
  coord_selector.update_roi_params() is called. This stores the current
  ROI boundaries into the coord_selector:
  ```python
  roi_offset_x = roi_x1
  roi_offset_y = roi_y1
  roi_width  = roi_x2 - roi_x1
  roi_height = roi_y2 - roi_y1
  ```
- These ROI params are saved alongside the galvo matrix in the
  calibration file.

**Recommendation:** Always calibrate with the same ROI state (active or
inactive) that you will use during multi-capture imaging. Calibrating
without ROI and then activating ROI before scanning is acceptable, but
calibrating with one ROI and scanning with a different ROI will shift the
pixel-to-angle mapping and produce misaligned grid points.

---

## 11. Calibration Persistence

Galvo calibration is automatically saved after every successful calibration
run and automatically loaded on every application startup.

**Save location:** The project root directory (same folder as new_GUI_sc_55.py)

**Files written:**

| File | Format | Purpose |
|---|---|---|
| galvo_calibration.pkl | Python pickle (binary) | Primary store — loaded first on startup |
| galvo_calibration_config.json | JSON (text) | Fallback store — human-readable |

**Calibration data structure saved:**

```python
{
    'galvo_matrix':    np.ndarray (rows, cols, 2),  # the angle grid
    'grid_rows':       int,                          # e.g. 3
    'grid_cols':       int,                          # e.g. 3
    'matrix_type':     str,                          # "regression"
    'roi_offset_x':    int,                          # ROI x1 at time of calibration
    'roi_offset_y':    int,                          # ROI y1 at time of calibration
    'roi_width':       int,                          # ROI width at time of calibration
    'roi_height':      int,                          # ROI height at time of calibration
    'timestamp':       str,                          # ISO 8601 datetime
    'camera_resolution': (1920, 1440)                # fixed for this camera
}
```

**Load order on startup:**

```
CoordinateSelector.__init__()
    → config_manager.load_binary_calibration()   # try .pkl first
    → if None: config_manager.load_calibration() # fallback to .json
    → if calibration found: populate matrix, grid_rows, grid_cols, matrix_type
    → print: "[Calibration] Loaded 3x3 regression calibration"
    → root.after(1000, show_calibration_load_status)
      → status bar: "Loaded 3×3 calibration"
```

If neither file exists on startup:
```
print: "[Calibration] No saved calibration found"
```
The galvo controls work but multi-capture with galvo enabled will show
an error: "Galvo is enabled but no calibration matrix is loaded."

---

## 12. Calibration Backup and Restore

The Calibration Management window (opened via the Calibration section of
the GUI) provides named backup and restore functionality.

### Creating a Backup

```
1. Open Calibration Management window.
2. Confirm the status display shows a loaded calibration 
   (e.g., "✓ CALIBRATION LOADED — Grid: 3 × 3").
3. Click "💾 Backup Current".
4. A dialog asks for a backup name.
   Default: calibration_backup_YYYYMMDD_HHMMSS
   You can type any name, e.g.: "session_20260115_3x3_forepaw"
5. Click OK.
6. Two files are created in the project root directory:
     session_20260115_3x3_forepaw.pkl
     session_20260115_3x3_forepaw.json
7. The status display refreshes and shows the backup count increased.
```

**The backup preserves:**
- The full galvo angle matrix (all rows×cols×2 values)
- Grid dimensions, matrix type, ROI parameters, timestamp
- Camera resolution (1920×1440)

---

### Restoring a Backup

```
1. Open Calibration Management window.
2. Click "📂 Restore Backup".
3. A selection dialog lists all backup names (files that are not 
   galvo_calibration.pkl/.json).
4. Click the name of the backup to restore.
5. Click "Restore Selected".
6. The .pkl file is loaded (preferred over .json if both exist).
7. coord_selector.matrix is updated with the restored values.
8. The main calibration files (galvo_calibration.pkl/.json) are 
   overwritten with the restored data.
9. Status bar: "Calibration restored (3×3)"
```

**Note:** Restoring a backup immediately replaces the active calibration
in memory and on disk. The previously active calibration is overwritten.
If you want to keep both, create a backup of the current calibration
before restoring an older one.

---

## 13. Viewing Calibration Status

The Calibration Management window shows a status panel updated in real time:

```
==================================================
✓ CALIBRATION LOADED
   Grid: 3 × 3
   Type: regression
   Shape: (3, 3, 2)

File Status:
  • JSON file: ✓ Found
  • Binary file: ✓ Found
  • Backup files: 2 found

ROI Settings (if applicable):
  • Active: True
  • Region: (240,180) to (1680,1260)
==================================================
```

Click "📊 Refresh Status" at any time to update this display with the
current in-memory calibration and file system state.

The startup status is also shown in the main window status bar,
approximately 1 second after launch:
- "Loaded 3×3 calibration" — if auto-load succeeded
- "No calibration — run Auto Calibrate" — if no file found

---

## 14. Home Galvo (Return to Start)

The Home Galvo function (Galvo Home button) moves both galvo motors to
the angle stored at matrix[0,0] — the top-left position of the calibrated
grid.

```python
angle_x, angle_y = coord_selector.matrix[0, 0]
send_galvo_motor1_direct(angle_x)   # X motor
time.sleep(2.0)                     # 2s settle
send_galvo_motor2_direct(angle_y)   # Y motor
time.sleep(2.0)                     # 2s settle
```

If no calibration matrix is loaded, the fallback is (0.0°, 0.0°).

**Use cases:**
- Verify beam is near the expected starting position before a scan
- Return beam to a known position after manual dial adjustment
- Quick visual confirmation that the calibration is approximately correct
  (if the beam appears at the top-left of your intended scan area, the
  calibration is likely valid)

**Prerequisites:** Galvo must be enabled. If galvo is disabled, the home
function shows: "Enable Galvo first to move to starting point."

---

## 15. Validating a Calibration

After calibration, perform these checks before relying on it for data collection:

**Check 1 — Console matrix printout.**
Inspect the matrix printed to the console. For a symmetric scan area:
- Corner values should have the largest absolute angles
- The centre value should be near (0°, 0°) or the mechanical centre
- Values should change monotonically across rows and columns

**Check 2 — Home galvo and observe.**
Click Galvo Home. The beam should appear at the top-left of your intended
scan region. If it appears elsewhere, the calibration click sequence was
likely performed in the wrong order.

**Check 3 — Send individual angles.**
Use the Motor X and Motor Y dials to manually send angles from the matrix
to specific grid positions. Observe the beam: it should move to the expected
image position.

**Check 4 — Run a test multi-capture with 1 frame.**
Set frames_count = 1 and trigger a single-exposure multi-capture. Watch the
beam move across each grid position in sequence. If the positions are in the
wrong order or at the wrong locations, recalibrate.

**Recalibration triggers:**
- Any change to the optical path (lens, filter, mirror angle)
- Camera physically repositioned
- Galvo controller powered off and back on (some controllers reset their
  zero-angle reference on power cycle)
- The beam is observed to be consistently offset from expected positions
  by more than ~5% of the field width

---

## 16. Troubleshooting Calibration Issues

### Error: "Need exactly N points for a N×N calibration grid"

The number of collected image_points or galvo_points does not match
rows × cols. This happens if:
- The calibration was started with a different grid size than the one set
  in the GUI — set the grid size before clicking Auto Calibrate
- A click was registered outside the image boundary and silently ignored —
  the boundary check prints "[Calibration] Click outside image area, ignored."
  in the console; look for this and ensure all clicks land inside the image

---

### Error: "Mismatch between image points and galvo points"

This should not occur in normal operation but can happen if the on_click
handler is called while the calibration index is inconsistent. Restart
the calibration from the beginning (click Auto Calibrate again — it resets
image_points and galvo_points to empty lists at the start).

---

### Calibration does not auto-load on startup

Both galvo_calibration.pkl and galvo_calibration_config.json are missing.
This occurs after:
- First install (no calibration ever performed)
- Files manually deleted
- Application run from a different working directory

Run a new calibration. After successful completion, the files will be
created and will auto-load on all subsequent startups.

---

### Beam does not move when clicking Set in ManualAngleInput

Galvo is not enabled. Click Enable Galvo in the Multi-Capture window first.
The Set button calls send_galvo_motor1/2_direct() which checks galvo_enabled
and silently skips if it is False.

---

### Calibration completes but beam positions during scan are wrong

Most likely causes:
1. The calibration was performed with ROI active but scanning is done with
   different ROI boundaries (or vice versa)
2. The click points were not at the laser beam position — e.g., clicking
   a reflection or a different feature than the beam spot
3. Grid was clicked in the wrong order — must follow row-major order
   (1,1) → (1,2) → (1,3) → (2,1) ... → (3,3)

Recalibrate, paying careful attention to clicking the actual beam centroid
and following the sequence indicated in the status bar.

---

### Scale calibration result seems wrong

If the px/mm factor is much larger or smaller than expected:
1. Confirm you entered the distance in millimetres, not centimetres
2. Confirm the line was drawn to the full length of the reference feature,
   not a partial length
3. Check that the live preview was running and stable when you drew the
   line (a frozen frame from a previous exposure setting may have a
   different effective scale than the current live feed)

---

*Document version: January 2026 - Platform v2.1*
