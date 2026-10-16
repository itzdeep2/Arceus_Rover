# Surveillance rover: architecture, power and wiring

## Architecture

```
 3S LiPo 11.1V
      |
  [fuse + switch]
      +----------------------+
      |                      |
  L298N 12V             LM2596 IN
      |                      |
  4x DC motors          5.10V rail: Uno, ESP32-CAM, L298N logic,
      ^                           DHT11, MQ-2, NEO-6M
      | ENA IN1 IN2 ENB IN3 IN4
+-----+-----------------------+  57600 8N1   +----------------------+
| Arduino Uno R3              | D8 -[1k/2k]->| ESP32-CAM GPIO3 (RX) |
| motors + 5 sensors          |<-- D0 (RX)   | GPIO1 (TX)           |
+-----------------------------+              +----------+-----------+
                                                        | Wi-Fi AP
                                                  phone or laptop
                                          :80 dashboard + /ws   :81 /stream
```

The Uno owns the motors and all five sensors. The ESP32-CAM owns Wi-Fi, the camera and the dashboard. They share one UART pair at 57600 baud. The Uno sends one JSON line per second, the ESP32-CAM sends single-character drive commands back.

Commands: F forward, B backward, L pivot left, R pivot right, S stop. The Uno stops by itself if no command arrives for 500 ms, so the dashboard repeats the held command every 150 ms.

## Pin mapping: motor driver

```
+-------------------+--------------------+-----------------------------------+
| From              | To                 | Note                              |
+-------------------+--------------------+-----------------------------------+
| Uno D5            | L298N ENA          | PWM left side, remove ENA jumper  |
| Uno D6            | L298N IN1          | left direction                    |
| Uno D7            | L298N IN2          | left direction                    |
| Uno D10           | L298N ENB          | PWM right side, remove ENB jumper |
| Uno D11           | L298N IN3          | right direction                   |
| Uno D12           | L298N IN4          | right direction                   |
| L298N OUT1 / OUT2 | left motor pair    | two motors in parallel            |
| L298N OUT3 / OUT4 | right motor pair   | two motors in parallel            |
| L298N GND         | common ground rail | shared with Uno and ESP32-CAM     |
+-------------------+--------------------+-----------------------------------+
```

## Pin mapping: sensors

```
+--------------+-------------------------+------------------------------------+
| From         | To                      | Note                               |
+--------------+-------------------------+------------------------------------+
| Uno D2       | DHT11 DATA              | 10k pull-up to 5V on a bare sensor |
| Uno A0       | MQ-2 AO                 | analog output                      |
| Uno A4 (SDA) | BMP280 SDA + BH1750 SDA | shared I2C bus                     |
| Uno A5 (SCL) | BMP280 SCL + BH1750 SCL | shared I2C bus                     |
| Uno D3 (RX)  | NEO-6M TX               | GPS NMEA into the Uno              |
| Uno D4 (TX)  | NEO-6M RX               | optional, through 1k/2k divider    |
| Uno 3.3V     | BMP280 VCC + BH1750 VCC | keeps I2C pull-ups at 3.3V         |
| Uno 5V rail  | DHT11, MQ-2, NEO-6M VCC | from the LM2596 rail               |
| BMP280 SDO   | GND                     | I2C address 0x76                   |
| BMP280 CSB   | 3.3V                    | selects I2C mode                   |
| BH1750 ADDR  | GND                     | I2C address 0x23                   |
+--------------+-------------------------+------------------------------------+
```

## Pin mapping: Uno to ESP32-CAM link

```
+-----------------------+--------------------------+-------------------------------------+
| From                  | To                       | Note                                |
+-----------------------+--------------------------+-------------------------------------+
| Uno D8 (TX)           | 1k, then ESP32-CAM GPIO3 | top leg of the divider              |
| ESP32-CAM GPIO3 (U0R) | 2k to GND                | bottom leg, node sits at 3.33V      |
| ESP32-CAM GPIO1 (U0T) | Uno D0 (RX)              | 3.3V is read as HIGH by the AVR     |
| Uno D9                | not connected            | dummy RX of the link SoftwareSerial |
| ESP32-CAM 5V          | 5V rail                  | 470uF low-ESR cap next to it        |
| ESP32-CAM GND         | common ground rail       |                                     |
+-----------------------+--------------------------+-------------------------------------+
```

## Power distribution

```
+-------------+------------------------+---------------------------------------+
| From        | To                     | Note                                  |
+-------------+------------------------+---------------------------------------+
| LiPo +      | 5A fuse, then switch   | 3S 11.1V nominal                      |
| switch out  | L298N 12V + LM2596 IN+ |                                       |
| LiPo -      | L298N GND + LM2596 IN- | common ground rail                    |
| LM2596 OUT+ | 5V rail                | trim to 5.10V before connecting loads |
| 5V rail     | L298N 5V terminal      | remove the 5V-EN jumper               |
| 5V rail     | Uno 5V pin             | do not also plug in USB               |
+-------------+------------------------+---------------------------------------+
```

Set the LM2596 to 5.10V with nothing attached, then connect the loads. Put 0.1uF ceramic capacitors across each motor's terminals. The L298N drops roughly 2V per channel, so the motors see about 9V at full duty on a fresh pack. Use a low-voltage alarm on the LiPo and stop at 3.3V per cell.

## Logic levels

The Uno drives D8 at 5V and ESP32 GPIOs are not 5V tolerant. A 1k series resistor with a 2k resistor to ground drops 5V to 3.33V at GPIO3. The other direction needs no shifter: ESP32-CAM GPIO1 outputs 3.3V, which the ATmega328P reads as HIGH.

The BMP280 and BH1750 modules are powered from the Uno 3.3V pin so their I2C pull-ups idle at 3.3V while the Uno still reads that as HIGH.

## Flashing

GPIO1 and GPIO3 are also the ESP32-CAM programming UART, and the Uno D0 pin is its USB UART. Unplug the D0 wire before uploading to the Uno. Unplug the GPIO1 and GPIO3 wires before flashing the ESP32-CAM, and tie GPIO0 to GND only for the flash.
