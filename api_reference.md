# docs/api_reference.md — API Reference

**LSCI Platform v2.1 · TEBI Lab · IIT Bombay**

Complete class and function reference for `new_GUI_sc_55.py`.
All parameters, return values, side-effects, and cross-references
are derived directly from the source code.

---

## Table of Contents

1. [Module-Level Globals](#1-module-level-globals)
2. [Module-Level Functions](#2-module-level-functions)
   - detect_com_ports
   - find_device_by_description
   - auto_detect_ports
   - camera_process
   - combined_function
3. [class Commands](#3-class-commands)
4. [class ConfigurationManager](#4-class-configurationmanager)
5. [class CalibrationManager](#5-class-calibrationmanager)
6. [class CoordinateSelector](#6-class-coordinateselector)
7. [class CalibrationTool](#7-class-calibrationtool)
8. [class ManualAngleInput](#8-class-manualangleinput)
9. [class IntensityProfileTool](#9-class-intensityprofiletool)
10. [class ForepawStimulatorApp](#10-class-forepawstimulatorapp)
11. [class IntegratedGUI](#11-class-integratedgui)
    - 11.1 Constructor and State
    - 11.2 Camera Methods
    - 11.3 Display and Rendering Methods
    - 11.4 Speckle Video Methods
    - 11.5 ROI Methods
    - 11.6 Galvo Methods
    - 11.7 Galvo Calibration Methods
    - 11.8 Relay Methods
    - 11.9 Z-axis Methods
    - 11.10 XYZ Stage Methods
    - 11.11 Laser Monitor Methods
    - 11.12 Multi-Capture Methods
    - 11.13 NiDAQ Stimulator Methods
    - 11.14 Data Save Methods
    - 11.15 Utility and UI Methods

---

## 1. Module-Level Globals

```python
command_queue = multiprocessing.Queue(maxsize=10)
```
IPC channel from the main process to the camera process.
Messages are integers (Commands constants) or tuples.
Backpressure: `put()` blocks when queue is full (10 items).

```python
data_queue = multiprocessing.Queue(maxsize=10)
```
IPC channel from the camera process back to the main process.
Messages are `(str, data)` tuples. See Commands section for message types.

```python
script_dir = os.path.dirname(os.path.abspath(__file__))
os.add_dll_directory(script_dir)
```
Executed at module load time. Adds the script directory to the Windows
DLL search path so that `scTDClib.dll` (and its dependencies) are found
when `import scTDC` loads the C extension.

---

## 2. Module-Level Functions

---

### `detect_com_ports()`

```python
def detect_com_ports() -> list[str]
```

Lists all currently available serial COM port device strings.

**Returns:** `list[str]` — e.g. `['COM3', 'COM4', 'COM7']`.
Returns an empty list if no ports are found.

**Uses:** `serial.tools.list_ports.comports()`

---

### `find_device_by_description(description_keywords)`

```python
def find_device_by_description(description_keywords: list[str]) -> str | None
```

Scans all COM ports and returns the device string of the first port
whose USB description contains any of the supplied keywords
(case-insensitive substring match).

**Parameters:**
- `description_keywords` — `list[str]`: Keywords to match against
  `port.description`. Example: `['CP2102', 'FT232R', 'USB-to-Serial']`

**Returns:** `str` COM port string (e.g. `'COM4'`) if a match is found,
`None` otherwise.

---

### `auto_detect_ports()`

```python
def auto_detect_ports() -> dict[str, str]
```

Detects all three required hardware COM ports by USB descriptor keyword
matching. Falls back to positional assignment if description matching fails.

**Returns:** `dict` with keys:
- `'relay_galvo'` — COM port for Relay + Galvo + Z-axis (Modbus shared bus)
- `'xyz_stage'`   — COM port for XYZ stereotaxic stage
- `'laser'`       — COM port for laser current monitor Arduino

Keys may be absent if fewer than the required number of ports are available.

**Keyword patterns used:**

| Key | Detection keywords |
|---|---|
| `relay_galvo` | `'USB Serial Device'`, `'CP2102'`, `'FT232R'`, `'USB-to-Serial'` |
| `xyz_stage`   | `'USB Serial Port'`, `'CH340'`, `'Prolific'`, `'SERIAL'` |
| `laser`       | `'Arduino'`, `'USB Serial'`, `'CP2102'` |

**Fallback:** If description matching fails, assigns the 1st, 2nd, and 3rd
available ports to relay_galvo, xyz_stage, and laser respectively.

**Side-effects:** Prints detection results to stdout.

---

### `camera_process(command_queue, data_queue)`

```python
def camera_process(
    command_queue: multiprocessing.Queue,
    data_queue:    multiprocessing.Queue
) -> None
```

Entry point for the spawned camera child process. Initialises the scTDC
SDK, then runs a blocking command dispatch loop until `Commands.EXIT` is received.

**Parameters:**
- `command_queue` — Receives commands from the main process
- `data_queue`    — Sends frames and status messages back to the main process

**Internal state:**
- `lib`         — `scTDC.scTDClib()` instance
- `camera`      — `scTDC.Camera(inifilepath="tdc_gpx3.ini", autoinit=False)`
- `pipe`        — Frame pipe created by `camera.add_frame_pipe()`
- `initialized` — `bool` flag; prevents double-initialisation

**Command dispatch:**

| Command received | Action |
|---|---|
| `Commands.INITIALIZE_CAM` | `camera.initialize()` → `camera.add_frame_pipe()` → puts `("INFO", msg)` |
| `(Commands.START_PREVIEW, exposure_us)` | Loops: set_exposure → do_measurement → pipe.read() → puts `("FRAME", img)` |
| `Commands.STOP_PREVIEW` | Breaks out of the preview loop (polled via `command_queue.empty()`) |
| `(Commands.GET_SPECKLE_FRAME, exposure_us, N)` | Captures N frames, computes K², puts `("SPECKLE", speckle_uint16)` |
| `Commands.EXIT` | `camera.remove_pipe()` → `camera.deinitialize()` → `break` |

**Data queue messages produced:**

| Type tag | Data type | Condition |
|---|---|---|
| `"FRAME"` | `np.ndarray` uint16 (H,W) | Every preview frame |
| `"SPECKLE"` | `np.ndarray` uint16 (H,W) | After GET_SPECKLE_FRAME completes |
| `"INFO"` | `str` | Successful camera initialization |
| `"ERROR"` | `str` | Any exception or initialization failure |

**Speckle contrast formula:**
```python
stack   = np.stack(frames, axis=0).astype(np.uint16)
mean    = np.mean(stack, axis=0)
std     = np.std(stack,  axis=0)
speckle = (((std / (mean + 1e-6)) ** 2) * 4095).astype(np.uint16)
```

**Notes:**
- Must be called in a spawned child process only — never called directly
- Camera SDK DLL must be accessible via `os.add_dll_directory(script_dir)`
  (set at module level before this function is defined)
- `pipe.close()` is called after each GET_SPECKLE_FRAME to release the pipe

---

### `combined_function(matrix_C, matrix_D, matrix_A, matrix_B, rows, cols)`

```python
def combined_function(
    matrix_C: list[float],   # galvo X angles (ground truth)
    matrix_D: list[float],   # galvo Y angles (ground truth)
    matrix_A: list[float],   # image X pixel coordinates
    matrix_B: list[float],   # image Y pixel coordinates
    rows: int = 3,
    cols: int = 3
) -> np.ndarray
```

Fits two linear regression models mapping image pixel coordinates to
galvo motor angles, then generates a uniformly spaced grid of predicted
angles covering the image extent of the provided calibration points.

**Parameters:**
- `matrix_C` — Ground-truth galvo X angles for each calibration point
- `matrix_D` — Ground-truth galvo Y angles for each calibration point
- `matrix_A` — Image X pixel coordinates for each calibration point
- `matrix_B` — Image Y pixel coordinates for each calibration point
- `rows`     — Number of rows in the output grid
- `cols`     — Number of columns in the output grid

All four lists must have equal length (≥ 3 for a valid 2D linear fit).

**Returns:** `np.ndarray` of shape `(rows, cols, 2)` where
`result[i, j] = (predicted_galvo_X, predicted_galvo_Y)` for the
grid position at row i, column j.

**Raises:** `ValueError` if the four input lists have unequal lengths.

**Regression models:**
```
Galvo_X = β₀ + β₁ × Image_X + β₂ × Image_Y
Galvo_Y = γ₀ + γ₁ × Image_X + γ₂ × Image_Y
```

**Grid generation:** x_vals and y_vals are evenly spaced from
`min(matrix_A)` to `max(matrix_A)` and `min(matrix_B)` to `max(matrix_B)`
respectively. The outer loop iterates rows (y), inner loop iterates columns (x).

---

## 3. class Commands

```python
class Commands:
    INITIALIZE_CAM    = 1
    START_PREVIEW     = 2
    STOP_PREVIEW      = 3
    RUN_MULTI_CAPTURE = 4   # reserved; orchestration is in the GUI thread
    EXIT              = 5
    GET_SPECKLE_FRAME = 6
```

Namespace class containing integer constants used as IPC command codes.
Has no `__init__` and is never instantiated — used as `Commands.INITIALIZE_CAM`.

---

## 4. class ConfigurationManager

Manages reading and writing of galvo calibration data in both JSON and
pickle (binary) formats.

```python
class ConfigurationManager:
    def __init__(self, config_file: str = "galvo_calibration_config.json")
```

**Parameters:**
- `config_file` — Filename for the JSON calibration store.
  Resolved relative to the script directory (`os.path.abspath(__file__)`).

**Attributes:**
- `config_file` — `str`: Filename (basename only)
- `config_dir`  — `str`: Absolute path to the script directory
- `full_path`   — `str`: Absolute path to the JSON file

---

### `save_calibration(calibration_data)`

```python
def save_calibration(self, calibration_data: dict) -> bool
```

Serialises `calibration_data` to `self.full_path` as indented JSON.
Converts `numpy.ndarray` values at key `'galvo_matrix'` to Python lists
via `.tolist()` for JSON compatibility.

**Returns:** `True` on success, `False` on any exception.

---

### `load_calibration()`

```python
def load_calibration(self) -> dict | None
```

Loads calibration from the JSON file at `self.full_path`. Converts the
`'galvo_matrix'` list back to `numpy.ndarray` via `np.array()`.

**Returns:** `dict` on success, `None` if the file does not exist
or an exception occurs.

---

### `save_binary_calibration(calibration_data, filename)`

```python
def save_binary_calibration(
    self,
    calibration_data: dict,
    filename: str = "galvo_calibration.pkl"
) -> bool
```

Serialises `calibration_data` to a pickle file using `pickle.dump()`.
Stores `numpy.ndarray` objects natively (no dtype conversion).

**Returns:** `True` on success, `False` on exception.

---

### `load_binary_calibration(filename)`

```python
def load_binary_calibration(
    self,
    filename: str = "galvo_calibration.pkl"
) -> dict | None
```

Loads calibration from the specified pickle file.

**Returns:** `dict` on success, `None` if file not found or error.

---

### `get_backup_files()`

```python
def get_backup_files(self) -> list[str]
```

Scans `self.config_dir` for backup files — any `.pkl` or `.json` files
that do not start with `"galvo_calibration."`.

**Returns:** Sorted list of backup base names (without extension).
Example: `['calibration_backup_20260115_143022', 'session_forepaw']`

---

### `get_calibration_status()`

```python
def get_calibration_status(self) -> dict
```

Checks the file system for calibration file existence and backup count.

**Returns:** `dict` with keys:
- `'json_exists'`   — `bool`
- `'pkl_exists'`    — `bool`
- `'json_path'`     — `str` absolute path
- `'pkl_path'`      — `str` absolute path
- `'backup_count'`  — `int`
- `'backup_files'`  — `list[str]` of backup base names

---

## 5. class CalibrationManager

Provides the GUI dialog for backup, restore, and status display of
galvo calibration files. Holds a `ConfigurationManager` internally.

```python
class CalibrationManager:
    def __init__(self, gui_instance: IntegratedGUI)
```

**Parameters:**
- `gui_instance` — Reference to the `IntegratedGUI` instance (for
  accessing `coord_selector`, `roi_active`, `roi_x1/y1/x2/y2`,
  and `root`)

---

### `open_dialog()`

```python
def open_dialog(self) -> None
```

Opens the Calibration Management Toplevel window (470×440 px).
If already open, raises the existing window to the foreground.
Calls `update_status_display()` on open to populate the status panel.

---

### `update_status_display()`

```python
def update_status_display(self) -> None
```

Refreshes the read-only text panel in the management window with
the current in-memory calibration state and file system status.
Reads from `self.gui.coord_selector` and `self.config_manager.get_calibration_status()`.

---

### `backup_calibration()`

```python
def backup_calibration(self) -> None
```

Prompts the user for a backup name (default: `calibration_backup_YYYYMMDD_HHMMSS`).
Saves the current `coord_selector.matrix` and metadata to
`<name>.pkl` and `<name>.json` in the project root directory.
Updates the status panel after save.

---

### `restore_calibration()`

```python
def restore_calibration(self) -> None
```

Opens a Listbox selection window populated with all backup base names
from `get_backup_files()`. On selection confirmation, loads the chosen
backup (PKL preferred over JSON) into `self.gui.coord_selector` and
overwrites the primary calibration files. Updates status display.

---

## 6. class CoordinateSelector

Manages the galvo angle matrix for grid scanning. Responsible for
auto-loading calibration on startup, running the grid scan thread,
and saving calibration after each successful calibration run.

```python
class CoordinateSelector:
    def __init__(self, gui_instance: IntegratedGUI)
```

**Attributes:**

| Attribute | Type | Default | Description |
|---|---|---|---|
| `gui` | `IntegratedGUI` | — | Reference to parent GUI |
| `grid_rows` | `int` | 3 | Number of rows in the calibration grid |
| `grid_cols` | `int` | 3 | Number of columns in the calibration grid |
| `running` | `bool` | False | Whether the grid scan thread is active |
| `matrix` | `np.ndarray` or `None` | None | Shape (rows,cols,2): galvo angle grid |
| `index` | `int` | 0 | Current grid scan position (for resume) |
| `matrix_type` | `str` or `None` | None | `'regression'` or `'hybrid'` |
| `roi_offset_x` | `int` | 0 | X offset of ROI at calibration time |
| `roi_offset_y` | `int` | 0 | Y offset of ROI at calibration time |
| `roi_width` | `int` | 1920 | ROI width at calibration time |
| `roi_height` | `int` | 1440 | ROI height at calibration time |

---

### `load_saved_calibration()`

```python
def load_saved_calibration(self) -> bool
```

Called from `__init__`. Tries PKL first, then JSON fallback. Populates
`self.matrix`, `self.grid_rows`, `self.grid_cols`, `self.matrix_type`.

**Returns:** `True` if calibration was loaded, `False` otherwise.

---

### `save_calibration_to_file()`

```python
def save_calibration_to_file(self) -> bool
```

Saves `self.matrix` plus all metadata (grid dimensions, ROI params,
timestamp, camera resolution) to both PKL and JSON files via
`ConfigurationManager`.

**Returns:** `True` on success, `False` if `self.matrix` is `None` or save fails.

---

### `update_roi_params()`

```python
def update_roi_params(self) -> None
```

Reads the current ROI state from the GUI and updates
`roi_offset_x`, `roi_offset_y`, `roi_width`, `roi_height`.
Called at the start of each multi-capture session to record ROI context.

---

### `load_matrix(matrix)`

```python
def load_matrix(self, matrix: np.ndarray) -> bool
```

Sets `self.matrix` to the provided array, reads grid dimensions from
`matrix.shape[0]` and `matrix.shape[1]`, sets `matrix_type = "regression"`,
and immediately calls `save_calibration_to_file()`.

**Parameters:**
- `matrix` — `np.ndarray` of shape `(rows, cols, 2)`

**Returns:** `True` always (save errors are logged but not propagated).

---

### `get_galvo_coords(x, y)`

```python
def get_galvo_coords(self, x: int, y: int) -> np.ndarray
```

Finds the closest galvo angle pair in `self.matrix` to the given
image coordinates by nearest-neighbour search in the angle space.

**Parameters:**
- `x`, `y` — Image pixel coordinates (ROI-relative; ROI offsets are
  added internally)

**Returns:** `np.ndarray` of shape `(2,)`: `[galvo_angle_X, galvo_angle_Y]`

**Raises:** `RuntimeError` if `self.matrix is None`.

---

### `start()`

```python
def start(self) -> None
```

Initiates the grid scan in a new daemon thread. Resets `self.index = 0`.
Shows error dialog if `self.matrix` is not loaded.

---

### `stop()`

```python
def stop(self) -> None
```

Sets `self.running = False`. The scan thread will exit at the top of
its next iteration and save `self.index` for resume.

---

### `resume()`

```python
def resume(self) -> None
```

Restarts the scan thread from `self.index` (the last incomplete position).
No-op if the scan is already running or is fully complete.

---

### `run()` *(internal thread target)*

```python
def run(self) -> None
```

Grid scan loop. For each grid position (row-major order from `self.index`):
1. Reads galvo angles from `self.matrix[row, col]`
2. Moves Motor 1 (X), waits 2s, moves Motor 2 (Y)
3. Captures `latest_live_frame` and saves as PNG
4. Appends an entry to `coordinate_log.docx` in the save directory
5. Updates GUI progress and status via `root.after()`

On completion, sets `self.running = False` and shows "Grid scan completed ✅".

---

## 7. class CalibrationTool

Interactive ruler tool for spatial scale calibration (px/mm).

```python
class CalibrationTool:
    def __init__(self, gui_instance: IntegratedGUI)
```

**Attributes:**
- `gui`          — Reference to `IntegratedGUI`
- `manual_angle_input` — `ManualAngleInput` instance (shared)
- `start_point`  — `tuple(x, y)` or `None`: canvas coords of line start
- `end_point`    — `tuple(x, y)` or `None`: canvas coords of line end
- `line_id`      — Tkinter canvas item ID of the drawn line, or `None`
- `active`       — `bool`: Whether the tool is currently active

---

### `activate()`

```python
def activate(self) -> None
```

Sets `self.active = True`. Binds `<ButtonPress-1>`, `<B1-Motion>`,
and `<ButtonRelease-1>` events to the live image canvas.
Updates status bar text.

---

### `deactivate()`

```python
def deactivate(self) -> None
```

Sets `self.active = False`. Unbinds all mouse events. Clears the
drawn line from the canvas. Resets status bar to "Ready".

---

### `open_manual_angles()`

```python
def open_manual_angles(self) -> None
```

Opens the `ManualAngleInput` window for the current grid size
(`gui.cal_grid_rows` × `gui.cal_grid_cols`).

---

### `clear_line()`

```python
def clear_line(self) -> None
```

Deletes the drawn canvas line (if any) and resets
`start_point`, `end_point`, and `line_id` to `None`.

---

### `calculate_calibration()` *(called from on_button_release)*

```python
def calculate_calibration(self) -> None
```

Converts canvas coordinates to full-resolution image coordinates
using `display_padding` and `display_scale`, computes Euclidean
pixel distance, prompts for real-world length via `simpledialog.askfloat`,
sets `gui.calibration_factor`, calls `gui.save_calibration()`, and
shows a confirmation dialog.

**Formula:**
```python
x1 = (start_point[0] - pad_left) / scale
pixel_distance = sqrt((x2-x1)² + (y2-y1)²)
factor = pixel_distance / mm_value   # px per mm
```

---

## 8. class ManualAngleInput

Provides the Hybrid Angle Input Toplevel window for pre-specifying
galvo angles at specific grid positions before running Auto Calibrate.

```python
class ManualAngleInput:
    def __init__(self, gui_instance: IntegratedGUI)
```

**Attributes:**
- `gui`           — Reference to `IntegratedGUI`
- `window`        — `tk.Toplevel` or `None`
- `angle_entries` — `dict{(row,col): {'var': StringVar, 'widget': Entry}}`
- `grid_rows`     — `int`
- `grid_cols`     — `int`
- `manual_points` — `dict{(row,col): (float, float)}` of saved angle pairs

---

### `open_window(rows, cols)`

```python
def open_window(self, rows: int = None, cols: int = None) -> None
```

Opens the Toplevel window (530×550 px). If already open, raises it.
Creates a rows×cols grid of entry fields, each with Set and Clear buttons.

**Entry format:** `"X_angle,Y_angle"` — e.g. `"10.5,-5.2"`

---

### `set_dials_from_entry(row, col)`

```python
def set_dials_from_entry(self, row: int, col: int) -> None
```

Reads the angle string from entry `(row, col)`, parses `X,Y` floats,
sets `galvo_dial_X` and `galvo_dial_Y` to those values, and
if `galvo_enabled`, sends the angles to the motors.

---

### `clear_entry(row, col)`

```python
def clear_entry(self, row: int, col: int) -> None
```

Clears the StringVar for entry `(row, col)` and removes it from
`self.manual_points`.

---

### `save_manual_points()`

```python
def save_manual_points(self) -> None
```

Iterates all `angle_entries`, parses non-empty `"X,Y"` strings,
and populates `self.manual_points` and `gui.manual_angle_points`
(the dict used by the calibration click loop). Updates status label.

---

### `set_from_current_dials()`

```python
def set_from_current_dials(self) -> None
```

Prompts the user for a `(row, col)` position and fills that cell
with the current dial readings from `galvo_dial_X` and `galvo_dial_Y`.

---

### `preview_combined_points()`

```python
def preview_combined_points(self) -> None
```

Shows a dialog summarising how many grid cells have manual angles
pre-set vs. how many will require dial-based calibration.

---

## 9. class IntensityProfileTool

Line-draw intensity analysis tool for both live and speckle frames.

```python
class IntensityProfileTool:
    def __init__(self, gui_instance: IntegratedGUI)
```

**Attributes:**
- `gui`                  — Reference to `IntegratedGUI`
- `active`               — `bool`
- `mode`                 — `str`: `'live'` or `'speckle'`
- `start_point`          — `tuple(x,y)` or `None`: canvas start
- `end_point`            — `tuple(x,y)` or `None`: canvas end
- `line_id`              — Canvas item ID or `None`
- `profile_window`       — `tk.Toplevel` or `None`
- `fig`                  — `matplotlib.figure.Figure` or `None`
- `ax1`, `ax2`           — `matplotlib.axes.Axes` (profile + histogram)
- `canvas`               — `FigureCanvasTkAgg` or `None`
- `last_intensity_profile` — `dict` or `None`: last computed profile data

---

### `activate(mode)`

```python
def activate(self, mode: str = 'live') -> None
```

Sets `self.active = True` and `self.mode`. Binds `<ButtonPress-1>`,
`<B1-Motion>`, `<ButtonRelease-1>` to the appropriate canvas
(`live_image_canvas` or `speckle_image_canvas`).

---

### `deactivate()`

```python
def deactivate(self) -> None
```

Sets `self.active = False`. Unbinds canvas events. Clears the drawn
line. Closes `profile_window` if open.

---

### `draw_line(event)` *(mouse event handler)*

```python
def draw_line(self, event) -> None
```

On `<ButtonRelease-1>`: calls `compute_profile()` to extract pixel
intensities along the drawn line. On `<B1-Motion>`: updates the
red line rendering on the canvas.

---

### `compute_profile()`

```python
def compute_profile(self) -> None
```

Converts canvas coordinates to image coordinates using `display_padding`
and `display_scale`. Samples pixel intensities along the line using
`np.linspace`-generated integer coordinate pairs. Stores results in
`self.last_intensity_profile`. Calls `show_intensity_profile()`.

**Profile data stored:**
```python
{
    'distances':  np.ndarray,   # pixel distance from start to each sample
    'intensities': np.ndarray,  # pixel value at each sample
    'start_x', 'start_y',       # int: line start in image coords
    'end_x', 'end_y',           # int: line end in image coords
    'line_length': float,       # Euclidean pixel distance
    'num_points': int,          # number of samples
    'mode': str                 # 'live' or 'speckle'
}
```

---

### `show_intensity_profile(distances, intensities, ...)`

```python
def show_intensity_profile(
    self,
    distances: np.ndarray,
    intensities: np.ndarray,
    start_x: int, start_y: int,
    end_x: int, end_y: int
) -> None
```

Opens or updates `profile_window` (800×800 px). Renders:
- `ax1`: Line plot of intensity vs. distance (blue, linewidth=2)
- `ax2`: Histogram of intensities (50 bins) with statistics annotation

Statistics shown: Mean, Std, Max, Min.

---

### `save_profile()`

```python
def save_profile(self) -> None
```

Saves `last_intensity_profile` in three formats to `gui.save_directory`:
- `.mat` — `scipy.io.savemat` with all profile fields + statistics
- `.png` — `fig.savefig()` of the two-panel matplotlib figure
- `.docx` — `python-docx` document with metadata table and embedded plot

---

## 10. class ForepawStimulatorApp

Standalone NiDAQ forepaw electrical stimulator control panel.
Runs in its own Toplevel window (495×370 px).

```python
class ForepawStimulatorApp:
    def __init__(self, root: tk.Tk)
```

**Parameters:**
- `root` — A new `tk.Toplevel` window passed as the root

**Default parameter values:**
- `total_sets` = 1
- `pulses_per_set` = 90
- `pulse_duration` = 0.0003 s (0.3 ms)
- `inter_pulse_interval` = 0.1664 s (166.4 ms)
- `inter_set_interval` = 30 s

---

### `start_stimulation_with_reminder()`

```python
def start_stimulation_with_reminder(self) -> None
```

Shows a safety confirmation dialog reminding the user to set the
stimulator hardware to direct digital input mode. Calls
`start_stimulation()` if confirmed.

---

### `start_stimulation()`

```python
def start_stimulation(self) -> None
```

Reads all parameter fields, validates that all values are positive,
resets progress counters, and starts `run_stimulation()` in a daemon thread.

---

### `stop_stimulation()`

```python
def stop_stimulation(self) -> None
```

Sets `self.is_running = False`. The stimulation thread exits at the
end of the current pulse cycle (within one `pulse_duration`).

---

### `run_stimulation()` *(thread target)*

```python
def run_stimulation(self) -> None
```

Main stimulation loop:
```
for each set (1 to total_sets):
    for each pulse (1 to pulses_per_set):
        PyDAQmx write HIGH (value=1)
        time.sleep(pulse_duration)
        PyDAQmx write LOW (value=0)
        time.sleep(inter_pulse_interval)
    time.sleep(inter_set_interval)
    update progress bar via root.after()
```

Uses `PyDAQmx.Task()` on `"Dev1/port0/line0"` with
`DAQmx_Val_ChanForAllLines`. Calls `stimulation_complete()` on finish.

---

### `update_progress(current_set)`

```python
def update_progress(self, current_set: int) -> None
```

Updates progress bar, progress label, time elapsed, and set counter
on the main thread via `root.after()`.

---

### `stimulation_complete()`

```python
def stimulation_complete(self) -> None
```

Resets button states, shows completion dialog.
Called from `run_stimulation()` after all sets finish.

---

## 11. class IntegratedGUI

Master application class. Owns all state, hardware connections, and GUI
widgets. Instantiated once in `__main__`.

```python
class IntegratedGUI:
    def __init__(self, root: tk.Tk)
```

---

### 11.1 Constructor and State

**Hardware connections established in `__init__`:**
- `self.client2` — `ModbusSerialClient` for relay + galvo + z-axis
- `self.relay_client = self.galvo_client = self.zaxis_client = self.client2`
- `self.shared_modbus_lock` — `threading.Lock()` protecting `client2`
- `self.xyz_client` — `ModbusSerialClient` for XYZ stage (None until connected)
- `self.command_process` — `multiprocessing.Process(target=camera_process)`

**Key state variables:**

| Attribute | Type | Description |
|---|---|---|
| `latest_live_frame` | `np.ndarray` uint16 | Raw full-res camera frame (1440,1920) |
| `latest_speckle_frame` | `np.ndarray` | ROI-cropped speckle display frame |
| `latest_speckle_frame_raw` | `np.ndarray` | Full-frame raw K² output |
| `latest_speckle_stack` | `list` | Rolling buffer of live frames for speckle video |
| `previewing` | `bool` | Live preview active flag |
| `speckle_video_active` | `bool` | Speckle video thread active flag |
| `galvo_enabled` | `bool` | Gate for all galvo commands |
| `stimulation_running` | `bool` | NiDAQ stimulation active flag |
| `exposure_time` | `tk.IntVar` | Current exposure in µs |
| `fps` | `tk.IntVar` | Frame stack depth for speckle video |
| `calibration_factor` | `tk.DoubleVar` | Spatial scale in px/mm |
| `roi_active` | `tk.BooleanVar` | ROI enabled state |
| `roi_x1/y1/x2/y2` | `tk.IntVar` × 4 | ROI boundaries (full-frame px) |
| `display_scale` | `float` | Canvas-to-raw pixel ratio |
| `display_padding` | `tuple` | `(pad_left, pad_top, content_w, content_h)` |
| `capture_paused` | `bool` | Multi-capture pause flag |
| `capture_abort` | `bool` | Multi-capture abort flag |
| `capture_control` | `threading.Condition` | Pause/abort synchronisation with T7 |
| `save_directory` | `tk.StringVar` | Output directory path |
| `exposure_times` | `tk.StringVar` | Comma-separated exposure list |
| `frames_count` | `tk.IntVar` | Frames per exposure for multi-capture |
| `image_points` | `list` | Pixel coords collected during calibration |
| `galvo_points` | `list` | Angle pairs collected during calibration |
| `manual_angle_points` | `dict` | `{(row,col): (ax,ay)}` for hybrid calibration |
| `relay_states` | `list[bool]` × 4 | Current relay ON/OFF states |
| `detected_ports` | `dict` | COM port assignments |

---

### 11.2 Camera Methods

#### `initialize_camera()`
```python
def initialize_camera(self) -> None
```
Puts `Commands.INITIALIZE_CAM` on `command_queue`. Shows a waiting
dialog. The camera process responds with `("INFO", ...)` or `("ERROR", ...)`.

#### `start_preview()`
```python
def start_preview(self) -> None
```
Puts `(Commands.START_PREVIEW, self.exposure_time.get())` on `command_queue`.
Sets `self.previewing = True`.

#### `stop_preview()`
```python
def stop_preview(self) -> None
```
Puts `Commands.STOP_PREVIEW` on `command_queue`.
Sets `self.previewing = False`. Also stops speckle video if active.

#### `update_exposure()`
```python
def update_exposure(self) -> None
```
Stops the current preview, waits 100 ms, restarts with the new
`exposure_time` value. Used when the exposure entry field is updated.

#### `start_frame_listener()`
```python
def start_frame_listener(self) -> None
```
Starts the frame listener daemon thread (T1) targeting `listen_for_frames()`.

#### `listen_for_frames()`
```python
def listen_for_frames(self) -> None
```
Runs in T1. Blocks on `data_queue.get()`. Dispatches:
- `"FRAME"` → stores to `latest_live_frame`, renders to live canvas
- `"SPECKLE"` → stores to `latest_speckle_frame_raw`, appends to stack
- `"INFO"` → `messagebox.showinfo`
- `"ERROR"` → `messagebox.showerror`

---

### 11.3 Display and Rendering Methods

#### `resize_with_aspect_ratio(image, target_w, target_h)`
```python
def resize_with_aspect_ratio(
    self,
    image: np.ndarray,
    target_w: int,
    target_h: int
) -> tuple[np.ndarray, float, tuple]
```
Resizes `image` to fit within `(target_w, target_h)` while preserving
the 4:3 aspect ratio. Returns:
- Resized image array (with letterbox padding filled to black)
- `scale` — float ratio (resized / original pixel size)
- `(pad_left, pad_top, content_w, content_h)` — layout info

Updates `self.display_scale` and `self.display_padding`.

#### `draw_mm_ruler(image)`
```python
def draw_mm_ruler(self, image: np.ndarray) -> np.ndarray
```
Overlays horizontal and vertical mm-tick rulers on `image` using
`calibration_factor` and `display_scale`. Also overlays ROI
dimension text (px and mm) when ROI is active. Returns the annotated image.

#### `update_coords(event)`
```python
def update_coords(self, event) -> None
```
Mouse motion callback on the live canvas. Converts canvas coordinates
to full-resolution image coordinates and optionally to ROI-relative
coordinates. Updates the status bar and the coord display in Camera Controls.

#### `render_colorbar_image(height, width, vmin, vmax, cmap)`
```python
def render_colorbar_image(
    self,
    height: int = 670,
    width: int = 60,
    vmin: float = 0.0,
    vmax: float = 1.0,
    cmap: str = 'viridis'
) -> np.ndarray
```
Generates a vertical gradient colorbar image as a uint8 RGB array.
Rendered with 10 evenly spaced tick labels from `vmin` to `vmax`.

---

### 11.4 Speckle Video Methods

#### `start_speckle_video()`
```python
def start_speckle_video(self) -> None
```
Sets `speckle_video_active = True`, starts the speckle video daemon
thread (T3) targeting `capture_speckle_video()`.

#### `stop_speckle_video()`
```python
def stop_speckle_video(self) -> None
```
Sets `speckle_video_active = False`. T3 exits on its next iteration check.

#### `capture_speckle_video()`
```python
def capture_speckle_video(self) -> None
```
Thread target (T3). Loop while `speckle_video_active`:
1. Sends `(GET_SPECKLE_FRAME, exposure, N)` to `command_queue`
2. Waits for `"SPECKLE"` from `data_queue`
3. Appends to `latest_speckle_stack`
4. Computes `1/K² × contrast_factor`, applies viridis colormap
5. Pushes rendered image to `speckle_image_canvas` via `root.after(0, ...)`
6. Updates colorbar

---

### 11.5 ROI Methods

#### `apply_roi(frame)`
```python
def apply_roi(self, frame: np.ndarray) -> np.ndarray
```
Returns `frame[y1:y2, x1:x2]` using the current ROI variables.
Enforces minimum size of 10×10 px.

#### `toggle_roi()`
```python
def toggle_roi(self) -> None
```
Validates ROI coordinates (x2 > x1, y2 > y1). Enables or disables ROI.
Updates status label. Triggers display refresh.

#### `reset_roi()`
```python
def reset_roi(self) -> None
```
Sets `roi_x1=0, roi_y1=0, roi_x2=1920, roi_y2=1440`.
Sets `roi_active = False`. Updates status.

---

### 11.6 Galvo Methods

#### `toggle_galvo()`
```python
def toggle_galvo(self) -> None
```
Toggles `self.galvo_enabled` between `True` and `False`.
Updates the Enable/Disable Galvo button text and colour.

#### `send_galvo_motor1()` / `send_galvo_motor2()`
```python
def send_galvo_motor1(self) -> None
def send_galvo_motor2(self) -> None
```
Reads the current dial value from `galvo_dial_X` or `galvo_dial_Y`
and calls the corresponding `_direct` method.

#### `send_galvo_motor1_direct(angle)` / `send_galvo_motor2_direct(angle)`
```python
def send_galvo_motor1_direct(self, angle: float) -> None
def send_galvo_motor2_direct(self, angle: float) -> None
```
Writes `angle` (degrees) to the appropriate galvo motor via Modbus.
Runs in a daemon thread. Acquires `shared_modbus_lock`.
Retries up to 3 times on error (delays: 0.4s, 0.6s, 0.8s).
Updates `galvo_motorX_status` or `galvo_motorY_status` label on success.

**Modbus payload for Motor 1 (X):**
`write_registers(address=0, values=[0, 3, HI_X, LO_X, 0, 0, 0], slave=2)`

**Modbus payload for Motor 2 (Y):**
`write_registers(address=0, values=[0, 5, 0, 0, HI_Y, LO_Y, 0], slave=2)`

#### `home_galvo_to_point1()`
```python
def home_galvo_to_point1(self) -> None
```
Moves both motors to `coord_selector.matrix[0, 0]`.
Fallback to `(0.0, 0.0)` if no matrix is loaded.
Checks `galvo_enabled` first.

#### `float_to_registers(val)`
```python
def float_to_registers(self, val: float) -> list[int]
```
Encodes `val` as a big-endian IEEE 754 float32 and returns the two
16-bit unsigned integer words: `[high_word, low_word]`.
```python
packed = struct.pack('>f', val)
return list(struct.unpack('>HH', packed))
```

---

### 11.7 Galvo Calibration Methods

#### `auto_calibrate_from_clicks()`
```python
def auto_calibrate_from_clicks(self) -> None
```
Starts the interactive calibration click loop.
- Reads `cal_grid_rows`, `cal_grid_cols` for grid dimensions
- Reads `manual_angle_points` for any pre-set angles
- Binds `on_click` to `live_image_canvas`
- For each point in row-major order: collects image click + galvo angle
- After last click: calls `generate_hybrid_calibration()`

#### `generate_hybrid_calibration()`
```python
def generate_hybrid_calibration(self) -> None
```
Called after all calibration clicks are collected. Fits two
`LinearRegression` models, generates the galvo matrix, loads it
into `coord_selector`, and auto-saves. Shows summary dialog.

#### `generate_dynamic_calibration()`
```python
def generate_dynamic_calibration(self) -> None
```
Alternative calibration path using `combined_function()`.
Produces the same output as `generate_hybrid_calibration()` but
uses the standalone `combined_function` module-level function.

#### `save_calibration()`
```python
def save_calibration(self) -> None
```
Deactivates the CalibrationTool (scale calibration) and updates the
ruler overlay after the `calibration_factor` has been set.
Also triggers a display refresh.

#### `show_calibration_load_status()`
```python
def show_calibration_load_status(self) -> None
```
Scheduled via `root.after(1000)` at startup. Updates the status bar
with either "Loaded N×M calibration" or "No calibration — run Auto Calibrate".

---

### 11.8 Relay Methods

#### `open_relay_control()`
```python
def open_relay_control(self) -> None
```
Opens the 4-channel Relay Control Toplevel window.

#### `toggle_relay(channel)`
```python
def toggle_relay(self, channel: int) -> None
```
Toggles `relay_states[channel]`, then writes all four relay states
to Modbus register 7, slave 2:
```python
payload = [1, channel] + [int(s) for s in relay_states] + [0, 0]
client2.write_registers(address=7, values=payload, slave=2)
```

---

### 11.9 Z-axis Methods

#### `move_zaxis_up()` / `move_zaxis_down()`
```python
def move_zaxis_up(self) -> None
def move_zaxis_down(self) -> None
```
Starts a daemon thread targeting `_zaxis_move_command(41)` or `_zaxis_move_command(57)`.

#### `_zaxis_move_command(value)`
```python
def _zaxis_move_command(self, value: int) -> None
```
Acquires `shared_modbus_lock`, writes `[value]` to register 6, slave 2.

---

### 11.10 XYZ Stage Methods

#### `open_motor_controls()`
```python
def open_motor_controls(self) -> None
```
Opens or shows the Stereotaxic Stage Control Toplevel window.

#### `connect_xyz_stage()`
```python
def connect_xyz_stage(self) -> None
```
Creates a new `ModbusSerialClient` on `detected_ports['xyz_stage']`
at 9600 baud and calls `.connect()`. Updates status label.

#### `move_axis(axis, direction)`
```python
def move_axis(self, axis: str, direction: str) -> None
```
Looks up the command code from the 3×2×3 command map
(`axis ∈ {'X','Y','Z'}`, `direction ∈ {'up','down'}`,
`mode ∈ {'Fast','Slow','SV Precise'}`).
Starts a daemon thread targeting `_xyz_move_command(cmd, axis, direction, mode)`.

#### `_xyz_move_command(cmd, axis, direction, mode)`
```python
def _xyz_move_command(
    self,
    cmd: int,
    axis: str,
    direction: str,
    mode: str
) -> None
```
Writes `[cmd]` to register 1, slave 1 on `xyz_client`.

---

### 11.11 Laser Monitor Methods

#### `start_laser_monitor()`
```python
def start_laser_monitor(self) -> None
```
Starts the laser serial daemon thread (T2) targeting `read_laser_serial()`.
Also schedules `update_laser_display()` via `root.after(1000)`.

#### `read_laser_serial()`
```python
def read_laser_serial(self) -> None
```
Thread target (T2). Opens `serial.Serial(laser_port, 19200, timeout=1)`.
Reads lines in a loop; parses `"Current: <float>"` lines and puts the
numeric string into `laser_serial_queue`. Exits when
`laser_monitor_stop_event` is set.

#### `update_laser_display()`
```python
def update_laser_display(self) -> None
```
Drains `laser_serial_queue`, parses float values, updates
`live_current_str` StringVar (shown in header bar).
Reschedules itself via `root.after(1000)` if the root window exists.

---

### 11.12 Multi-Capture Methods

#### `run_multi_capture_window()`
```python
def run_multi_capture_window(self) -> None
```
Opens the Multi-Capture & Galvo Toplevel window.
Contains: exposure time entry, frames per exposure, save directory,
data type checkboxes, stimulator enable, progress bar, galvo dial controls,
and calibration section.

#### `start_recon_capture_thread(window)`
```python
def start_recon_capture_thread(self, window: tk.Toplevel) -> None
```
Validates configuration, resets `capture_paused` and `capture_abort`,
and starts the multi-capture daemon thread (T7).

#### `pause_capture()` / `resume_capture()` / `abort_capture()`
```python
def pause_capture(self) -> None
def resume_capture(self) -> None
def abort_capture(self) -> None
```
Sets the corresponding flags and notifies `capture_control` Condition.
T7 checks these flags after each save.

#### `generate_timing_report()`
```python
def generate_timing_report(self) -> None
```
Creates a DOCX timing report using `python-docx` containing
session metadata, per-exposure capture and save timing tables,
galvo movement summary, and efficiency metrics.
Saves to `<save_directory>/<timestamp>_capture_log.docx`.

---

### 11.13 NiDAQ Stimulator Methods

#### `open_nidaq_controller()`
```python
def open_nidaq_controller(self) -> None
```
Opens a new `tk.Toplevel` and passes it to `ForepawStimulatorApp.__init__()`.

#### `nidaq_start_stimulation()` / `nidaq_stop_stimulation()`
```python
def nidaq_start_stimulation(self) -> None
def nidaq_stop_stimulation(self) -> None
```
Used during multi-capture stimulation sync. Sets `stimulation_running`,
starts `_run_stimulation_pattern()` in T8, or stops it.

#### `_run_stimulation_pattern()`
```python
def _run_stimulation_pattern(self) -> None
```
Thread target (T8). Generates digital pulses on `Dev1/port0/line0`
using parameters from the NiDAQ Controller window.
Checks `stimulation_running` flag between each pulse.

---

### 11.14 Data Save Methods

#### `save_live_image()`
```python
def save_live_image(self) -> None
```
Saves `latest_live_frame` to `save_directory`:
- 16-bit PNG via `_save_png_16bit()`
- 8-bit PNG (normalised preview)
- MAT file with live image + metadata

#### `save_speckle_image()`
```python
def save_speckle_image(self) -> None
```
Saves the current speckle frame in three formats:
- 16-bit PNG
- MAT (speckle_contrast float + statistics)
- Viridis PNG via `plt.imsave(cmap='viridis', vmin=0, vmax=50)`

Also saves a second MAT containing the full raw frame stack.

#### `_save_png_16bit(frame, path)`
```python
def _save_png_16bit(self, frame: np.ndarray, path: str) -> bool
```
Primary: `Image.fromarray(frame.astype(np.uint16), mode='I;16').save(path)`
Fallback: `cv2.imwrite(str(path), frame.astype(np.uint16))`

**Returns:** `True` on success, `False` on error.

---

### 11.15 Utility and UI Methods

#### `float_to_registers(val)`
See [11.6 Galvo Methods](#116-galvo-methods).

#### `show_calibration_factors()`
```python
def show_calibration_factors(self) -> None
```
Shows `messagebox.showinfo` with reference px/mm values for standard
optical tube sizes (10, 25, 40, 50 mm).

#### `show_calibration_help()`
```python
def show_calibration_help(self) -> None
```
Shows `messagebox.showinfo` explaining the regression calibration method.

#### `browse_save_directory()`
```python
def browse_save_directory(self) -> None
```
Opens `filedialog.askdirectory()`. Validates write access via
`os.access(directory, os.W_OK)` before setting `save_directory`.

#### `show_detected_ports()`
```python
def show_detected_ports(self) -> None
```
Shows a dialog listing all detected COM ports and their assigned roles.

#### `show_specs()`
```python
def show_specs(self) -> None
```
Opens a 600×750 Toplevel with a scrollable read-only text area
showing full platform specifications (camera, algorithms, hardware, performance).

#### `show_help()`
```python
def show_help(self) -> None
```
Opens a 700×900 Toplevel with a scrollable user guide covering
all major features, workflow, and troubleshooting steps.

#### `show_about()`
```python
def show_about(self) -> None
```
Opens a 550×700 Toplevel with lab information, developer credits,
release date, version, and feature list. Loads `logo.png` if present.

#### `show_scrollable_message(title, message)`
```python
def show_scrollable_message(self, title: str, message: str) -> None
```
Generic utility: opens a 500×300 Toplevel with a `ScrolledText`
widget showing `message`. Read-only.

#### `exit_gui()`
```python
def exit_gui(self) -> None
```
Confirmation dialog → stops laser monitor thread → cancels Tk callbacks
→ puts `Commands.EXIT` on command_queue → joins camera process (timeout=5s)
→ terminates camera process if still alive → calls `root.quit()`.

---

## Entry Point

```python
if __name__ == "__main__":
    multiprocessing.set_start_method("spawn", force=True)
    root = tk.Tk()
    app = IntegratedGUI(root)
    root.mainloop()
```

`set_start_method("spawn", force=True)` must be called before any
`multiprocessing.Process` is created. `force=True` overrides any
previously set start method (relevant when running from an IDE that
may set `"fork"` by default on Linux).

---

*Document version: January 2026 · Platform v2.1*
