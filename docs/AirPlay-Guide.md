# AirPlay: ESP32-WROVER-E AirPlay receiver

An AirPlay 1 (RAOP) audio receiver for an original ESP32-WROVER-E with a PCM5102A DAC.
Built from https://github.com/jptrsn/esp-airsync (v0.2.4 + the changes listed in section 7).

> **AirPlay 1 only.** It appears as a normal AirPlay speaker on iPhone, iPad and Mac.
> It is *not* AirPlay 2.

## Status: what is and is not verified

| Item | Status |
|---|---|
| Compiles with ESP-IDF 5.5.1 for the original ESP32 | Verified (GitHub Actions build) |
| Image headers say ESP32 (not S3), PSRAM on, 4 MB flash | Verified from the build files |
| `AirPlay.bin` fits its partition (about 1.07 MB of 4.13 MB, 26%) | Verified |
| Boots, joins Wi-Fi, appears as `AirPlay` on Apple devices, plays audio | Confirmed by the owner on real hardware |
| Playback stable (a 2+ minute serial log showed 0 new underruns, 0 silent frames) | Confirmed by log, after the router placement fix |
| Stereo left/right correct | **Not yet tested** |
| Stable for an hour or more; recovers after a Wi-Fi outage | **Not yet tested** |

Anything marked "not yet tested" should be checked before you rely on it.

---

## 1. Wiring (PCM5102A to ESP32-WROVER-E)

| PCM5102A pin | ESP32 pin | Notes |
|---|---|---|
| BCK | **GPIO 33** | bit clock |
| LCK (LRCK / WS) | **GPIO 25** | left/right clock |
| DIN | **GPIO 32** | audio data |
| SCK | **GND** | no master clock is used, so SCK must be tied low |
| GND | GND | common ground |
| VIN | 3.3 V or 5 V | check the label on your DAC board |

Do not use GPIO 6 to 11 (flash) or GPIO 16 and 17 (PSRAM) for anything. The pins above are safe.

Many PCM5102A boards have small solder pads on the back. If you get silence, check that
SCK is grounded and that the XSMT pad is high (unmuted); the board's own markings decide how.
The pins can be changed later on the device's web page, without reflashing.

---

## 2. Files you need

From the downloaded build (also in the `firmware/` folder of the project):

- **`AirPlay-full-flash.bin`**: one combined file that contains everything. Use this one.
- `AirPlay.bin`, `bootloader.bin`, `partition-table.bin`: the same
  thing in separate pieces, for tools that want them one by one (addresses are in `flasher_args.json`).

> **Always flash the combined file at `0x0`.** The app now lives at `0x10000` (it used to be at
> `0x20000`), so flashing `AirPlay.bin` on its own onto a board with the older layout will not boot.
> Leave "Erase all" off to keep your saved Wi-Fi settings.

---

## 3. Flashing (no terminal, in your browser)

**Warning: from step 5 on, this erases and replaces the firmware currently on the board
(your old Squeezelite-ESP32).** Do not continue unless you are happy with that
(see "Backup" below).

What you need: a computer with **Chrome or Edge** (Safari and Firefox cannot talk to USB
serial devices), and a USB cable that carries **data**, not just charge. If the board never
shows up, the cable is the first suspect.

1. Plug the ESP32 board into the computer with the USB cable.
2. In Chrome or Edge, open Espressif's browser flasher, the
   [esptool-js demo page](https://espressif.github.io/esptool-js/).
3. Click **Connect** and pick the port that appears (it is often named something like
   "USB Serial", "CP210x" or "CH340"). If several appear, unplug the board, look again, plug it
   back in, and choose the one that just appeared.
4. If it says it cannot connect, put the board in download mode: **hold the BOOT button, tap
   EN (reset), then let go of BOOT**, then click Connect again. (Many boards do this by themselves.)
5. Choose the flash address **`0x0`** and select **`AirPlay-full-flash.bin`**.
6. *Recommended once, because the board previously ran different firmware:* click
   **Erase Flash** first. This removes the old firmware and its saved settings.
7. Click **Program** and wait until it reports 100% and finishes. Do not unplug it.
8. Press the **EN** button (or unplug and replug) so the board restarts.

### Backup (optional)
Your old firmware can be reinstalled from its original installer, so a backup is only
needed if you made special settings you want to keep. A true copy needs Python's `esptool`:
`esptool.py -p <port> read_flash 0 0x400000 old-backup.bin`. Skip this if that sounds like
too much; nothing in this project requires it.

### If flashing fails
- Wrong port or no port: try another USB cable and another USB port.
- "Failed to connect" or timeout: use the BOOT and EN button sequence in step 4.
- Fails partway: choose a slower speed (115200 baud) and retry.
- Board does not start afterwards: erase flash, then program again.

---

## 4. First start

1. Within about a minute the board creates a Wi-Fi network named **`AirPlay-Setup-XXXXXX`**.
   **It has no password** (the upstream README mentions one, but the code leaves the setup
   network open). Connect your phone or computer to it.
2. Open **http://192.168.4.1** in a browser.
3. Enter your **2.4 GHz** Wi-Fi name and password (the ESP32 cannot use 5 GHz).
4. Check that the pin settings show BCK 33, WS 25, DOUT 32, and that the device name is `AirPlay`.
5. Save, then restart the board (EN button). It joins your Wi-Fi.
6. After a minute, **AirPlay** should show in the AirPlay menu:
   - iPhone or iPad: Control Center, then the AirPlay icon.
   - Mac: the AirPlay icon in the menu bar or inside Music.
   The board is also reachable as `airplay.local` on your network.

### Test list (tell me what happens)
1. Does `AirPlay-Setup-XXXXXX` appear? (Proves the firmware boots.)
2. Does `AirPlay` appear in the AirPlay list after the Wi-Fi setup?
3. Does sound play? Is left on the left and right on the right? (Use a stereo test track.)
4. Does it stay clean for an hour or more? Does it recover after you briefly turn off the router?

### Serial log (optional)
Any serial monitor at **115200 baud** works. Every 10 seconds it prints a line starting with `diag:`
showing Wi-Fi state and signal, free internal memory, free PSRAM, playback state and sample rate.
Every 5 seconds during playback the audio code also prints a health line that includes
buffer underruns. Warning signs: `UNDERRUN`, free internal memory steadily falling, or the
Wi-Fi state dropping to `DOWN`.

**Privacy note:** the upstream code prints the Wi-Fi password in plain text in the serial log at
the moment you save it. Do not post that part of a log publicly.

---

## 5. Troubleshooting

| Problem | Try |
|---|---|
| No port appears | Different USB cable (data cable) and port; install the CP210x or CH340 driver if your OS needs one |
| Flash fails to connect | BOOT + EN sequence (step 4 of section 3) |
| No `AirPlay-Setup` network | Wait 1 to 2 minutes; erase flash and program again |
| Not shown in the AirPlay list | Phone and board must be on the same network; the router must allow mDNS/Bonjour; check the router's client isolation setting |
| Appears, connects, but silence | Recheck wiring and SCK to GND; check the DAC's XSMT pad; confirm pin values on the web page |
| Left/right swapped or one channel dead | Swap or recheck BCK/LCK wiring |
| Short dropouts or silences | This was a Wi-Fi delivery problem in testing (many resend requests in the `rtp health:` line, few recovered). Move the board and/or router so they have a clear line of sight, set the router's 2.4 GHz band to a 20 MHz width on channel 1, 6 or 11, and check again. Healthy playback shows `underruns` not rising and `silent=0 discarded=0` |
| Crackling | Try a better USB power supply and shorter DAC wires |
| Reboots in a loop | Send me the first lines of the serial log after reset |

---

## 6. Build instructions (to rebuild yourself)

- Upload the project to a GitHub repository. The workflow `.github/workflows/build.yml`
  builds with ESP-IDF v5.5.1 for the `esp32` target and publishes the files above.
- Or locally: install ESP-IDF v5.5.1, then `idf.py set-target esp32` and `idf.py build`.
  The dependencies `mdns` and `esp-dsp` are downloaded automatically from Espressif's component registry.
- Partition layout (`partitions.csv`): nvs 0x9000 (24 KB), phy_init 0xF000, one `factory` app
  partition at 0x10000 (about 3.94 MB, the rest of the 4 MB flash). There is no OTA; firmware is
  always flashed over USB.
- ESP-IDF settings are in `sdkconfig.defaults` and `sdkconfig.defaults.esp32`.

---

## 7. What is in the firmware

**Unchanged from upstream:** the RAOP/AirPlay 1 receiver, ALAC decoding, jitter buffering with
timing correction, I2S output, software volume, web configuration page and (the over-the-air update feature was removed in this build; see section 6).

**Changed for this build:**
- Project and binary named AirPlay; device name `AirPlay`; Wi-Fi hostname `airplay`; mDNS `airplay.local`.
- I2S pins default to BCK 33, WS 25, DOUT 32 (upstream: 26, 25, 22).
- Wi-Fi power-save is turned off, to avoid latency jitter in the stream.
- The 10-second `diag:` health line was added.
- Over-the-air (OTA) update removed: no update page, no update web handler, no rollback code,
  and a single large app partition instead of two 1.75 MB slots.
- Setup network renamed from `ESP-AIRSYNC-XXXXXX` to `AirPlay-Setup-XXXXXX`.
- Web page titles renamed.
- Flash and PSRAM defaults documented for the WROVER-E.

**How audio flows:** AirPlay client, then RTSP and RTP receive, then ALAC decode, then a ring buffer
in PSRAM (timed against the sender's clock with drift correction), then the output task on core 1,
then I2S at 44.1 kHz, 16-bit stereo, then the PCM5102A. Large buffers live in PSRAM; small
allocations stay in internal RAM. Buffer sizes were not changed.

**Memory note:** the ESP32 can directly use only 4 MB of the 8 MB PSRAM. The upstream
project needs about 2.7 MB (its own figure, not measured here), so this is enough.

**Preparing for Spotify later:** audio leaves the AirPlay code through one function
(`audio_output_callback` in `main/main.c`, which calls `i2s_output_write`). A shared audio
manager can later sit at that point so that a Spotify Connect receiver (cspot) feeds the same I2S
output. Nothing of that is implemented in this build.
