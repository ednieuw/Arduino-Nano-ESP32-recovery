# Arduino Nano ESP32 – Recovering a Stalled Board

This guide describes how a stalled or non-booting Arduino Nano ESP32
(ESP32-S3) is restored to a working board that can be used via the Arduino
IDE. It also describes how loose `.bin` files can be flashed using the
**Flash Download Tool**.

> **Worth knowing:** the actual ROM bootloader of the ESP32-S3 chip is
> embedded in the chip itself and cannot be overwritten by any firmware.
> Recovery is therefore always possible.

## 0. Quick overview

| 1. Open the tool | 2. Download mode + load files |
|---|---|
| ![Flash Download Tool - first screen with ChipType, WorkMode and LoadMode](FDTsrc/chiptype.jpg) | ![Flash Download Tool - four bin files loaded at the correct addresses](FDTsrc/UploadBinBoot.jpg) |
| Open the Flash Download Tool. On the first screen, select **ChipType: ESP32-S3**, **WorkMode: Develop**, **LoadMode: UART**, then click OK. | Put the board into download mode by shorting **B1** to **GND** and pressing the **RST** button. Then load `bootloader.bin`, `partitions.bin`, `boot_app0.bin` and the sketch `.bin` at their respective addresses and click **START**. |

The full explanation of each step is provided in the chapters below.

## 1. Putting the board into download mode

On a stalled board, or when no valid firmware is present, the automatic
Arduino reset (toggling of the DTR/RTS lines) does not function. According
to the official [Arduino Nano ESP32 cheat sheet](https://docs.arduino.cc/tutorials/nano-esp32/cheat-sheet/),
there are two methods for putting the board into a recovery mode manually:

### Option A – Arduino Bootloader mode (software-based, no jumper required)

1. Press **RESET**, then press it again as soon as the RGB LED starts flashing (double reset).
2. The board is in bootloader mode once the green LED pulses slowly.

This method depends on the Arduino bootloader partition, which performs the
double-reset detection. If this partition has been overwritten by
third-party firmware, that detection logic is absent and the board is not
enumerated as a serial COM port. In that case, Option B is required.

### Option B – ROM Boot mode via the B1 pin (functions independently of the firmware present)

> Note: according to Arduino, the correct pin is named **B1**, not B0. On
> the Nano ESP32, B1 = GPIO0 (the standard ESP32 download pin); B0 = GPIO46
> serves a different function.

1. Short the **GND** pin to the **B1** pin using a jumper wire. The RGB LED turns green.
2. Keep the GND–B1 connection in place and briefly press the white **RST** button on top of the board.
3. Remove the connection between GND and B1. The RGB LED remains lit, now in purple: the board is in firmware download mode.

Source: [Arduino Help Center – Reset the Arduino bootloader on the Nano ESP32](https://support.arduino.cc/hc/en-us/articles/9810414060188-Reset-the-Arduino-bootloader-on-the-Nano-ESP32).

Check Windows Device Manager to see which COM port appears. On the Nano
ESP32, the port number may change when the board switches from normal mode
to download mode (for example COM6 → COM5), because the USB descriptor of
the ROM bootloader differs from that of the application firmware.

> **Shortest official route (alternative to the Flash Download Tool):**
> put the board into download mode using Option B, in the Arduino IDE
> select **Tools → Programmer → Esptool**, run **Tools → Burn Bootloader**
> (erases the flash), then **Sketch → Upload Using Programmer**. See the
> same [Help Center article](https://support.arduino.cc/hc/en-us/articles/9810414060188-Reset-the-Arduino-bootloader-on-the-Nano-ESP32)
> for the full procedure.

> **Note:** this only works if the board has not stalled completely. If a
> different or foreign ESP32 bootloader has ended up on the board (for
> example because the factory/bootloader partition was overwritten using
> esptool, esp-idf or PlatformIO), the Arduino IDE's Upload button no
> longer responds, and **only the Flash Download Tool** (with the 4
> separate files at the correct addresses) still works. See the following
> real-world examples:
>
> - [Arduino Forum – BIN flashing Nano ESP32 problem](https://forum.arduino.cc/t/bin-flashing-nano-esp32-problem/1320594): confirms that if the factory partition has been overwritten, only the esptool/jumper method still works.
> - [odelayIO/Recovering-Bricked-Arduino-Nano-ESP32 (GitHub)](https://github.com/odelayIO/Recovering-Bricked-Arduino-Nano-ESP32): recovery of a Nano ESP32 that could no longer be programmed via the IDE, using esptool outside the IDE.

> As long as *no* valid application firmware is present in flash, the boot
> sequence fails and the RTC watchdog triggers a continuous reset cycle.
> As a result, the COM port keeps appearing and disappearing. A COM port
> should only be selected *after* the procedure of Option A or B above has
> been carried out.

## 2. Installing the Flash Download Tool

Espressif's official "Flash Download Tool" is a Windows GUI application
that writes `.bin` files directly to specific flash addresses, without
requiring Python or esptool.

- Official download page: [espressif.com – Support / Download / Tools](https://www.espressif.com/en/support/download/other-tools)
- Official User Guide (with screenshots): [docs.espressif.com – Flash Download Tool User Guide](https://docs.espressif.com/projects/esp-test-tools/en/latest/esp32/production_stage/tools/flash_download_tool.html)
- Direct download of the zip file: [dl.espressif.com/public/flash_download_tool.zip](https://dl.espressif.com/public/flash_download_tool.zip)
- Local copy (already downloaded for this project): [FDTsrc/flash_download_tool.zip](FDTsrc/flash_download_tool.zip)

Extract the zip archive to a folder of choice and start `flash_download_tool.exe`.

### Selecting the chip and mode

![Selecting chip type and work mode in the Flash Download Tool](FDTsrc/chiptype.jpg)

Select **ChipType: ESP32-S3**, **WorkMode: Develop**, **LoadMode: UART**, then click OK.

### Main screen of the tool

![Main interface of the Flash Download Tool](FDTsrc/main_interface.jpg)

Once the chip and mode have been selected, the main screen opens on the SPIDownload tab.

## 3. Flashing a single .bin file

To write a single `.bin` file only (for example to test whether the
connection works, or to update just the sketch while the bootloader and
partitions are already correct):

![SPIDownload tab with one bin file at address 0x10000](FDTsrc/UploadBin.jpg)

The SPIDownload tab with one file entered at address `0x10000`.

1. Click the **...** button next to the first row and select the required `.bin` file.
2. Enter the address in the field alongside it (for example `0x10000` for a sketch — see the table below).
3. Tick the checkbox in front of that row.
4. Select the correct **COM** port at the bottom and a **BAUD** rate of, for example, 921600.
5. Put the board into download mode (see step 1) and click **START** immediately afterwards.

## 4. Full recovery: bootloader + partitions + application

If the flash is completely empty (for example after an `erase_flash`),
more than just the sketch is required. In the Arduino IDE, first use
**Sketch → Export Compiled Binary** on a simple sketch (e.g. Blink) for the
"Arduino Nano ESP32" board. This produces the following files in the
sketch folder:

![Sketch menu in Arduino IDE showing Export Compiled Binary](FDTsrc/ExportBin.jpg)

Arduino IDE: **Sketch → Export Compiled Binary** (Alt+Ctrl+S).

| File | Address | Description |
|---|---|---|
| `*.bootloader.bin` | `0x0` | Second-stage bootloader |
| `*.partitions.bin` | `0x8000` | Partition table |
| `boot_app0.bin` | `0xe000` | OTA selection data |
| `*.ino.bin` | `0x10000` | The sketch itself (application) |

Add all 4 rows to the Flash Download Tool with their respective file and
address, tick all 4 checkboxes, put the board into download mode and click
**START**.

![SPIDownload tab with all 4 files entered and ticked](FDTsrc/UploadBinBoot.jpg)

All 4 files added and ticked (application, bootloader, partitions, boot_app0) — ready to click START.

> **Shortest route:** instead of flashing these 4 files manually, the
> **Upload** button in the Arduino IDE can simply be used once the board is
> in download mode and the COM port is visible — the IDE then
> automatically writes all 4 files to the correct addresses itself.

## 5. Testing the connection without flashing (optional, using esptool)

To check beforehand whether the board responds, without using the GUI tool:

```bash
pip install esptool
python -m esptool --chip esp32s3 --port COM5 --connect-attempts 10 chip_id
```

If this responds with chip information, the connection is functioning and
flashing can proceed (step 3 or 4).

---

Sources: [Arduino – Nano ESP32 Cheat Sheet / User Manual](https://docs.arduino.cc/tutorials/nano-esp32/cheat-sheet/) ·
[Arduino Help Center – Reset the Arduino bootloader on the Nano ESP32](https://support.arduino.cc/hc/en-us/articles/9810414060188-Reset-the-Arduino-bootloader-on-the-Nano-ESP32) ·
[Espressif – Download Tools](https://www.espressif.com/en/support/download/other-tools) ·
[Espressif – Flash Download Tool User Guide](https://docs.espressif.com/projects/esp-test-tools/en/latest/esp32/production_stage/tools/flash_download_tool.html)

Screenshots in `FDTsrc/` are original screen captures, with the exception of the tool's main screen, which is taken from the official Espressif documentation.
Instruction to revive a Arduino Nano ESP32 with Espressif Flash Download Tool
