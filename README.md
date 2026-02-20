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

  #Main LED
  - platform: gpio
    pin: 23
    id: fan_led

fan:
  - platform: speed
    output: pwm_output
    name: "$friendly_name"
    id: $fan_name
    restore_mode: RESTORE_DEFAULT_OFF

    on_speed_set:
      lambda: |-
        int s = id($fan_name).speed;

        if(s != id(fan_speed)){
          id(fan_speed) = s;
          id(set_fan_freq).publish_state(s);
        }

        if (s == id(speed_1)) id(current_step) = 1;
        else if (s == id(speed_2)) id(current_step) = 2;
        else if (s == id(speed_3)) id(current_step) = 3;
        else id(current_step) = 0;

    on_turn_on:
      - output.turn_on: fan_led #Main LED
      - lambda: |-
          id(set_fan_freq).publish_state(id($fan_name).speed);

    on_turn_off:
      - output.ledc.set_frequency:
          id: pwm_output
          frequency: !lambda 'return int(0);'
      - logger.log: "Fan Turned Off!"
      - output.turn_off: fan_led #Main LED

sensor:
  - platform: pulse_meter
    pin: 
      number: 19
      mode: INPUT_PULLUP
    unit_of_measurement: 'RPM'
    accuracy_decimals: 0
    id: fan_tach
    name: $friendly_name Fanspeed #HA entity
    filters:
      - multiply: 0.5
      - throttle: 1s

  - platform: template
    id: set_fan_freq
    name: "$friendly_name Fan Frequency"  #HA entity
    filters:
      - multiply: 3
      - lambda: |-
          if (id($fan_name).state){
            if(x > 300) { // limit max frequency
              return 300;
              
            } else if(x < 36 && x > 3) { // limit low frequency
              return 36;
              
            } else if(x <= 3) {
              return 0;
              
            } else {
              return x;
              
            }
          } else {
            return 0;
          }
    on_value:
      then:
        - output.ledc.set_frequency:
            id: pwm_output
            frequency: !lambda 'return int(x);'
        - logger.log: "on_value called"
        

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
              id(current_step) = 0;
              auto call = id($fan_name).turn_off();
              call.perform();

      # ---- SHORT PUSH -> SPEED SELECT ----
      - timing:
          - ON for 50ms to 1s
        then:
          - lambda: |-
              
              if(!id($fan_name).state){
                id(current_step) = 1; 
                auto call = id($fan_name).turn_on();
                call.set_speed(id(speed_1));
                call.perform();

                
                id(fan_speed) = id(speed_1);
                id(set_fan_freq).publish_state(id(speed_1));
                return;  
              }

              
              id(current_step)++;
              if (id(current_step) > 3) id(current_step) = 0;

              int speed_val = 0;
              if (id(current_step) == 1) speed_val = id(speed_1);
              if (id(current_step) == 2) speed_val = id(speed_2);
              if (id(current_step) == 3) speed_val = id(speed_3);

              if (id(current_step) == 0) {
                auto call = id($fan_name).turn_off();
                call.perform();
              } else {
                auto call = id($fan_name).turn_on();
                call.set_speed(speed_val);
                call.perform();
              }

              
              id(fan_speed) = id($fan_name).speed;
              id(set_fan_freq).publish_state(id($fan_name).speed);



  - platform: template                      #Ha entity
    name: "$friendly_name Speed 1"
    lambda: |-
      if (id(set_fan_freq).state >= 60 and id(set_fan_freq).state < 100 and id($fan_name).state) {
        return true;
      } else {
        return false;
      }
      
  - platform: template                      #Ha entity
    name: "$friendly_name Speed 2"
    lambda: |-
      if (id(set_fan_freq).state >= 100 and id(set_fan_freq).state < 200 and id($fan_name).state) {
        return true;
      } else {
        return false;
      }

  - platform: template                      #Ha entity
    name: "$friendly_name Speed 3"
    lambda: |-
      if (id(set_fan_freq).state >= 200 and id($fan_name).state) {
        return true;
      } else {
        return false;
      }      



globals:
  - id: current_step
    type: int
    restore_value: yes
    initial_value: '0'

  - id: fan_speed
    type: int
    restore_value: yes
    initial_value: '0'
    
  - id: speed_1
    type: int
    restore_value: yes
    initial_value: '33'
    
  - id: speed_2
    type: int
    restore_value: yes
    initial_value: '66'
    
  - id: speed_3
    type: int
    restore_value: yes
    initial_value: '100'


