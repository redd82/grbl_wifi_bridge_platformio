# GRBL Wi-Fi Bridge for ESP32-S3

This firmware lets LightBurn (or another GRBL client) communicate with a laser over Wi-Fi without modifying the laser:

```text
LightBurn  -- Wi-Fi / raw TCP -->  ESP32-S3  -- USB host / serial -->  GRBL laser
```

The ESP32-S3 creates a Wi-Fi access point and relays bytes between one TCP client and the laser's USB serial connection. GRBL continues to run on the laser; the bridge does not interpret or transform G-code.

## Hardware

The documented setup uses an ESP32-S3 development board with a UART USB connection and a native USB connection, and an Acmer S1 laser with a CH340 USB-serial interface.

- Use the board's UART USB port to flash the firmware and view logs.
- Connect the laser to the ESP32-S3 native USB port using a USB-C OTG adapter and the laser's USB cable. This is the USB host connection (GPIO19 D- and GPIO20 D+ on the documented board).
- Check the board's native-port VBUS wiring. Some boards do not supply 5 V from that port, so the laser's USB interface may need an appropriate 5 V VBUS source. Follow the board's electrical documentation and connect grounds; do not assume the port can power the laser.

Board USB connector roles and VBUS wiring vary. See [GUIDE_PLATFORMIO.txt](GUIDE_PLATFORMIO.txt) for notes on the documented board setup.

## Build and Flash

This is an ESP-IDF project packaged for PlatformIO, not an Arduino sketch. Install PlatformIO (the VS Code extension or CLI) and open this project directory. The component manifest requires ESP-IDF 5.1 or newer; the first build may download the toolchain and USB components.

```sh
pio run                         # build
pio run -t upload               # flash
pio device monitor              # view serial logs
pio run -t upload -t monitor    # flash and open the monitor
```

Flash and monitor through the UART port. If PlatformIO selects the wrong serial port, set `upload_port` and `monitor_port` in `platformio.ini`. The serial monitor baud rate is 115200.

## Connect

The current defaults are defined near the top of `src/main.cpp`:

| Setting | Default |
| --- | --- |
| Wi-Fi network (SSID) | `Acmer-S1` |
| Wi-Fi password | `AcmerS1234` |
| Access point channel | 6 (2.4 GHz) |
| ESP32-S3 address | `192.168.4.1` |
| TCP port | `23` |
| GRBL serial | `115200` baud, 8 data bits, no parity, 1 stop bit |

**Change the default Wi-Fi password before using the bridge outside a trusted environment.** The password is compiled into the firmware; change `AP_PASS` in `src/main.cpp` and rebuild. Use at least 8 characters.

1. Power the ESP32-S3 and connect the laser to its native USB host port.
2. Open the serial monitor. Look for the access-point startup message and `Waiting for laser on USB...`; after connecting the laser, look for `Laser connected`.
3. Join the `Acmer-S1` Wi-Fi network from the computer running LightBurn.
4. In LightBurn, add or configure a GRBL device using a Wi-Fi/TCP connection to `192.168.4.1`, port `23`.
5. Connect and check that the laser responds to status queries and commands.

The access point is limited to 2.4 GHz. The computer may lose internet access while connected to this network. LightBurn does not automatically discover the bridge; configure its IP address and port directly.

## Behavior and Limitations

- The TCP service is a raw byte stream; it does not use HTTP or add message framing.
- One TCP client is handled at a time.
- Client-to-laser data is written to the USB serial device; the bridge does not parse GRBL commands or real-time characters.
- Laser responses are buffered and forwarded to the connected client. Responses are discarded when no client is connected, and USB data may be dropped if the receive buffer fills.
- The USB host uses 115200 8N1, keeps DTR/RTS low, and waits for the laser to be unplugged or reconnected.
- This is a dedicated access point, not a bridge to an existing Wi-Fi network.

## Troubleshooting

- **No `Laser connected` message:** Check that the laser is on the native USB host port, the OTG adapter is correct, and the board supplies the required VBUS power.
- **LightBurn cannot connect:** Confirm the computer joined `Acmer-S1` and the configured address and port are `192.168.4.1` and `23`.
- **No GRBL response or garbled data:** Confirm the laser's serial baud rate is 115200 and check its USB power and cable.
- **Laser resets on connection:** The firmware holds DTR/RTS low; if resets continue, check the USB-serial board behavior and the `set_control_line_state` call in `src/main.cpp`.
- **Component download or build failure:** Ensure the first build has internet access, then retry `pio run`. The USB host dependencies are declared in `src/idf_component.yml`.

## Project Status

The PlatformIO build and upload commands completed successfully in the current workspace. Laser communication and LightBurn operation still require verification on the target hardware.