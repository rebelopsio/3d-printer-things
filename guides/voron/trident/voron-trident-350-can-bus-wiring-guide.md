# Voron Trident 350 Complete Wiring Guide: Octopus Pro, CAN Bus, and EBB36

**This guide covers every wire, connector, and pin assignment for a self-sourced Voron Trident 350×350 build using a BTT Octopus Pro v1.1 mainboard, Raspberry Pi 4, BTT U2C v2.1, BTT EBB36 v1.2 CAN toolhead, Voron StealthBurner with CW2 extruder, BTT ADXL345 v2.0, and BTT Knomi 2 display.** All wire gauges, lengths, connector types, and MCU pin assignments are specified for the 350mm frame size. The CAN bus architecture reduces the toolhead umbilical from ~17 wires to just 4, dramatically simplifying wiring while improving reliability.

> ⚠️ **Critical safety notice**: This build involves **120V/240V AC mains wiring** for the heated bed. AC wiring should only be performed by individuals with electrical competence. Always verify your work with a multimeter before applying power. Never work on AC wiring while the printer is plugged in.

---

## 1. Power system: PSU, grounding, and safety

### Main 24V power supply

The recommended PSU is the **Mean Well LRS-200-24** (200W, 24V, 8.8A). For 230V regions, the **RSP-200-24** variant includes power factor correction and universal AC input. The LRS-200-24 has a physical **110V/230V selector switch** on the side — setting this incorrectly will destroy the PSU on first power-up.

**Mains AC wiring** from the IEC C14 inlet to the PSU uses **16AWG (1.25mm²) minimum** stranded wire. Route Live (L) through the inlet's integrated fuse and rocker switch, then to the PSU L terminal. Neutral (N) goes directly to the PSU N terminal. Earth (PE/⏚) connects to the PSU earth terminal, then continues to the printer frame via a **serrated star washer** for reliable contact. Use **Wago 221-series** lever nuts for all AC junction points.

| AC Component | Wire Gauge | Connector |
|---|---|---|
| IEC C14 to PSU (L, N) | 16AWG min (14AWG preferred) | Spade terminals or Wago 221 |
| Earth to frame | 16AWG | Ring terminal + serrated washer on M5 bolt |
| IEC inlet fuse | 5A for 120V / 3A for 230V | Glass fuse, integrated in inlet |

**24V DC distribution** from PSU to the Octopus Pro uses **16AWG silicone-jacketed wire**, approximately **30–40cm** for a typical 350 electronics bay layout. The Octopus Pro has three power input screw terminals that all share a common ground:

| Terminal | Voltage | Purpose | Wire Gauge |
|---|---|---|---|
| **PWR** | DC 24V | Board logic, fans, heaters, onboard regulators | 16AWG |
| **MOTOR_POWER** | DC 24V | Stepper driver power (supports up to 60V for HV drivers) | 16AWG |
| **BED_IN** | DC 24V | Heated bed MOSFET output (unused with AC bed + SSR) | Not connected for AC bed |

Connect **both PWR and MOTOR_POWER** positive terminals to PSU V+, and their ground terminals to PSU V−. Use **bootlace ferrules** on all wire ends entering screw terminals. For an AC heated bed, **BED_IN is left disconnected** — the SSR handles bed heating separately.

### Onboard voltage regulators

The Octopus Pro v1.1 includes onboard DC-DC converters: **12V at 4A**, **5V at 8A**, and **3.3V at 1A**. The 5V rail can power the Raspberry Pi through the dedicated Pi power header or through the UART header's 5V pins. However, a **dedicated 5V/3A USB-C power supply** for the Pi is more reliable and isolates the Pi from board power transients. If powering the Pi from the Octopus 5V rail, use the provided header and **always remove the USB 5V power supply jumper** near the USB-C port to prevent conflict between USB 5V and the onboard regulator.

### Grounding and safety

**Frame grounding is mandatory.** Connect earth ground from the PSU PE terminal to the frame using **16AWG wire** with a ring terminal and serrated star washer. For AC heated beds, **ground the aluminum bed plate** independently — the Trident bed has an M4 mounting point specifically for the PE cable with serrated washer. If using multiple DC power supplies, connect all DC V− terminals together to establish a common voltage reference.

---

## 2. AC heated bed and SSR wiring for 350mm

A 350×350mm Voron Trident requires an **AC silicone heater pad rated at 650–750W**, matched to your mains voltage (120V or 230V). AC heating is strongly recommended over DC for the 350 build — a DC bed heater at this size would draw **30+ amps** at 24V, exceeding the Octopus Pro's BED_OUT rating of 300W.

### SSR selection and wiring

Use a quality SSR: **Omron G3NA-210B-DC5** (10A) or **Crydom D2425** (25A). Cheap counterfeit SSRs can fail in the closed position, causing uncontrolled heating and fire. Mount the SSR on a **metal bracket with thermal compound** for heat dissipation.

**DC control side** (low voltage, from Octopus Pro):

- Connect the **HE1 header** on the Octopus Pro (pin **PA3**) to SSR DC input terminals (pins 3 and 4)
- HE1 positive → SSR pin 3 (+), HE1 negative → SSR pin 4 (−)
- **Wire gauge**: 22–24AWG (low-current control signal, ~20mA)
- **Cable length**: ~20–30cm within electronics bay

**AC load side** (mains voltage):

- AC Live from PSU/inlet → SSR AC input terminal (pin 1)
- SSR AC output terminal (pin 2) → bed heater Live wire
- AC Neutral connects directly to bed heater (bypasses SSR)
- **Wire gauge**: **14AWG minimum** (12AWG recommended for 120V/750W = 6.25A)
- Route AC wires separately from DC wiring; use Wago 221 connectors for junctions

**Klipper configuration**: In `[heater_bed]`, set `heater_pin: PA3`. Calculate `max_power` as SSR continuous rating divided by actual bed current. For the Omron G3NA-210B at 4A continuous (no heatsink): `max_power: 0.64` for a 750W/120V bed.

### Bed thermistor

The bed thermistor is an **NTC 100K B3950**, typically pre-installed on the silicone heater pad. It connects to the **TB header (pin PF3)** on the Octopus Pro via a **2-pin JST-XH** connector. Use **24AWG** wire, routed through the Z cable chain (**100cm** for all Trident sizes). Add a **thermal fuse rated 115–125°C** in series with the bed heater AC wiring as a hardware safety backup against thermal runaway.

---

## 3. CAN bus topology: Pi → U2C → EBB36

The CAN bus architecture is the backbone of this build's toolhead communication. Instead of routing 17+ individual wires through drag chains, a **4-wire umbilical** carries all toolhead signals digitally.

### Signal chain

```
Raspberry Pi 4 ──USB-C──▶ BTT U2C v2.1 ──CAN bus (4-wire)──▶ BTT EBB36 v1.2
   (Klipper host)          (USB-to-CAN bridge)                   (Toolhead MCU)
```

The **U2C v2.1** (STM32G0B1 MCU, CandleLight firmware) converts USB to CAN bus protocol. It is powered entirely by the Pi's USB port — **no separate 24V power is required for the U2C itself**. The 24V screw terminals on the U2C are **pass-through only**, providing a convenient way to route 24V power to the CAN connector for the EBB36. The U2C presents as a `gs_usb` device on Linux.

### U2C v2.1 connections

| Port | Connection | Cable |
|---|---|---|
| USB-C (CAN_IN) | Raspberry Pi USB-A port | USB-A to USB-C, ~30cm |
| CAN_OUT (Molex Microfit 3.0 2×2) | To EBB36 umbilical | CANH, CANL, +24V, GND |
| 24V screw terminal | From 24V PSU | 16–18AWG, ~30cm |

The U2C has **three electrically-identical CAN output connectors** (JST, Molex Microfit 3.0, and screw terminal) — use whichever suits your cable. The **Molex Microfit 3.0 2×2** connector is most common because it matches the EBB36's input connector directly.

### Umbilical cable specifications

The CAN umbilical carries exactly **4 wires**:

| Wire | Function | Gauge | Notes |
|---|---|---|---|
| **CANH** | CAN High signal | 22–24AWG | Must be twisted pair with CANL |
| **CANL** | CAN Low signal | 22–24AWG | Must be twisted pair with CANH |
| **+24V** | Power to EBB36 | 20AWG minimum (18AWG preferred) | Carries all toolhead current |
| **GND** | Ground return | 20AWG minimum (18AWG preferred) | Carries all toolhead current |

**Cable length for 350 build**: **750–1000mm** from electronics bay to toolhead, including service loop.

**Recommended cables**: IGUS Chainflex CF9-05-04 (4-conductor, 20AWG, shielded, rated for continuous flex) or IGUS CF113-018-D (6-conductor with dedicated twisted pairs). For DIY cables, use **PTFE/FEP-insulated wire** — avoid PVC insulation inside enclosed chambers.

**Color convention** (varies by manufacturer): Red = +24V, Black = GND, Yellow = CANH, Green = CANL.

### Umbilical connectors

At the **toolhead end**: The EBB36 uses a **Molex Microfit 3.0 2×2 (4-pin)** connector for CAN+power input. Crimp Molex Microfit 3.0 female pins onto your cable ends and insert into the 2×2 housing. **Pin-for-pin miswiring will permanently damage the EBB36** — verify with a multimeter before connecting.

For a **detachable connection point** at the frame (optional but recommended), install a **GX16-4 aviation connector** (4-pin) at the rear top of the A-drive motor mount. Pin assignment: Pin 1 = +24V, Pin 2 = GND, Pin 3 = CANH, Pin 4 = CANL.

### Termination resistors (120Ω)

CAN bus requires **exactly two 120Ω termination resistors**, one at each end of the bus. Install the **120R jumper cap** on both the U2C v2.1 and the EBB36 v1.2. With both jumpers installed and the system unpowered, measure resistance between CANH and CANL — it should read **~60Ω** (two 120Ω resistors in parallel).

### CAN bus speed

All devices must use the **same CAN bus speed**. BTT precompiled firmware defaults to **1,000,000 bps (1 Mbit/s)**. Configure the Linux CAN interface on the Pi:

```
# /etc/network/interfaces.d/can0
allow-hotplug can0
iface can0 can static
    bitrate 1000000
    up ifconfig $IFACE txqueuelen 1024
```

---

## 4. Octopus Pro v1.1 complete pin assignments and jumper settings

![Voron Trident + Octopus Pro Wiring Diagram](https://raw.githubusercontent.com/VoronDesign/Voron-Documentation/main/build/electrical/images/trident_octopus_wiring_no_stepstick.png)
*Official Voron Trident wiring diagram for BTT Octopus Pro — from the VoronDesign/Voron-Documentation GitHub repository. In a CAN bus build, the toolhead wiring shown here terminates at the U2C instead, and the 4-wire umbilical replaces the drag chain.*

### Critical V1.1 pin changes

The Octopus Pro **v1.1 changed four pins** from v1.0. Using v1.0 configurations on v1.1 hardware will cause incorrect heater control and potential safety hazards.

| Signal | V1.0 Pin | **V1.1 Pin** |
|---|---|---|
| HE0 (Hotend Heater 0) | PA2 | **PA0** |
| HE2 (Hotend Heater 2) | PB10 | **PB0** |
| MOTOR3 Enable | PA0 | **PA2** |
| Neopixel/RGB | PB0 | **PB10** |

### Required jumper configuration

Before connecting any wires, configure all jumpers. For **TMC2209 UART mode** under each used driver socket: insert **one jumper only** in the rightmost position (closest to motor connector), and remove the other three. **Remove ALL DIAG jumpers** — leaving them installed will interfere with endstop function. **Remove the USB 5V power jumper** to prevent conflict between USB 5V from the Pi and the onboard 5V regulator. Set all fan voltage jumpers to **V_FUSED (VIN/24V)** for standard 24V fans. Set each motor power jumper to **RIGHT/VIN** for TMC2209 — the LEFT position routes MOTOR_POWER up to 60V and will destroy TMC2209 drivers.

### Stepper driver slot assignments

| Role | Driver Slot | STEP | DIR | ENABLE | UART | DIAG |
|---|---|---|---|---|---|---|
| **B Motor** (CoreXY, left) | DRIVER0 / MOTOR0 | PF13 | PF12 | PF14 | PC4 | PG6 |
| **A Motor** (CoreXY, right) | DRIVER1 / MOTOR1 | PG0 | PG1 | PF15 | PD11 | PG9 |
| **Z0** (Front Left) | DRIVER2 / MOTOR2_1 | PF11 | PG3 | PG5 | PC6 | PG10 |
| **Z1** (Rear Center) | DRIVER3 / MOTOR3 | PG4 | PC1 | **PA2** ⚠️ | PC7 | PG11 |
| **Z2** (Front Right) | DRIVER4 / MOTOR4 | PF9 | PF10 | PG2 | PF2 | PG12 |
| Extruder (unused with CAN) | DRIVER6 / MOTOR6 | PE2 | PE3 | PD4 | PE1 | PG14 |

⚠️ **Z1 enable pin PA2 is V1.1-specific** (was PA0 on V1.0). Uncomment `enable_pin: !PA2` for V1.1 in your Klipper config.

### Heater, fan, thermistor, and endstop pin map

| Function | Header | Pin (V1.1) | Notes |
|---|---|---|---|
| Hotend heater | HE0 | **PA0** | Unused with CAN (EBB36 handles hotend) |
| **Bed SSR control** | HE1 | **PA3** | DC control signal to SSR |
| Case lighting (optional) | HE2 | **PB0** | PWM output for LED strips |
| Spare heater | HE3 | PB11 | Available |
| **Controller/electronics fans** | FAN2 | **PD12** | 3× 6020 skirt fans via splitter PCB |
| **Exhaust fan** | FAN3 | **PD13** | Rear chamber exhaust |
| **Nevermore filter** (optional) | FAN4 | **PD14** | Activated carbon filter fans |
| **Bed thermistor** | TB | **PF3** | NTC 100K B3950 |
| Chamber thermistor | T1 | **PF5** | NTC 100K, mounted in chamber |
| **X endstop** | STOP_0 | **PG6** | Microswitch, normally closed |
| **Y endstop** | STOP_1 | **PG9** | Microswitch, normally closed |
| **Neopixel** | RGB (J37) | **PB10** | For case LEDs (SB LEDs on EBB36) |

### PT100/PT1000 support (MAX31865 DIP switch)

| Configuration | SW1 | SW2 | SW3 | SW4 |
|---|---|---|---|---|
| 2-wire PT100 | ON | ON | ON | OFF |
| 2-wire PT1000 | ON | ON | OFF | ON |
| 4-wire PT100 | OFF | ON | ON | OFF |
| 4-wire PT1000 | OFF | ON | OFF | ON |

---

## 5. Stepper motors: wiring, gauges, and lengths for 350 build

The standard BOM specifies **LDO-42STH48-2004MAH** (NEMA 17, 48mm body, 2.0A, 1.8°) for A/B gantry motors. Use **24AWG silicone-jacketed** wire throughout. The table below gives estimated wire lengths for the 350 build — **add 15–20cm service loop** to each.

| Motor | Estimated Length |
|---|---|
| B motor (gantry left rear) | ~65–85cm |
| A motor (gantry right rear) | ~65–85cm |
| Z0 (front left) | ~75–95cm |
| Z1 (rear center, shortest run) | ~40–65cm |
| Z2 (front right) | ~75–95cm |

**Board side connectors**: The Octopus Pro motor headers use **JST-XH 4-pin (2.5mm pitch)**. For mid-wire disconnects, use **Molex Microfit 3.0 4-pin** (rated 5A, keyed for error-free reconnection). There is **no universal color standard** for stepper motor wires — always verify coil pairs with a multimeter in continuity mode before crimping. Pins 1+2 are Coil A, pins 3+4 are Coil B. If a motor runs backward, reverse it in Klipper config (`dir_pin: !PF12`) rather than re-pinning.

---

## 6. Toolhead wiring: everything on the EBB36 v1.2

![BTT EBB36 v1.1/v1.2 Official PIN Diagram](https://raw.githubusercontent.com/bigtreetech/EBB/master/EBB%20CAN%20V1.1%20and%20V1.2%20%28STM32G0B1%29/EBB36%20CAN%20V1.1%20and%20V1.2/Hardware/EBB36%20CAN%20V1.1%26V1.2-PIN.png)
*Official BTT EBB36 v1.1/v1.2 pinout diagram — from the bigtreetech/EBB GitHub repository. Your v1.2 board uses this pinout. Note that the hotend heater is on **PB13** (not PA2 as on V1.1) — this is the critical safety fix for the STM32 DFU boot issue.*

With CAN bus, **every toolhead component terminates at the EBB36** — nothing runs back to the Octopus Pro from the toolhead. The EBB36 v1.2 uses an **STM32G0B1CBT6** MCU with an onboard **TMC2209** stepper driver.

### EBB36 v1.2 complete pinout

| Function | Pin | Connector Type | Wire Gauge | Notes |
|---|---|---|---|---|
| **CAN + Power input** | PB0/PB1 (CAN), VIN/GND | Molex Microfit 3.0 2×2 | 20AWG power, 22AWG signal | 4-pin: CANH, CANL, +24V, GND |
| **Extruder stepper** | Step=PD0, Dir=PD1, En=PD2 | JST-XH 4-pin | 24AWG | Onboard TMC2209 (UART=PA15, addr 00) |
| **Hotend heater** | **PB13** | Screw terminal (ferrules) | 20AWG | ⚠️ V1.2: PB13 (was PA2 on V1.1), max 5A |
| **Hotend thermistor** | PA3 | JST-XH 2-pin | 24AWG | 4.7K pullup (NTC 100K); jumper for 2.2K (PT1000) |
| **Part cooling fan** | PA1 (FAN0) | JST-XH 2-pin | 24AWG | PWM capable, max 1A, 24V |
| **Hotend cooling fan** | PA0 (FAN1) | JST-XH 2-pin | 24AWG | PWM capable, max 1A, 24V |
| **TAP probe signal** | **PB6** (Endstop 3) | JST-PH 3-pin | 24AWG | 5V + GND + Signal; PB6 is 5V-tolerant |
| **Neopixel LEDs** | PD3 | JST-XH 3-pin | 24AWG | +5V, Data (PD3), GND |
| **ADXL345** (onboard) | CS=PB12, SCLK=PB10, MOSI=PB11, MISO=PB2 | Soldered to PCB | — | No external wiring needed |
| Endstop 1 | PB7 | JST-PH | 24AWG | Available |
| Endstop 2 | PB5 | JST-PH | 24AWG | Available |
| I2C | SDA=PB3, SCL=PB4 | Header | — | For filament sensor or DIY |

⚠️ **V1.2 safety fix**: The hotend heater pin changed from PA2 to **PB13** because PA2 goes HIGH during STM32 DFU bootloader mode, which would uncontrollably heat the hotend. Always verify you are using the V1.2 pin in your Klipper config (`heater_pin: EBBCan:PB13`).

The hotend heater cartridge (40W or 60W at 24V) connects to the **HE0 screw terminal** using **20AWG silicone-jacketed wire** with bootlace ferrules — polarity does not matter for resistive heaters. The hotend thermistor (NTC 100K or PT1000) connects to **TH0 (PA3)** via JST-XH 2-pin. Both part cooling fan (5015 blower → FAN0/PA1) and hotend cooling fan (4010 axial → FAN1/PA0) use **24AWG, 2-wire JST-XH 2-pin** connectors.

### Voron TAP probe on EBB36

Voron TAP replaces the traditional Z endstop entirely. The OptoTap PCB (V2 or V2.4) connects to the EBB36 endstop header with +5V, GND, and Signal on **PB6** (Endstop 3). Klipper config: `[probe] pin: ^EBBCan:PB6`. The `^` enables the internal pull-up resistor. PB6 is **5V-tolerant** on the STM32G0B1, making it safe for all OptoTap versions. In your Klipper stepper_z config, set `endstop_pin: probe:z_virtual_endstop` and `homing_retract_dist: 0`.

### StealthBurner Neopixel LEDs

The StealthBurner LED PCB uses **WS2812B (RGB)** or **SK6812 (RGBW)** addressable LEDs: **1 logo LED + 2 nozzle LEDs** standard, or **10 LEDs total** for the "Rainbow Barf" variant. Connect the 3-wire cable (+5V, Data, GND) to the EBB36 Neopixel header: top pin = +5V, middle pin = Data (**PD3**), bottom pin = GND.

```ini
[neopixel sb_leds]
pin: EBBCan:PD3
chain_count: 3          # 10 for Rainbow Barf
color_order: GRBW       # GRB for WS2812B RGB
```

The CW2 extruder stepper connects to the EBB36 motor port via **JST-XH 4-pin**, 24AWG wire. Klipper references: `step_pin: EBBCan:PD0`, `dir_pin: EBBCan:PD1`, `enable_pin: !EBBCan:PD2`, `uart_pin: EBBCan:PA15`.

---

## 7. Endstops, probes, and homing

Use **D2F-series** microswitches (Omron D2F-5 or D2F-01) in **normally closed (NC)** configuration for both X and Y endstops. Both route through the Y cable chain at **~170cm** for the 350 build.

| Endstop | Octopus Pin | Connector | Length (350) |
|---|---|---|---|
| X endstop | STOP_0 (**PG6**) | JST-XH 2-pin | ~170cm (through Y chain) |
| Y endstop | STOP_1 (**PG9**) | JST-XH 2-pin | ~170cm (through Y chain) |

With TAP, no separate Z endstop is needed. The nozzle itself probes the bed. Restrict probing temperature to **150°C maximum** to protect the print surface.

For **sensorless homing** on X/Y (never Z): install DIAG jumpers for DRIVER0 and DRIVER1, configure `endstop_pin: tmc2209_stepper_x:virtual_endstop`, and use macros that reduce motor current to ~0.7A during homing. Start with `driver_SGTHRS: 255` and decrease until reliable (typical range: 100–140).

---

## 8. Fans: headers, voltages, and assignments

| Octopus Header | Pin | Assignment | Voltage |
|---|---|---|---|
| FAN2 | PD12 | **Electronics bay fans** (3× 6020 via splitter) | 24V |
| FAN3 | PD13 | **Chamber exhaust fan** (1× 6020) | 24V |
| FAN4 | PD14 | **Nevermore filter** (2× 5015 or 4010) | 24V |
| FAN0 | PA8 | **Bed fans** (optional) | 24V |
| FAN1 | PE5 | Spare | — |
| FAN5 | PD15 | Spare | — |

Configure the electronics bay fans as `[controller_fan]` in Klipper so they activate when steppers are energized. Each fan header has an individual 3-pin voltage jumper — set all to **24V (V_FUSED)**. If using Noctua 12V fans, change only that specific header's jumper.

---

## 9. Raspberry Pi 4 and communication links

| Pi USB Port | Connected Device | Cable | Purpose |
|---|---|---|---|
| USB-A #1 | BTT Octopus Pro v1.1 | USB-A to USB-C, ~30cm | Klipper MCU serial |
| USB-A #2 | BTT U2C v2.1 | USB-A to USB-C, ~30cm | CAN bus bridge |
| USB-A #3 | BTT ADXL345 v2.0 (temporary) | USB-A to USB-C | Resonance testing |
| USB-A #4 | Webcam (optional) | USB-A | Print monitoring |

Use a **dedicated 5V/3A USB-C power supply** for the Pi to isolate it from Octopus Pro power rail transients. If instead powering from the Octopus 5V rail, **remove the USB 5V power jumper** on the Octopus Pro to prevent backfeeding.

The **BTT Knomi 2 connects via WiFi** (ESP32, 802.11 b/g/n 2.4GHz) — the USB-C port on the Knomi is for firmware flashing only, not runtime communication. Power it via its **MX1.25 (1.25mm pitch) connector** from any convenient 5V–24V source in the StealthBurner body. It queries the Klipper/Moonraker API over WiFi and requires **gcode macro wrappers** in `printer.cfg` for M109, M190, G28, and BED_MESH_CALIBRATE.

---

## 10. ADXL345 v2.0 and resonance compensation

The **BTT ADXL345 v2.0** connects to the Raspberry Pi via USB serial (not SPI). The **EBB36 v1.2 has an onboard ADXL345** via SPI that serves as the primary accelerometer for X-axis calibration. The standalone ADXL345 v2.0 is most useful **mounted on the bed** for Y-axis resonance measurement.

```ini
[adxl345]
cs_pin: EBBCan:PB12
spi_software_sclk_pin: EBBCan:PB10
spi_software_mosi_pin: EBBCan:PB11
spi_software_miso_pin: EBBCan:PB2
axes_map: x,y,z
```

Mount the standalone ADXL345 v2.0 on the **StealthBurner side bracket** using the included M3 hardware and silicone dampers. Adjust `axes_map` based on physical orientation.

---

## 11. Wire management, routing, and drag chains

With the EBB36 CAN toolhead, the X-axis drag chain (which would carry 15+ wires) is **eliminated entirely**. The 4-wire CAN umbilical runs as a free-hanging cable with bungee-style support from the top rear frame extrusion. The Y cable chain is retained for X/Y endstop wires (4 wires, 170cm for 350); the Z cable chain is always retained (100cm).

### Recommended wire lengths for 350 build

All lengths include ~200mm service loop.

| Wire Run | Length | Gauge |
|---|---|---|
| CAN umbilical (electronics bay to toolhead) | 800–1000mm | 20AWG power, 22AWG signal |
| A/B motor to Octopus | 700–900mm | 24AWG |
| Z0/Z2 motor to Octopus | 800–1000mm | 24AWG |
| Z1 motor to Octopus | 500–700mm | 24AWG |
| X/Y endstops (through Y chain) | 1700mm | 24AWG |
| Bed thermistor (through Z chain) | 1000mm | 24AWG |
| SSR to Octopus HE1 | 200–300mm | 22AWG |
| PSU to SSR (AC side) | 300–400mm | 14AWG |
| SSR to bed heater (AC) | 1000–1200mm | 14AWG |
| Electronics bay fans to Octopus | 200–400mm | 24AWG |
| Exhaust fan to Octopus | 400–600mm | 24AWG |
| Chamber thermistor to Octopus | 500–800mm | 24AWG |

Use **silicone-jacketed, high-strand-count wire** for all moving applications. Use **PTFE or FEP-insulated wire** for the CAN umbilical and hotend-area connections. **Never use PVC wire inside an enclosed heated chamber** — it degrades and off-gasses above 60°C. Strain relief: PG7 cable gland at the A-drive mount for the umbilical, and a zip tie 10–20mm from the EBB36 Molex connector on the toolhead side.

---

## 12. Complete connector reference table

| Connector Type | Pitch | Wire Gauge | Max Current | Used For | Crimping Tool |
|---|---|---|---|---|---|
| **JST-XH** | 2.5mm | 22–26AWG | 3A | Octopus Pro: endstops, thermistors, fans, motor headers; EBB36: motor, fans, thermistor, Neopixel | IWISS IWS-3220M or Engineer PA-09 |
| **JST-PH** | 2.0mm | 22–26AWG | 2A | EBB36: endstop/probe headers; StealthBurner PCB fan connectors | IWISS IWS-3220M or Engineer PA-09 |
| **Molex Microfit 3.0** | 3.0mm | 20–24AWG | 5A | Mid-wire disconnects (motors, heaters, endstops); EBB36 CAN+power input (2×2); CAN umbilical | IWISS IWS-3220M or Molex 63819-0000 |
| **MX1.25** (JST MX) | 1.25mm | 26–28AWG | 1A | Knomi 2 power connector | Fine-pitch crimper or pre-made cable |
| **GX16-4** | — | 18–22AWG | 5A | Optional umbilical disconnect at frame | Solder to pins |
| **Wago 221** | — | 12–24AWG | 20A | All AC mains wire junctions | Tool-free lever operation |
| **Bootlace ferrules** | — | 10–28AWG | Per gauge | All screw terminal connections (Octopus Pro power, EBB36 heater, PSU) | Self-adjusting ferrule pliers |
| **Spade/ring terminals** | — | 12–18AWG | Per gauge | SSR screw terminals, PSU terminals, frame ground | Standard terminal crimper |
| **USB-C** | — | — | — | Octopus Pro data, U2C data, ADXL345 data, Pi power, EBB36 firmware flash | — |

The **IWISS IWS-3220M** is the single most versatile crimping tool for this build, handling JST-PH, JST-XH, and Molex Microfit 3.0 pins. The **Engineer PA-09** or **PA-21** produces higher-quality JST crimps specifically. Always perform a tug test on every crimp.

> ⚠️ **Never solder wires inserted into screw terminals.** Solder cold-flows under clamping pressure over time, loosening the connection, increasing resistance, and potentially causing arcing or fire. Always use ferrules for screw terminals.

---

## Conclusion: key takeaways for a successful build

The CAN bus architecture fundamentally simplifies this build. By replacing the 15+ wire X-axis drag chain with a **4-wire umbilical** to the EBB36, you gain reliability, easier maintenance, and cleaner routing. The Octopus Pro v1.1's **PA2/PA0 pin swap** from v1.0 is the single most common source of configuration errors — triple-check that your Klipper config uses V1.1 pin assignments, especially `enable_pin: !PA2` for Z1 and `heater_pin: PA0` for HE0.

Before first power-on, verify with a multimeter: no short between 24V and GND on the Octopus Pro, correct PSU voltage switch setting (110V/230V), SSR DC control polarity, and **60Ω across CANH/CANL** confirming both CAN termination resistors are active. Leave the hotend heater and bed SSR **disconnected** until Klipper firmware is successfully loaded.

A single bad crimp in the CAN umbilical will cause intermittent communication failures that are extremely difficult to diagnose. Crimp carefully, test every connection, and label every wire before routing it into the printer.
