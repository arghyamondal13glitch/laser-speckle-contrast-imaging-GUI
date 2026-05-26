# docs/user_guide.md — User Guide

**LSCI Platform v2.1 · TEBI Lab · IIT Bombay**

This guide is a complete walkthrough of every panel, window, button, and
workflow in the platform. It is written for daily lab use. Calibration
procedures are in `calibration_guide.md`; hardware wiring is in
`hardware_setup.md`.

---

## Table of Contents

1. [Main Window Layout](#1-main-window-layout)
2. [Header Bar](#2-header-bar)
3. [Quick Actions Panel](#3-quick-actions-panel)
4. [Live Preview Panel](#4-live-preview-panel)
5. [Speckle Contrast View Panel](#5-speckle-contrast-view-panel)
6. [Speckle Contrast Video Controls](#6-speckle-contrast-video-controls)
7. [Intensity Profile Tools](#7-intensity-profile-tools)
8. [Camera Feed Toggle](#8-camera-feed-toggle)
9. [Status Bar](#9-status-bar)
10. [Camera Controls Window](#10-camera-controls-window)
    - 10.1 Exposure Time
    - 10.2 Calibration Factor
    - 10.3 Z-axis Focus Control
    - 10.4 ROI Settings
11. [Multi-Capture & Galvo Window](#11-multi-capture--galvo-window)
    - 11.1 Galvo Motor Controls
    - 11.2 Calibration Section
    - 11.3 Multi-Capture Configuration
    - 11.4 Running a Multi-Capture Session
    - 11.5 Pause, Resume, and Abort
12. [Stereotaxic Stage Control Window](#12-stereotaxic-stage-control-window)
13. [Relay Control Window](#13-relay-control-window)
14. [NiDAQ Forepaw Stimulator](#14-nidaq-forepaw-stimulator)
15. [Calibration Management Window](#15-calibration-management-window)
16. [Intensity Profile Output Window](#16-intensity-profile-output-window)
17. [Companion Tools](#17-companion-tools)
18. [Standard Session Workflow](#18-standard-session-workflow)
19. [Keyboard and Mouse Reference](#19-keyboard-and-mouse-reference)
20. [Troubleshooting During a Session](#20-troubleshooting-during-a-session)

---

## 1. Main Window Layout

The main window occupies 95% of the screen and is divided into four zones:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  HEADER BAR                                                              │
│  [Platform Title]  [Laser: 450.00 mA]  [Calibrate Scale] [NiDAQ]       │
│  [Relay] [Help] [Specs] [About] [Exit GUI]                              │
├─────────────────────────┬───────────────────────────────────────────────┤
│  QUICK ACTIONS          │  DISPLAY AREA                                 │
│  (left sidebar,         │  ┌──────────────────┐  ┌───────────────────┐ │
│   scrollable)           │  │  Live Preview     │  │ Speckle Contrast  │ │
│                         │  │  [contrast] [img] │  │ [C-axis] [img]    │ │
│  Initialize Camera      │  │                  │  │ [colorbar]        │ │
│  Open Camera Controls   │  └──────────────────┘  └───────────────────┘ │
│  Start Preview          │                                               │
│  Stop Preview           │                                               │
│  Stereotaxic Platform   │                                               │
│  Multi-Capture & Galvo  │                                               │
│  Spatial-Temporal       │                                               │
│  Launch Speckle Tracker │                                               │
│  ─────────────────────  │                                               │
│  Speckle Video Controls │                                               │
│  Intensity Profile      │                                               │
│  Image Capture          │                                               │
│  Feed Toggle            │                                               │
├─────────────────────────┴───────────────────────────────────────────────┤
│  STATUS BAR — coordinates | operation state | error messages            │
└─────────────────────────────────────────────────────────────────────────┘
```

The left sidebar is scrollable — scroll down to see controls below the
visible area if the window is shorter than all the controls.

---

## 2. Header Bar

The header bar (dark blue, #003366) is always visible across the top.

### Platform title
"Speckle Contrast Imaging Platform" — static label on the left.

### Laser Input Current display
Shows the live drive current of the imaging laser in mA, updated every
1 second from the laser Arduino serial monitor. Yellow text on blue.

| Display | Meaning |
|---|---|
| `450.00 mA` | Laser running at 450 mA |
| `--- mA` | Laser serial port not connected or not responding |
| `0.00 mA` | Laser at zero current (powered but beam off) |

### Header bar buttons (right side, left to right)

| Button | Opens / Does | Notes |
|---|---|---|
| Calibrate Scale | Activates the ruler calibration tool on the live canvas | Draws a green line; see calibration_guide.md Section 2 |
| NiDAQ Controller | Opens the Forepaw Stimulator window | See Section 14 |
| Relay | Opens the 4-channel relay control window | See Section 13 |
| Help | Opens a brief in-app help dialog | Summarises key controls |
| Specs | Opens the platform specifications dialog | Lists camera, laser, and optical specs |
| About | Opens the About dialog | Lab, developer, version info, logo |
| Exit GUI | Closes the application with confirmation | Stops camera process, serial threads, and Tk loop |

---

## 3. Quick Actions Panel

The left sidebar contains all primary workflow controls, arranged in order
of typical use from top to bottom.

### 3.1 Initialize Camera
Sends `Commands.INITIALIZE_CAM` to the camera process. A dialog appears:
"Initializing camera. Please wait..." 

After initialization completes, the camera process sends back
`("INFO", "Camera initialized...")` and the status bar is updated.
Start Preview and Start Speckle Video buttons unlock after this step.

**Only click once per session.** If the camera is already initialized, a
dialog reminds you: "Camera already initialized."

---

### 3.2 Open Camera Controls
Opens the Camera Controls window (500×650 px Toplevel) with:
- Exposure time entry
- Calibration factor entry
- Z-axis up/down buttons
- ROI coordinate entry and enable toggle

See Section 10 for a full walkthrough of this window.

---

### 3.3 Start Preview
Sends `(Commands.START_PREVIEW, exposure_us)` to the camera process.
The camera begins delivering frames. The frame listener thread (T1) picks
them up and renders them on the live canvas.

- Button state: greyed out while previewing
- Stop Preview becomes available
- Default exposure: 1500 µs

**Must be clicked after Initialize Camera.** If clicked before initialization,
nothing happens (the button is disabled until initialization completes).

---

### 3.4 Stop Preview
Sends `Commands.STOP_PREVIEW`. Stops the live frame loop in the camera
process. Also stops the speckle video if it is running.

After stopping preview:
- The live canvas shows the last received frame (frozen)
- Start Preview re-enables

---

### 3.5 Stereotaxic Platform
Opens the XYZ stage control window. See Section 12.

---

### 3.6 Multi-Capture & Galvo
Opens the main data acquisition window, which contains:
- Galvo motor dial controls
- Galvo calibration buttons
- Multi-capture configuration and run controls

See Section 11 for the full walkthrough.

---

### 3.7 Spatial-Temporal Contrast
Launches `ST_GUI_3.py` as a separate process via `subprocess.Popen`.
This is a standalone post-processing tool for spatiotemporal speckle
contrast analysis of previously saved data. It does not require the
camera to be initialized. The script must be present in the project
root directory.

---

### 3.8 Launch Speckle Tracker
Launches `live_contrast_GUI_2.py` as a separate process.

**Important startup order:** This tool must be launched BEFORE clicking
Initialize Camera in the main GUI. Both tools share the camera hardware
and cannot both hold an active camera connection simultaneously.

A reminder dialog appears when you click this button:
> "Starting Live Speckle Tracker needs to be started before initialization
>  with the Main GUI. Or else Restart the GUI and then directly Launch
>  Live Speckle Tracker."

---

## 4. Live Preview Panel

The left display panel shows the raw 12-bit camera output.

### Live Contrast slider
A vertical slider on the left edge of the live panel, range 0.000–1.000.

This slider adjusts the display brightness of the live image without
changing the actual exposure or the raw data. The calculation applied
to each pixel value is:

```
adjusted = 0.5 + (normalized - 0.5) × contrast_factor
display_img = clip(adjusted × 255, 0, 255).astype(uint8)
```

At 1.000 (default): full dynamic range is displayed.
At lower values: contrast is compressed toward mid-grey.

**The raw frame data is never modified** — only the display copy is
affected. Saves, intensity profiles, and speckle computation all use
the unmodified raw data regardless of slider position.

### Live image canvas
Displays the incoming 12-bit frame rescaled to fit the available panel
width (target 900×670 px) while preserving the 4:3 aspect ratio of
the 1920×1440 sensor. Any remaining area is black letterboxing.

**Overlaid elements:**
- Horizontal and vertical mm scale rulers (when calibration_factor > 0)
- ROI dimensions in px and mm (when ROI is active)

**Mouse events on the live canvas:**
- Moving the mouse: updates coordinate display in Camera Controls window
  and in the status bar
- Click and drag (when Calibrate Scale is active): draws a green ruler line
- Click and drag (when Intensity Profile is active): draws a red profile line
- Click (during galvo calibration): registers a calibration point

### Coordinate display
When the mouse is over the live canvas (and Camera Controls is open):
- Without ROI: `Image Coordinates → X: 847, Y: 612`
- With ROI active: `ROI Coordinates → X: 607, Y: 432 | Original → X: 847, Y: 612`

When calibration_factor > 0, the status bar also shows the mm equivalent.

---

## 5. Speckle Contrast View Panel

The right display panel shows the real-time speckle contrast map.

### C-Axis slider (1/K²)
A vertical slider on the left edge of the speckle panel, labelled
"C-Axis (1/k²)", range 0.000–1.000.

This scales the computed 1/K² map before display:
```
display = (1.0 / K²) × contrast_factor
```
Higher values = brighter display = more dynamic range visible.

At 0.000 the display is completely black. Start at 1.000 and reduce
only if the image is overexposed (saturated to yellow-green).

### Speckle image canvas
Displays the 1/K² map rendered through the viridis colormap.

**Colour interpretation (viridis):**
- Dark purple / blue: low 1/K², meaning high K², meaning low flow speed
- Yellow / green: high 1/K², meaning low K², meaning high flow speed
- Regions with moving red blood cells appear brighter

The ruler overlay is also applied to the speckle canvas using the same
calibration factor as the live panel.

### Vertical colorbar
A 60 px wide gradient strip to the right of the speckle image, showing
the viridis colour scale from 0.000 (bottom) to 1.000 (top). Ten tick
labels are rendered along the side. The colorbar updates whenever the
speckle video frame is refreshed.

---

## 6. Speckle Contrast Video Controls

Located in the "Speckle Contrast Video" LabelFrame in the Quick Actions panel.

### No. of Frames for Speckle Video
Entry field bound to `fps` (IntVar, default 10).

This is the rolling buffer depth — the number of most recent live frames
stacked together to compute the speckle contrast map. Despite being labelled
with fps, it is actually a frame count.

| Value | Effect |
|---|---|
| 10 | Fast response, noisier contrast map |
| 50 | Smoother map, slower to respond to flow changes |
| 100 | Very smooth map, suitable for slow flows |

Click **Enter** after changing this value. If the speckle video is already
running, it stops and restarts with the new depth automatically.

### Start Speckle Video
Starts the speckle video thread (T3). Requires Start Preview to be running
first. The button disables itself while speckle video is active.

### Stop Speckle Video
Stops the speckle video thread. The last rendered frame remains frozen
on the speckle canvas.

---

## 7. Intensity Profile Tools

Located in the "Live/Speckle Intensity Profile and Image Capture" LabelFrame.

### Live Intensity Profile (toggle)
**Click once:** Activates line-draw mode on the live canvas. Status bar:
"Intensity Profile: Click and drag to draw an imaginary line on the live image"

**Draw a line:** Click and hold at the start, drag to the end, release.
A red line appears on the canvas. When you release, the profile is computed
from `latest_live_frame` and the Intensity Profile Output window opens.
See Section 16 for the output window details.

**Click again (same button):** Deactivates the tool and removes the line.

### Full Live Intensity
Computes and displays the full-frame pixel intensity histogram and statistical
summary of the current `latest_live_frame`. Does not require drawing a line —
it analyses the entire image (or ROI if active).

### Speckle Intensity Profile (toggle)
Same as Live Intensity Profile but operates on `latest_speckle_frame`
(the 8-bit normalized display version of the 1/K² map). Line drawn on
the speckle canvas in red.

### Full Speckle Intensity
Full-frame intensity analysis of the current speckle frame.

### Save Live Image
Saves `latest_live_frame` (raw uint16) to the current save directory:
- `live_YYYYMMDD_HHMMSS.png` — 16-bit PNG
- `live_YYYYMMDD_HHMMSS_8bit.png` — 8-bit preview
- `live_YYYYMMDD_HHMMSS.mat` — MATLAB struct

If no save directory is set, a dialog prompts you to choose one.

### Save Speckle Image
Saves the current speckle contrast frame in multiple formats:
- `speckle_YYYYMMDD_HHMMSS.png` — 16-bit PNG (uint16 normalised)
- `speckle_YYYYMMDD_HHMMSS.mat` — MATLAB struct with `speckle_contrast` field
- Viridis-coloured preview PNG via `plt.imsave`

---

## 8. Camera Feed Toggle

Located in the "Camera Feed Toggle" LabelFrame at the bottom of Quick Actions.

Two checkboxes control which panels are shown in the display area:

| Checkboxes checked | Display |
|---|---|
| Live Feed only | Only the live canvas is shown, expanded to full width |
| Speckle Feed only | Only the speckle canvas is shown, expanded to full width |
| Both checked | Both canvases side by side (default and recommended) |
| Neither checked | Display area is blank |

Toggling these checkboxes calls `update_display()` which repacks the
display widgets. The ruler overlay refreshes 100 ms after the toggle.

---

## 9. Status Bar

The status bar runs along the bottom of the main window (sunken relief,
left-aligned text). It is updated by many operations across the application:

| Shown text | Trigger |
|---|---|
| `Ready` | Default state, mouse left canvas, most operations complete |
| `X: 847, Y: 612` | Mouse moving over live canvas (no ROI) |
| `ROI Coordinates → X: 607, Y: 432 | Original → X: 847, Y: 612` | Mouse over canvas with ROI active |
| `Calibration: Draw an imaginary line...` | Scale calibration tool active |
| `Intensity Profile: Click and drag...` | Intensity tool active |
| `Moving to point 3/9 — Row:2 Col:1 — X:0.12 Y:-5.03` | Galvo grid scan in progress |
| `Grid scan completed ✅` | Galvo scan finished |
| `Loaded 3×3 calibration` | Calibration auto-loaded at startup |
| `No calibration — run Auto Calibrate` | No saved calibration found |
| `Invalid ROI: X2 must be greater than X1` | ROI validation error (clears after 3 s) |
| `Galvo calibration completed ✅ (3×3)` | Calibration run completed |

Error states appear in red; warnings in orange; success states in green.
Status bar text automatically resets to "Ready" after most operations.

---

## 10. Camera Controls Window

Opened by: Quick Actions → Open Camera Controls (500×650 px)

### 10.1 Exposure Time

**Field:** "Exposure Time (us):" — entry bound to `exposure_time` (IntVar,
default 1500).

**Workflow:**
1. Type a new value in the entry field
2. Click **Enter**

This calls `update_exposure()`, which:
- Sends `STOP_PREVIEW` to the camera process
- Waits 100 ms
- Sends `(START_PREVIEW, new_exposure_us)`

The live feed pauses for ~100–200 ms and resumes at the new exposure.

**Typical exposure values:**

| Exposure (µs) | Use case |
|---|---|
| 500 | Fast flow measurement (arteries) |
| 1000 | Moderate flow |
| 1500 | Default — general cortical imaging |
| 2000–3000 | Slow flow / high speckle contrast regions |
| 5000+ | Very slow flow or dark samples |

For multi-capture experiments, the exposure is set automatically by
the multi-capture thread from the exposure_times list.

---

### 10.2 Calibration Factor

**Field:** "Calibration Factor (pixels/mm):" — entry bound to
`calibration_factor` (DoubleVar, default 0.0).

You can type a known value directly into this field (e.g., 148.80) and
click **Save** — this is equivalent to running the CalibrationTool and
is useful when the factor is known from a previous session.

Click **ℹ️** to open the reference values dialog showing the lab's
measured calibration factors for different tube sizes.

After saving, the mm ruler overlay on both canvases updates immediately.

---

### 10.3 Z-axis Focus Control

Two buttons in the Z-axis section:

| Button | Command | Modbus call |
|---|---|---|
| Move Up | `move_zaxis_up()` | `write_registers(address=6, values=[41], slave=2)` |
| Move Down | `move_zaxis_down()` | `write_registers(address=6, values=[57], slave=2)` |

Each click sends one Modbus command and moves the camera focus stage by
one step. Click and hold is not supported — click repeatedly for larger
focus adjustments.

The Z-axis COM port and slave ID labels in this section show:
"Z-axis COM: COM4 (fixed)" and "Z-Axis Slave ID: 2 (fixed)". These refer
to the shared Modbus bus (Adapter A), not a dedicated port.

---

### 10.4 ROI Settings

ROI (Region of Interest) crops the sensor output to a rectangular sub-region.
When active, the crop is applied in the display, in speckle computation,
and in all saves.

**Coordinate system:**

```
(0,0)──────────────────►X (0–1920)
 │
 │    (x1,y1)────────────┐
 │       │    ROI region │
 │       └────────────(x2,y2)
 │
 ▼ Y (0–1440)
```

**Fields:** X1 (0–1920), Y1 (0–1440), X2 (0–1920), Y2 (0–1440)

**Constraints enforced on Enable:**
- X2 must be > X1 (otherwise: "Invalid ROI" error dialog, ROI reverts to inactive)
- Y2 must be > Y1 (same)
- Values outside 0–1920 (X) or 0–1440 (Y) show a warning but are accepted

**Minimum ROI size:** 10×10 pixels (enforced in `apply_roi()`)

**Workflow:**
1. Type X1, Y1, X2, Y2 coordinate values
2. Check **Enable ROI**
3. ROI status label turns green: `ROI: ACTIVE (240,180)-(1680,1260)`
4. Live preview and speckle view both switch to the cropped region
5. Ruler ticks rescale to the cropped dimensions

**Reset ROI:** Click **Reset ROI** to restore full-frame dimensions
(0, 0, 1920, 1440) and deactivate ROI.

**Coordinate display with ROI active:**
When hovering over the live canvas with ROI on, both the ROI-relative
coordinates and the original full-frame coordinates are shown:
`ROI Coordinates → X: 607, Y: 432 | Original → X: 847, Y: 612`

**Close behaviour:** Closing the Camera Controls window unbinds the mouse
motion events from the live canvas and deactivates any active intensity
tool. The ROI state (active/inactive and coordinates) is preserved.

---

## 11. Multi-Capture & Galvo Window

Opened by: Quick Actions → Multi-Capture & Galvo

This is the primary data acquisition window. It contains four main sections:
Galvo Motor Controls, Calibration, Multi-Capture Configuration, and
Run Controls.

### 11.1 Galvo Motor Controls

**Enable Galvo / Disable Galvo toggle button**
Must be clicked before any galvo commands will execute. When enabled,
the button shows "Disable Galvo" (and the internal `galvo_enabled = True`).
All galvo functions silently skip if `galvo_enabled = False`.

**Motor X dial (Galvo Motor 1 / X-axis)**
tkdial.Dial widget, range -40.0° to +40.0°, step 0.1°.
Turning the dial sends the angle to Motor 1 (X-axis) via Modbus.

**Motor Y dial (Galvo Motor 2 / Y-axis)**
tkdial.Dial widget, same range. Controls the Y-axis mirror.

**Motor X status / Motor Y status labels**
Update to show the last successfully sent angle:
`Motor X: 12.34°` / `Motor Y: -5.67°`

**Galvo Home button**
Moves both motors to the matrix[0,0] position from the calibration grid
(top-left of scan area). Fallback: (0.0°, 0.0°) if no calibration loaded.

---

### 11.2 Calibration Section

**Grid size selectors**
Two dropdown menus (or spinboxes) for rows and columns:
- Rows: 2 or 3
- Columns: 2 or 3
Together these define the grid size for both calibration and multi-capture.

**Manual Angles button**
Opens the ManualAngleInput dialog (see calibration_guide.md Section 5).
Use this before running Auto Calibrate if you want to pre-specify some or
all galvo angles.

**Auto Calibrate button**
Starts the click-based calibration sequence. See calibration_guide.md
Sections 4–9 for the full procedure.

**Calibration Management button**
Opens the CalibrationManager window for backup and restore. See Section 15.

**Calibration status display**
A small text area showing the currently loaded calibration:
```
Grid: 3×3  |  Type: regression  |  Status: Loaded ✓
```

---

### 11.3 Multi-Capture Configuration

**Exposure Times field**
Comma-separated list of exposure values in microseconds.
Default: `500,1000,1500,2000,2500,3000`

Example: `1000,2000` captures at two exposures per grid point.
All values must be positive integers separated by commas.

**Number of Frames field**
IntVar, default 100. The number of frames stacked per capture to compute
the speckle contrast. Higher values produce smoother K² maps but take longer.

**Data Type checkboxes**
Three independent checkboxes:
- `[ ] Dark` — captures with no illumination
- `[ ] Baseline` — captures before any stimulation
- `[ ] Stimulation` — captures during stimulation

Check all three for a complete experimental run. Each selected type
creates its own subdirectory in the save folder.

**Enable Stimulator checkbox**
When checked, the NiDAQ stimulation pattern runs in parallel with the
stimulation-type captures. The stimulation pattern uses the parameters
last set in the NiDAQ Controller window. If unchecked, stimulation captures
are taken without delivering electrical stimulation (useful for sham controls).

**Save Directory**
Click **Browse** to select the output directory. All captures from this
session are saved under a timestamped subdirectory inside the chosen folder.

---

### 11.4 Running a Multi-Capture Session

**Pre-run checklist:**
```
[ ] Camera initialized and preview running
[ ] Galvo enabled
[ ] Calibration loaded (status shows "Loaded ✓")
[ ] Save directory selected
[ ] Exposure times entered (comma-separated)
[ ] Number of frames set
[ ] At least one data type checked
[ ] NiDAQ parameters set if stimulation is enabled
```

**Click Start Multi-Capture.**

The multi-capture thread (T7) starts and the window shows a progress bar.

**Execution sequence per data type:**
```
For each selected data type (dark → baseline → stimulation):
  For each grid point in coord_selector.matrix (row-major order):
    Send galvo to (angle_x, angle_y) — Motor 1, sleep 2s, Motor 2, sleep 2s, sleep 4s
    For each exposure in exposure_times:
      Set exposure → restart preview → sleep 1s
      Send GET_SPECKLE_FRAME command (N frames)
      Wait for SPECKLE response
      Save PNG (16-bit) + PNG (8-bit) + MAT to typed subdirectory
  If stimulation type AND Enable Stimulator checked:
    Start NiDAQ pulse pattern in T8 concurrently with captures
```

**Timing per exposure:** Approximately 2–5 seconds depending on N_frames
and exposure time.

**Timing per grid point:** ~8s galvo settling + (exposures × capture time)

**Timing per complete run (example — 3×3 grid, 6 exposures, 100 frames):**
- Galvo moves: 9 points × 8s = 72s
- Captures: 9 × 6 × ~3s = 162s
- Saves: ~0.5s per capture = 27s
- Approximate total: ~4–5 minutes per data type

---

### 11.5 Pause, Resume, and Abort

While a multi-capture session is running:

| Button | Effect |
|---|---|
| Pause | Sets `capture_paused = True`; the thread finishes its current save, then waits on `capture_control.Condition` |
| Resume | Notifies the Condition; capture continues from the next exposure |
| Abort | Sets `capture_abort = True`; the thread exits after the current save |

After abort, a timing report is still generated for all captures completed
before the abort.

**Timing report:** At the end of each session (or after abort), a DOCX
file is written to the save directory:
`YYYYMMDD_HHMMSS_capture_log.docx`

This document contains:
- Session metadata (date, grid size, data types, exposure list, frame count)
- Per-exposure capture duration and FPS table
- Galvo move timing summary
- Image save timing summary
- Total session duration
- Performance efficiency metric (capture time as % of total time)

---

## 12. Stereotaxic Stage Control Window

Opened by: Quick Actions → Stereotaxic Platform

Controls the XYZ motorised stereotaxic stage for specimen positioning.

### Connection
Click **Connect** to establish the Modbus connection to the XYZ stage.
The port used is `detected_ports['xyz_stage']` (auto-detected at startup).
Connection status is shown next to the button.

### Speed mode selector
Three options (radio buttons or dropdown):
- **Fast** — maximum motor speed, for gross repositioning
- **Slow** — reduced speed
- **SV Precise** — step-and-verify mode, for final positioning

### Movement buttons
Six directional buttons, one per axis and direction:

| Button | Axis | Direction | Fast code | Slow code | Precise code |
|---|---|---|---|---|---|
| Move Right | X | + | 137 | 138 | 140 |
| Move Left | X | − | 201 | 202 | 204 |
| Move Up | Y | + | 145 | 146 | 148 |
| Move Down | Y | − | 209 | 210 | 212 |
| Come Upward | Z | + | 161 | 162 | 164 |
| Go Downward | Z | − | 225 | 226 | 228 |

Each button click sends one Modbus command via `xyz_client.write_registers
(address=1, values=[cmd], slave=1)`. Click repeatedly for larger moves.

**Note on Y-axis labels:** "Move Up" and "Move Down" are labelled to match
the physical stage orientation in the TEBI Lab setup. The button labelled
"Move Up" sends the stage in the direction that moves the specimen upward
toward the camera.

---

## 13. Relay Control Window

Opened by: Header Bar → Relay

Controls the four-channel relay board on the shared RS-485 bus.

### Layout
Four rows, one per relay channel. Each row contains:
- Channel label: "Relay 1", "Relay 2", etc.
- Toggle button labelled **ON** (green) or **OFF** (grey)
- Status indicator

### Toggling a relay
Click the toggle button for a channel. The button immediately updates its
label and colour. A Modbus write is sent:

```python
relay_states[channel] = not relay_states[channel]
payload = [1, channel, R0, R1, R2, R3, 0, 0]
write_registers(address=7, values=payload, slave=2)
```

All four relay states are sent in every write — this is a batch update,
not a single-channel command.

**Typical relay assignments (lab-specific, not hardcoded):**
- Relay 1: Laser power
- Relay 2: Illumination ring light
- Relay 3: Stimulator power supply
- Relay 4: Spare

Turn off laser relay (Relay 1) before any preparation steps that require
hands in the beam path.

---

## 14. NiDAQ Forepaw Stimulator

Opened by: Header Bar → NiDAQ Controller

### Safety dialogs
**On open:** A dialog reminds you to set the stimulator hardware to
direct digital input mode.

**Before each run:** A confirmation dialog asks:
"Confirm stimulation settings? [parameters shown] — Proceed?"
Click YES to start, NO to cancel.

### Parameter fields

| Field | Variable | Default | Description |
|---|---|---|---|
| Number of Sets | total_sets | — | How many stimulus sets to deliver |
| Pulses per Set | pulses_per_set | — | Pulses within one set |
| Pulse Duration (s) | pulse_duration | — | Duration of each HIGH pulse |
| Inter-Pulse Interval (s) | inter_pulse_interval | — | Gap between pulses in a set |
| Inter-Set Interval (s) | inter_set_interval | — | Gap between sets |

Example configuration for a standard somatosensory activation protocol:
- Sets: 10
- Pulses per set: 5
- Pulse duration: 0.003 s (3 ms)
- Inter-pulse interval: 0.050 s (50 ms)
- Inter-set interval: 10.0 s

### Start / Stop Stimulation
**Start:** Triggers the confirmation dialog, then starts T8 (stimulation thread).
The progress bar shows completed sets, with a time-elapsed counter and set
count (e.g., "Set 3 / 10").

**Stop:** Sets `stimulation_running = False`. The stimulation thread
exits at the end of the current pulse cycle (at most one pulse_duration
after clicking Stop).

### Progress display
- Progress bar: 0 to total_sets
- Time elapsed label: "Elapsed: 00:23"
- Set counter: "Set 3 / 10"
- Completion dialog when all sets finish

### Integration with Multi-Capture
When **Enable Stimulator** is checked in the Multi-Capture window, the
stimulation pattern is triggered automatically at the start of each
stimulation-type capture rather than manually from this window.

---

## 15. Calibration Management Window

Opened by: Multi-Capture & Galvo → Calibration Management button

### Status panel
Shows the current in-memory calibration state:
```
✓ CALIBRATION LOADED
   Grid: 3 × 3
   Type: regression
   Shape: (3, 3, 2)

File Status:
  • JSON file: ✓ Found
  • Binary file: ✓ Found
  • Backup files: 2 found

ROI Settings (if applicable):
  • Active: False
```

### Buttons

| Button | Action |
|---|---|
| 📊 Refresh Status | Updates the status panel from the current in-memory state and file system |
| 💾 Backup Current | Saves the current matrix to a named .pkl + .json backup file |
| 📂 Restore Backup | Opens a list of backups; restoring overwrites the active calibration |
| ❌ Close | Closes the window |

For full backup and restore procedures, see calibration_guide.md Section 12.

---

## 16. Intensity Profile Output Window

Opened automatically when a line is drawn with the Live or Speckle
Intensity Profile tool.

### Layout
A two-panel matplotlib figure embedded in a Toplevel window:
- **Left panel:** The image (live or speckle) with the drawn line shown in red
- **Right panel:** The intensity profile along the line (pixel value vs. distance in px)

### Profile computation
The profile is extracted by sampling pixel values along the Bresenham line
between the start and end points. The number of sample points equals the
Euclidean pixel distance of the line rounded to the nearest integer.

### Export buttons

| Button | Saves |
|---|---|
| Export MAT | `profile_YYYYMMDD_HHMMSS.mat` — MATLAB struct with `intensity`, `x_coords`, `y_coords`, `distance_px` fields |
| Export PNG | `profile_YYYYMMDD_HHMMSS.png` — the two-panel matplotlib figure |
| Export DOCX | `profile_YYYYMMDD_HHMMSS.docx` — statistics report: mean, std, min, max, range, profile plot embedded |

All exports go to the current save directory. If no save directory is set,
a file picker dialog opens.

---

## 17. Companion Tools

### Live Speckle Tracker (live_contrast_GUI_2.py)
A separate standalone real-time speckle monitoring application. Connects
directly to the camera hardware — see Section 3.8 for the critical launch
ordering requirement.

### Spatial-Temporal Analysis (ST_GUI_3.py)
A post-processing application for computing spatiotemporal speckle contrast
from data saved by the main platform. Does not connect to the camera.
Open this tool after imaging is complete to analyse the saved .mat or .png files.

---

## 18. Standard Session Workflow

The following sequence describes a typical baseline + stimulation imaging session.

```
SETUP (do once per day, or after any hardware change)
──────────────────────────────────────────────────────
1. Power on hardware in order (relay board, galvo, stage, laser Arduino)
2. Connect all USB cables to PC
3. Activate virtual environment: lsci_env\Scripts\activate
4. Launch: python new_GUI_sc_55.py
5. Console: confirm COM port detection for all 3 devices
6. Header bar: laser current appears (e.g., 450.00 mA)

INITIALIZATION
──────────────
7. Quick Actions → Initialize Camera → wait for "Camera initialized" dialog
8. Header → Relay → Turn ON relays 1–4
9. Quick Actions → Start Preview → confirm live image appears
10. Open Camera Controls → set exposure time for your sample

FOCUSING AND POSITIONING
─────────────────────────
11. Camera Controls → Z-axis: Move Up / Down to focus
12. Stereotaxic Platform → Connect → position specimen with Move buttons
13. Observe live image; confirm blood vessels are visible

CALIBRATION (if not already loaded from previous session)
──────────────────────────────────────────────────────────
14. Header → Calibrate Scale → draw line over reference → enter mm → confirm
15. Multi-Capture & Galvo → Enable Galvo → Auto Calibrate → click all 9 points
16. Confirm "Calibration completed ✅" dialog

ROI (optional)
────────────────
17. Camera Controls → ROI Settings → enter X1,Y1,X2,Y2 → Enable ROI
18. Confirm live image crops to the selected region

SPECKLE VIDEO CHECK
────────────────────
19. Quick Actions → enter frame depth (e.g., 50) → Enter
20. Start Speckle Video → confirm viridis map appears in right panel
21. Verify vasculature visible as brighter regions; adjust C-axis slider
22. Stop Speckle Video (multi-capture will restart it automatically)

DATA COLLECTION — DARK
───────────────────────
23. Turn OFF laser relay (Relay 1) to eliminate illumination
24. Multi-Capture & Galvo → check Dark only → Start Multi-Capture
25. Wait for all exposures × grid points to complete
26. Turn laser relay back ON

DATA COLLECTION — BASELINE
────────────────────────────
27. Multi-Capture & Galvo → check Baseline only → Start Multi-Capture
28. Wait for completion

DATA COLLECTION — STIMULATION
────────────────────────────────
29. NiDAQ Controller → set stimulation parameters → close window
30. Multi-Capture & Galvo → check Stimulation → check Enable Stimulator
31. Start Multi-Capture
32. Stimulation pulses deliver automatically with each capture

WRAP-UP
────────
33. Stop Preview
34. Header → Relay → Turn OFF all relays
35. Review timing report DOCX in the save directory
36. Header → Exit GUI → confirm quit

POST-PROCESSING
────────────────
37. Quick Actions → Spatial-Temporal Contrast → ST_GUI_3.py opens
38. Load saved .mat files for spatiotemporal speckle contrast analysis
```

---

## 19. Keyboard and Mouse Reference

The platform is primarily mouse-driven. There are no dedicated keyboard
shortcuts beyond standard Tkinter entry field behaviour. The following
mouse interactions are context-sensitive:

| Context | Action | Effect |
|---|---|---|
| Live canvas — normal | Mouse move | Update coordinates in status bar and Camera Controls |
| Live canvas — scale calibration active | Click + drag | Draw green ruler line |
| Live canvas — intensity profile active | Click + drag | Draw red profile line → profile window opens on release |
| Live canvas — galvo calibration active | Left click | Register calibration point |
| Speckle canvas — speckle profile active | Click + drag | Draw red profile line on speckle image |
| Any entry field | Tab | Move to next field |
| Any entry field | Return / Enter | Confirms value if button is bound; otherwise no effect |
| Dialog windows | Escape | Closes most Toplevel dialogs |

---

## 20. Troubleshooting During a Session

### Live image is frozen / not updating
1. Check Start Preview was clicked and is not greyed out
2. Check the camera process is still running — look for it in Task Manager
   (a second python.exe process)
3. Stop Preview → Start Preview to restart the frame loop
4. If frames still do not arrive: reinitialize (Stop Preview → Initialize Camera
   is not available after first init; restart the application)

---

### Speckle view is black or uniform
1. Confirm Start Preview is running — speckle video requires live frames
2. C-Axis slider must be > 0.000 (default 1.000)
3. At least 2 frames must be in the buffer — wait 1–2 seconds after
   clicking Start Speckle Video
4. Check laser is on and relay is ON
5. Try increasing the frame depth (e.g., from 10 to 50) for a smoother map

---

### Multi-capture stops mid-session unexpectedly
Check the console window for error messages. Common causes:
- Camera process died (check Task Manager): restart application
- Modbus error on galvo move: check RS-485 cable and power
- data_queue timeout: increase exposure_time or reduce N_frames to stay
  within the 10-second Modbus timeout

The abort flag is checked after each save, so any incomplete captures will
have a partial timing report in the DOCX.

---

### Relay button clicked but device does not respond
1. Confirm the relay board is powered on
2. Check RS-485 cable between relay board and USB adapter
3. Console should print Modbus errors if the write failed
4. Verify slave ID = 2 on the relay board hardware

---

### Galvo dial moved but beam does not move
1. Click Enable Galvo — all galvo commands are silently skipped
   when galvo_enabled = False
2. Check Modbus connection: relay should still work if the bus is up
3. Verify the galvo controller is powered and slave ID = 2

---

### NiDAQ stimulation starts but no current is delivered to animal
1. Check Dev1/port0/line0 → stimulator input cable
2. Confirm stimulator is in direct digital input mode (not edge trigger)
3. Check signal ground connection (DAQ GND to stimulator GND)
4. Measure the signal on Dev1/port0/line0 with a multimeter — it should
   toggle between 0V and 3.3V (or 5V) during the pulse ON phase

---

### Save directory error on capture
The save directory must exist and be writable. If the directory was on a
network share that disconnected during the session, the save will fail.
Always use a local drive for data collection and copy to the network
share afterwards.

---

*Document version: January 2026 - Platform v2.1*
