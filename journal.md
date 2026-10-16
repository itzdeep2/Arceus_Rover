# Arceus Rover — Engineering Journal

## Session 1: Architecture, Pin Mapping & Power Planning

**Date:**  16/10/2026 3.15 PM 
**Time spent:** 2 hours 05 minutes

### What I worked on

I established the hardware foundations and low-level wiring blueprint for the Arceus Rover project, documented in `docs/WIRING.md`[cite: 1]. 

Before committing to a custom carrier PCB layout, I mapped out every pin interface between the Arduino Uno R3, the ESP32-CAM, the L298N motor driver, the LM2596 step-down regulator, and all five onboard sensors (DHT11, MQ-2, BMP280, BH1750, and NEO-6M)[cite: 1]. 

### What I learned

* **Power rails cannot be an afterthought:** Planning the power budget revealed how sensitive the ESP32-CAM is to instantaneous current draw during Wi-Fi transmission bursts (often exceeding 300–400mA), highlighting why decoupling capacitors and an independent buck converter rail are critical[cite: 1].
* **Mixed-voltage buses require careful planning:** The BMP280 and BH1750 sensors run on 3.3V[cite: 1]. Powering them from the Uno’s 3.3V pin keeps their I2C bus pull-ups naturally clamped at 3.3V, which the Uno can safely read on its A4/A5 pins without dedicated level shifter ICs[cite: 1].

### Next steps

- Implement the basic Arduino Uno motor control loop and verify L298N drive sequences[cite: 1].
- Add serial packet parsing with the 500ms safety timeout on the Uno[cite: 1].
- Begin laying out the custom carrier PCB schematic in EasyEDA/KiCad to integrate the level shifter, power distribution rails, and module headers into a single board.

### Evidence

- Complete reference pinout and power diagrams committed to [`docs/WIRING.md`](docs/WIRING.md)[cite: 1].
