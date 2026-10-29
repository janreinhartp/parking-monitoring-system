# Parking Monitoring System: LoRa Sensor Node Design

## 1. Purpose

Design a solar-powered sensor node for each parking bay. Each node measures the distance to a vehicle or the empty parking surface, determines occupancy, and sends compact status messages to the LoRa HAT on the Raspberry Pi 4B gateway.

The Pi manages the HDMI entrance display and sends updates to Cloudflare Workers over HTTPS through a 4G router. There is no separate receiver board between the node and the Pi.

Shared rules live in [system-architecture.md](system-architecture.md). Packet bytes live in [radio-protocol.md](radio-protocol.md). This document covers the node hardware, power, firmware modules, and node acceptance checks.

## 2. Selected hardware

| Component | Selected specification |
|---|---|
| Development board | Seeed Studio Wio-E5 mini |
| MCU and radio | STM32WLE5JC with integrated sub-GHz LoRa radio |
| Distance sensor | A02YYUW waterproof ultrasonic sensor |
| Solar panel | 6 V, 10 W nominal |
| Battery | 1-cell LiFePO4, 3.2 V nominal, 20 Ah |
| Solar charger | LiFePO4-compatible charging controller |
| Main board supply | Low-quiescent-current boost converter, sized for the Wio-E5 mini input range |
| Sensor supply | Regulated rail matching the A02YYUW datasheet, with high-side power switching |
| Battery monitoring | Wio-E5 mini ADC with a properly designed voltage divider |
| Sensor power switching | P-channel MOSFET high-side switch with NPN gate driver, subject to circuit validation |
| Antenna | Matching antenna for the selected module frequency and connector |
| Enclosure | Outdoor-rated enclosure with suitable cable glands and antenna placement |

### Official references

- Wio-E5 mini: https://wiki.seeedstudio.com/LoRa_E5_mini/
- Wio-E5 datasheet and resources: https://www.seeed.cc/product/lora-series-module-en
- A02YYUW datasheet: obtain the datasheet for the exact purchased sensor revision.

## 3. Important power-supply requirement

The Wio-E5 mini development board specifies a 3.7 V to 5 V supply. The 1-cell LiFePO4 battery operates below this range during much of its discharge cycle.

Therefore:
- Do not connect the battery directly to the Wio-E5 mini board's supply input.
- Use a boost converter with a regulated output within the board's specified input range.
- Select a converter with low quiescent current, good efficiency at light loads, and enough transient current for radio transmission.
- Verify the actual board input current in active, receive, transmit, and sleep states.
- Account for the onboard USB-to-UART circuitry and indicator LED when estimating power consumption.
- Do not assume the module's published 2.1 µA sleep figure represents the current drawn by the complete development board.

The A02YYUW sensor should have its own switched, regulated supply. Confirm the required voltage, startup time, operating current, and UART logic levels before finalizing the circuit.

## 4. Functional requirements

1. Measure the ultrasonic sensor at approximately one-second intervals.
2. Filter readings to reduce false occupancy transitions.
3. Determine whether the parking bay is VACANT, OCCUPIED, UNKNOWN, or SENSOR_FAULT.
4. Send occupancy changes promptly to the gateway.
5. Send periodic health messages when occupancy remains unchanged.
6. Report distance, battery voltage, sensor health, and a sequence number.
7. Detect missed acknowledgments and communication failures.
8. Support bounded retries without draining the battery during an outage.
9. Enter low-power states between measurement and transmission cycles.
10. Recover after resets, temporary radio loss, and sensor faults.

## 5. Occupancy detection

Implement per-slot calibration and hysteresis.

- VACANT: valid measurements indicate an empty bay.
- OCCUPIED: valid measurements indicate a vehicle is present.
- UNKNOWN: readings are stale, inconclusive, or insufficient to determine occupancy.
- SENSOR_FAULT: the sensor is disconnected, returning invalid data, or failing diagnostics.

Use multiple consecutive readings before changing occupancy state. Configure separate thresholds for entering and leaving OCCUPIED.

A sensor fault or stale measurement must never be interpreted as VACANT.

Store calibration values and occupancy thresholds in non-volatile memory. Provide a configuration method for commissioning each parking bay.

## 6. Sensor interface and power control

The A02YYUW uses a UART interface. The Wio-E5 mini exposes UART-capable pins.

Proposed UART connection, subject to confirmation against the exact board pinout:
- A02YYUW TX to Wio-E5 mini USART2 RX, PA3.
- A02YYUW RX to Wio-E5 mini USART2 TX, PA2, only if the selected sensor mode requires commands.
- Sensor VCC to the switched regulated sensor supply.
- Sensor GND to system GND.

Keep the Wio-E5 mini's onboard USB-to-UART interface available for development and diagnostics. Avoid assigning the sensor to pins already used by the onboard USB bridge.

### High-side sensor switching

Use a P-channel MOSFET to switch the sensor supply. An NPN transistor can drive the MOSFET gate if the selected circuit requires it.

The circuit must include:
- A MOSFET with suitable voltage rating and low on-resistance at the available gate drive.
- Correct gate pull-up and base resistor values.
- Decoupling capacitors near the sensor.
- Safe UART pin states before sensor power is removed.
- Protection against back-powering through UART or control pins.

Verify the circuit with the actual datasheets before wiring. Measure sensor-off current during hardware validation.

## 7. LoRa communication

Use the integrated LoRa radio in the Wio-E5 mini. No external LoRa transceiver board is required.

### Radio configuration

- Use private point-to-point LoRa communication between sensor nodes and the receiver.
- Configure the frequency, bandwidth, spreading factor, coding rate, and transmit power consistently across all nodes and the receiver.
- Verify permitted operating frequencies and radio settings for the Philippine deployment before ordering antennas or fixing radio parameters.
- Select the antenna for the actual operating frequency and install it outside any RF-shielding enclosure where practical.
- Calculate packet airtime before choosing heartbeat intervals and retries.

The factory AT firmware is primarily intended for LoRaWAN operation. For the selected private point-to-point architecture, develop firmware for the STM32WLE5 using the appropriate STM32 development tools and LoRa radio driver. Do not assume the default AT firmware provides the complete custom point-to-point protocol required by this project.

### Message structure

Use the v1 packet in [radio-protocol.md](radio-protocol.md): a 12-byte `STATUS` or `HEARTBEAT`, and a 5-byte `ACK` from the gateway. `STATUS` is sent on a confirmed occupancy transition and is acknowledged. `HEARTBEAT` is not acknowledged.

`CONFIG` and `CONFIG_ACK` are not in v1. Set node ID, thresholds, radio settings, and the heartbeat interval over the debug serial link and store them in non-volatile memory.

The gateway rejects duplicate and stale sequence numbers and stores the last accepted sequence per node.

## 8. Transmission strategy

Target an end-to-end occupancy update time of 1 to 3 seconds under normal operating conditions.

Initial firmware behavior:
1. Wake and measure approximately once per second. Do not transmit every sample.
2. Send a `STATUS` packet when occupancy is confirmed to have changed.
3. Send a `HEARTBEAT` when occupancy is unchanged, default every 60 seconds, using the slot offset in [radio-protocol.md](radio-protocol.md).
4. Transmit each `STATUS` sequence at most 3 times. Wait a random 250–750 ms between attempts.
5. Stop retrying that sequence after the third transmission. Set the `ACK_EXHAUSTED` flag on the next heartbeat.
6. Do not retransmit continuously during a gateway outage.
7. Track the last `STATUS` sequence that received an `ACK`.

Choose the final heartbeat interval, radio parameters, and retry policy after calculating airtime and checking the applicable radio regulations.

## 9. Solar and battery system

Initial power-system selection:
- 6 V, 10 W solar panel
- 3.2 V nominal, 20 Ah LiFePO4 battery
- LiFePO4-compatible solar charging controller
- Low-quiescent-current boost converter for the Wio-E5 mini board input
- Switched sensor power supply
- Battery protection and a fuse near the battery positive terminal
- ADC battery-voltage measurement with a suitable divider

The charger must support the actual panel characteristics and LiFePO4 charge profile. Include overcharge, over-discharge, and short-circuit protection appropriate to the selected battery.

The panel and battery are preliminary selections, not a verified energy balance. Actual performance depends on board current, sensor operating time, LoRa airtime, sunlight, shading, temperature, and converter efficiency.

## 10. Firmware structure

Organize the firmware into these modules:

- `main`: initialization and scheduling
- `sensor_manager`: UART acquisition and sensor power control
- `occupancy`: filtering, calibration, hysteresis, and state transitions
- `lora_manager`: radio configuration, packets, acknowledgments, and retries
- `power_manager`: battery voltage monitoring and low-power transitions
- `diagnostics`: sensor errors, radio errors, and watchdog handling
- `config_manager`: node ID, thresholds, radio settings, heartbeat configuration, and the next LoRa `sequence` in non-volatile memory

Use STM32 development tools and a supported STM32WL LoRa radio driver. Keep tasks non-blocking where practical and use timed scheduling rather than long delays.

## 11. Initial acceptance criteria

- Correct occupancy detection at the intended sensor mounting height and angle
- Occupancy updates delivered within the 1 to 3 second target under normal radio conditions
- No false VACANT state during sensor faults or communication loss
- Correct handling of duplicate packets and sequence numbers
- Recovery after radio interruption or power cycling
- Verified battery voltage measurement
- Measured board current in active, receive, transmit, and sleep states
- Measured sensor startup, operating, and switched-off current
- Tested communication at the maximum expected installation distance
- Verified solar charging and battery autonomy using measured energy consumption

## 12. Items to finalize before wiring

1. Confirm the Wio-E5 mini version and its permitted radio-frequency configuration.
2. Confirm the A02YYUW sensor revision, supply voltage, UART frame format, baud rate, and startup time.
3. Confirm the Wio-E5 mini pinout and available ADC pin.
4. Select the boost converter and verify its current consumption at light load.
5. Finalize the sensor power-switch circuit.
6. Select the matching antenna and connector.
7. Implement and test custom point-to-point LoRa firmware.
8. Calculate packet airtime and configure collision avoidance.
9. Measure full-board sleep current before finalizing the battery-life estimate.

## Architecture summary

A02YYUW sensor → switched sensor supply → Wio-E5 mini (STM32WLE5 + integrated LoRa) → LoRa HAT on the Raspberry Pi 4B → HDMI entrance display, and HTTPS sync to Cloudflare Workers through the 4G router.