# FRC2158 CAN Analyzer

**Team 2158's pit tool for the robot's CAN bus: every SystemCore, CTRE and REV device, live, in your browser.**

![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue) ![Platform: Windows](https://img.shields.io/badge/platform-Windows-lightgrey) ![License: MIT](https://img.shields.io/badge/license-MIT-green)

The FRC2158 CAN Analyzer is Team 2158's pit tool for the robot's CAN bus. It connects through the RH-02-PRO USB
adapter, or runs with no hardware on a built-in demo robot or a saved log. The page lists every device it hears
(SystemCore, CTRE, REV and others) and compares them with the team's device map. It flags missing, unknown and
duplicate IDs and gives the robot a CAN health score. It also shows bus load and heartbeat timing, decodes frames
into plain words, and can make a device blink so you can find it on the robot.

## Quick start

```bash
pip install pyserial
python can_analyzer.py --bus demo ui     # try it with no robot and no adapter
python can_analyzer.py ui                # then pick the RH-02-PRO's COM port on the page
```

The page opens at http://127.0.0.1:8766. On Windows you can also double-click `start_can_analyzer.cmd`.

## How it works

The analyzer decodes every frame for SystemCore, REV Robotics and CTRE devices. It reads the FRC CAN ID (device
type, manufacturer, API class and index, device number), the SystemCore heartbeat (enabled, mode, alliance, match
time) and the REV SPARK status values. It also checks every device the robot should have against the team's device
map (Engineering Standards Appendix C). It is built as a small Python
server (standard library + pyserial) and one web page, `ui/index.html`, styled to the Team 2158 Brand Guide V1.1.

## The adapter

The Jhoinrch RH-02-PRO is a CANable 2.0 design and ships with the CANable **SLCAN firmware**: it shows up on Windows
as a USB serial port (**COMx**, Device Manager -> Ports), no vendor DLL. The analyzer opens it with:

- `C` (close), `S8` (1 Mbit/s, the FRC rate), `Y<n>` (CAN FD data rate, only when an FD rate is picked), then
  `L` (**listen-only**, the default) or `O` (normal).
- Frames: `T` (29-bit, every FRC device), `t` (11-bit), `D` / `B` (CAN FD without / with bit-rate switch), `R` / `r`
  (remote). A BEL from the adapter (a refused command) is counted and logged.

If it was reflashed to candleLight: python-can with `gs_usb` (needs pyusb and a WinUSB driver).

**Wiring:** CANH (yellow) and CANL (green) into any spot on the bus (a spare Weidmuller or a CAN board's pass-
through), GND to the CAN ground if the adapter's isolated side wants it. The robot's two 120 Ω terminators are
already there: leave the adapter's terminator switch **off** unless it is at the very end of the bus. Robot off,
about 60 Ω across CANH-CANL.

**Listen-only** never drives the bus (it does not even ACK). On a robot that is what you want: the other devices
ACK each frame. Untick it only on a bench with **one** device and the adapter, where nothing else would ACK.

**SystemCore has several buses** (CAN 0 to 4 in the Standards). The adapter sees one bus at a time: pick that bus
under **Robot bus** so only its devices are expected.

## Install

Python 3.10+ on Windows and pyserial (`pip install pyserial`). From this folder:

```bash
python can_analyzer.py list          # COM ports; the RH-02-PRO is marked "CANable / RH-02"
```

## Use

```bash
python can_analyzer.py ui                      # opens http://127.0.0.1:8766; connect from the page
python can_analyzer.py --channel COM7 ui       # connects to the RH-02-PRO at once
python can_analyzer.py --bus demo ui           # the demo robot: no adapter, no robot
python can_analyzer.py --bus replay --log logs/capture_20261120_181500.log ui
python can_analyzer.py --channel COM7 dump     # decoded frames on the console
python can_analyzer.py decode 0204180B         # what an ID means (add the data bytes to decode them)
```

Or double-click `start_can_analyzer.cmd`. Port 8766 (the bench simulator keeps 8765), local only.

### The page

- **Header:** the Can Man and the title; pills for the **SystemCore heartbeat** (green Teleop / Autonomous / Test,
  amber Disabled, red Lost: devices treat 100 ms without it as disabled), **frames per second and bus load**,
  **Record** (every frame to `logs/capture_<date>_<time>.log`, candump format) and the **connection**.
- **Connection** (the dropdown under the connection pill): **RH-02-PRO** (pick the COM port; a CANable-type port is
  pre-selected), **python-can** (any interface), **Demo robot**, or **Replay** (a log from `logs/`, or Open file...;
  ¼×, 1×, 4×). Bit rate 1 Mbit/s (FRC) by default; Classic CAN or a CAN FD data rate (1 to 8 Mbit/s) for a SystemCore
  CAN FD bus; listen-only.
- **Health** (header pill and the first tile): one robot CAN health number. It starts at the share of expected
  devices heard, then loses 20 points with no SystemCore heartbeat (10 for a heartbeat gap over 100 ms in the last
  minute), 10 per possible duplicate ID, 5 or 10 for bus load over 60 % or 80 %, and 2 per device not in the map.
  Green from 90 %, amber from 70 %, red below; hover it for what took points off.
- **Tiles are links:** Health and Missing / Not in map / Duplicate IDs filter the cards to those devices (the yellow tag clears it), SystemCore opens its card, Expected devices shows all, Bus opens the Frames tab.
- **Device** pulldown (Devices tab): pick one device, or click its card, to see that device alone with its frames
  live (each CAN ID, data with the changing bytes lit, period, count); click a frame to open it in the byte
  explorer. **Problems only** hides the devices that are OK.
- **Actions** (in the single-device view): **Identify (blink)** sends the device its Identify command so its LED
  blinks. CTRE and REV do not publish that frame, so the analyzer learns it once per model: press **Teach Identify**,
  then, with the robot disabled, click Blink / Identify for that device in Phoenix Tuner X or the REV Hardware Client
  within 20 s. The first new frame to that device is saved in `identify.json` and used for every device of that model
  (**Forget** clears it). Sending needs the RH-02-PRO connected with **Listen-only unticked**. A device that is NOT IN
  MAP gets **Add to device map** (name and robot bus).
- **SystemCore card:** everything in the heartbeat (mode, alliance, match time and number, tournament, watchdog,
  replay, FTC motor override, the Driver Station clock once it is set) plus what the analyzer measures: heartbeat
  period and jitter, gaps over 100 ms (each one makes every device disable itself), time in the current mode, mode
  changes, time enabled, and SystemCore's Disable broadcasts (ID 0, no data). Manufacturer codes 2 and 3 follow the
  2027 spec (Limelight, FIRST).
- **Devices:** tiles for SystemCore mode, expected devices present (n / m on the selected bus), missing, not in the
  map, possible duplicate IDs and bus load; then one card per device, grouped SystemCore / CTRE / REV / Other:
  the device number large (Agency FB, as the guide wants robot numbers), the map's name, the likely product, its
  rate and when it was last heard, and the decoded values (heartbeat fields; SPARK applied output, faults, velocity,
  temperature, bus voltage, current, position). Card colours: green OK, red **SILENT** (heard, then nothing for 1 s)
  or **NOT HEARD** (in the map for this bus, never seen), amber **NOT IN MAP** (a wrong ID, a factory ID 0 SPARK, or
  a device to add) and **CHECK ID** (status frames about twice as fast as the device sends them: two devices share
  the ID).
- **Frames:** **By ID** (one row per CAN ID: product, API, data, **Decoded** in plain words (or a grey hint when the maker has not published the layout), period with the device's default in brackets, red
  when twice as slow or twice as fast, count, last seen) or **Stream** (every frame, newest at the bottom, decoded);
  filter by family or text; Pause; **Save log** (the last 20 000 frames).
- **Byte explorer** (for payloads nobody publishes, CTRE's above all): pick a CAN ID (or click a row in By ID) and
  the page keeps its recent frames. Each byte gets a card with its range and a live trace; **Hide constant** drops
  bytes that never change, **Sort smoothest first** puts counters and sensor values on top. The candidate table tries
  every 8, 16, 24 and 32 bit integer (little and big endian, signed and unsigned) and float32 at each offset, with a
  sparkline, so you can move a mechanism and watch which candidate follows it. **Snapshot A / B** marks the bytes that
  changed between two moments (wheel turned, robot enabled). The **bit-field probe** reads any start bit and length.
  **Name a field** saves what you found (type, offset or bits, scale, offset, unit, note) to `fieldmap.json` for that
  frame kind (type, maker, API class, index), so the same frame on every device of that model shows the named value
  in the device cards, By ID and Stream. **CSV** exports the captured frames for a spreadsheet.
- **Product pictures:** click a product name (device card, By ID, Stream) to see it. The pictures are rendered from
  the makers' STEP models: CTRE Device-CADs (Falcon 500, Talon FXS, Pigeon 2.0, CANrange, CANdle, CANdi, PDP, PCM),
  WCP (Kraken X60, Kraken X44, CANcoder, ThroughBore powered by CANcoder) and REV (NEO Vortex with SPARK Flex,
  SPARK MAX, Power Distribution Hub, Pneumatic Hub, Servo Hub; REV's models have no colours, so they show in grey).
- **Status lights:** the same pop-up has the LED table: Talon FX (Kraken X60, X44, Falcon 500) from CTRE's Phoenix 6
  hardware reference, SPARK Flex from REV's docs: each code, what it means and what to do. A device that is NOT HEARD or SILENT gets a
  one-line LED hint on its card.
- **Decoder:** paste an ID (and data) to see its five fields bit by bit and the decoded payload; **Build an ID** from
  the fields for code, filters or Phoenix Tuner X.
- **Device map:** the robot's expected devices (name, device type, maker, CAN ID or blank for any, robot bus), saved
  to `devmap.json`. **Reset to Standards** puts back Appendix C: swerve module n = drive n1 (Kraken X60), steer n2
  (Kraken X44), CANcoder n3; front on CAN 0, rear on CAN 1, the Pigeon 2.0 alone on CAN 2. Add the mechanism motors
  once they have IDs.
- **Log:** connects, disconnects, quiet buses ("no frames for 3 s"), refused commands.
- **Dark** switches to a dark pit theme (remembered by the browser).

## What is decoded, and where it comes from

| Frames | Decoded | Source |
|---|---|---|
| Every 29-bit ID | device type, manufacturer, API class / index, device number, likely product | WPILib, FRC CAN Device Specifications |
| `0x01011840` SystemCore heartbeat | enabled, autonomous, test, watchdog, alliance, match time / number, tournament | same (byte 8 = match time, byte 5 = mode bits) |
| Broadcast (type 0, maker 0) | Disable, System Halt, Reset, Heartbeat, Sync... | same |
| REV SPARK MAX / Flex, API class 6 index 0-6 | Status 0-6: applied output, faults, velocity, temperature, voltage, current, positions, encoders | REV, SPARK MAX Control Interfaces; scales as REVLib (V /128, A /32) |
| REV SPARK, API class 46 | named "Status n (firmware 2025+, likely)", not decoded | REV has not published that layout |
| CTRE (Talon FX / Kraken, CANcoder, Pigeon 2.0, CANrange, CANdle) | ID fields and rates; fields you name in the byte explorer | CTRE does not publish Phoenix 6 payloads (not in Phoenix6-Examples or the Linux example either): find them with the byte explorer |
| Any frame kind | the fields in `fieldmap.json` | you, from the byte explorer |

The product names are a best guess from (device type, maker): two products can share a pair (a Talon FX and a Talon
FXS both report Motor Controller / CTRE).

**Duplicate IDs:** the check needs a frame with a known default period (SPARK status 0-6 and the heartbeat). Two
CTRE devices on one ID are usually caught by Phoenix Tuner X instead.

## Tests

```bash
python -m unittest test_can_analyzer
```

22 tests, no adapter or robot needed: the ID fields, the heartbeat and SPARK layouts both ways, the SLCAN driver
against a fake RH-02-PRO (open sequence `C S8 Y5 L`, frames, BEL), candump logs, the device checks (missing,
duplicate, not in map, the bus filter), the byte explorer (field extraction, named fields), and the web server end to
end on the demo robot (record, replay, decode, watch, fields, product pictures).

## Files

| File | What |
|---|---|
| `can_analyzer.py` | the command line: `ui`, `list`, `dump`, `decode` |
| `buses.py` | the RH-02-PRO (SLCAN on pyserial), python-can, replay; candump log lines |
| `frc_can.py` | the FRC CAN tables and decoders |
| `analyzer.py` | the capture engine: by-ID table, devices, bus load, heartbeat, the device map checks, recording |
| `robot_sim.py` | the demo robot (its CTRE payloads are made up but structured, to practise the byte explorer on) |
| `webui.py` | the local web server and its API |
| `ui/index.html` | the page (brand fonts in `ui/fonts`, the Can Man in `ui/brand`) |
| `devmap.json` | the device map (the page edits it) |
| `fieldmap.json` | fields named in the byte explorer (created on the first save) |
| `ui/img/` | product pictures, rendered from the makers' STEP models |
| `logs/` | recordings and logs opened for replay |

## License

MIT License. See [LICENSE](LICENSE). Questions: andersonroboticsinfo@gmail.com.
