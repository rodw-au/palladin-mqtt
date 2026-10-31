Paladin-MQTT
This is an Arduino IDE sketch for an ESP32 (I used WROOM32) that allows control of a Paladin hot water diverter from Home assistant using MQTT
The commands are documented in the INO file.
Link to esp32 https://link.amazon/B0gsCEaz1

I mounted it in a 3D printed case externally  so I could plug a USB cable in to modify the script but now its working well, you could mount it internally
<img width="311" height="644" alt="Screenshot 2026-11-01 05 14 03" src="https://github.com/user-attachments/assets/eea5271b-b1d0-45f0-a107-a6fc5fb4ca9d" />
<img width="1216" height="684" alt="paladin-board" src="https://github.com/user-attachments/assets/013f0967-6f7b-4d77-bb7c-b40de2c649e8" />
<img width="385" height="684" alt="paladin-esp32" src="https://github.com/user-attachments/assets/42fbd4c1-ecaa-44e3-877b-f9a2bed6c12b" />

The wiring just needs you to solder some pins to an internal board and use +-5 volts to power the esp32. Connect the esp32 serial port to 2 the rx and tx pins (TX to TX, RX to RX)
```
// Serial2 Pins for Paladin Interface
#define RXD2 16  // Connect to Paladin RX
#define TXD2 17  // Connect to Paladin TX
```
You will also need to set your wifi credentials and your Home Assistant MQTT user name and password in the source code.
Monitor the device on the Arduino IDE serial port monitor.
