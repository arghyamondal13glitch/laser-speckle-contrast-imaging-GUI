# CHANGELOG

**Small Animal Imaging Platform (SAIP)**
TEBI Lab · IIT Bombay
Developer: Arghya Mondal · PI: Dr. Hari M. Varma

All notable changes to this platform are documented in this file.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

> **Note on reconstruction:** This changelog is reconstructed from
> the source code, the About dialog, the Specs window, inline comments,
> and the companion script versioning (live_contrast_GUI_2.py,
> ST_GUI_3.py). Entries marked *(inferred)* are derived from version
> number suffixes, feature comparisons between script iterations, and
> naming conventions, not from explicit commit history.

---

## [2.1] — January 2026

**Current production release.**
This version represents the fully integrated platform combining real-time
LSCI, multiprocessing camera isolation, hybrid galvo calibration, NiDAQ
stimulation synchronisation, and comprehensive data management.

### Added

**Hybrid Galvo Calibration System**
- Four interoperable calibration modes: Manual (pre-typed angles), Dial-based
  (physical dial positioning), Hybrid (mixed), and Auto Calibrate (click-only)
- `ManualAngleInput` Toplevel window with rows×cols angle entry table
- Per-cell Set, Clear, and Set-from-Current-Dials controls
- Preview Combined dialog showing manual vs dial-based point counts
- `auto_calibrate_from_clicks()` unified click-loop entry point routing
  between modes based on content of `manual_angle_points` dict
- Per-point confirmation dialog for dial-based points; auto-advance for
  pre-typed manual points
- `combined_function()` module-level regression helper callable independently
  of the GUI

**Rectangular Grid Support**
- Calibration and multi-capture grids now support 2×2, 2×3, 3×2, and 3×3
  configurations (previously 3×3 only)
- `cal_grid_rows` and `cal_grid_cols` selectors added to Multi-Capture window
- Row-major scan order enforced and documented (y outer, x inner)

**Calibration Management System**
- `CalibrationManager` class with Toplevel management dialog (470×440 px)
- Named backup: saves current calibration to `<name>.pkl` + `<name>.json`
- Named restore: Listbox selection of all backups; PKL preferred over JSON
- Status panel showing grid dimensions, type, shape, file existence, ROI params
- Refresh Status button for on-demand file system re-check

**ROI Coordinate Mapping Improvements**
- ROI offsets (`roi_x1`, `roi_y1`) added to every calibration click coordinate
  so calibration operates in camera-absolute pixel space regardless of ROI state
- `update_roi_params()` called at multi-capture start to snapshot current ROI
  into the calibration file metadata
- Dual-coordinate display in status bar: ROI-relative and original full-frame
  coordinates shown simultaneously when ROI is active

**Dynamic Ruler Overlay**
- `draw_mm_ruler()` renders horizontal and vertical mm tick marks directly
  on the live canvas image array, compensating for display scale and padding
- Ruler labels update dynamically when `calibration_factor` changes
- ROI active dimensions shown as overlay text (px and mm) on both canvases

**Speckle Contrast Video — Viridis + Colorbar**
- Live `1/K² × C` computation in rolling buffer thread with configurable depth
- Vertical colorbar with 10 tick labels rendered alongside the speckle canvas
- `render_colorbar_image()` method generating the gradient strip
- C-axis slider (0.000–1.000) for display scaling independent of raw data
- Frame buffer managed as `latest_speckle_stack` rolling list

**Dual-Mode Intensity Profile Tool**
- `IntensityProfileTool` supports both `'live'` and `'speckle'` modes
- Line drawn in red on the selected canvas; Bresenham-approximated sampling
- Two-panel output window: intensity profile line plot + histogram
- Statistical annotation: Mean, Std, Max, Min in plot title
- Three export formats: `.mat` (scipy), `.png` (fig.savefig), `.docx` (python-docx)

**NiDAQ Forepaw Stimulator Integration**
- `ForepawStimulatorApp` class with full Toplevel control panel (495×370 px)
- Configurable: total sets, pulses per set, pulse duration, inter-pulse
  interval, inter-set interval
- 1 ms timing precision via busy-wait loop
- Progress bar with elapsed time and set counter
- Safety confirmation dialog on every run start
- Digital output on `Dev1/port0/line0` via `PyDAQmx.Task`
- Stimulation sync with multi-capture: starts in T8 concurrently with
  stimulation-type captures when Enable Stimulator is checked

**Multi-Capture Data Type Classification**
- Three independently checkable data types: Dark, Baseline, Stimulation
- Each type creates its own subdirectory: `dark_data/`, `base_data/`,
  `stimulation_data/`
- File prefix matches data type: `dark_`, `base_`, `stim_`

**Advanced Timing Analysis**
- `generate_timing_report()` produces a DOCX document after every session
- Seven-section report: Session Summary, Per-Exposure Capture Statistics,
  Galvo Movement Timing, Image Save Statistics, Frame Capture Statistics,
  Performance Efficiency Metric, Raw Timing Data table
- Capture efficiency = capture_time / total_session_time × 100 %

**Pause / Resume / Abort for Multi-Capture**
- `capture_control` threading.Condition for safe inter-thread signalling
- Pause stops after current save; Resume restarts from next exposure
- Abort exits after current save; timing report still generated

**Laser Current Monitor Thread**
- Dedicated daemon thread (T2) reading ASCII `"Current: <float>\n"` at
  19200 baud from the laser Arduino
- `laser_serial_queue` for thread-safe handoff to the Tk `root.after` updater
- `laser_monitor_stop_event` threading.Event for clean shutdown
- Header bar shows live mA value, updated every 1 second

**Camera Feed Toggle Panel**
- Checkboxes to show/hide Live Feed and Speckle Feed independently
- Layout repacking on toggle with 100 ms ruler refresh delay

**Companion Script Integration**
- Quick Actions buttons launch `live_contrast_GUI_2.py` and `ST_GUI_3.py`
  via `subprocess.Popen`
- Start-before-initialization reminder dialog for the Live Speckle Tracker

### Changed

**Multiprocessing Architecture (from threading)**
- Camera process isolated in a spawned child process (`set_start_method("spawn")`)
- `camera_process(command_queue, data_queue)` handles all scTDC SDK calls
- GUI never calls camera SDK functions directly; all access via IPC queues
- `command_queue` and `data_queue` both `maxsize=10` for backpressure control
- Eliminates scTDC DLL thread-safety issues that caused frame drops in v2.0

**Shared Modbus Bus — Single Client with Lock**
- Relay, Galvo, and Z-axis now share one `ModbusSerialClient` (`client2`)
- `shared_modbus_lock = threading.Lock()` serialises all writes
- `relay_client = galvo_client = zaxis_client = client2` (aliases)
- Eliminates "port already open" errors from attempting three serial
  connections to the same RS-485 adapter

**Galvo Float32 Encoding**
- Angles now encoded as IEEE 754 big-endian float32 split into two uint16
  Modbus registers via `float_to_registers(val)`
- Previously used fixed-point integer encoding which limited angular precision

**Coordinate System for Calibration Clicks**
- Click coordinates converted from canvas display space to full-resolution
  image space using `display_padding` and `display_scale` before storage
- ROI offsets added to produce camera-absolute coordinates
- Previously calibration operated in display pixel space, causing
  miscalibration when the window was resized

**Calibration Persistence — Dual Format**
- Both `.pkl` (pickle, preferred) and `.json` (human-readable) written
  simultaneously after every calibration run
- Load order: PKL first, JSON fallback, graceful degradation if neither found
- Previously only JSON was written, losing numpy dtype information

**XYZ Stage — Separate Modbus Client**
- XYZ stage now uses a dedicated `xyz_client` on a separate COM port
  rather than the shared `client2` bus
- No lock needed for XYZ writes (independent serial connection)

**_save_png_16bit — PIL Primary + cv2 Fallback**
- Primary path: `Image.fromarray(frame.astype(np.uint16), mode='I;16').save(path)`
- Fallback: `cv2.imwrite(str(path), frame.astype(np.uint16))`
- Previously only cv2 was used; PIL mode='I;16' preserves exact uint16 values

**GUI Window Sizing**
- Window sized to 95% of screen dimensions at startup via
  `root.geometry(f"{int(sw*0.95)}x{int(sh*0.95)}")`)
- Previously fixed 1400×900 px which was too small on 1080p screens

### Fixed

- Galvo "phantom move" on startup: galvo commands now gated by `galvo_enabled`
  flag; all methods silently return early if galvo is not enabled
- Calibration click registered outside letterbox area: boundary check added
  in `on_click()` handler before processing any calibration point
- ROI toggle leaving stale canvas ruler ticks: ruler is redrawn 100 ms after
  every ROI state change via `root.after(100, ...)`
- Camera process orphaned on abnormal exit: `exit_gui()` sends `Commands.EXIT`,
  joins with timeout=5s, and terminates the process if still alive
- `"--- mA"` displayed permanently if laser serial port disconnects mid-session:
  `laser_monitor_stop_event` now checked in the readline loop; empty reads
  are silently skipped rather than causing the thread to crash

### Dependencies Added

- `tkdial >= 0.3` — for galvo dial widgets
- `customtkinter >= 5.2` — for styled buttons in galvo control panel
- `PyDAQmx >= 1.4` — for NiDAQ stimulator
- `scikit-learn >= 1.3` — for `LinearRegression` in calibration regression

---

## [2.0] — Mid 2025 *(inferred)*

Major restructuring of the original single-threaded GUI into a
multiprocessing architecture. First version to support galvo scanning
and automated multi-capture.

### Added *(inferred)*

- `Commands` class for camera IPC command codes
- `camera_process()` function as spawned child process (initial version
  with threading — see v2.1 for multiprocessing fix)
- `CoordinateSelector` class for galvo grid management
- `ConfigurationManager` class for JSON calibration persistence
- Basic galvo dial controls (Motor X, Motor Y) with ±40° range
- Initial 3×3 grid calibration (dial-based only)
- Multi-capture loop: automated exposure sweep per grid point
- Frame stacking speckle contrast (K² = (std/mean)² × 4095 formula)
- ROI selection (X1, Y1, X2, Y2 entry fields)
- `CalibrationTool` for interactive px/mm ruler calibration
- Live preview with adjustable contrast slider
- Save to 16-bit PNG and MAT file formats
- `ForepawStimulatorApp` initial version (standalone, not yet synced
  with multi-capture)
- Z-axis motor control (up/down buttons)
- Relay control (4-channel Modbus)
- XYZ stereotaxic stage control (initial version, same Modbus bus)
- Laser current monitor (initial ASCII serial reader)

### Changed *(inferred)*

- Camera previously accessed directly from the GUI thread; moved to
  background thread to prevent UI freezing during frame acquisition
- Calibration data previously stored only in session memory; JSON
  persistence added in this version

### Known Issues Addressed in 2.1

- DLL thread-safety: Camera SDK not safe when called from a Python thread;
  fixed in 2.1 by moving to a spawned child process
- Three serial clients on same COM port: relay, galvo, and z-axis each
  opened the same port independently causing "port already open" errors;
  fixed in 2.1 with shared client and lock
- Calibration click coordinates in display space: not corrected for
  window resize or letterboxing; fixed in 2.1 with display_padding system

---

## [1.3] — Early 2025 *(inferred)*

Third major iteration of the single-file GUI. First version to include
a speckle contrast view and exposure time controls.

### Added *(inferred)*

- Right-panel speckle contrast display (greyscale, no colormap)
- Multi-exposure capability: capture at multiple µs values in sequence
- Exposure time entry field and update mechanism
- Basic coordinate display (pixel x, y on mouse hover)
- `save_directory` selection via `filedialog.askdirectory()`
- 8-bit PNG preview saves alongside 16-bit captures
- `.mat` file export using `scipy.io.savemat`
- Basic status bar messages

### Changed *(inferred)*

- Camera frame acquisition moved from blocking call to a background thread
  (first use of `threading.Thread` for camera)
- Live canvas resized from fixed 640×480 to aspect-ratio-preserving layout

---

## [1.2] — Late 2024 *(inferred)*

Second iteration. Added hardware controls beyond the camera.

### Added *(inferred)*

- Relay board control (initial Modbus implementation)
- Galvo motor control (initial version — integer commands, not float32)
- Z-axis motor control (initial version)
- XYZ stage control (initial version)
- COM port selection via dropdown (not yet auto-detected)
- Basic ROI (crop to fixed region, not user-configurable)

### Changed *(inferred)*

- Modbus library upgraded from `minimalmodbus` to `pymodbus`
  (inferred from current pymodbus v3.x API usage throughout the codebase)

---

## [1.1] — Mid 2024 *(inferred)*

First iteration with user-configurable parameters.

### Added *(inferred)*

- Exposure time entry field
- Frame count entry for stack averaging
- Save path entry
- Basic `about` dialog with developer information
- Error dialogs for camera initialization failures

### Changed *(inferred)*

- Camera frame display switched from `cv2.imshow` (external window) to
  Tkinter canvas rendering (integrated into the main GUI window)

---

## [1.0] — Early 2024 *(inferred)*

Initial working prototype. Basic camera preview GUI.

### Added *(inferred)*

- Tkinter main window with live camera preview
- `scTDC` camera initialization and single-frame acquisition
- Initialize Camera and Start/Stop Preview buttons
- Raw 12-bit to uint8 normalisation for display
- Console debug output for camera state

---

## Companion Script Version History

The main platform ships with two companion tools. Their versioning
is tracked separately, indicated by their filename suffixes.

### live_contrast_GUI_2.py — Live Speckle Tracker v2

| Version | Status | Notes |
|---|---|---|
| v1 (`live_contrast_GUI.py`) | Superseded | Original live speckle tracker; basic contrast display |
| **v2 (`live_contrast_GUI_2.py`)** | **Current** | Improved layout; must be launched before camera initialization in the main GUI |

### ST_GUI_3.py — Spatial-Temporal Analysis v3

| Version | Status | Notes |
|---|---|---|
| v1 (`ST_GUI.py`) | Superseded | Initial spatiotemporal speckle analysis |
| v2 (`ST_GUI_2.py`) | Superseded | Added batch file loading |
| **v3 (`ST_GUI_3.py`)** | **Current** | Full post-processing tool; reads .mat files saved by the main platform |

---

## Platform Naming History

| Name | File | Notes |
|---|---|---|
| Small Animal Imaging Platform (SAIP) | Various early scripts | Initial working name used internally |
| Cerebral Blood Flow Imaging Platform | — | Used in early lab documentation |
| Speckle Contrast Imaging Platform | `new_GUI_sc_*.py` | Current naming; `sc` = speckle contrast; numeric suffix = iteration |
| **Small Animal Imaging Platform V2.1** | `new_GUI_sc_55.py` | Official release name shown in About dialog; iteration 55 |

The `new_GUI_` prefix distinguishes this script from an earlier
`GUI_sc_*.py` series which used a different widget layout and lacked
the multiprocessing camera isolation.

---

## Development Statistics (v2.1)

| Metric | Value |
|---|---|
| Total lines of code | 6,416 |
| Number of classes | 8 |
| Number of threads at runtime | Up to 9 (T0–T8) |
| Number of OS processes at runtime | 2 (main + camera) |
| Hardware devices integrated | 6 (camera, galvo, relay, z-axis, XYZ stage, laser, NiDAQ) |
| Serial/Modbus COM ports in use | 3 |
| Output file formats | 7 (.png 16-bit, .png 8-bit, .png viridis, .mat, .docx, .json, .pkl) |
| Calibration modes | 4 (manual, dial, hybrid, auto) |
| Supported grid sizes | 4 (2×2, 2×3, 3×2, 3×3) |
| Python dependencies (pip) | 13 + 2 vendor (scTDC, NI-DAQmx) |
| Release date | January 2026 |
| Licence | Academic Use Only |

---

*Maintained by: Arghya Mondal — TEBI Lab, IIT Bombay*
*Contact: 30006166@iit.ac.in*
