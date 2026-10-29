# Sensor Node — Bill of Materials (BOM)

## 1. Project Overview

**Purpose:** Detect whether a parking bay is occupied and transmit its status wirelessly to the Raspberry Pi gateway using point-to-point LoRa.

**Design:**
- Microcontroller/radio: Seeed Studio Wio-E5 mini
- Vehicle detection: A02YYUW waterproof ultrasonic sensor
- Power: Solar panel + single-cell LiFePO4 battery
- Communication: Private point-to-point LoRa
- Installation: Outdoor, weather-resistant enclosure
- Operating mode: Periodic measurement and transmission, with low-power operation where practical

## 2. BOM — One Sensor Node

| No. | Component | Specification | Qty. | Estimated Unit Cost (PHP) |
|---:|---|---|---:|---:|
| 1 | LoRa development board | Seeed Studio Wio-E5 mini | 1 | ₱1,700 |
| 2 | Ultrasonic sensor | A02YYUW, waterproof, UART output | 1 | ₱900–₱1,200 |
| 3 | Solar panel | 6 V, 10 W | 1 | ₱500–₱900 |
| 4 | Battery | 3.2 V, 20 Ah LiFePO4 | 1 | ₱1,900–₱3,400 |
| 5 | Solar charge controller | Designed for a 1-cell LiFePO4 battery and compatible with the panel | 1 | ₱300–₱800 |
| 6 | DC-DC boost converter | Low-quiescent-current, regulated 5 V output | 1 | ₱100–₱300 |
| 7 | Sensor power switch | Logic-level MOSFET load switch | 1 | ₱50–₱150 |
| 8 | Battery protection | LiFePO4-compatible protection circuit, if not built into the battery | 1 | ₱150–₱300 |
| 9 | Outdoor enclosure | UV-resistant, IP65 or better | 1 | ₱300–₱800 |
| 10 | Cable glands | Weather-resistant, sized for wiring | 2–4 | ₱50–₱150 |
| 11 | Mounting hardware | Brackets, stainless screws, fasteners | 1 set | ₱150–₱400 |
| 12 | Wiring and connectors | Suitable wire, terminals, heat-shrink tubing | 1 set | ₱150–₱350 |
| 13 | Antenna | Compatible with the selected legal LoRa frequency | 1 | ₱150–₱400 |

## 3. Estimated Cost per Node

| Cost category | Estimated amount |
|---|---:|
| Electronics and sensor | ₱2,650–₱3,350 |
| Solar power system | ₱2,800–₱5,400 |
| Enclosure and installation materials | ₱650–₱1,700 |
| Antenna | ₱150–₱400 |
| **Estimated total** | **₱6,250–₱10,850** |

*These are preliminary budgeting estimates, not confirmed supplier quotations. Shipping, labor, and spare parts are excluded.*

## 4. Power and Wiring Requirements

### Power path

Solar panel → LiFePO4-compatible solar charge controller → 3.2 V LiFePO4 battery → regulated power supply → Wio-E5 mini.

The ultrasonic sensor should have a separately switched power connection so the controller can turn it off between readings if the sensor supports reliable power cycling.

**Important:** Verify the exact Wio-E5 mini board revision and its input-voltage range. Do not connect the battery directly to the board unless the documented input range explicitly supports the battery's full voltage range.

The charge controller must support a single-cell LiFePO4 battery. A generic 12 V solar controller or a standard 4.2 V lithium-ion charger is not an appropriate substitute.

## 5. Sensor Node Operation

1. Wake the controller on its configured measurement interval.
2. Power the A02YYUW sensor.
3. Wait for the sensor to stabilize and collect valid UART readings.
4. Determine the bay status using calibrated distance thresholds and filtering.
5. Transmit the bay ID, status, battery voltage, sequence number, and diagnostic information over LoRa.
6. Switch off the sensor if appropriate.
7. Return to a low-power state until the next measurement cycle.

### Status values

- `VACANT`
- `OCCUPIED`
- `UNKNOWN`
- `SENSOR_FAULT`

Invalid readings must not be interpreted as `VACANT`.

## 6. Before Purchasing

- Verify Philippine NTC rules for the intended LoRa frequency and transmit power.
- Confirm the A02YYUW's UART voltage levels and wiring against the exact sensor variant.
- Test vehicle detection at the intended mounting height and angle.
- Measure actual current consumption before finalizing the solar panel and battery.
- Confirm the battery's protection circuitry and the solar controller's charging profile.
- Purchase and test **one complete node first** before buying components for all parking bays.

## 7. Initial Prototype Acceptance Criteria

- Correctly identifies an empty bay and a parked vehicle under expected installation conditions.
- Reports `UNKNOWN` or `SENSOR_FAULT` when readings are invalid.
- Transmits data reliably to the gateway at the installation site.
- Recovers after a temporary loss of radio communication.
- Charges safely from the solar panel.
- Operates through nighttime and expected low-sun periods, verified through actual power measurements.
