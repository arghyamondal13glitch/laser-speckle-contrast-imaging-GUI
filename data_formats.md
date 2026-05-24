# docs/data_formats.md — Data Formats Reference

**LSCI Platform v2.1 · TEBI Lab · IIT Bombay**

This document is the complete specification for every file format the
platform reads and writes — output images, MATLAB data files, timing
reports, intensity profiles, and calibration stores.

---

## Table of Contents

1. [File Type Overview](#1-file-type-overview)
2. [Output Directory Structure](#2-output-directory-structure)
3. [File Naming Conventions](#3-file-naming-conventions)
4. [16-bit PNG — Speckle Contrast Image](#4-16-bit-png--speckle-contrast-image)
5. [8-bit PNG — Preview Image](#5-8-bit-png--preview-image)
6. [Viridis PNG — False-Colour Speckle Preview](#6-viridis-png--false-colour-speckle-preview)
7. [MAT File — Multi-Capture Speckle](#7-mat-file--multi-capture-speckle)
8. [MAT File — Manual Speckle Save](#8-mat-file--manual-speckle-save)
9. [MAT File — Live Image Save](#9-mat-file--live-image-save)
10. [MAT File — Intensity Profile](#10-mat-file--intensity-profile)
11. [DOCX — Capture Timing Report](#11-docx--capture-timing-report)
12. [DOCX — Intensity Profile Report](#12-docx--intensity-profile-report)
13. [JSON — Galvo Calibration](#13-json--galvo-calibration)
14. [PKL — Galvo Calibration (Binary)](#14-pkl--galvo-calibration-binary)
15. [Reading Files in MATLAB](#15-reading-files-in-matlab)
16. [Reading Files in Python](#16-reading-files-in-python)
17. [File Size Reference](#17-file-size-reference)

---

## 1. File Type Overview

| Extension | Format | Written by | Purpose |
|---|---|---|---|
| `.png` (uint16) | 16-bit PNG, mode I;16 | Multi-capture, save_speckle_image | Primary speckle contrast data |
| `.png` (uint8) | 8-bit PNG, mode L or RGB | All save functions | Quick-view preview |
| `.png` (viridis) | 8-bit RGB, matplotlib imsave | save_speckle_image | False-colour publication image |
| `.mat` | MATLAB v5 binary | scipy.io.savemat | MATLAB-compatible data export |
| `.docx` | Office Open XML | python-docx | Timing logs, profile reports |
| `.json` | UTF-8 JSON text | ConfigurationManager | Human-readable calibration |
| `.pkl` | Python pickle, protocol 4 | ConfigurationManager | Binary calibration store |

---

## 2. Output Directory Structure

Every multi-capture session creates a timestamped folder tree inside
the chosen save directory:

```
<save_directory>/
└── (all files land directly in save_directory during a session)
    ├── dark_data/
    │   ├── dark_YYYYMMDD_HHMMSS_500us.png          16-bit speckle contrast
    │   ├── dark_YYYYMMDD_HHMMSS_500us_8bit.png     8-bit preview
    │   ├── dark_YYYYMMDD_HHMMSS_500us.mat           MATLAB struct
    │   ├── dark_YYYYMMDD_HHMMSS_1000us.png
    │   ├── dark_YYYYMMDD_HHMMSS_1000us_8bit.png
    │   ├── dark_YYYYMMDD_HHMMSS_1000us.mat
    │   └── ... (one triplet per exposure per grid point)
    │
    ├── base_data/
    │   └── base_YYYYMMDD_HHMMSS_<exp>us.png / _8bit.png / .mat
    │
    ├── stimulation_data/
    │   └── stim_YYYYMMDD_HHMMSS_<exp>us.png / _8bit.png / .mat
    │
    └── YYYYMMDD_HHMMSS_capture_log.docx            timing report

Manual saves (not part of multi-capture) go directly into save_directory:
    ├── live_YYYYMMDD_HHMMSS.png
    ├── live_YYYYMMDD_HHMMSS_8bit.png
    ├── live_YYYYMMDD_HHMMSS.mat
    ├── speckle_YYYYMMDD_HHMMSS.png
    ├── speckle_YYYYMMDD_HHMMSS.mat
    ├── speckle_YYYYMMDD_HHMMSS_viridis.png
    ├── profile_YYYYMMDD_HHMMSS.mat
    ├── profile_YYYYMMDD_HHMMSS.png
    └── profile_YYYYMMDD_HHMMSS.docx

Calibration files (always in project root directory):
    ├── galvo_calibration.pkl
    ├── galvo_calibration_config.json
    ├── <backup_name>.pkl
    └── <backup_name>.json
```

---

## 3. File Naming Conventions

### Multi-capture files

```
{prefix}_{timestamp}_{exposure}us.png
{prefix}_{timestamp}_{exposure}us_8bit.png
{prefix}_{timestamp}_{exposure}us.mat
```

| Token | Values | Example |
|---|---|---|
| `{prefix}` | `dark_`, `base_`, `stim_` | `dark_` |
| `{timestamp}` | `YYYYMMDD_HHMMSS` | `20260115_143022` |
| `{exposure}` | Integer microseconds | `1500` |

Full example: `dark_20260115_143022_1500us.png`

When multiple grid points are captured, each point creates its own
set of files with the timestamp advancing by a few seconds between
each point. Grid point number is not embedded in the filename — files
are distinguished by their timestamps, which increase monotonically
through the grid scan.

### Manual save files

```
live_{timestamp}.png
live_{timestamp}_8bit.png
live_{timestamp}.mat
speckle_{timestamp}.png
speckle_{timestamp}.mat
speckle_{timestamp}_viridis.png
```

### Timing report

```
{timestamp}_capture_log.docx
```

The timestamp on the report is the session start time — earlier than
any individual file timestamp within the session.

### Profile files

```
profile_{timestamp}.mat
profile_{timestamp}.png
profile_{timestamp}.docx
```

### Calibration files

```
galvo_calibration.pkl               (primary binary, always in project root)
galvo_calibration_config.json       (primary JSON, always in project root)
{backup_name}.pkl                   (backup, project root)
{backup_name}.json                  (backup, project root)
```

Backup names are set by the user in the Calibration Management window.
Default: `calibration_backup_YYYYMMDD_HHMMSS`.

---

## 4. 16-bit PNG — Speckle Contrast Image

**Written by:** `_save_png_16bit(frame, path)`

**Pixel values:** uint16, range 0–4095 (12-bit data in a 16-bit container)

**Content:** The raw K² speckle contrast map, scaled to 12-bit:
```
value[px] = ((std / (mean + 1e-6))² × 4095).astype(uint16)
```
Higher values = higher K² = lower flow speed.
Lower values = lower K² = higher flow speed.

**Save method:**

```python
# Primary: PIL I;16 mode (little-endian 16-bit pixels)
img = Image.fromarray(frame.astype(np.uint16), mode='I;16')
img.save(path)

# Fallback (if PIL fails): OpenCV
cv2.imwrite(str(path), frame.astype(np.uint16))
```

**Dimensions:** Full sensor resolution 1920×1440 px,
or ROI dimensions if ROI was active at capture time.

**Colour space:** Greyscale (single channel). No colour table embedded.

**Metadata:** None stored inside the PNG. All metadata is in the
companion .mat file.

**Opening in ImageJ/FIJI:**
- File → Open → select .png
- Set type to 16-bit if not auto-detected
- Image → Adjust → Brightness/Contrast to visualise dynamic range
- Note: values in the 0–4095 range will appear very dark in a 16-bit
  0–65535 display without contrast adjustment

**Opening in MATLAB:**
```matlab
img = imread('dark_20260115_143022_1500us.png');  % uint16
imagesc(img); colormap gray; colorbar;
```

---

## 5. 8-bit PNG — Preview Image

**Written by:** Multi-capture save loop, save_live_image, save_speckle_image

**Pixel values:** uint8, range 0–255

**Content (multi-capture / speckle):**
The 16-bit speckle contrast map linearly scaled to 8-bit:
```python
preview = (frame / 4095.0 * 255).astype(np.uint8)
```

**Content (live image):**
The raw 12-bit live frame linearly scaled to 8-bit:
```python
preview = (frame / 4095.0 * 255).astype(np.uint8)
```

**Purpose:** Quick visual check without needing 16-bit-aware software.
These files are not suitable for quantitative analysis — use the .mat
or 16-bit .png for that.

**Filename suffix:** `_8bit.png`

---

## 6. Viridis PNG — False-Colour Speckle Preview

**Written by:** `save_speckle_image()` via `plt.imsave()`

**Pixel values:** uint8 RGB (3 channels)

**Content:** The 1/K² speckle display array rendered through the
viridis colormap, normalised to the display range (vmin=0, vmax=50):
```python
plt.imsave(path, speckle_img, cmap='viridis', vmin=0, vmax=50)
```

**Purpose:** Publication-ready false-colour image. Colours directly
correspond to the on-screen display (viridis, yellow = high 1/K² = fast flow).

**Filename suffix:** `_viridis.png`

**Note:** The vmax=50 normalisation is a fixed display parameter, not an
adaptive scale. Regions where 1/K² > 50 will be clipped to yellow. Adjust
vmax in the source if your data has a different dynamic range.

---

## 7. MAT File — Multi-Capture Speckle

**Written by:** Multi-capture thread via `scipy.io.savemat()`

**File:** `dark_YYYYMMDD_HHMMSS_1500us.mat` (or `base_`, `stim_` prefix)

**MATLAB struct name:** `data` (the variable is named `data` when loaded)

**Fields:**

| Field name | MATLAB type | Shape | Description |
|---|---|---|---|
| `speckle_contrast` | uint16 | (H, W) | Raw K² map, values 0–4095 |
| `exposure_us` | double | scalar | Exposure time in microseconds |
| `num_frames` | double | scalar | Number of stacked frames (N) |
| `timestamp` | char | string | ISO 8601 datetime of capture |
| `roi_active` | logical | scalar | 1 if ROI was applied, 0 if not |
| `roi_coords` | double | (1, 4) | [x1, y1, x2, y2] in full-frame px |
| `calibration_factor` | double | scalar | Scale in pixels per millimetre |
| `capture_duration_s` | double | scalar | Duration of frame capture in seconds |
| `save_duration_s` | double | scalar | Duration of file save in seconds |
| `data_type` | char | string | One of: 'dark', 'baseline', 'stimulation' |
| `grid_point` | double | (1, 2) | [row, col] index of galvo grid position (0-indexed) |
| `galvo_angle` | double | (1, 2) | [angle_x, angle_y] in degrees at this grid point |

**MATLAB load example:**
```matlab
s = load('dark_20260115_143022_1500us.mat');
img = s.data.speckle_contrast;    % uint16 array
exp = s.data.exposure_us;         % e.g. 1500
imagesc(double(img)); colormap hot; colorbar;
title(sprintf('K^2 map  |  %d us  |  %s', exp, s.data.timestamp));
```

**Coordinate note:** When ROI is active, `speckle_contrast` has shape
(roi_height, roi_width), not (1440, 1920). The full-frame coordinates
of the ROI region are given in `roi_coords`.

---

## 8. MAT File — Manual Speckle Save

**Written by:** `save_speckle_image()` via `scipy.io.savemat()`

This produces two .mat files in a single save operation:

### File 1: Primary speckle .mat

**Filename:** `speckle_YYYYMMDD_HHMMSS.mat`

**Fields:**

| Field name | Type | Shape | Description |
|---|---|---|---|
| `speckle_contrast` | float32 | (H, W) | 1/K² × C_factor (display values) |
| `speckle_contrast_raw` | uint16 | (H, W) | Normalised uint16 version |
| `timestamp` | char | string | ISO 8601 datetime |
| `roi_active` | logical | scalar | ROI state at save time |
| `roi_coords` | double | (1, 4) | [x1, y1, x2, y2] |
| `calibration_factor` | double | scalar | px/mm |
| `contrast_factor` | double | scalar | C-axis slider value at save time |
| `mean_speckle` | double | scalar | Mean of speckle_contrast array |
| `std_speckle` | double | scalar | Std dev of speckle_contrast array |
| `min_speckle` | double | scalar | Minimum value |
| `max_speckle` | double | scalar | Maximum value |

### File 2: Raw frame stack .mat

**Filename:** `speckle_YYYYMMDD_HHMMSS_raw_stack.mat`

**Fields:**

| Field name | Type | Shape | Description |
|---|---|---|---|
| `raw_stack` | uint16 | (N, H, W) | All N raw live frames used for speckle computation |
| `num_frames` | double | scalar | N |
| `timestamp` | char | string | ISO 8601 datetime |
| `exposure_us` | double | scalar | Exposure at time of capture |

**MATLAB load example:**
```matlab
s = load('speckle_20260115_143022.mat');
K2_inv = s.speckle_contrast;          % float32, 1/K^2 x C
imagesc(K2_inv); colormap viridis; colorbar;

s2 = load('speckle_20260115_143022_raw_stack.mat');
raw = s2.raw_stack;                   % uint16, shape (N, H, W)
K2 = (std(double(raw), 0, 1) ./ (mean(double(raw), 1) + 1e-6)).^2;
```

---

## 9. MAT File — Live Image Save

**Written by:** `save_live_image()` via `scipy.io.savemat()`

**Filename:** `live_YYYYMMDD_HHMMSS.mat`

**Fields:**

| Field name | Type | Shape | Description |
|---|---|---|---|
| `live_image` | uint16 | (H, W) | Raw 12-bit frame, values 0–4095 |
| `timestamp` | char | string | ISO 8601 datetime |
| `exposure_us` | double | scalar | Exposure time in microseconds |
| `roi_active` | logical | scalar | ROI state at save time |
| `roi_coords` | double | (1, 4) | [x1, y1, x2, y2] |
| `calibration_factor` | double | scalar | px/mm |

**MATLAB load example:**
```matlab
s = load('live_20260115_143022.mat');
img = s.live_image;                   % uint16 raw 12-bit
figure; imagesc(double(img)); colormap gray; colorbar;
title(sprintf('Live frame | %d us', s.exposure_us));
```

---

## 10. MAT File — Intensity Profile

**Written by:** `IntensityProfileTool.save_profile()` via `scipy.io.savemat()`

**Filename:** `profile_YYYYMMDD_HHMMSS.mat`

**Fields:**

| Field name | Type | Shape | Description |
|---|---|---|---|
| `distances` | double | (N,) | Pixel distance from start point along line (0 to line_length) |
| `intensities` | double | (N,) | Pixel intensity value at each sample point |
| `start_x` | double | scalar | Line start X in image coordinates |
| `start_y` | double | scalar | Line start Y in image coordinates |
| `end_x` | double | scalar | Line end X in image coordinates |
| `end_y` | double | scalar | Line end Y in image coordinates |
| `line_length` | double | scalar | Euclidean pixel distance from start to end |
| `num_points` | double | scalar | Number of sample points N (= round(line_length)) |
| `mode` | char | string | 'live' or 'speckle' |
| `mean_intensity` | double | scalar | Mean of intensities array |
| `std_intensity` | double | scalar | Std dev of intensities array |
| `max_intensity` | double | scalar | Maximum intensity along line |
| `min_intensity` | double | scalar | Minimum intensity along line |
| `timestamp` | char | string | ISO 8601 datetime of save |

**Sampling method:**
Points are sampled by linear interpolation along the Bresenham-approximated
line using integer pixel indices:
```python
x_coords = np.linspace(start_x, end_x, num_points, dtype=int)
y_coords = np.linspace(start_y, end_y, num_points, dtype=int)
intensities = [image[y, x] for x, y in zip(x_coords, y_coords)]
```

Out-of-bounds points are assigned intensity 0.

**MATLAB load example:**
```matlab
s = load('profile_20260115_143022.mat');
plot(s.distances, s.intensities, 'b-', 'LineWidth', 2);
xlabel('Distance (px)'); ylabel('Intensity');
title(sprintf('Profile: mean=%.1f  std=%.1f', s.mean_intensity, s.std_intensity));
```

---

## 11. DOCX — Capture Timing Report

**Written by:** `generate_timing_report()` via `python-docx`

**Filename:** `YYYYMMDD_HHMMSS_capture_log.docx`

**Triggered by:** Completion (or abort) of a multi-capture session.

**Content structure:**

```
Document title: "Multi-Capture Timing Analysis Report"

Section 1 — Session Summary
    Date and time of session
    Total session duration (HH:MM:SS)
    Grid configuration (rows × cols)
    Data types collected (dark / baseline / stimulation)
    Exposure times list (µs)
    Frames per exposure
    ROI status and coordinates
    Save directory path

Section 2 — Per-Exposure Capture Statistics (table)
    Columns: Exposure (µs) | Captures | Mean Duration (s) | Std Dev (s) | FPS achieved
    One row per unique exposure time

Section 3 — Galvo Movement Timing
    Number of grid points visited
    Mean galvo move time (s)
    Total galvo time (s)
    Percentage of session time spent moving galvo

Section 4 — Image Save Statistics (table)
    Columns: Exposure (µs) | Mean Save Time (s) | Total Save Time (s)
    One row per unique exposure time

Section 5 — Frame Capture Statistics
    Total frames requested
    Total frames captured
    Total capture time (s)
    Overall capture FPS across session

Section 6 — Performance Efficiency Metric
    Capture time / Total session time × 100 = % efficiency
    Interpretation: higher = more time in capture, less in overhead

Section 7 — Raw Timing Data (table, one row per individual capture)
    Columns: Data type | Grid point | Exposure (µs) | Duration (s) | FPS | Save time (s)
```

**Opening:** Any version of Microsoft Word, LibreOffice Writer,
or Google Docs (upload) can open .docx files.

**Programmatic reading in Python:**
```python
from docx import Document
doc = Document('20260115_143022_capture_log.docx')
for para in doc.paragraphs:
    print(para.text)
for table in doc.tables:
    for row in table.rows:
        print([cell.text for cell in row.cells])
```

---

## 12. DOCX — Intensity Profile Report

**Written by:** `IntensityProfileTool.save_profile()` via `python-docx`

**Filename:** `profile_YYYYMMDD_HHMMSS.docx`

**Content structure:**

```
Document title: "Intensity Profile Report — {mode} Image"

Section 1 — Profile Metadata
    Date and time
    Mode (Live / Speckle Contrast)
    Line coordinates: ({start_x},{start_y}) → ({end_x},{end_y})
    Line length: {line_length:.1f} pixels
    Number of sample points: {num_points}
    ROI active: Yes/No

Section 2 — Statistical Summary (table)
    Metric         | Value
    ───────────────┼─────────
    Mean           | {mean:.2f}
    Std Dev        | {std:.2f}
    Maximum        | {max}
    Minimum        | {min}
    Range          | {max-min}
    Coefficient of | {std/mean × 100:.1f}%
    Variation      |

Section 3 — Embedded Plot
    The matplotlib two-panel figure (intensity profile + histogram)
    embedded as an inline image inside the document.
```

---

## 13. JSON — Galvo Calibration

**Written by:** `ConfigurationManager.save_calibration()`

**Filename:** `galvo_calibration_config.json` (default)
or `{backup_name}.json` (backups)

**Encoding:** UTF-8, human-readable indented JSON.

**Schema:**

```json
{
    "galvo_matrix": [
        [[-12.34, 9.87], [-0.12, 9.91], [12.10, 9.89]],
        [[-12.38, 0.03], [-0.08, -0.01], [12.06, 0.00]],
        [[-12.41, -9.81], [-0.09, -9.93], [12.03, -9.86]]
    ],
    "grid_rows": 3,
    "grid_cols": 3,
    "matrix_type": "regression",
    "roi_offset_x": 0,
    "roi_offset_y": 0,
    "roi_width": 1920,
    "roi_height": 1440,
    "timestamp": "2026-01-15T14:30:22.453821",
    "camera_resolution": [1920, 1440]
}
```

**Field descriptions:**

| Field | Type | Description |
|---|---|---|
| `galvo_matrix` | 3D nested list [rows][cols][2] | galvo_matrix[i][j] = [angle_X, angle_Y] in degrees |
| `grid_rows` | int | Number of rows in grid (2 or 3) |
| `grid_cols` | int | Number of cols in grid (2 or 3) |
| `matrix_type` | string | "regression" (from Auto Calibrate) or "hybrid" (from Hybrid Calibration) |
| `roi_offset_x` | int | roi_x1 at time of calibration (0 if no ROI) |
| `roi_offset_y` | int | roi_y1 at time of calibration (0 if no ROI) |
| `roi_width` | int | ROI width at calibration time (1920 if no ROI) |
| `roi_height` | int | ROI height at calibration time (1440 if no ROI) |
| `timestamp` | string | ISO 8601 with microseconds |
| `camera_resolution` | list [int, int] | Fixed: [1920, 1440] |

**Serialisation note:** `galvo_matrix` is a `numpy.ndarray` in memory.
During save it is converted to a Python nested list via `.tolist()` for
JSON compatibility. During load it is converted back to `numpy.ndarray`
via `np.array(data["galvo_matrix"])`.

**Manually editing the JSON:**
The JSON file can be edited in any text editor to correct individual
angles. After editing, restart the application — it will load the
edited file at startup. Ensure all angle values remain in the range
[-40.0, 40.0] and that the list dimensions match grid_rows × grid_cols × 2.

---

## 14. PKL — Galvo Calibration (Binary)

**Written by:** `ConfigurationManager.save_binary_calibration()`

**Filename:** `galvo_calibration.pkl` (default)
or `{backup_name}.pkl` (backups)

**Format:** Python pickle, protocol 4 (compatible with Python 3.8+)

**Serialised object:** A Python `dict` with the same keys as the JSON
format (Section 13), but with `galvo_matrix` stored as a `numpy.ndarray`
(not a list). This means the PKL file preserves the numpy dtype (float64)
and shape (rows, cols, 2) exactly without conversion.

**Load precedence:** On application startup, `CoordinateSelector.__init__()`
tries to load the PKL file first. If it does not exist or is corrupt, it
falls back to the JSON. The PKL is always preferred because it avoids the
numpy array serialisation/deserialisation round-trip.

**Corruption risk:** Pickle files are sensitive to Python version changes.
If you upgrade Python (e.g., 3.10 → 3.11), the PKL will still load because
the dict and numpy protocol-4 format is forward-compatible. However, if the
numpy version changes significantly and the array dtype format changes, the
PKL may fail to load. In that case, the JSON fallback will succeed.

**Do not edit PKL files directly.** Use the JSON file for manual edits.

**Programmatic inspection:**
```python
import pickle, numpy as np

with open('galvo_calibration.pkl', 'rb') as f:
    calib = pickle.load(f)

print(calib.keys())
print(calib['grid_rows'], 'x', calib['grid_cols'])
print('Matrix shape:', calib['galvo_matrix'].shape)
print('Matrix:\n', calib['galvo_matrix'])
```

---

## 15. Reading Files in MATLAB

### 16-bit PNG

```matlab
% Read 16-bit speckle contrast PNG
img = imread('dark_20260115_143022_1500us.png');   % uint16, shape [H W]
% Values range 0-4095 (K^2 scaled to 12-bit)

% Display with correct contrast
figure;
imagesc(double(img), [0 4095]);
colormap hot; colorbar;
title('Speckle Contrast K^2');

% Compute K (speckle contrast scalar)
K2 = double(img) / 4095;    % normalise to 0-1
```

### MAT file — multi-capture

```matlab
s = load('dark_20260115_143022_1500us.mat');
img = s.data.speckle_contrast;          % uint16
exp_us = s.data.exposure_us;           % double scalar
t = s.data.timestamp;                  % char string
roi = s.data.roi_coords;               % [x1 y1 x2 y2]

% Convert to K (speckle contrast)
K2_norm = double(img) / 4095;          % normalised K^2
K = sqrt(K2_norm);                     % speckle contrast K

figure;
subplot(1,2,1); imagesc(double(img)); colormap hot; colorbar; title('K^2 raw');
subplot(1,2,2); imagesc(K, [0 1]);    colormap hot; colorbar; title('K');
sgtitle(sprintf('%d us | %s', exp_us, t));
```

### Batch loading a session

```matlab
% Load all dark captures at 1500us
files = dir('dark_data/dark_*_1500us.mat');
for i = 1:length(files)
    s = load(fullfile('dark_data', files(i).name));
    stack(:,:,i) = double(s.data.speckle_contrast);
end
% stack is now (H, W, N) where N = number of grid points

mean_img = mean(stack, 3);
figure; imagesc(mean_img); colormap hot; colorbar;
title('Mean K^2 across grid points');
```

### Intensity profile

```matlab
s = load('profile_20260115_143022.mat');
figure;
plot(s.distances, s.intensities, 'b-', 'LineWidth', 1.5);
xlabel('Distance (pixels)');
ylabel('Intensity');
title(sprintf('Profile: mean=%.1f, std=%.1f', s.mean_intensity, s.std_intensity));
grid on;
```

---

## 16. Reading Files in Python

### 16-bit PNG

```python
import numpy as np
from PIL import Image

# Method 1: PIL (preferred — preserves uint16 exactly)
img = np.array(Image.open('dark_20260115_143022_1500us.png'))
# dtype: uint16, shape: (H, W), values: 0-4095
print(f"Shape: {img.shape}, dtype: {img.dtype}, max: {img.max()}")

# Method 2: OpenCV
import cv2
img_cv = cv2.imread('dark_20260115_143022_1500us.png', cv2.IMREAD_UNCHANGED)
# Note: OpenCV returns uint16 for 16-bit PNG with IMREAD_UNCHANGED

# Display with matplotlib
import matplotlib.pyplot as plt
plt.figure()
plt.imshow(img, cmap='hot', vmin=0, vmax=4095)
plt.colorbar(label='K² (scaled)')
plt.title('Speckle Contrast Map')
plt.show()
```

### MAT file

```python
import scipy.io as sio
import numpy as np

s = sio.loadmat('dark_20260115_143022_1500us.mat', squeeze_me=True)
# squeeze_me=True removes singleton dimensions

img = s['data']['speckle_contrast'].item()      # uint16 array
exp = float(s['data']['exposure_us'].item())    # scalar
ts  = str(s['data']['timestamp'].item())        # string

print(f"Exposure: {exp} µs  |  Shape: {img.shape}  |  Time: {ts}")

import matplotlib.pyplot as plt
plt.imshow(img.astype(float), cmap='inferno', vmin=0, vmax=4095)
plt.colorbar(); plt.title(f'K² @ {exp} µs'); plt.show()
```

### Batch processing in Python

```python
import scipy.io as sio
import numpy as np
from pathlib import Path

data_dir = Path('dark_data')
files = sorted(data_dir.glob('dark_*_1500us.mat'))

stack = []
for f in files:
    s = sio.loadmat(str(f), squeeze_me=True)
    stack.append(s['data']['speckle_contrast'].item().astype(np.float32))

stack = np.stack(stack, axis=0)       # shape: (N, H, W)
mean_map = stack.mean(axis=0)         # mean K² across grid points
std_map  = stack.std(axis=0)          # spatial variability

import matplotlib.pyplot as plt
fig, axes = plt.subplots(1, 2, figsize=(12, 5))
axes[0].imshow(mean_map, cmap='hot'); axes[0].set_title('Mean K²'); 
axes[1].imshow(std_map,  cmap='hot'); axes[1].set_title('Std K²')
plt.tight_layout(); plt.show()
```

### JSON calibration

```python
import json
import numpy as np

with open('galvo_calibration_config.json', 'r') as f:
    calib = json.load(f)

matrix = np.array(calib['galvo_matrix'])   # shape: (rows, cols, 2)
rows = calib['grid_rows']
cols = calib['grid_cols']

print(f"Grid: {rows}×{cols}")
for i in range(rows):
    for j in range(cols):
        ax, ay = matrix[i, j]
        print(f"  ({i+1},{j+1}): X={ax:.2f}°, Y={ay:.2f}°")
```

### PKL calibration

```python
import pickle
import numpy as np

with open('galvo_calibration.pkl', 'rb') as f:
    calib = pickle.load(f)

matrix = calib['galvo_matrix']       # numpy float64 (rows, cols, 2)
print(f"Loaded: {calib['grid_rows']}×{calib['grid_cols']}")
print(f"Type: {calib['matrix_type']}")
print(f"Saved at: {calib['timestamp']}")
```

---

## 17. File Size Reference

Approximate file sizes per capture (1920×1440 full frame, ROI-inactive):

| File | Approximate size |
|---|---|
| 16-bit PNG (lossless, greyscale) | 1.5–3.5 MB (varies with content entropy) |
| 8-bit PNG preview | 0.4–1.2 MB |
| Viridis PNG (RGB) | 1.5–4.0 MB |
| MAT — multi-capture (single frame) | ~5.6 MB (uint16, 1920×1440 + metadata) |
| MAT — speckle with raw stack (N=10) | ~56 MB (uint16 stack) |
| MAT — live image | ~5.5 MB |
| MAT — intensity profile (1000 pts) | <0.1 MB |
| DOCX — timing report | 20–80 KB |
| DOCX — profile report with plot | 150–400 KB |
| JSON — calibration | <5 KB |
| PKL — calibration | <10 KB |

**Storage planning (multi-capture session, 3×3 grid, 6 exposures):**

Per data type (dark / baseline / stimulation):
- 9 grid points × 6 exposures = 54 captures
- Each capture: 16-bit PNG (~2.5 MB) + 8-bit PNG (~0.8 MB) + MAT (~5.6 MB) = ~9 MB
- 54 × 9 MB = ~486 MB per data type
- Three data types + timing report ≈ ~1.5 GB per complete session

Use an SSD for write performance. At 5.5 MB per frame × 100 frames per
capture = ~550 MB/s write burst is typical. An SSD with sustained write
speed >200 MB/s is recommended for sessions with N_frames ≥ 100 and 6+
exposure times.

---

*Document version: January 2026 - Platform v2.1*
