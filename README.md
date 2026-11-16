# ESP32 Host Power & State Controller (HPSC)

[![GitHub release (latest by date)](https://img.shields.io/github/v/release/zazhigaev/esp32-host-power-state-controller?style=flat-square&color=blue)](https://github.com/zazhigaev/esp32-host-power-state-controller/releases)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg?style=flat-square)](LICENSE)
[![Platform: ESP32-C3](https://img.shields.io/badge/Platform-ESP32--C3-orange.svg?style=flat-square)](https://www.espressif.com/en/products/socs/esp32-c3)
[![Framework: Arduino / PlatformIO](https://img.shields.io/badge/Framework-PlatformIO-lightgrey.svg?style=flat-square)](https://platformio.org/)

**ESP32 Host Power & State Controller (HPSC)** is an out-of-band hardware power management and state monitoring controller built on the **ESP32-C3 Super Mini**. Designed primarily for industrial/embedded platforms (such as the IPC-R300), it bridges physical power control lines and Embedded Controller (EC) UART telemetry with a responsive Web UI and Home Assistant via MQTT.

Unlike basic smart relays, HPSC parses real-time POST codes and ACPI power state transitions directly from the host's EC bus to accurately track boot progression, host uptime, and system hangs—enabling autonomous hardware watchdog recovery.

## Key Features

* **Out-of-Band Power Control:** Galvanically isolated (optocoupler) actuation of `Power SW` (short pulse & 5-second hard force-off) and `Reset` headers.

* **EC / POST Code Telemetry:** Real-time parsing of UART logs from the motherboard's Embedded Controller, decoding AMI / Intel FSP POST checkpoints and hardware fault codes.

* **Accurate State & Uptime Tracking:**

  * Tracks distinct platform states: `Offline (S4/S5)`, `Booting (POST)`, `Online (S0)`, and `Boot Hung`.

  * Maintains independent **Host Uptime** (resilient against non-zero runtime POST events) and **ESP32 Uptime**.

  * Detects power loss via UART bus silence and RX line state.

* **Autonomous Hardware Watchdog:** Automatically detects POST hangs (stalled >30s during boot) and executes a staged recovery sequence (soft ACPI power-off pulse $\rightarrow$ fallback hardware reset $\rightarrow$ power-on).

* **Safe Auto-Power-On:** Optional delayed power-on upon AC restoration, strictly suppressed if the host is already running or was manually shut down.

* **Asynchronous Web Interface & Captive Portal:**

  * Zero-configuration AP mode (`HPSC-Config-AP`) with non-blocking Wi-Fi scanning, DHCP/Static IP setup, and mDNS (`http://hpsc.local`).

  * Protected by HTTP Basic Authentication with thread-safe credential updates and physical `BOOT` button (GPIO9) password reset.

* **Home Assistant & MQTT Integration:** Real-time state publishing, JSON telemetry payload, LWT (Last Will and Testament), and remote power/reset command topics.

## Hardware & Pinout

Built around an **ESP32-C3 Super Mini** powered via the motherboard's `5VSB` (5V Standby) rail so the controller remains active while the host is powered down.

| ESP32-C3 Pin | Function | Connection / Notes | 
| ----- | ----- | ----- | 
| **GPIO 1** | `PIN_PWR` | Optocoupler driving motherboard `PWR_SW` header (Active HIGH) | 
| **GPIO 3** | `PIN_RESET` | Optocoupler driving motherboard `RESET_SW` header (Active HIGH) | 
| **GPIO 4** | `UART_RX` | Connects to Host EC / Debug `TX` (115200 8N1, internal pulldown) | 
| **GPIO 5** | `UART_TX` | Connects to Host EC / Debug `RX` (115200 8N1) | 
| **GPIO 8** | `PIN_LED` | Onboard status LED (Fast blink in AP mode, solid off/idle in STA) | 
| **GPIO 9** | `PIN_BOOT` | Hardware `BOOT` button — hold for 3s at power-up to reset HTTP credentials | 

> **Safety Note:** Always use optocouplers (e.g., PC817 / LTV-817) between the ESP32 GPIOs and the motherboard front-panel headers to prevent ground loops and protect the MCU. Ensure UART voltage levels match (3.3V TTL).

## Architecture & Firmware Design

The firmware (`v6.4`) is engineered for continuous 24/7 operation on single-core RISC-V FreeRTOS:

* **Thread-Safe Command Queue:** Web server callbacks (`AsyncWebServer`) and MQTT handlers never touch NVS flash or hardware GPIOs directly. All actions are dispatched via a FreeRTOS `SystemCommand` queue (`cmdQueue`) to `loopTask`.

* **Atomic State Protection:** Shared state (`PcStatus`), authentication caches, and Wi-Fi scan buffers are guarded by FreeRTOS mutexes and critical sections (`portMUX_TYPE`) to prevent race conditions and heap fragmentation.

* **Non-Blocking Execution:** All timing operations (optocoupler pulses, watchdog recovery stages, Wi-Fi scans, and delayed restarts) use non-blocking state machines.

## Getting Started

### 1. Building & Flashing

This project uses [PlatformIO](https://platformio.org/).

```bash
# Clone the repository
git clone https://github.com/zazhigaev/esp32-host-power-state-controller.git
cd esp32-host-power-state-controller

# Build and upload firmware
pio run -t upload
```

### 2. Initial Wi-Fi Setup (Captive Portal)

1. On first boot (or when no Wi-Fi credentials are stored), the ESP32 starts an Access Point:

   * **SSID:** `HPSC-Config-AP`

   * **Password:** `P@$$w0rd`

2. Connect to the AP and open `http://192.168.4.1` (or follow the captive portal prompt).

3. Select your Wi-Fi network, choose **DHCP** or **Static IP**, and click **Save & Connect**.

4. Once connected to your LAN, access the dashboard at `http://hpsc.local` or its assigned IP address.

   * **Default Web Login:** `admin` / `admin` *(Change immediately via the Security panel in the Web UI)*.

> **Forgot Web Password?** Hold the onboard **BOOT (GPIO 9)** button while powering on or resetting the ESP32-C3 for 3 seconds until the LED flashes rapidly. Credentials will reset to `admin` / `admin`.

## MQTT & Home Assistant Integration

Configure your MQTT broker IP, port, and credentials directly from the Web UI under **MQTT Settings**.

### Published Telemetry Topics

| Topic | Payload Example | Description | 
| ----- | ----- | ----- | 
| `homeassistant/sensor/hpsc/status` | `online`, `booting`, `offline`, `hung`, `restarting` | Current host state (LWT: `offline`) | 
| `homeassistant/sensor/hpsc/post_code` | `00000000`, `00000099` | Latest hex POST code from EC | 
| `homeassistant/sensor/hpsc/post_desc` | `OS Running / Heartbeat` | Human-readable POST description | 
| `homeassistant/sensor/hpsc/pwr_state` | `S0 (Active)`, `S4/S5 (Off)` | ACPI / power rail state | 
| `homeassistant/sensor/hpsc/host_uptime` | `145230` | Host continuous online uptime (seconds) | 
| `homeassistant/sensor/hpsc/esp_uptime` | `512400` | Controller uptime (seconds) | 
| `homeassistant/sensor/hpsc/state` | `{"status":"online","post_hex":"00000000",...}` | Combined JSON state payload | 

### Command Topics

| Topic | Accepted Payloads | Action | 
| ----- | ----- | ----- | 
| `homeassistant/switch/hpsc/power/set` | `PRESS`, `PULSE`, `ON` | Triggers a 200ms `Power SW` pulse | 
| `homeassistant/switch/hpsc/power/set` | `FORCE_OFF` | Triggers a 5000ms hard power-off hold | 
| `homeassistant/switch/hpsc/reset/set` | `PRESS`, `RESET`, `ON` | Triggers a 200ms hardware `Reset` pulse | 

### Example Home Assistant Configuration (`configuration.yaml`)

```yaml
mqtt:
  sensor:
    - name: "Server Host Status"
      state_topic: "homeassistant/sensor/hpsc/status"
      icon: "mdi:server"

    - name: "Server POST Code"
      state_topic: "homeassistant/sensor/hpsc/post_code"
      value_template: "0x{{ value }}"
      icon: "mdi:chip"

    - name: "Server POST Description"
      state_topic: "homeassistant/sensor/hpsc/post_desc"
      icon: "mdi:text-box-outline"

    - name: "Server Host Uptime"
      state_topic: "homeassistant/sensor/hpsc/host_uptime"
      unit_of_measurement: "s"
      device_class: "duration"

  button:
    - name: "Server Power Pulse"
      command_topic: "homeassistant/switch/hpsc/power/set"
      payload_press: "PRESS"
      icon: "mdi:power"

    - name: "Server Hard Power Off"
      command_topic: "homeassistant/switch/hpsc/power/set"
      payload_press: "FORCE_OFF"
      icon: "mdi:power-plug-off"

    - name: "Server Hardware Reset"
      command_topic: "homeassistant/switch/hpsc/reset/set"
      payload_press: "RESET"
      icon: "mdi:restart"
```

## License

This project is licensed under the **MIT License** — see the [LICENSE](LICENSE) file for details.