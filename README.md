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

[pic]

MCU programming: The board supports UART programming only via the TX and RX pin headers on the right side of the ESP module (there is no USB interface on the board). Therefore, a **USB-to-TTL converter is required** to upload firmware. USB-TTL converters are cheap and broadly available.

[pic]

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
| SW1        | Tact Push Button, 6x6mm            | Main control button      |
| SW2        | Unknown                            | Filter reset button      |
| LED1       | OF-SMD2012B, 0805, Blue            | Control LED              |
| LED2       | OF-SMD2012R, 0805, Red             | Filter LED               |
| R2         | 100R, 0805                         | Control LED resistor     |
| R3         | 100R, 0805                         | Filter LED resistor      |





