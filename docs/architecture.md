# docs/architecture.md — System Architecture

**LSCI Platform v2.1 · TEBI Lab · IIT Bombay**

This document covers the complete architectural design of the platform: the process model, inter-process communication, threading model, class structure, Modbus bus topology, data flow from camera sensor to saved file, and the shutdown sequence.

---

## Table of Contents

1. [Top-Level Design Philosophy](#1-top-level-design-philosophy)
2. [Process Model](#2-process-model)
3. [Inter-Process Communication IPC](#3-inter-process-communication-ipc)
4. [Threading Model](#4-threading-model)
5. [Class Structure and Relationships](#5-class-structure-and-relationships)
6. [Hardware Communication Layer](#6-hardware-communication-layer)
7. [Data Flow Camera to Screen](#7-data-flow-camera-to-screen)
8. [Data Flow Camera to Saved File](#8-data-flow-camera-to-saved-file)
9. [Speckle Contrast Computation Pipeline](#9-speckle-contrast-computation-pipeline)
10. [Calibration Architecture](#10-calibration-architecture)
11. [Key State Variables](#11-key-state-variables)
12. [Startup and Shutdown Sequences](#12-startup-and-shutdown-sequences)

---

## 1. Top-Level Design Philosophy

The platform is a single Python file application (new_GUI_sc_55.py, ~6,400 lines). This was a deliberate choice for deployment simplicity in a lab environment — no packages to install from the local repository, no build step, and the file is self-contained.

Three architectural decisions govern everything else:

**Decision 1 — Camera in a separate process, not a thread.**
The scTDC camera SDK is not thread-safe at the DLL level. Wrapping it in a multiprocessing.Process with start_method="spawn" gives it an entirely isolated Python interpreter and memory space. The GUI never calls any camera SDK function directly; it only sends commands and receives frames through multiprocessing queues.

**Decision 2 — A single shared Modbus client, protected by a lock.**
The Relay controller, Galvo X/Y motors, and Z-axis are physically wired to the same RS-485 bus and share one USB-serial adapter. Rather than opening three serial connections to the same port (which fails), one ModbusSerialClient instance is created at startup and shared among all three hardware systems. All writes are wrapped in with self.shared_modbus_lock to prevent concurrent Modbus transactions from corrupting bus framing.

**Decision 3 — All UI updates happen on the main thread.**
Background threads and the camera process are not permitted to write to Tkinter widgets directly. They communicate with the GUI via:
- data_queue.put(...) from the camera process to the frame listener thread
- self.root.after(0, lambda: widget.config(...)) from worker threads to the Tk event loop

---

## 2. Process Model

The application runs as two OS processes:

```
MAIN PROCESS (Python PID A)
  Tkinter event loop (root.mainloop())
  IntegratedGUI instance — all GUI widgets, hardware drivers, data logic
  Background threads (see Section 4)
  Modbus clients (relay/galvo/z-axis on shared bus, xyz stage)
  Serial thread (laser current monitor)

  command_queue (maxsize=10)  ──────────────────────►  CAMERA PROCESS
  data_queue    (maxsize=10)  ◄──────────────────────

CAMERA PROCESS (Python PID B — spawned at startup)
  camera_process(command_queue, data_queue)
  scTDC.scTDClib()
  scTDC.Camera(inifilepath="tdc_gpx3.ini", autoinit=False, lib=lib)
  Frame pipe (camera.add_frame_pipe())
  Blocking loop on command_queue.get()
```

**Why spawn and not fork?**
On Windows, fork is not available — spawn is the only valid start method. The code sets this explicitly:

```python
if __name__ == "__main__":
    multiprocessing.set_start_method("spawn", force=True)
```

spawn creates a clean Python interpreter with no inherited file handles or DLL state, which is critical because scTDClib.dll must be loaded fresh in the child process. The os.add_dll_directory(script_dir) call at module level ensures the DLL is found in the spawned child as well.

**Queue capacities:**
Both command_queue and data_queue are created with maxsize=10. This provides backpressure: if the GUI consumes frames too slowly, data_queue.put() in the camera process blocks rather than leaking memory. If no command arrives, the camera process blocks on command_queue.get() with no CPU spin.

---

## 3. Inter-Process Communication IPC

### 3.1 Commands (Main to Camera)

All commands use the Commands class as a namespace:

```python
class Commands:
    INITIALIZE_CAM    = 1   # One-shot: initialize SDK, create frame pipe
    START_PREVIEW     = 2   # Tuple: (Commands.START_PREVIEW, exposure_us)
    STOP_PREVIEW      = 3   # One-shot: break preview loop
    RUN_MULTI_CAPTURE = 4   # (orchestrated by GUI thread directly)
    EXIT              = 5   # One-shot: deinitialize SDK and exit process
    GET_SPECKLE_FRAME = 6   # Tuple: (Commands.GET_SPECKLE_FRAME, exposure_us, num_frames)
```

Command routing in the camera process:

```
command_queue.get()
    INITIALIZE_CAM      -> camera.initialize(), add_frame_pipe()
                        -> data_queue.put(("INFO", "Camera initialized..."))

    (START_PREVIEW, exp) -> loop:
                               camera.set_exposure_and_frames(exp, 1)
                               camera.do_measurement()
                               meta, img = pipe.read()
                               data_queue.put(("FRAME", img.copy()))
                               check for STOP_PREVIEW -> break

    (GET_SPECKLE_FRAME, exp, N)
                        -> camera.set_exposure_and_frames(exp, N)
                           collect N frames into stack[]
                           K2 = (std(stack) / (mean(stack) + 1e-6))^2 * 4095
                           data_queue.put(("SPECKLE", K2_uint16))

    EXIT                -> camera.deinitialize() -> break
```

### 3.2 Messages (Camera to Main)

| message_type | data | Handled by |
|---|---|---|
| "FRAME" | np.ndarray uint16 (1920x1440) | Frame listener -> live canvas |
| "SPECKLE" | np.ndarray uint16 (K2 map) | Frame listener -> speckle canvas |
| "INFO" | str | messagebox.showinfo + unlock preview button |
| "ERROR" | str | messagebox.showerror |
| "PROGRESS" | int (frame delta) | progress bar increment |

---

## 4. Threading Model

All threads are daemon threads (daemon=True) and terminate automatically with the main process.

```
MAIN PROCESS

[T0] Tkinter Event Loop  (main thread)
     Processes widget events, runs root.mainloop(), executes root.after() callbacks

[T1] Frame Listener Thread  (permanent — started in __init__)
     start_frame_listener() -> listen_for_frames()
     Blocks on data_queue.get()
     Dispatches: FRAME -> live canvas, SPECKLE -> speckle canvas,
                 INFO/ERROR -> messagebox, PROGRESS -> progress bar
     Stores: latest_live_frame (raw full-res), latest_speckle_frame,
             latest_speckle_stack (rolling buffer)

[T2] Laser Current Monitor Thread  (permanent — started in __init__)
     start_laser_monitor() -> read_laser_serial()
     serial.Serial(laser_port, 19200, timeout=1)
     Reads ASCII "Current: <float>" lines
     Puts value into laser_serial_queue (thread-safe Queue)
     Controlled by: laser_monitor_stop_event (threading.Event)
     UI side: update_laser_display() polls queue via root.after(1000)

[T3] Speckle Video Thread  (on-demand — while speckle_video_active)
     capture_speckle_video()
     Sends (GET_SPECKLE_FRAME, exposure, N) to command_queue
     Waits for ("SPECKLE", data) on data_queue
     Appends to latest_speckle_stack rolling buffer
     Computes 1/K^2 x contrast_factor -> viridis colormap -> uint8 RGB
     Pushes to speckle canvas via root.after(0, ...)

[T4] Galvo Move Threads  (transient — one per move command)
     send_galvo_motor1/2_direct(angle)
     Acquires shared_modbus_lock
     Writes 7 registers to client2 (address=0, slave=2)
     Float32 big-endian -> two 16-bit words
     Up to 3 retries with 0.4s, 0.6s, 0.8s increasing delay

[T5] XYZ Stage Move Threads  (transient — one per move command)
     _xyz_move_command(cmd)
     xyz_client.write_registers(address=1, values=[cmd], slave=1)
     No lock needed (separate Modbus client instance)

[T6] Z-axis Move Threads  (transient — one per move command)
     _zaxis_move_command(value)
     Acquires shared_modbus_lock
     client2.write_registers(address=6, values=[value], slave=2)

[T7] Multi-Capture Thread  (on-demand — one per capture session)
     Outer loop: data types {dark, baseline, stimulation}
     Middle loop: galvo grid points from coord_selector.matrix
     Inner loop: exposure times
     Uses capture_control (threading.Condition) for pause/abort
     8s total galvo settling per grid point (2+2+4 seconds)
     Sends GET_SPECKLE_FRAME -> waits for SPECKLE response
     Saves PNG + MAT per exposure per grid point per data type
     Generates DOCX timing report after all captures

[T8] NiDAQ Stimulation Thread  (on-demand)
     _run_stimulation_pattern()
     PyDAQmx.Task() on "Dev1/port0/line0"
     Busy-waits with time.sleep(0.001) for 1ms pulse precision
     Continuous ON/OFF while stimulation_running == True
     Stopped by nidaq_stop_stimulation() setting stimulation_running=False
```

**Thread-safety mechanisms:**

| Mechanism | Type | Guards |
|---|---|---|
| shared_modbus_lock | threading.Lock | All client2.write_registers() calls (T4, T6, relay toggle) |
| capture_control | threading.Condition | Pause/resume/abort for T7 |
| laser_monitor_stop_event | threading.Event | Clean shutdown signal to T2 |
| laser_serial_queue | queue.Queue | T2 to T0 handoff for laser current values |
| root.after(0, lambda: ...) | Tk callback | T3, T7, T8 to T0 widget mutations |
| command_queue, data_queue | multiprocessing.Queue | Main process <-> Camera process |

---

## 5. Class Structure and Relationships

```
new_GUI_sc_55.py

Commands                     Integer namespace for IPC commands
detect_com_ports()           Lists available COM ports via pyserial
find_device_by_description() Matches port to device by USB descriptor keyword
auto_detect_ports()          Returns {relay_galvo, xyz_stage, laser} -> COM port
camera_process()             Spawned child process — owns scTDC SDK exclusively

ConfigurationManager
    save_calibration()           -> JSON (numpy -> list)
    load_calibration()           <- JSON (list -> numpy)
    save_binary_calibration()    -> pickle .pkl
    load_binary_calibration()    <- pickle
    get_backup_files()           lists non-default .pkl/.json backups
    get_calibration_status()     returns dict {json_exists, pkl_exists, backups}

CalibrationManager               Toplevel dialog for backup/restore
    has-a ConfigurationManager
    holds-ref-to IntegratedGUI

CoordinateSelector               Galvo grid scan engine + calibration store
    matrix: np.ndarray (rows, cols, 2) — galvo angle grid
    grid_rows, grid_cols, matrix_type
    start(), stop(), resume()    grid scan control
    load_matrix()                receives result from generate_dynamic_calibration()
    save/load_calibration_to/from_file()
    update_roi_params()
    Auto-loads calibration on __init__

CalibrationTool                  Interactive ruler calibration (px/mm)
    activate()                   binds mouse draw on live_image_canvas
    deactivate()
    open_manual_angles()         opens ManualAngleInput dialog

ManualAngleInput                 Toplevel table for typing all galvo angles
    populates gui.manual_angle_points {(row,col): (angle_x, angle_y)}

IntensityProfileTool             Line-draw intensity analysis
    activate(mode)               mode = "live" or "speckle"
    deactivate()
    Exports: .mat, .png, .docx

ForepawStimulatorApp             NiDAQ pulse stimulator panel
    Owns Toplevel window
    run_stimulation()            runs complete session in its own thread
    State: total_sets, pulses_per_set, pulse_duration, inter_pulse_interval

IntegratedGUI                    Master application class
    __init__():
        auto_detect_ports()
        ModbusSerialClient(relay_galvo_port) -> self.client2
            self.relay_client = self.galvo_client = self.zaxis_client = client2
        shared_modbus_lock = threading.Lock()
        multiprocessing.Process(camera_process).start()
        IntensityProfileTool, CalibrationTool, CalibrationManager, CoordinateSelector
        start_laser_monitor()    -> T2
        create_widgets()
        start_frame_listener()   -> T1
```

---

## 6. Hardware Communication Layer

```
PC USB ports
  USB-Serial A (CP2102/FT232R)  ->  RS-485 Bus A (shared: relay + galvo + z-axis)
    auto-detected, fallback COM4        Relay Board   slave=2, addr=7
    ModbusSerialClient (client2)        Galvo X/Y     slave=2, addr=0
    9600 baud, 8N1, timeout=10s        Z-axis         slave=2, addr=6
    Protected by shared_modbus_lock

  USB-Serial B (CH340/Prolific)  ->  XYZ Stereotaxic Stage
    auto-detected, fallback COM3        slave=1, addr=1
    ModbusSerialClient (xyz_client)     9600 baud
    Separate client instance, no lock needed

  USB-Serial C (Arduino/CP2102)  ->  Laser Current Monitor
    auto-detected, fallback COM7        19200 baud ASCII
    serial.Serial                       format: "Current: 450.00\n"

  NI-DAQmx Card  ->  Forepaw Stimulator
    Dev1/port0/line0                    digital HIGH=1 / LOW=0
    PyDAQmx.Task
```

### 6.1 Modbus Register Map — Shared RS-485 Bus (Slave ID 2)

| Register Address | Device | Values Written | Effect |
|---|---|---|---|
| 0 | Galvo Motor X | [0, 3, HI_X, LO_X, 0, 0, 0] | Send float32 X angle; Y=0 |
| 0 | Galvo Motor Y | [0, 5, 0, 0, HI_Y, LO_Y, 0] | Send float32 Y angle; X=0 |
| 6 | Z-axis | [41] | Move camera focus up |
| 6 | Z-axis | [57] | Move camera focus down |
| 7 | Relay all 4 ch | [1, index, R0, R1, R2, R3, 0, 0] | Batch toggle, Ri in {0,1} |

### 6.2 Float32 Encoding for Galvo Angles

```python
def float_to_registers(self, val):
    packed = struct.pack('>f', val)           # 4 bytes, IEEE 754, big-endian
    return list(struct.unpack('>HH', packed)) # [high_word, low_word]
# Example: 15.5 degrees -> 0x41780000 -> [0x4178, 0x0000]
```

### 6.3 XYZ Stage Command Codes (Slave ID 1, Register Address 1)

| Axis | Direction | Fast | Slow | SV Precise |
|---|---|---|---|---|
| X | Right (+) | 137 | 138 | 140 |
| X | Left (-)  | 201 | 202 | 204 |
| Y | Up (+)    | 145 | 146 | 148 |
| Y | Down (-)  | 209 | 210 | 212 |
| Z | Up (+)    | 161 | 162 | 164 |
| Z | Down (-)  | 225 | 226 | 228 |

---

## 7. Data Flow Camera to Screen

Hot path executed for every frame during live preview:

```
CAMERA PROCESS                          MAIN PROCESS [T1]
camera.set_exposure_and_frames(exp, 1)
camera.do_measurement()
meta, img = pipe.read()                <- blocks (12-bit uint16, 1920x1440)
data_queue.put(("FRAME", img.copy()))
                                        |
                [1] latest_live_frame = data.copy()   <- preserve raw full-res
                [2] if roi_active: display_data = apply_roi(data)
                [3] normalize uint16 -> float [0,1]
                    apply contrast slider:
                        adjusted = 0.5 + (norm - 0.5) x contrast_factor
                        display_img = (adjusted x 255).astype(uint8)
                [4] resize_with_aspect_ratio(display_img, 900px, 670px)
                    -> resized_data (letterboxed canvas image)
                    -> scale (float: raw px / display px ratio)
                    -> padding_info (pad_left, pad_top, content_w, content_h)
                    stored in: self.display_scale, self.display_padding
                [5] draw_mm_ruler(resized_data)
                    -> horizontal ticks at: calibration_factor x (content_w/orig_w) px spacing
                    -> vertical ticks similarly
                    -> ROI dimension overlay in px and mm
                [6] Image.fromarray(data_with_ruler)
                    ImageTk.PhotoImage -> live_image_canvas.config(image=...)
```

**Invariant:** latest_live_frame is always the raw uint16 frame. Display modifications only occur on local copies. Save functions and analysis tools always see unmodified sensor data.

---

## 8. Data Flow Camera to Saved File

Path during an automated multi-capture session [T7]:

```
for data_type in [dark, baseline, stimulation]:
  for (angle_x, angle_y) in coord_selector.matrix:
      send_galvo_motor1_direct(angle_x)  sleep(2.0)
      send_galvo_motor2_direct(angle_y)  sleep(2.0)
      sleep(4.0)                         [8s total settle per grid point]

      for exposure in exposure_times:    [e.g. 500, 1000, 1500, 2000 us]
          exposure_time.set(exposure)
          update_exposure()              [restart preview with new exposure]
          sleep(1.0)                     [camera settle]

          command_queue.put((GET_SPECKLE_FRAME, exposure, N_frames))
          <- camera captures N frames, computes K2 = (std/mean)^2 x 4095
          <- data_queue receives ("SPECKLE", K2_uint16)
          frame = latest_speckle_frame_raw
          if roi_active: frame = apply_roi(frame)

          Save 16-bit PNG:  _save_png_16bit(frame, path)
                             PIL Image.fromarray(arr, mode='I;16').save()
                             fallback: cv2.imwrite()

          Save 8-bit preview PNG:
                             normalize uint16 -> uint8 -> Image.save()

          Save MAT:          scipy.io.savemat(path, {
                                 'speckle_contrast': frame,
                                 'exposure_us', 'num_frames', 'timestamp',
                                 'roi_active', 'roi_coords',
                                 'calibration_factor', 'capture_duration_s',
                                 'save_duration_s'})

          Log timing: capture_times, exposure_capture_times, image_save_times

After all captures:
  generate_timing_report() -> python-docx with:
      session metadata, per-exposure timing table (capture vs save),
      galvo movement timing, frame capture stats, efficiency metrics
  -> <save_dir>/YYYYMMDD_HHMMSS_capture_log.docx

Output structure:
  <save_dir>/
    dark_data/          dark_YYYYMMDD_HHMMSS_500us.png  + .mat
    base_data/          base_*.png + *.mat
    stimulation_data/   stim_*.png + *.mat
    *_capture_log.docx
```

---

## 9. Speckle Contrast Computation Pipeline

Two computation contexts with different formulas:

### 9.1 Camera Process — K-squared (GET_SPECKLE_FRAME)

Used for multi-capture and on-demand speckle captures.

```python
stack   = np.stack(frames, axis=0).astype(np.uint16)  # shape: (N, H, W)
mean    = np.mean(stack, axis=0)                        # shape: (H, W)
std     = np.std(stack,  axis=0)                        # shape: (H, W)
speckle = (((std / (mean + 1e-6)) ** 2) * 4095).astype(np.uint16)
#            K^2 scaled to 12-bit range
```

Epsilon 1e-6 prevents division by zero in dark or zero-intensity regions.

### 9.2 Main Process — Real-time Video (1/K-squared display)

Used for the speckle video canvas during start_speckle_video().

```python
stack    = np.stack(latest_speckle_stack[-N:], axis=0)
std_img  = np.nanstd(stack,  axis=0)
mean_img = np.nanmean(stack, axis=0)
K        = std_img / (mean_img + 1e-8)     # speckle contrast
C        = self.contrast_factor.get()       # C-axis slider 0.000-1.000
display  = (1.0 / K**2) * C               # 1/K^2 x C  (proportional to flow speed)
```

Display rendering from float display array:
- normalize min/max -> uint8
- apply viridis colormap (matplotlib.cm.viridis) -> RGB uint8
- resize to 900x670 via resize_with_aspect_ratio()
- push to speckle_image_canvas via root.after(0, ...)
- render_colorbar_image() draws vertical gradient with 10 tick labels alongside canvas

---

## 10. Calibration Architecture

### 10.1 Spatial Calibration (px/mm)

```
CalibrationTool.activate()
  -> binds mouse draw events on live_image_canvas
  -> user draws line over known physical feature
  -> dialog: "Enter real-world length in mm"
  -> calibration_factor = pixel_distance / real_mm (DoubleVar)
  -> applied in draw_mm_ruler() for ruler ticks and update_coords() for status bar

Not persisted to disk. Reference values from optics documentation:
  25mm tube: 148.8 px/mm    50mm tube: 295.5 px/mm
  40mm tube: 232.7 px/mm    10mm tube: 59.7-105.6 px/mm
```

### 10.2 Galvo Calibration (image pixel -> galvo angle, linear regression)

```
Data collected during calibration:
  image_points[i] = (pixel_x, pixel_y)   <- mouse clicks on live image
  galvo_points[i] = (angle_x, angle_y)   <- dial readings or manual entry

Regression (two independent LinearRegression models):
  X_train = np.column_stack(image_points)     shape (N, 2)
  model_X.fit(X_train, [p[0] for p in galvo_points])
  model_Y.fit(X_train, [p[1] for p in galvo_points])

  Galvo_X = b0 + b1*Image_X + b2*Image_Y
  Galvo_Y = g0 + g1*Image_X + g2*Image_Y

Grid generation (evenly spaced in image coordinate space):
  x_vals = np.linspace(x_min, x_max, cols)
  y_vals = np.linspace(y_min, y_max, rows)
  galvo_matrix[i,j] = (model_X.predict([[x,y]]), model_Y.predict([[x,y]]))

Persistence:
  ConfigurationManager.save_calibration()         -> JSON
  ConfigurationManager.save_binary_calibration()  -> pickle (.pkl)
  CoordinateSelector.__init__() auto-loads on startup (pkl preferred)

Backup/Restore via CalibrationManager dialog:
  Save Backup   -> copies .pkl/.json to <name>_<timestamp>.pkl/.json
  Restore       -> copies chosen backup -> galvo_calibration.pkl/.json
```

Four calibration modes all feed into generate_dynamic_calibration() -> LinearRegression.fit() -> coord_selector.load_matrix():

| Mode | Angle source | Image click source |
|---|---|---|
| Manual | ManualAngleInput table | User clicks each position |
| Dial-based | Physical galvo dials | User clicks beam spot in live image |
| Hybrid | Mix of pre-typed and dial-adjusted | User clicks all points |
| Auto Calibrate | Dials + optional manual_angle_points dict | User clicks all points |

---

## 11. Key State Variables

| Variable | Type | Purpose |
|---|---|---|
| latest_live_frame | np.ndarray uint16 | Raw full-res camera frame, never modified in-place |
| latest_speckle_frame | np.ndarray | ROI-cropped display speckle frame |
| latest_speckle_frame_raw | np.ndarray | Full-frame raw K2 output from camera process |
| latest_speckle_stack | list of np.ndarray | Rolling buffer for speckle video computation |
| previewing | bool | Live preview running state |
| speckle_video_active | bool | Speckle video thread running state |
| stimulation_running | bool | NiDAQ pulse pattern running state |
| galvo_enabled | bool | Gate checked by all galvo command functions |
| roi_active | tk.BooleanVar | Whether ROI crop is applied |
| roi_x1/y1/x2/y2 | tk.IntVar x4 | ROI boundaries in full-frame pixel space |
| calibration_factor | tk.DoubleVar | Spatial scale in px/mm |
| exposure_time | tk.IntVar | Camera exposure in microseconds |
| fps | tk.IntVar | Frame stack depth for speckle video |
| display_scale | float | Ratio: raw pixels -> display pixels (set per-frame) |
| display_padding | tuple | (pad_left, pad_top, content_w, content_h) in display px |
| capture_paused | bool | Pause flag read by T7 via capture_control |
| capture_abort | bool | Abort flag read by T7 via capture_control |
| capture_control | threading.Condition | Synchronizes pause/resume/abort with T7 |
| client2 | ModbusSerialClient | Shared Modbus bus (relay + galvo + z-axis) |
| shared_modbus_lock | threading.Lock | Serializes all writes to client2 |
| xyz_client | ModbusSerialClient | Separate Modbus client for XYZ stage |
| detected_ports | dict | Maps device names to COM port strings |
| image_points | list of tuple | Pixel coords collected during galvo calibration |
| galvo_points | list of tuple | Angle pairs collected during galvo calibration |
| relay_states | list of bool x4 | Current ON/OFF state per relay channel |
| manual_angle_points | dict | {(row,col): (angle_x, angle_y)} for hybrid calibration |

---

## 12. Startup and Shutdown Sequences

### 12.1 Startup Sequence

```
__main__ block
  [1] multiprocessing.set_start_method("spawn", force=True)
  [2] root = tk.Tk()
  [3] IntegratedGUI(root)
        [3a] os.add_dll_directory(script_dir)
        [3b] auto_detect_ports()  -> self.detected_ports
        [3c] ModbusSerialClient(relay_galvo_port, 9600).connect()
             self.relay_client = self.galvo_client = self.zaxis_client = client2
        [3d] multiprocessing.Process(camera_process).start()
        [3e] CoordinateSelector(self)  -> auto-loads calibration from disk
        [3f] create_widgets()          -> full widget tree built
        [3g] start_frame_listener()    -> T1 daemon thread
        [3h] start_laser_monitor()     -> T2 daemon thread
             root.after(1000, update_laser_display)
        [3i] root.after(1000, show_calibration_load_status)
  [4] root.mainloop()
```

### 12.2 Shutdown Sequence

```
User clicks "Exit GUI" -> messagebox.askokcancel("Quit")
  [1] laser_monitor_stop_event.set()       T2 exits its serial read loop
  [2] laser_serial_thread.join(timeout=1.0)
  [3] root.after_cancel(laser_after_id)    cancel repeating Tk callback
  [4] command_queue.put(Commands.EXIT)     camera process cleans up SDK
  [5] command_process.join(timeout=5)      wait up to 5s for clean exit
  [6] if still alive: command_process.terminate()
  [7] root.quit()                          exits mainloop()
  [8] All daemon threads T1-T8 terminate automatically with the process
```

Note: If the main process is killed externally (e.g., via Task Manager), the spawned camera process may become an orphan because Commands.EXIT was never sent. Terminate it manually via Task Manager or: taskkill /F /PID <camera_pid>. Normal application exit always goes through exit_gui() which runs the full sequence above.

---

*Document version: January 2026 - Platform v2.1*
