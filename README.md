# SATVIK-26

**A custom STM32H743 flight controller, designed from scratch for ArduPilot.**

36 × 36 mm, 30.5 mm stack mounting, 4-layer. My first PCB, built for Hack Club Macondo.

---
## What this is

SATVIK-26 is a flight controller: the computer that reads a drone's sensors and decides how fast to spin each motor. It runs [ArduPilot](https://ardupilot.org/) on an STM32H743VIT6, a 480 MHz Cortex-M7.

Anyone can just buy a flight controller and assemble a drone, but I wanted to understand one. There is a large gap between knowing that an IMU talks over Serial Peripheral Interface contrast to having placed that IMU's decoupling capacitor yourself, knowing why it sits exactly where it sits. The only way I know to cross that gap is to draw the schematic and route the board.

This repo is the complete hardware design: schematic, layout, gerbers, BOM, and the reasoning behind the decisions.

---
## Specifications

- MCU - STM32H743VIT6, LQFP-100, 480 MHz Cortex-M7, 2 MB flash/1 MB RAM
- IMU	- ICM-42688-P (TDK InvenSense), SPI1
- Barometer - BMP388 (Bosch), I2C2
- Storage - microSD on SDMMC1 for onboard logging
- USB-C with USBLC6-2SC6 ESD protection, DFU accessible
- Input - 2 to 6S direct battery, SMBJ26A TVS clamp
- 5 V rail - TPS54560 buck, 15 µH inductor, SS56 catch diode
- 3.3 V rail - AP2112K-3.3 LDO from the 5 V rail
- Sensing - VBAT divider and ESC current sense into ADC1
- Motor outputs - 8 PWM channels
- Serial - 6 UARTs on JST-GH connectors
- Board - 36 × 36 mm, 4-layer FR4, 1.6 mm, 3 mm corner radius
- Mounting - 4 × M3 at 30.5 mm (standard FC stack)
- Components - 80 footprints, 55 top and 25 bottom
- Weight - 40 g

---
## What makes this not a copy

**Dual airframe, 8 PWM outputs.** Most 5-inch quad boards break out four motor outputs. I mapped eight timer channels to headers so the same board can fly a quad now and a fixed-wing later, where you need servo channels for ailerons, elevator and rudder on top of throttle. Four outputs would have made this a dead end. Eight cost me real routing space on a 36 mm board, and I decided the flexibility was worth it.

**FPV designed in, not bolted on.** A dedicated UART is broken out for a video transmitter, and the 5 V rail is sized with headroom to feed a camera and VTX rather than just flight electronics. Retrofitting FPV onto a board that never planned for it means splicing into power and hunting for a spare serial port.

**A custom ArduPilot hwdef.** ArduPilot does not know what is wired where on a board it has never seen. I am writing a `hwdef.dat` describing every peripheral, pin function and timer assignment, and the build system compiles firmware specifically for your hardware. That port is part of this project. The board is not finished until it boots real firmware.

For the record: Macondo's flight controller guide builds a **rocket** FC on a different MCU with a battery charger IC. This is a drone FC on an H7, no charger, different power tree, different sensors.

---
## Design decisions

- **TPS54560 buck, rated to 60V.** On 4S that looks like overkill, and cheaper 28 V parts would work. I took the headroom deliberately. 6S is 25.2 V charged, uncomfortably close to a 28 V absolute maximum, and a motor decelerating hard dumps energy back into the rail.
- **Entire power section moved to the back.** I could not route the board with everything on top. Rather than grow it to 40 × 40 and lose stack compatibility, the buck, inductor, LDO, filter caps and thirteen control passives all went to the bottom. That freed the top for MCU fanout, and it put the noisiest thing on the board on the opposite side of a solid ground plane from the IMU, crystal and USB pair. A space fix that turned out to be the right electrical call.
- **IMU as close to the mounting centroid as possible.** The further a sensor sits from the centre of rotation, the more frame vibration it reads as acceleration that is not real. Vibration is the most common cause of bad behaviour in custom ArduPilot builds.
- **Lower KV motors chosen on purpose.** 1855 KV instead of the common 2400 KV. Lower prop RPM means less vibration reaching an IMU sitting on a board I designed myself with no proven mounting. It is also more efficient, which buys flight time. 2400 KV would give roughly 9:1 thrust-to-weight, which is race-quad territory and pointless for a GPS platform flying Loiter and Auto. It also makes tuning the IMU harder.

---
## Stackup
Four layers: **F.Cu** signal, **In1.Cu** GND plane, **In2.Cu** +3V3 plane, **B.Cu** signal and power section.

+3V3 touches 38 pads spread across the whole board. The net that is everywhere gets the plane. VBAT_RAW became short traces, which is all a local 8-pad net needs.

---
## The work

The tidy version above is not how it felt.
Placing components looks like the easy part. It is not. Almost anything fits on a board if all you care about is fitting. I built layouts that passed every clearance check and were physically impossible to route, with pads jammed against the board edge and no path out at all. Each one had to be scrapped and rebuilt.

The final routing push ran **18 hours straight, 1pm on July 22nd to 6am on July 23rd, without a break.** By the end: 1114 track segments, 89 vias, zero unconnected pads.

Some NotSoFun blunders:
- The IMU had been deleted from the board entirely, and I had not noticed
- The crystal had been deleted too
- Thirteen buck and LDO support components stranded on the wrong side after the power flip, meaning a switching regulator's feedback network would have had to via across the board
- A netclass rule silently matching nothing, so every "power" trace was drawing at 0.2 mm instead of 0.6 mm

AND YES... after so much work it is quite nail-biting to put it on JLCPCB to see if your work actually computes... the real test is still far off 

---

## Things I got wrong

- **Connector pin order.** My connectors run 5V, GND, TX, RX. Most off-the-shelf GPS modules and telemetry radios use Pixhawk order with ground on the _last_ pin. Stock cables do not line up, and plugging one in without checking could push 5 V into a UART pin. Fixable by re-pinning cables, so the board still works as designed, but it is a genuine mistake and first on the rev 2 list. Every cable gets continuity-tested before it goes anywhere near power.
- **Thermal reliefs on the ground pours.** Ground connected to pads through four thin spokes, which exists to stop the plane wicking heat away during _hand_ soldering. On a dense board several pads could not fit the minimum spoke count. This board is reflow assembled, so thermal relief was simply the wrong setting. Switched to solid, which is better electrically and thermally.
- **Traces do not follow components.** Move a part after routing and the copper stays behind, silently disconnected. I found this out by doing it. Placement gets finished properly before routing starts.

---
## What I would do differently

- **Stop designing version one like it is the last version.** Most of my pain came from trying to make the first board complete instead of making it _work_. A first revision exists to prove the core design is sound and to teach you what you got wrong. Every extra feature is another thing that can be wrong, and another thing you have to route.
- **Smaller connectors.** JST-GH is the right professional choice and it is what real flight controllers use, but it is physically large and it ate a serious amount of board edge and routing space on a 36 mm board. On a first iteration I would take something more compact, or simply break out fewer UARTs.
- **Skip the gimmicks.** Status LEDs and their resistors, extra headers, nice-to-haves. None of them made the board work. All of them competed for space against things that did.
- **Plan the escape routes before finalizing placement.** The lesson that cost the most time. A component is not placed correctly just because it fits and clears its neighbours. It is placed correctly when every one of its pads has a realistic path to where it needs to go.

---
## Airframe and flight dynamics

| Item                        | Mass       |
| --------------------------- | ---------- |
| 5-inch carbon frame         | 120 g      |
| 4 × 2207 motors             | 132 g      |
| 4-in-1 ESC                  | 14 g       |
| SATVIK-26 FC                | 40 g       |
| GPS, compass, mast          | 18 g       |
| RC receiver                 | 15 g       |
| Telemetry radio (air)       | 15 g       |
| Propellers                  | 18 g       |
| Wiring, standoffs, hardware | 35 g       |
| **Dry**                     | **407 g**  |
| 4S 1850 mAh LiPo            | 244 g      |
| **All-up weight**           | **~651 g** |
|                             |            |
___
## Airframe parts

| Part            | Chosen component                    |
| --------------- | ----------------------------------- |
| Frame           | TBS Source One V6, 5 inch           |
| Motors          | iFlight XING2 2207 1855 KV (×4)     |
| ESC             | Sequre Blueson A2 6S 65A, AM32      |
| Propellers      | Gemfan 51466 tri-blade, 5 inch      |
| GPS and compass | Sequre M10-252Q with QMC5883        |
| Telemetry       | Micoair 915 MHz LoRa pair           |
| RC link         | FlySky FS-i6X with FS-iA6B receiver |
| Battery         | GNB 1850 mAh 4S 120C, XT60 (×2)     |
| FC mounting     | Rubber or TPU soft mounts           |
With XING2 2207 1855 KV motors on 4S: roughly **6.8:1 thrust-to-weight**, hover throttle around **38 %**, estimated **5-8 minutes** of mixed GPS flying.

---
## Status

- [x] Schematic complete, ERC clean
- [x] Component placement
- [x] 4-layer routing complete, 0 unconnected pads
- [x] DRC clean
- [x] Gerbers verified in JLCPCB's viewer
- [x] BOM and pick-and-place prepared
- [ ] Boards ordered
- [ ] ArduPilot `hwdef` written and firmware compiled
- [ ] Bring-up
- [ ] Airframe assembly
- [ ] Maiden hover
- [ ] Loiter and Auto missions

**Bring-up is planned in order:** visual inspection, then a current-limited bench supply with the limit set low, then every rail verified with a multimeter before a battery touches the board, then USB enumeration and DFU, then flash ArduPilot, then confirm each peripheral in Mission Planner one at a time. Motors tested with **propellers removed**.

---
## Regulatory
Over 250 g, so under Transport Canada rules this must be registered and flown by a certified pilot. That means you (& I) am taking the Small Basic Exam and registering the aircraft.

Basic certification allows uncontrolled (Class G) airspace only, 30 m from bystanders, below 400 ft, within visual line of sight. Most of Toronto is controlled airspace, which needs Advanced certification and a manufacturer safety-assurance declaration, something a homebuilt aircraft cannot obtain by definition. So fly out of city, many apps show you the proper zones.

---
## Tools

**[KiCad 10](https://www.kicad.org/)** for schematic and layout. **[STM32CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html)** for pin mapping. [**ArduPilot**](https://ardupilot.org/) as target firmware, with SITL for development during parts lead time. **[JLCPCB](https://jlcpcb.com/)** for 4-layer fabrication and assembly.

---
CREDITS TO [OBSIDIAN](https://obsidian.md/), an .md file editor and notes taking app, for easing the process of writing in this file format.