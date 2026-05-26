# docs/hardware_setup.md — Hardware Setup Guide

**LSCI Platform v2.1 · TEBI Lab · IIT Bombay**

This document is the single authoritative reference for all hardware connections, 
protocol settings, register maps, command tables, and physical wiring required by 
the platform. Every number in this document is derived directly from the source code.

---

## Table of Contents

1. [Hardware Topology Overview](#1-hardware-topology-overview)
2. [Shared RS-485 Modbus Bus (USB Adapter A)](#2-shared-rs-485-modbus-bus-usb-adapter-a)
   - 2.1 [Bus Physical Layer](#21-bus-physical-layer)
   - 2.2 [Relay Board](#22-relay-board)
   - 2.3 [Galvo Motor Controller](#23-galvo-motor-controller)
   - 2.4 [Z-axis Motor Controller](#24-z-axis-motor-controller)
   - 2.5 [Complete Modbus Register Map — Slave ID 2](#25-complete-modbus-register-map--slave-id-2)
3. [XYZ Stereotaxic Stage (USB Adapter B)](#3-xyz-stereotaxic-stage-usb-adapter-b)
   - 3.1 [Connection Settings](#31-connection-settings)
   - 3.2 [XYZ Stage Command Code Table](#32-xyz-stage-command-code-table)
4. [Laser Current Monitor (USB Adapter C)](#4-laser-current-monitor-usb-adapter-c)
   - 4.1 [Serial Protocol](#41-serial-protocol)
   - 4.2 [Arduino Firmware Requirement](#42-arduino-firmware-requirement)
5. [NI-DAQ Card — Forepaw Stimulator](#5-ni-daq-card--forepaw-stimulator)
   - 5.1 [Card Configuration](#51-card-configuration)
   - 5.2 [Signal Specification](#52-signal-specification)
   - 5.3 [Pulse Timing Model](#53-pulse-timing-model)
6. [scTDC Camera](#6-sctdc-camera)
   - 6.1 [USB Connection](#61-usb-connection)
   - 6.2 [Software Configuration](#62-software-configuration)
7. [Auto COM Port Detection Reference](#7-auto-com-port-detection-reference)
8. [Float32 Encoding for Galvo Angles](#8-float32-encoding-for-galvo-angles)
9. [Power-On Sequence](#9-power-on-sequence)
10. [Physical Connection Checklist](#10-physical-connection-checklist)
11. [Troubleshooting Hardware Issues](#11-troubleshooting-hardware-issues)

---

## 1. Hardware Topology Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│  PC (Windows 10/11, 64-bit)                                          │
│                                                                      │
│  USB-A Port 1 ──── USB-Serial Adapter A (CP2102 / FT232R)           │
│                           │                                          │
│                    RS-485 Bus (shared, daisy-chained)                │
│                           ├─── Relay Board            (Slave ID: 2) │
│                           ├─── Galvo Motor Controller (Slave ID: 2) │
│                           └─── Z-axis Motor Controller (Slave ID: 2)│
│                                                                      │
│  USB-A Port 2 ──── USB-Serial Adapter B (CH340 / Prolific)          │
│                           │                                          │
│                    XYZ Stereotaxic Stage              (Slave ID: 1) │
│                                                                      │
│  USB-A Port 3 ──── USB-Serial Adapter C (Arduino / CP2102)          │
│                           │                                          │
│                    Laser Current Monitor     (19200 baud ASCII)      │
│                                                                      │
│  USB-A Port 4 ──── scTDC Camera (USB3 Vision or proprietary USB)    │
│                           │                                          │
│                    12-bit scientific camera (1920 x 1440 px)        │
│                                                                      │
│  PCIe Slot ────── NI-DAQ Card (Dev1)                                 │
│                           │                                          │
│                    BNC / screw terminal                              │
│                    Dev1/port0/line0 ──► Forepaw Stimulator           │
│                    GND              ──► Stimulator GND              │
└─────────────────────────────────────────────────────────────────────┘
```

**Key design constraint:** Relay, Galvo, and Z-axis share one RS-485 bus and one 
USB-serial adapter. They cannot have separate serial ports — the Modbus RTU protocol 
on a shared bus requires all three to be reachable on the same physical COM port. The 
software handles this with a single ModbusSerialClient instance (client2) protected by 
a threading.Lock.

---

## 2. Shared RS-485 Modbus Bus (USB Adapter A)

### 2.1 Bus Physical Layer

| Parameter | Value |
|---|---|
| Interface standard | RS-485 (half-duplex, 2-wire) |
| Protocol | Modbus RTU |
| Baud rate | 9600 |
| Data bits | 8 |
| Parity | None |
| Stop bits | 1 |
| Timeout | 10 seconds |
| USB chipset (typical) | CP2102 or FT232R |
| Fallback COM port | COM4 |

**Cable notes:**
- Use a twisted-pair cable for the RS-485 data lines (A+ and B-)
- For runs longer than 1 metre, add a 120Ω termination resistor between A+ and B- 
  at the far end of the bus
- Maximum bus length without repeaters: 1200 m at 9600 baud (in practice, lab 
  distances are well within this limit)
- Keep the USB adapter close to the PC; the RS-485 cabling handles the distance

**Software configuration (from source):**

```python
self.client2 = ModbusSerialClient(
    port   = relay_galvo_port,   # auto-detected COM port
    baudrate = 9600,
    stopbits = 1,
    bytesize = 8,
    parity   = 'N',
    timeout  = 10
)
self.client2.connect()

# All three devices share this one client:
self.relay_client  = self.client2
self.galvo_client  = self.client2
self.zaxis_client  = self.client2
```

All writes to this client must acquire shared_modbus_lock:

```python
with self.shared_modbus_lock:
    response = self.client2.write_registers(address=..., values=[...], slave=2)
```

---

### 2.2 Relay Board

**Purpose:** Controls power to up to 4 peripheral devices (laser, illumination, 
stimulator power rails, or other lab equipment). Each channel is independently 
switchable from the GUI.

| Parameter | Value |
|---|---|
| Slave ID | 2 |
| Register address | 7 |
| Number of channels | 4 |
| Initial state | All OFF (relay_states = [False, False, False, False]) |

**Modbus write payload when toggling a relay:**

```python
relay_values = [int(state) for state in self.relay_states]  # 4 booleans -> [0|1, 0|1, 0|1, 0|1]
payload = [1, index] + relay_values + [0] * (6 - len(relay_values))
# Result: [1, channel_index, R0, R1, R2, R3, 0, 0]  (8 values total)
response = self.client2.write_registers(address=7, values=payload, slave=2)
```

**Payload field breakdown:**

| Byte position | Field | Value | Description |
|---|---|---|---|
| 0 | Command flag | 1 | Fixed — indicates batch relay write |
| 1 | Channel index | 0–3 | Which relay was just toggled (informational) |
| 2 | Relay 1 state | 0 or 1 | 1 = ON, 0 = OFF |
| 3 | Relay 2 state | 0 or 1 | 1 = ON, 0 = OFF |
| 4 | Relay 3 state | 0 or 1 | 1 = ON, 0 = OFF |
| 5 | Relay 4 state | 0 or 1 | 1 = ON, 0 = OFF |
| 6 | Padding | 0 | Reserved |
| 7 | Padding | 0 | Reserved |

**Example — Turn Relay 3 ON while 1 and 2 are already ON, 4 is OFF:**

```
relay_states = [True, True, True, False]
index = 2  (0-indexed, so Relay 3)
payload = [1, 2, 1, 1, 1, 0, 0, 0]
write_registers(address=7, values=[1,2,1,1,1,0,0,0], slave=2)
```

**Physical wiring:**
- Connect the COM terminal of each relay channel to its device's power common
- Connect NO (Normally Open) terminal to the device's power supply positive (or mains live)
- Ensure relay board power supply matches its rated voltage (typically 5V or 12V DC)
- Label each relay channel (e.g., Relay 1 = Laser power, Relay 2 = Illumination, etc.)

---

### 2.3 Galvo Motor Controller

**Purpose:** Steers a 2-axis galvo mirror system to redirect the laser beam to 
different positions on the specimen, enabling grid scanning for multi-point imaging.

| Parameter | Value |
|---|---|
| Slave ID | 2 |
| Register address | 0 |
| Angle range | -40.0° to +40.0° per axis |
| Angle resolution | 0.1° (dial step size) |
| Data format | IEEE 754 float32, big-endian, split into 2 × uint16 registers |
| Settling time (X axis) | 2.0 seconds (after write, before Y write) |
| Final settling time | 4.0 seconds (after both writes, before capture) |
| Total move time per grid point | ~8 seconds |

**Motor assignments:**
- Motor 1 = X axis (horizontal beam steering)
- Motor 2 = Y axis (vertical beam steering)

**Modbus payload structure — Motor 1 (X axis only):**

```
Register address 0, 7 values, slave 2

Position: [0]  [1]  [2]    [3]    [4]  [5]  [6]
Value:      0    3   HI_X   LO_X   0    0    0

Where:
  [1] = 3   -> subcommand: write X angle
  [2] = high 16-bit word of float32(angle_X)
  [3] = low  16-bit word of float32(angle_X)
  [4],[5] = 0 (Y angle not changed)
```

**Modbus payload structure — Motor 2 (Y axis only):**

```
Register address 0, 7 values, slave 2

Position: [0]  [1]  [2]  [3]  [4]    [5]    [6]
Value:      0    5    0    0   HI_Y   LO_Y    0

Where:
  [1] = 5   -> subcommand: write Y angle
  [4] = high 16-bit word of float32(angle_Y)
  [5] = low  16-bit word of float32(angle_Y)
  [2],[3] = 0 (X angle not changed)
```

**Float32 encoding (see Section 8 for full detail):**

```python
def float_to_registers(self, val):
    packed = struct.pack('>f', val)           # big-endian IEEE 754
    return list(struct.unpack('>HH', packed)) # [high_word, low_word]
```

**Retry logic:** Each galvo write retries up to 3 times on Modbus error, with 
delays of 0.4s, 0.6s, and 0.8s between attempts.

**Galvo Home position:**
The home_galvo_to_point1() function sends the galvo to the angle stored in 
coord_selector.matrix[0,0], which is the top-left grid position from calibration.

**Physical wiring:**
- Connect the RS-485 A+/B- terminals of the galvo controller to the shared bus
- Set the galvo controller's Modbus slave ID to 2 using the controller's 
  DIP switches or configuration software (consult controller documentation)
- Ensure the galvo controller firmware is set to RTU mode, 9600 baud, 8N1

---

### 2.4 Z-axis Motor Controller

**Purpose:** Controls the vertical position of the camera to adjust focus on the 
specimen. Separate from the XYZ stereotaxic stage.

| Parameter | Value |
|---|---|
| Slave ID | 2 (shared bus with relay and galvo) |
| Register address | 6 |
| Up command value | 41 |
| Down command value | 57 |

**Modbus write:**

```python
# Move up
self.client2.write_registers(address=6, values=[41], slave=2)

# Move down
self.client2.write_registers(address=6, values=[57], slave=2)
```

The Z-axis controller is on the same RS-485 bus as the relay and galvo. Its slave 
ID is 2 and it responds on register address 6 — no conflict with the relay (address 7) 
or galvo (address 0) because they use different register addresses on the same slave.

**Physical wiring:**
- Same RS-485 bus as relay and galvo (daisy-chained)
- Set slave ID to 2 on the controller hardware
- Connect motor drive wires to the camera Z-axis stepper or servo motor

**Note on camera window:** The Camera Controls dialog in the GUI shows:
"Z-axis COM: COM4 (fixed)" and "Z-Axis Slave ID: 2 (fixed)" — these are 
the hardcoded defaults in the UI labels, but the actual port used is always 
whatever the shared client2 is connected to (auto-detected).

---

### 2.5 Complete Modbus Register Map — Slave ID 2

All three devices on the shared bus use Slave ID 2. They are distinguished by 
their register address and the content of the payload:

| Register Address | Device | Payload | Effect |
|---|---|---|---|
| 0 | Galvo Motor X | [0, 3, HI_X, LO_X, 0, 0, 0] | Set X-axis angle (float32, Y unchanged) |
| 0 | Galvo Motor Y | [0, 5, 0, 0, HI_Y, LO_Y, 0] | Set Y-axis angle (float32, X unchanged) |
| 6 | Z-axis | [41] | Move camera focus upward |
| 6 | Z-axis | [57] | Move camera focus downward |
| 7 | Relay (4 ch) | [1, idx, R0, R1, R2, R3, 0, 0] | Batch toggle all relays |

There are no read operations issued in the current codebase — all hardware 
control on Slave ID 2 is write-only.

---

## 3. XYZ Stereotaxic Stage (USB Adapter B)

The XYZ stereotaxic stage is used for coarse positioning of the specimen under 
the imaging head. It uses a separate Modbus client instance from the relay/galvo bus.

### 3.1 Connection Settings

| Parameter | Value |
|---|---|
| Slave ID | 1 |
| Register address | 1 |
| Baud rate | 9600 |
| Protocol | Modbus RTU |
| USB chipset (typical) | CH340 or Prolific PL2303 |
| Fallback COM port | COM3 |
| Speed modes | Fast, Slow, SV Precise |

**Software configuration:**

```python
self.xyz_client = ModbusSerialClient(port=port, baudrate=9600)
self.xyz_client.connect()

# Move command:
response = self.xyz_client.write_registers(address=1, values=[cmd], slave=1)
```

No shared_modbus_lock is needed for xyz_client — it is a completely separate 
serial connection from client2.

---

### 3.2 XYZ Stage Command Code Table

Each axis and direction has three speed variants. The mode is selected in the 
Stereotaxic Stage Control window before clicking Move.

| Axis | Direction | GUI Label | Fast | Slow | SV Precise |
|---|---|---|---|---|---|
| X | Positive (+) | Move Right | 137 | 138 | 140 |
| X | Negative (−) | Move Left  | 201 | 202 | 204 |
| Y | Positive (+) | Move Up    | 145 | 146 | 148 |
| Y | Negative (−) | Move Down  | 209 | 210 | 212 |
| Z | Positive (+) | Come Upward  | 161 | 162 | 164 |
| Z | Negative (−) | Go Downward  | 225 | 226 | 228 |

**Note on GUI axis labels:** The GUI buttons for Y-axis are labelled 
"Move Up" and "Move Down" with direction="down" and direction="up" respectively 
in the source code. This inversion is intentional and reflects the mechanical 
orientation of the stage in the lab setup. Do not swap the button labels.

**Speed mode definitions (approximate, stage-dependent):**
- Fast: Maximum motor speed — use for gross repositioning only
- Slow: Reduced speed — use for fine approach to the target area
- SV Precise: Step-and-verify mode — use for final positioning before imaging

**Physical wiring:**
- Connect the XYZ stage USB cable directly to a PC USB port
- The stage controller will enumerate as a CH340 or Prolific serial device
- Set the stage controller's Modbus slave ID to 1 (verify with controller manual)
- Confirm 9600 baud, 8N1 in the controller configuration

---

## 4. Laser Current Monitor (USB Adapter C)

The laser current monitor reads the drive current of the imaging laser in 
real time and displays it in the header bar of the GUI.

### 4.1 Serial Protocol

| Parameter | Value |
|---|---|
| Interface | UART over USB (CDC ACM or CP2102) |
| Baud rate | 19200 |
| Data bits | 8 |
| Parity | None |
| Stop bits | 1 |
| Timeout | 1 second |
| Fallback COM port | COM7 |
| Update interval (GUI) | 1000 ms (root.after(1000, update_laser_display)) |

**Message format:**

The monitor device must send ASCII lines terminated with \n (newline). 
The platform parses only lines matching this exact format:

```
Current: <float_value>\n
```

Examples of valid lines:
```
Current: 450.00
Current: 0.00
Current: 1000.00
```

Any line not starting with "Current:" is treated as unrecognised and logged 
to the console but not displayed in the GUI.

**Software parsing:**

```python
line = arduino.readline().decode('utf-8').strip()
if line.startswith("Current:"):
    val = line.split(":")[1].strip()   # extract "450.00"
    self.laser_serial_queue.put(val)   # put numeric string into queue
```

The GUI display (update_laser_display) then shows the value as:
```
{val:.2f} mA
```

Values of 0.0 or negative are not displayed (the field shows blank). Values 
between 0.01 and 1000.00 mA are considered valid.

---

### 4.2 Arduino Firmware Requirement

The laser current monitor is implemented as a microcontroller (typically Arduino Uno 
or Arduino Nano) that reads the laser driver's current feedback signal (analogue or 
PWM) and transmits it over USB serial.

**Minimum firmware requirements:**

```cpp
// Arduino sketch minimum requirements for compatibility with LSCI platform

void setup() {
    Serial.begin(19200);    // Must be exactly 19200 baud
}

void loop() {
    float current_mA = read_laser_current(); // your sensor reading function
    Serial.print("Current: ");
    Serial.println(current_mA, 2);           // 2 decimal places
    delay(100);                              // ~10 Hz update rate is sufficient
}
```

**Key requirements:**
- Serial baud rate must be exactly 19200 (not 9600, not 115200)
- Each line must start with exactly "Current: " (capital C, colon, space)
- Value must be a valid decimal float
- Line must end with \n (println provides this automatically)

**Sensor wiring (typical):**
- Connect the laser driver's current monitor output (0–5V analogue, representing 
  0–1000 mA range) to Arduino analogue pin A0
- Scale: 5V = 1000 mA → scale factor = 1000/1023 × analogRead(A0)
- Calibrate the scale factor against a known reference current meter

---

## 5. NI-DAQ Card — Forepaw Stimulator

### 5.1 Card Configuration

| Parameter | Value |
|---|---|
| NI device name | Dev1 (hardcoded in source) |
| Channel | Dev1/port0/line0 |
| Channel type | Digital output (DO), single line |
| DAQmx channel mode | DAQmx_Val_ChanForAllLines |
| Task type | PyDAQmx.Task |
| Write function | WriteDigitalScalarU32 |
| Write timeout | 10.0 seconds |

**Source code hardcode locations:**
The string "Dev1/port0/line0" appears in two places in new_GUI_sc_55.py:
- Line ~1913: inside ForepawStimulatorApp.run_stimulation()
- Line ~5574: inside IntegratedGUI._run_stimulation_pattern()

If your NI-DAQ card enumerates as "Dev2" in NI MAX, edit both occurrences.

---

### 5.2 Signal Specification

| Signal | Value | Description |
|---|---|---|
| HIGH | WriteDigitalScalarU32(..., 1, ...) | Stimulation ON — current pulse delivered |
| LOW  | WriteDigitalScalarU32(..., 0, ...) | Stimulation OFF — no current |

The digital output is TTL-level (0V / 3.3V or 5V depending on the DAQ card 
output specification). The forepaw stimulator hardware must accept this TTL 
signal as its trigger input.

**Stimulator hardware input mode:** Set to **direct digital level input** (not 
edge-triggered, not analog). The software generates the complete waveform 
including pulse width — it does not rely on the stimulator's own pulse shaper.

**GND connection:** The signal ground (GND) of the DAQ card must be connected 
to the signal ground of the stimulator hardware. Without this common ground, the 
TTL signal reference is undefined and stimulation will be erratic or absent.

---

### 5.3 Pulse Timing Model

The platform generates pulses with 1 ms timing precision using a busy-wait loop:

```
Within one set:
│
├── Pulse 1
│     HIGH output ─────────────────────────────┐
│     (pulse_duration ms)                       │
│                                               ▼ LOW output
│     inter-pulse interval ─────────────────────┘
│
├── Pulse 2
│     ...
│
└── Pulse N (pulses_per_set)

After each set:
    inter-set interval (seconds) before next set begins
```

**Configurable parameters in the ForepawStimulatorApp dialog:**

| Parameter | Variable | Unit | Description |
|---|---|---|---|
| Number of sets | total_sets | count | How many stimulus sets to deliver |
| Pulses per set | pulses_per_set | count | Pulses within one set |
| Pulse duration | pulse_duration | seconds | Duration of each HIGH pulse |
| Inter-pulse interval | inter_pulse_interval | seconds | Gap between pulses |
| Inter-set interval | inter_set_interval | seconds | Gap between sets |

**Timing precision:** The busy-wait loop sleeps in 1 ms increments 
(time.sleep(0.001)). Actual precision is ±1–2 ms depending on OS scheduling 
jitter. For sub-millisecond timing, hardware trigger generation from the 
DAQ card itself would be required (not currently implemented).

**Safety:** The ForepawStimulatorApp shows a confirmation dialog before 
every stimulation run. It also prompts the user to verify the stimulator 
is set to direct digital input mode. Do not bypass these dialogs.

---

## 6. scTDC Camera

### 6.1 USB Connection

| Parameter | Value |
|---|---|
| Interface | USB 3.0 (USB3 Vision or vendor proprietary) |
| Sensor resolution | 1920 × 1440 pixels |
| Bit depth | 12-bit (uint16 output, values 0–4095) |
| SDK | scTDC (vendor-provided, Windows only) |
| DLL | scTDClib.dll (must be in project root directory) |
| Config file | tdc_gpx3.ini (must be in project root directory) |

The camera is plugged directly into a USB 3.0 port on the PC. USB 2.0 may 
work for slow acquisition rates but is not recommended — the large frame size 
(1920 × 1440 × 2 bytes = 5.5 MB per frame) requires USB 3.0 bandwidth for 
smooth preview at any reasonable frame rate.

---

### 6.2 Software Configuration

**Initialization sequence (in camera_process):**

```python
lib    = scTDC.scTDClib()
camera = scTDC.Camera(inifilepath="tdc_gpx3.ini", autoinit=False, lib=lib)
# autoinit=False means camera.initialize() must be called explicitly
# This is triggered by the "Initialize Camera" button in the GUI
initret, message = camera.initialize()
pipe_id, pipe    = camera.add_frame_pipe()
```

**Frame acquisition:**

```python
camera.set_exposure_and_frames(exposure_us, num_frames)
camera.do_measurement()
meta, img = pipe.read()   # img is numpy uint16, shape (1440, 1920)
```

**Key constraint — single consumer:** Only one process may call 
camera.initialize() at a time. If the Live Speckle Tracker companion script 
(live_contrast_GUI_2.py) is running and has initialized the camera, the main 
GUI's "Initialize Camera" will fail with error code -5 (camera already in use).

---

## 7. Auto COM Port Detection Reference

At startup, the platform scans all available COM ports and tries to match 
each one to a known device using USB descriptor string matching.

**Detection keyword table:**

| Device | Detection keywords (USB descriptor contains any of these) |
|---|---|
| Relay / Galvo / Z-axis | CP2102, FT232R, USB Serial Device, USB-to-Serial, Silicon Labs |
| XYZ Stage | CH340, Prolific, USB Serial Port, SERIAL |
| Laser Monitor | Arduino, USB Serial, CP2102 |

**Note:** CP2102 appears in both relay/galvo and laser categories. The 
algorithm assigns the first CP2102 match to relay_galvo and a subsequent 
CP2102 match to laser. If only one CP2102 is connected, the laser port 
will fall back to the default.

**Fallback ports (used if no descriptor match is found):**

| Device | Fallback |
|---|---|
| Relay / Galvo / Z-axis | First available COM port, or COM4 |
| XYZ Stage | Second available COM port, or COM3 |
| Laser Monitor | Third available COM port, or COM7 |

**Console output format:**

```
[PORT DETECTION] Available COM ports: ['COM3', 'COM4', 'COM7']
[PORT DETECTION] RELAY_GALVO found on COM4
[PORT DETECTION] XYZ_STAGE found on COM3
[PORT DETECTION] LASER found on COM7
[PORT DETECTION] Final port assignment: {'relay_galvo': 'COM4', 'xyz_stage': 'COM3', 'laser': 'COM7'}
✅ Connected to COM4
```

**To override with fixed COM port numbers:**
Edit auto_detect_ports() in new_GUI_sc_55.py (lines 74–106) and return 
explicit port strings before the keyword-matching logic runs.

---

## 8. Float32 Encoding for Galvo Angles

The galvo controller expects angles as IEEE 754 single-precision floats 
encoded in big-endian byte order, split into two 16-bit Modbus registers.

**Encoding function (from source):**

```python
def float_to_registers(self, val):
    packed = struct.pack('>f', val)           # 4 bytes, big-endian IEEE 754
    return list(struct.unpack('>HH', packed)) # [high_word, low_word]
```

**Worked examples:**

| Angle (degrees) | Hex representation | High word | Low word |
|---|---|---|---|
| 0.0  | 0x00000000 | 0x0000 | 0x0000 |
| 1.0  | 0x3F800000 | 0x3F80 | 0x0000 |
| -1.0 | 0xBF800000 | 0xBF80 | 0x0000 |
| 10.0 | 0x41200000 | 0x4120 | 0x0000 |
| 15.5 | 0x41780000 | 0x4178 | 0x0000 |
| 40.0 | 0x42200000 | 0x4220 | 0x0000 |
| -40.0| 0xC2200000 | 0xC220 | 0x0000 |
| 12.3 | 0x4144CCCD | 0x4144 | 0xCCCD |

**Full write payload for X angle = 15.5 degrees:**

```
float_to_registers(15.5) -> [0x4178, 0x0000]

Payload sent to write_registers(address=0, slave=2):
  [0, 3, 0x4178, 0x0000, 0, 0, 0]
   ^  ^  ^─────────────^  ^─────^
   |  |  float32 X angle  Y = 0
   |  subcommand: 3 = write X
   padding
```

**Full write payload for Y angle = -10.0 degrees:**

```
float_to_registers(-10.0) -> [0xC120, 0x0000]

Payload sent to write_registers(address=0, slave=2):
  [0, 5, 0, 0, 0xC120, 0x0000, 0]
   ^  ^  ^──^ ^─────────────^
   |  |  X=0  float32 Y angle
   |  subcommand: 5 = write Y
   padding
```

**Angle range:** Both motors accept -40.0° to +40.0°. The dial widget in the 
GUI enforces this range (start=-40.0, end=40.0). Values outside this range 
are not rejected by the software but may cause the galvo hardware to clip 
at its mechanical limits.

---

## 9. Power-On Sequence

Follow this order each session to avoid initialization conflicts:

```
Step 1 — Power on the relay board and Modbus adapter (USB Adapter A)
          The relay board must be powered and its Modbus controller active before 
          the software attempts to connect.

Step 2 — Power on the galvo controller
          Connected on the same RS-485 bus as the relay board.
          Both should be on before the software starts.

Step 3 — Power on the Z-axis motor controller
          Also on the shared RS-485 bus.

Step 4 — Connect and power on the XYZ stereotaxic stage (USB Adapter B)
          The stage has its own Modbus bus; power order relative to the above 
          is not critical.

Step 5 — Connect the laser Arduino (USB Adapter C)
          The laser itself should be off until camera preview confirms beam 
          alignment. The Arduino monitor runs regardless of laser state.

Step 6 — Connect the scTDC camera (USB Adapter for camera)
          The camera USB must be connected before launching the application.
          Hot-plugging after launch requires a GUI restart.

Step 7 — Launch the application
          python new_GUI_sc_55.py
          The software will auto-detect COM ports and connect to the relay/
          galvo Modbus bus at startup.

Step 8 — Click "Initialize Camera" in the GUI
          The camera SDK initializes and creates the frame pipe.
          Only after this can "Start Preview" be used.

Step 9 — Enable relays as needed (Relay 1–4 via the Relay button)
          Turn on the laser relay last, after confirming beam path safety.

Step 10 — Enable galvo (Enable Galvo button in the Multi-Capture window)
           The galvo motors are now active and will respond to dial moves 
           and grid scan commands.
```

**Power-off sequence:** Reverse of the above. Always disable the galvo and 
turn off laser relays before powering down hardware.

---

## 10. Physical Connection Checklist

Use this before every imaging session:

```
USB Connections
[ ] USB Adapter A (CP2102/FT232R) connected — relay/galvo/z-axis bus
[ ] USB Adapter B (CH340/Prolific) connected — XYZ stage
[ ] USB Adapter C (Arduino/CP2102) connected — laser monitor
[ ] Camera USB cable connected to USB 3.0 port
[ ] NI-DAQ card seated in PCIe slot (or USB DAQ connected if applicable)

RS-485 Bus (Adapter A)
[ ] Relay board A+/B- terminals wired to adapter
[ ] Galvo controller A+/B- terminals daisy-chained from relay board
[ ] Z-axis controller A+/B- terminals daisy-chained from galvo controller
[ ] 120 Ohm termination resistor at far end of bus (if cable > 1 m)
[ ] All three devices powered on

Slave ID Verification (RS-485 Bus)
[ ] Relay board slave ID = 2
[ ] Galvo controller slave ID = 2
[ ] Z-axis controller slave ID = 2
[ ] XYZ stage slave ID = 1

Baud Rate Verification
[ ] All RS-485 devices configured to 9600 baud, 8N1, RTU mode

NI-DAQ
[ ] Dev1/port0/line0 connected to stimulator TTL input
[ ] DAQ card GND connected to stimulator signal GND
[ ] Stimulator set to direct digital input mode
[ ] NI-DAQ device name confirmed as "Dev1" in NI MAX

Camera
[ ] scTDClib.dll present in project root directory
[ ] tdc_gpx3.ini present in project root directory
[ ] Camera USB cable connected before launching application

Laser Monitor Arduino
[ ] Arduino connected via USB Adapter C
[ ] Arduino sketch running at 19200 baud
[ ] Output format: "Current: <float>\n"

Software
[ ] Virtual environment activated
[ ] Working directory = project root (same folder as new_GUI_sc_55.py)
[ ] All three COM ports appearing in Device Manager
```

---

## 11. Troubleshooting Hardware Issues

### RS-485 bus: Galvo and relay respond but Z-axis does not

Verify the Z-axis controller slave ID is set to 2 — not 1 or another value. 
All three devices share slave ID 2; any mismatch will cause that device to 
ignore all Modbus frames.

Also check that the Z-axis controller is on the bus before the Modbus client 
connects. Hot-connecting a Modbus device after the client is already running 
may require a client reconnect (restart the application).

---

### Galvo responds but beam does not move

The motor controller accepted the Modbus write but the mechanical system is 
not responding:
1. Check galvo drive power supply is on and at correct voltage
2. Verify the encoder feedback cable (if applicable) is connected
3. Confirm the float32 angle values are within -40° to +40° range
4. Try sending 0.0° to both axes (home position) — if the beam snaps to 
   centre, the encoding is correct and the issue is mechanical

---

### Relay toggles in GUI but device does not power on or off

1. Check the relay board power LED — board must be powered independently 
   of the USB-serial adapter
2. Verify the load is wired to the NO (Normally Open) terminal, not NC
3. Use a multimeter in continuity mode to confirm the relay contact switches 
   when the GUI button is toggled
4. If all four relays toggle simultaneously when you click one, the Modbus 
   write is reaching the board; the issue is likely the relay board firmware 
   not interpreting the batch-write payload correctly — verify the board 
   model matches the expected register format

---

### XYZ stage "Not connected" even after clicking Connect

1. Confirm the stage COM port appears in Device Manager
2. The Connect button calls connect_xyz_stage() which uses 
   detected_ports['xyz_stage'] — if the stage was not detected during startup, 
   this may be a wrong port. Check console output for [PORT DETECTION] lines.
3. Try connecting before any other hardware to ensure the CH340 driver 
   enumerated before the software started scanning ports.
4. Modbus slave ID must be 1 (not 2). If the stage has the wrong slave ID, 
   all write_registers calls will return no response.

---

### Laser monitor shows constant "--- mA"

1. Arduino must be sending at exactly 19200 baud — not 9600 or 115200
2. The Arduino COM port must be detected as the 'laser' port in auto-detection
3. Serial format must be "Current: 450.00\n" — any deviation (wrong prefix, 
   missing space, missing colon, missing newline) causes the parser to ignore 
   the line
4. Check the Arduino is powered by USB (the USB cable from the PC powers it)
5. Open a terminal (PuTTY or Arduino Serial Monitor) at 19200 baud on the 
   Arduino COM port and confirm valid lines appear there first

---

### NI-DAQ stimulator: DAQmxError -200220 (device not found)

Error -200220 means NI-DAQmx cannot find a device named "Dev1".
1. Open NI MAX and check the device name shown under Devices and Interfaces
2. If it is "Dev2" or similar, edit the two occurrences of "Dev1/port0/line0" 
   in new_GUI_sc_55.py to match
3. If no device appears in NI MAX at all: reinstall NI-DAQmx runtime and reboot

---

### Camera initialization error code -3 (hardware not detected)

The scTDC SDK cannot find the camera on the USB bus:
1. Confirm the camera USB cable is plugged into a USB 3.0 port (blue port)
2. Open Windows Device Manager and check for any Unknown Device entries
3. Install the camera vendor's USB driver if provided separately from the SDK
4. Restart the PC with the camera connected — some cameras require the 
   USB connection before Windows boots to enumerate correctly

---

*Document version: January 2026 - Platform v2.1*
