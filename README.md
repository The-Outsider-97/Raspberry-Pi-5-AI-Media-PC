# Raspberry Pi 5 AI Media PC

This document captures the final hardware, capabilities, and how to assemble & use your Raspberry Pi 5–based media PC with vision and room-sensing.

## 1) Final specs
### Core & graphics
- Board: Raspberry Pi 5 (16 GB LPDDR4X)
- CPU/GPU: Broadcom BCM2712 — 4× Cortex-A76 (up to ~2.4 GHz) + VideoCore VII
- Hardware video decode: HEVC/H.265 up to 4K60; H.264 = software; AV1 = software
- Video outputs: 2× micro-HDMI 2.0 (using your Micro-HDMI→HDMI multifunction adapter to bring ports to one side)
- Networking: Gigabit Ethernet, dual-band Wi-Fi, Bluetooth 5.x
- USB: 2× USB 3.0 (UASP), 2× USB 2.0
- Audio out: over HDMI (Pi 5 has no 3.5 mm jack)

### Storage & AI acceleration
- Carrier: Double NVMe Base (dual M.2 via PCIe switch; Gen2 x1 uplink shared)
- SSD: Transcend TS512GMTE110S, 512 GB, NVMe, M.2 2280 (slot A)
- NPU: Hailo-8L (13 TOPS) module, M.2 2242 (slot B)

 | Note: Both devices share the Pi’s single PCIe lane via the switch. You still get hundreds of MB/s NVMe and full Hailo inference throughput for typical vision workloads. |

### Cooling & power
- Pi cooling: Raspberry Pi 5 Active Cooler
- Case airflow: 2× 30 mm 5 V fans (PWM via MOSFET, GPIO 18)
- PSU: Official 27 W USB-C (5.1 V / 5 A)

### Display, touch & audio
- Panel: Waveshare 13.3″ 1920×1080 capacitive-touch (60 Hz)
- Touch: USB-HID (single USB cable to the Pi)
- Speakers:
   - Primary media audio via HDMI to the display’s built-in speakers
   - Notifications via a separate 8 Ω 5 W speaker driven by your PAM8403 2×5 W Bluetooth 5.0 mini amp at 5 V

### Camera & sensors
- Camera: Official Raspberry Pi AI Camera (Sony IMX500; on-sensor AI capability)
- Ambient light: TSL25911 (a.k.a. TSL2591 class), I²C — high-sensitivity lux
- Environment: BME280, I²C — temperature, humidity, pressure
- Presence (radar): LD2410 mmWave, UART (5 V supply, 3.3 V-level UART)
- Distance: HC-SR04P ultrasonic (3.3–5.5 V supply; 3.3 V-safe ECHO when powered at 3.3 V)
- GPIO switching: Grove MOSFET module (for fans/LEDs or other DC loads)

### Sidecar microcontroller
- Board: Waveshare RP2350B-Plus-W (Pico-class, Wi-Fi/BT) — ideal for real-time sensor reads, button/LED I/O, and forwarding data to the Pi over USB-serial or MQTT.

### Servo + Driver
- Servo driver: 16-channel PWM Servo Driver Hat 
- Servo: 25kg DS3218MG 270 Servo motor (to pan the base of the ), MG90S Micro servo motor (to tilt the camera and the sensors)
- Power: Mascot 6V Plug-in adapter

## 2) What you can do with this PC
### Media & streaming (primary use)
- 4K movies (HEVC/H.265) with smooth hardware decoding; 1080p is effortless across services and files.
- Run as a media appliance with LibreELEC/Kodi, or as a desktop streamer with Raspberry Pi OS (64-bit) + Chromium/Firefox.
- Use the touchscreen for quick seeking and on-screen keyboard. Keep “system beeps” and notification chimes on the Bluetooth amp + 5 W speaker while show audio stays on HDMI.

### AI-powered vision
- Hailo-8L + AI Camera = real-time person/object detection, tracking, segmentation, etc., with low CPU load.
- Build presence-aware media: auto-pause when you leave, resume on return, show overlays (e.g., who’s at the door, weather tickers).

### Smart-room automation
- Adaptive brightness: TSL25911 controls screen backlight based on room lux.
- Comfort dashboard: BME280 feeds temperature/humidity/pressure trends.
- Presence fusion: combine LD2410 (micro-motion) + AI Camera + HC-SR04P distance for robust occupancy. E.g., wake the display as you approach; dim when idle.
- Physical controls: the Pico sidecar can handle buttons/encoders/LEDs and publish to the Pi (USB or MQTT).

### Maker & expansion
- PWM-control fans/LED strips via the MOSFET (GPIO 18).
- Add USB gear (capture cards, SDR, extra storage) on USB 3.
- Expose a local REST/MQTT API for sensor data & automations.

### Light gaming & retro
- Retro up to 16/32-bit very comfortably; heavier systems with tuning (remember, it’s a Pi).

## 3) Assembly & wiring
### Buses & pins (recommended mapping)

| Function      | Device           | Power | Pi pins/bus |
| ------------- | ---------------- | ----- | ----------- |
| I²C (sensors) | TSL25911, BME280 |3.3V   |GPIO 2 (SDA) / GPIO 3 (SCL) — I²C-1; unique addresses (TSL2591=0x29; BME280=0x76/0x77)|
| UART (mmWave) | LD2410           |5V     |GPIO 14 (TXD) / GPIO 15 (RXD) @ 3.3 V logic|
| Ultrasonic | HC-SR04P via RP2350B-Plus-W | 3.3V | GP15 = TRIG, GP14 = ECHO (safe at 3.3 V)|
| Fans (PWM) | 2× 30 mm 5 V fans via Grove MOSFET | 5V | GPIO 18 (PWM) → MOSFET gate; fan + → 5 V; fan – → MOSFET drain; MOSFET source → GND. Tie Pi GND and fan PSU GND. Add a flyback diode across each fan (recommended) |
| Audio (notifications) | PAM8403 BT 2×5 W @ 5 V → 8 Ω 5 W speaker | 5V | Pair over Bluetooth to Pi as an A2DP speaker. Keep HDMI as default sink; target notifications to BT sink (see §5.3) |
| Camera | Raspberry Pi AI Camera (IMX500) | 3.3V | CSI ribbon to Pi camera connector; route carefully around the NVMe base and HDMI adapter |

### PCIe & storage layout
- Slot A (Duo base) → NVMe SSD (boot device)
- Slot B → Hailo-8L NPU
- Update Pi firmware/bootloader so NVMe-boot via the switch works reliably. Bandwidth (Gen2 x1) is shared—still ample for media + AI.

### Power
- Pi 5 on the 27 W PSU (covers Pi + NVMe + USB).
- Display requires its own 12 V supply.
- Fans are 5 V (as per your note) and are MOSFET-switched from GPIO 18 for speed control.
- Bluetooth amp runs at 5 V; don’t power speakers from the Pi directly.

## 4) Software setup
### 4.1 Quick-start (Raspberry Pi OS, NVMe boot)
1. Flash Raspberry Pi OS (64-bit) to the NVMe (use Raspberry Pi Imager + USB M.2 enclosure, or flash to microSD then clone to NVMe).
2. Boot once, run sudo apt update && sudo apt full-upgrade.
3. Firmware/bootloader: sudo raspi-config → Advanced → Boot Order → NVMe/USB First. Reboot.
4. Enable interfaces: raspi-config → Interface Options → enable I²C, Serial (UART) (login shell off, serial port on) and Camera.
5. Install basics: sudo apt install -y git python3-pip i2c-tools minicom pavucontrol (PipeWire-Pulse is default on Bookworm).
6. Confirm devices:
   - I²C: sudo i2cdetect -y 1 (expect 0x29 and 0x76/0x77)
   - UART: minicom -b 256000 -o -D /dev/serial0 (LD2410 talks fast; adjust baud as needed)

```bash
LibreELEC path: If you want an appliance feel, flash LibreELEC (Kodi) to NVMe instead.
You’ll get best-in-class video playback; run AI/sensors on a separate Pi OS install or containers if needed.
```

### 4.2 Hailo & AI Camera
- Install the Raspberry Pi AI libraries / Hailo runtime per vendor docs. Typical flow:
   - rpicam-apps for capture → GStreamer / OpenCV → Hailo post-processing.
   - Use pre-compiled models (e.g., YOLO-family) for person/object detection.
- The IMX500 can run on-sensor models; use it to pre-filter (e.g., motion/person) and pass ROIs to Hailo for heavier inference.

### 4.3 Sensors & data flow
- I²C: Python with smbus2 or adafruit-circuitpython-* libs (TSL2591, BME280).
- LD2410: Use a Python lib or serial parser to read presence/motion/micro-motion.
- HC-SR04P: Use GPIO libs (gpiozero/RPi.GPIO) — if you powered at 3.3 V, you can read ECHO directly.
- Sidecar (RP2350B-Plus-W): Run MicroPython / Pico SDK to sample sensors at high-rate and publish over USB-serial or MQTT to the Pi.

```bash
Recommended pattern: run a tiny sensor daemon (Python) that publishes MQTT topics like room/lux, room/temp, presence/ld2410, presence/ai, distance/cm.
Home Assistant or a custom Node-RED/Kodi bridge can subscribe and act.
```

## 5) Audio & notifications
### 5.1 Outputs
- Media audio: HDMI → display’s built-in speakers (default)
- Notifications: Bluetooth PAM8403 2×5 W amp → 8 Ω 5 W speaker

### 5.2 Pair the Bluetooth amp
```bash
bluetoothctl
power on
agent on
scan on             # wait for your amp to appear
pair XX:XX:XX:XX:XX # replace with device MAC
trust XX:XX:XX:XX:XX
connect XX:XX:XX:XX:XX
quit
```

### 5.3 Route only alerts to the BT speaker (keep HDMI default)
- On Raspberry Pi OS (PipeWire with Pulse compatibility):
```bash
pactl list short sinks          # note sink names (e.g., alsa_output.hdmi..., bluez_output....a2dp-sink)
```
- Keep HDMI as the default. For a notification sound:
```bash
paplay -d bluez_output.XX_XX_XX_XX_XX_XX.a2dp-sink /usr/share/sounds/freedesktop/stereo/complete.oga
```
- In apps/scripts, point short “beep”/“alert” sounds to the BT sink while leaving players (Kodi/Chromium) on default HDMI.

## 6) Fan control (GPIO 18 → MOSFET → 5 V fans)
### 6.1 Simple PWM control (example)
- Use pigpio (accurate PWM) or kernel PWM:
```bash
sudo apt install -y pigpio
sudo systemctl enable --now pigpiod
pigs pfs 18 25000   # PWM freq 25 kHz (quiet)
pigs p 18 128       # 50% duty (0..255)
```

- Add a flyback diode across each fan (e.g., 1N5819) to protect from inductive kickback.

### 6.2 Auto fan curve (pseudo-flow)
- Read CPU temp from /sys/class/thermal/thermal_zone0/temp
- Map to duty cycle (e.g., 0% < 45 °C, 30% @ 55 °C, 60% @ 65 °C, 100% @ 75 °C)
- Run as a systemd service on boot.

## 7) Automation examples
- Adaptive brightness:
  
   Map TSL25911 lux → display brightness via ddcutil (if display supports DDC/CI) or your panel’s vendor tool; otherwise use a software overlay/desktop brightness.
- Presence-aware playback:
   - If presence/ld2410 == present OR presence/ai == person, keep screens on and audio normal.
   - If both clear for N seconds, pause playback and dim screen.
- Proximity wake:
  
   Use HC-SR04P distance < 100 cm to wake display and raise brightness.

## 8) Maintenance & tips
- Prefer HEVC/H.265 for 4K files; 4K H.264 is CPU-bound on Pi 5.
- Keep firmware updated (improves NVMe boot & PCIe quirks).
- Grounds common: if any sensor/fans draw from a separate 5 V/12 V source, tie grounds to the Pi when switching with the MOSFET.
- Cable management: route the camera ribbon away from the fan and HDMI adapter; dry-fit the Duo base, adapter, and ribbons before final mounting.
- Backups: image the NVMe after first clean setup so you can recover fast.

## 9) Known limitations
- Shared PCIe bandwidth: NVMe + Hailo share Gen2 x1; real-world NVMe ≈ ~300–450 MB/s when the NPU is also active (still far beyond microSD).
- DRM streaming caps: Some services limit resolution on ARM/Linux browsers; plan for 1080p on the web, and use local HEVC files for full 4K.
- No analog audio jack on Pi 5: use HDMI, USB audio, or your BT amp (as you’re doing).

## 10) Optional: software checklist (Raspberry Pi OS)
1. OS & updates: 64-bit Bookworm, fully updated
2. Interfaces: enable I²C / Serial / Camera in raspi-config
3. Media: Kodi (sudo apt install kodi), or Chromium/Firefox for streaming
4. AI: Hailo runtime + examples; rpicam-apps for capture
5. Sensors: Python libs (adafruit-circuitpython-tsl2591, adafruit-circuitpython-bme280, serial parser for LD2410)
6. Automation: MQTT broker (mosquitto), Node-RED or Home Assistant (optional)
7. Audio: pair BT amp; use pavucontrol or pactl to manage sinks
8. Fan service: pigpio + systemd fan curve script
9. Backlight control: brightness script driven by lux readings
10. Dashboards: kiosk web UI (local status, now-playing, environment)
