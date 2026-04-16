# Fixing "Homing Fail" & Firmware Update Loop on Creality Falcon 2 (12W/22W/40W)

If your Creality Falcon 2 has suddenly stopped moving, throws an instant `ALARM:6 (Homing fail)` without any physical travel, or reports `nan` in its Machine Position coordinates (`MPos`) when querying `?` via serial—and flashing the firmware via the SD Card fails to resolve it—your board is trapped in a corrupt boot loop.

Here is the exact step-by-step process to force the laser's ESP32-S3 brain to load a clean firmware image over USB.

## The Problem
The Creality Falcon 2 motherboard uses an Espressif ESP32-S3 microcontroller. Like many ESP chips, it supports Over-The-Air (OTA) updates and uses a partition table with a "Factory" slot and two "OTA" slots. When you update the firmware via the SD card, the bootloader updates an internal memory sector called `otadata` to permanently point to the newly downloaded software.

If that update corrupts, or if a bug inside the firmware (such as reading uninitialized Z-axis memory on a 2-axis machine) causes a `nan` (Not A Number) calculation, the board's floating-point math crashes. You cannot home, you cannot jog, and standard `$$` EEPROM resets (`$RST=*`) won't fix it if the firmware logic itself is flawed. 

Because the `otadata` partition forces the chip to keep booting from the corrupted OTA slot, standard USB firmware flashing tools normally fail to overwrite the *active* partition. We need to physically erase the `otadata` memory, forcing the laser back to its "Factory" slot where we will install a fresh, uncorrupted firmware update.

## Prerequisites
1. **USB-A to USB-C Cable**: Connect your computer directly to the Creality Falcon 2's USB-C port beside the SD card slot.
2. **Python & `esptool`**: Ensure you have Python 3 installed. Install the official Espressif flashing tool:
   ```bash
   pip install esptool
   ```
3. **Firmware `.bin` File**: Download the latest firmware for your specific Falcon 2 wattage from the [Creality Support page](https://www.creality.com/pages/download-creality-falcon). Extract the `.zip`/`.rar` to find the bare `.bin` file.

---

## Step-by-Step Fix

### Step 1: Put the Board into Boot Mode
1. Ensure the laser's primary power switch is turned **OFF**.
2. Locate the two small DIP switches on the side of the motherboard (usually near the ports or underneath a small access panel).
3. Flip the switch (or switches) into **BOOT / High** mode.
4. Plug the USB cable connecting the laser to your computer. The ESP32 will now show up as a serial device (e.g., `/dev/ttyACM0` on Linux/Mac, or `COM3` on Windows), but importantly, it will be in its raw bootloader mode rather than running GRBL. 

> [!WARNING]
> Ensure any programs like LightBurn, LaserGRBL, or CNCjs are completely **disconnected** and closed so they don't block the serial port.

### Step 2: Flash the Clean Firmware to the Factory Slot
We will write the newly downloaded `.bin` firmware straight into the 0x10000 offset (the standard ESP32 factory app slot).

Open your terminal or command prompt and run:
*(Replace `PORT` with your serial port, e.g., `/dev/ttyACM0` or `COM3`, and update the path to your firmware `.bin` file)*

```bash
esptool --port PORT write_flash 0x10000 /path/to/your/firmware.bin
```
Wait for the progress bar to reach 100% and successfully verify the hash.

### Step 3: Erase the OTA Data Partition
This is the magic step. The laser is still programmed to boot from the corrupted OTA slot. By wiping out the `otadata` memory block (which lives at standard ESP location `0xd000` with length `0x2000`), we wipe the bootloader's memory of the corrupted update, forcing it to load the clean factory firmware we just flashed.

In your terminal, run:
```bash
esptool --port PORT erase_region 0xd000 0x2000
```
This should complete almost instantly ("Flash memory region erased successfully in 0.1 seconds").

### Step 4: Reboot and Restore
1. Unplug the USB cable.
2. Flip the DIP switches **back to their original** normal operating positions.
3. Plug in the laser's main heavy-duty power supply brick and flip the main power switch **ON**.
4. Allow the machine to boot up (it will beep normally).

Connect back to LightBurn or CNCjs and type `$$` or `?`. You should no longer see `nan` under `MPos`, and the machine will now successfully home when commanded!

---
*If this guide fixed your dead Creality Falcon laser, feel free to share it with others in the community who find themselves trapped in the dreaded ALARM:6 Homing Fail loop.*
