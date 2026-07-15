# SATVIK-26
A custom STM32H743 flight controller for a 5-inch freestyle quadcopter, running ArduPilot. Designed dual-airframe (quad now) and FPV-ready.

## What makes it mine
- Dual-airframe design: 8 PWM outputs so the same board flies a quad or a fixed-wing with servos.
- FPV-ready: dedicated UART and filtered power broken out for a video transmitter, no redesign needed later.
- Custom ArduPilot hwdef port written for this board from scratch.

## Hardware
- MCU: STM32H743VIT6
- IMU: ICM-42688-P (SPI)
- Barometer: DPS310 / BMP388
- microSD (SDMMC), USB-C, 2-6S power input

## Status
In design. Schematic and PCB in progress.