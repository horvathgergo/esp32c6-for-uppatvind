<p align="left">
  <a href="https://ko-fi.com/horvathgergo">
    <img src="https://ko-fi.com/img/githubbutton_sm.svg" alt="Support me on Ko-fi">
  </a>
</p>
If you can, please support the development and testing of prototypes with €1.
Your kind support greatly helps cover components, PCBs, and tools.


# Smart Air Purifier ESP32-C6 (Project Uppatvind)

Smart controller for IKEA UPPÅTVIND Air Purifiers (ESP32 version)

# Hardware

![Image](https://github.com/user-attachments/assets/0d447e7e-4666-4d62-a054-76011d08d878)

Fan Controller v2.0 – Ready for Testing 😎

The board is based on the **ESP32-C6-WROOM-1N8** module, enabling support for **Wi-Fi 6, Bluetooth 5, Zigbee 3.0, and Thread**.

The previous voltage regulator has been replaced with a better alternative. The new **AP63203WU** operates at a higher switching frequency (>1 MHz) and requires **smaller and more cost-effective external components**.

![Image](https://github.com/user-attachments/assets/757d55d2-8a10-4db7-82bf-5cf7b4133f8b)

MCU programming: The board supports UART programming only via the TX and RX pin headers on the right side of the ESP module (there is no USB interface on the board). Therefore, a **USB-to-TTL converter is required** to upload firmware. USB-TTL converters are cheap and broadly available.

On the back of the board, there is a push button and a blue indicator LED, just like on the original board. Note: The filter reset LED and filter reset button are not implemented. I tested the board without them but intentionally left placeholders for their positions on the PCB for future use.

![Image](https://github.com/user-attachments/assets/d04a9067-6acb-4a9d-ad6e-93fc652556c3)

It is recommended to power the module via the VIN and GND pins during the initial setup. The board is compatible with a +5–24V supply.

Note: In serial bootloader mode, you must **supply at least +5V**. A standard +3.3V supply will not work, as voltage drop across the buck converter may prevent proper operation. Make sure to provide sufficient input voltage during flashing. Supplying +5V is safe, as the **VIN pin is not directly connected to the ESP chip**.

The ESP32-C6 enters serial bootloader mode when **GPIO9 (BOOT)** is held low during reset. It is recommended to use jumper caps on the **BOOT** and **RST** pin headers.

Flashing procedure:
1. Connect the USB-to-TTL converter to the board.
2. Power the device.
3. Place a jumper cap on the BOOT pin.
4. Place a jumper cap on the RST pin (both pins are now held low).
5. Remove the jumper cap from the RST pin.
6. Flash the firmware.
7. If flashing is successful, remove the jumper cap from the BOOT pin.
8. Restart the device.

# Components

| Designator | Items                              | Designation              |
|------------|------------------------------------|--------------------------|
| IC1        | ESP32-C6-WROOM-1-N8                | ESP modul                |
| C2         | 22uF, >6V, 1206                    | Decoupling capacitor     |
| C4         | 100nF, >6V, 0805                   | Decoupling capacitor     |
| R1         | 10K, 0805                          | IO9 resistor             |
| R4         | 10K, 0805                          | EN resistor              |
| R5         | 10K, 0805                          | IO8 resistor             |
| C1         | 10uF, >30V, 1206                   | Buck input capacitor     |
| C9         | 22uF, >6V, 1206                    | Buck output capacitor    |
| C11        | 22uF, >6V, 1206                    | Buck output capacitor    |
| C3         | 100nF, >6V, 0805                   | Buck bootstrap capacitor |
| L1         | HPI0630-4R7, L_6.7x7.0             | Buck inductor            |
| U1         | AP63203WU, TSOT-23-6               | Buck converter           |
| J1         | JST_XH 2P                          | Power                    |
| J2         | JST_XH 4P                          | Fan                      |
| F1         | 1A, 2410, SIBA SMD 157000          | Fuse                     |
| J3         | PinHeader_1x02_P2.54mm_Vertical    | UART                     |
| J4         | PinHeader_1x02_P2.54mm_Vertical    | RST                      |
| J5         | PinHeader_1x02_P2.54mm_Vertical    | BOOT                     |
| SW1        | Tact Push Button, 6x6mm, 5mm       | Main control button      |
| SW2        | K2-1102BG (not sure)               | Filter reset button      |
| LED1       | OF-SMD2012B, 0805, Blue            | Control LED              |
| LED2       | OF-SMD2012R, 0805, Red             | Filter LED               |
| R2         | 100R, 0805                         | Control LED resistor     |
| R3         | 100R, 0805                         | Filter LED resistor      |


# ESPHOME Configuration

The ESP32-C6 chip is now officially supported by ESPHome, so you can use its Thread capabilities without any issues. It has been tested with ESPhome version 2026.01 using the following example.

Example config:

```yaml
substitutions:
  friendly_name: Air Purifier
  fan_name: air_purifier

esphome:
  name: air-purifier
  

esp32:
  board: esp32-c6-devkitc-1
  framework:
    type: esp-idf

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "xxxxxxxxxxx"

ota:
  - platform: esphome
    password: "xxxxxxxxxxxxxxxxxx"

#THREAD
network:
  enable_ipv6: true

openthread:
  tlv: "YOUR TLV"


output:
  - platform: ledc
    pin: 18
    frequency: 150 Hz
    id: pwm_output
    min_power: 0.5
    max_power: 0.5
    zero_means_zero: true

  - platform: gpio
    pin: 23
    id: fan_led

fan:
  - platform: speed
    output: pwm_output
    name: ""
    id: $fan_name

    speed_count: 3
    preset_modes:
      - "Low"
      - "Medium"
      - "High"

    restore_mode: ALWAYS_OFF

    on_speed_set:
      then:
        - lambda: |-
            int s = id($fan_name).speed;

            // Auto-on:
            if (!id($fan_name).state && s > 0) {
              auto call = id($fan_name).turn_on();
              call.set_speed(s);
              call.perform();
              return;
            }

            // Frequency counter: 33, 66, 100 -> *3
            float freq = 0;
            if (s == 1) freq = 33 * 3;
            else if (s == 2) freq = 66 * 3;
            else if (s == 3) freq = 100 * 3;

            id(set_fan_freq).publish_state(freq);

    on_turn_on:
      then:
        - output.turn_on: pwm_output
        - output.turn_on: fan_led
        - lambda: |-
            int s = id($fan_name).speed;
            if (s <= 0) s = 1;

            float freq = 0;
            if (s == 1) freq = 33 * 3;
            else if (s == 2) freq = 66 * 3;
            else if (s == 3) freq = 100 * 3;

            id(set_fan_freq).publish_state(freq);

    on_turn_off:
      then:
        - output.turn_off: pwm_output
        - logger.log: "Fan Turned Off!"
        - output.turn_off: fan_led
        - lambda: id(set_fan_freq).publish_state(0);

sensor:
  - platform: pulse_meter
    pin:
      number: 19
      mode: INPUT_PULLUP
    unit_of_measurement: 'RPM'
    accuracy_decimals: 0
    id: fan_tach
    name: "$friendly_name Fanspeed"
    internal: true
    timeout: 2s
    filters:
      - multiply: 0.5
      - throttle: 1s

  - platform: template
    id: set_fan_freq
    name: "$friendly_name Fan Frequency"
    filters:
      - throttle: 100ms
    on_value:
      then:
        - output.ledc.set_frequency:
            id: pwm_output
            frequency: !lambda 'return (int)x;'
        - logger.log:
            format: "Set PWM frequency: %d"
            args: [ "(int)x" ]

binary_sensor:
  - platform: gpio
    id: fan_button
    pin:
      number: 21
      mode: INPUT_PULLUP
      inverted: true
    filters:
      - delayed_on: 30ms

    on_multi_click:
      # ---- LONG PUSH (>=2s) -> OFF ----
      - timing:
          - ON for at least 2s
        then:
          - lambda: |-
              auto call = id($fan_name).turn_off();
              call.perform();

      # ---- SHORT PUSH -> STEP ----
      - timing:
          - ON for 50ms to 1s
        then:
          - lambda: |-
              auto f = id($fan_name);

              // If OFF, then turn on with preset LOW (Speed 1)
              if (!f->state) {
                auto call = f->turn_on();
                call.set_speed(1);
                call.perform();
                return;
              }

              // If ON, then step: 1 -> 2 -> 3 -> OFF
              int s = f->speed;
              if (s < 1 || s > 3) s = 1;

              s++;
              if (s > 3) {
                // After 3 steps, OFF
                auto call = f->turn_off();
                call.perform();
              } else {
                auto call = f->turn_on();
                call.set_speed(s);
                call.perform();
              }

