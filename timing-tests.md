# USB2RIG vs Digirig: audio timing and data-mode tests

October 2026, VE3CA

## Summary

I tested three USB2RIG units against two Digirig Mobile units while developing a high-speed OFDM data mode for FM. Over a direct audio cable, the Digirigs were clean. The USB2RIGs had faults in their audio streams that don't show up as errors and aren't audible in voice, but they break fast data modes.

| | USB2RIG (units A, B, C) | Digirig Mobile (units D, E) |
|---|---|---|
| Clock difference between two units | about 44 ppm | within 1 ppm |
| Playback (to radio) | Drops or repeats a whole 1 ms block of audio 4–10 times a minute, with no error reported | No slips |
| Capture (from radio) | USB packet errors (`-71`) every 3.5–5 s; some shift the audio by about 0.5 ms | No errors |
| Timing wander | Steps of 0.5–1 ms, as above | About 20 µs rms, slow (under 0.25 Hz) |
| Resets | Unit C reset every 30–60 minutes, on any port or PC | None seen |
| modem73 OFDM, BPSK to QAM64, 48 frames per profile | 13–20 of 48 frames, unit pair | 48 of 48, every profile |
| VARA FM narrow, unregistered (capped at level 2) | 0.38–0.49 kbps net | 1.05 kbps net |
| VARA FM, registered | not tested | 9.5 kbps narrow, 16 kbps wide |
| 1200 baud APRS frames (Direwolf), 300 each way | **17% lost** with the USB2RIG sending, **6% lost** with it receiving | 0% lost both ways |
| Slow, robust modes (MFSK, modem73 ROBUST) | Clean | Clean |

**In short:** the USB2RIG is fine for voice and slow digital modes. For 1200 baud packet it works, but loses a noticeable share of frames. It is not suitable for fast data modes like VARA FM at its higher levels or OFDM. For those, use an interface with a clean audio clock, such as the Digirig.

## Hardware

| | USB2RIG | Digirig Mobile |
|---|---|---|
| Audio codec | C-Media CM108B, no crystal on the board (`0d8c:0012`) | C-Media CM108-family (`0d8c:0012`) |
| Serial (PTT/CAT) | CH340 (`1a86:7523`) | CP2102N (`10c4:ea60`) |
| Internal USB hub | Terminus FE1.1 (`1a40:0101`) | Microchip/SMSC USB2412 (`0424:2412`) |
| PTT | Serial RTS | Serial RTS |

The units are labelled A to E in the results. A, B and C are USB2RIGs, all unmodified. D and E are Digirig Mobiles.

## How the tests were done

- **Test PC:** Debian 12, kernel 6.1, x86-64. Early tests under WSL2 on Windows gave the same faults, so they aren't a WSL problem. Unit C was also tried on an Ubuntu 24.04 PC.
- **Cable:** each unit's audio output went to the other unit's mic input through a resistor pad. There was no radio. Levels were speaker 27, mic capture 16, AGC off. PipeWire/PulseAudio was stopped, so the cards were opened directly through ALSA.
- **Timing:** a steady 300 Hz tone was played through one unit and recorded on the other. The tone's phase over time shows the timing of the audio stream. A 1 kHz tone hides 1 ms slips, because a 1 ms slip is exactly one cycle at 1 kHz.
- **Capture errors:** the Linux USB audio driver reports dropped capture packets when its debug output is on:
  ```
  echo "func retire_capture_urb +p" | sudo tee /sys/kernel/debug/dynamic_debug/control
  dmesg -w | grep "active:"     # USB2RIG shows "frame 0 active: -71" every few seconds
  echo "func retire_capture_urb -p" | sudo tee /sys/kernel/debug/dynamic_debug/control
  ```
  Each timing jump in the 300 Hz recording was matched against these messages.
- **Data modes:**
  - modem73: an open-source OFDM modem. I used my fork, which adds test tools (`tests/phy_jitter`, `tools/phy_cable.sh`, `tools/slip_correlate.py`) at <https://github.com/cgrant417/modem73/tree/modem73fm>. Frames were sent through the cable and the intact ones counted.
  - VARA FM 4.4.0 under Wine 8.0: a scripted connect and transfer between two copies.
  - 1200 baud AFSK: Direwolf 1.6. `gen_packets` made 300 APRS position frames of about 90 bytes, numbered so each one could be identified, about 1 s apart. The recording was decoded with `atest`, and the distinct frames counted. Levels were set so Direwolf read an audio level of about 58 on every pairing (speaker 27, mic capture 24, AGC off; peaks around −10 dBFS).

## Results

### USB2RIG playback slips

With USB2RIG A playing into Digirig D, the recorded tone jumped by **about 1 ms**, in either direction. D logged no errors at all. So the USB2RIG is dropping or repeating whole 1 ms USB audio packets when it sends.

The rate varied: **4 per minute** in a 1-minute run, and **21 in 2 minutes** (10.5 per minute) in a run a day later.

This fits a codec with no crystal: it runs on its own clock, and when its buffer runs over or under, it skips or repeats a packet. Nothing reports an error.

### USB2RIG capture errors

When recording, the USB2RIGs logged `-71` (protocol error) on capture packets every **3.5–5 seconds**. Both units showed it. With Digirig D playing into USB2RIG A, A logged 18 errors in one run. All 4 timing jumps in that recording, of about −0.5 ms each, fell on those errors.

### Unit C

Unit C reset itself every **30–60 minutes**, and the resets followed it to a different USB port. Each reset put its mixer back to its defaults: mic gain +23 dB and **AGC on**, which overdrives data modes. On an Ubuntu 24.04 PC with older USB controllers, unit C gave `-75` (overflow) errors, and its capture stopped within 15–60 seconds.

### Digirig D and E

No slips and no capture errors in either direction. The two units' clocks were within 1 ppm of each other. The only timing movement was a slow wander of about 20 µs rms.

### modem73 OFDM over the cable

6 frames per mode and tone layout, BPSK to QAM64:

| Pair | 7.5 Hz tones | 30 Hz tones | 60 Hz tones |
|---|---|---|---|
| USB2RIG A ↔ USB2RIG B | 13/48 | 20/48 | 19/48 |
| Digirig D → USB2RIG A | 20/24 | 21/24 | 18/24 |
| USB2RIG A → Digirig D | 11/24 | 13/24 | 16/24 |
| Digirig D ↔ Digirig E | **48/48** | **48/48** | **48/48** |

- Frame losses didn't depend on how dense the modulation was: BPSK lost about as many frames as QAM16. Noise wasn't the problem; the timing jumps were.
- The USB2RIG did worse as the sender, because of its playback slips.
- On the Digirigs, every mode worked up to QAM1024 5/6 (13.3 kbps per frame), 24 of 24 frames.

The slow, robust modes (MFSK-8, MFSK-32R, RDM-300, RDMN-150) were clean on the USB2RIGs.

### VARA FM over the cable

| Interfaces | VARA | Net throughput | Highest level reached | VARA's S/N reading |
|---|---|---|---|---|
| USB2RIG A ↔ B | Narrow, unregistered | 0.38–0.49 kbps | 1–2 | −2 to 33 dB, jumping frame to frame |
| Digirig D ↔ E | Narrow, unregistered | 1.05 kbps | 2 (the unregistered limit) | 29–40 dB |
| Digirig D ↔ E | Narrow, registered | 9.5 kbps | 11 of 11 | 34–38 dB |
| Digirig D ↔ E | Wide, registered | 16 kbps | 13 of 13 | 30–40 dB |

The two unregistered rows compare like with like: both are capped at level 2. VARA got less than half the Digirig throughput over the USB2RIGs, and its S/N reading jumped around from frame to frame.

### 1200 baud packet (Direwolf) over the cable

300 frames each way, the same frames and audio level in every run:

| Sender → receiver | Frames decoded | Lost |
|---|---|---|
| Digirig E → Digirig D | 300/300 | 0% |
| Digirig D → Digirig E | 300/300 | 0% |
| USB2RIG A → Digirig D | 249/300 | **17%** |
| Digirig D → USB2RIG A | 282/300 | **6%** |

- **USB2RIG sending:** D's capture logged no errors, so the losses come from A's playback slips. A 2-minute 300 Hz tone run right afterwards showed about 10 slips per minute, which is about 50 in a 5-minute run, against 51 frames lost. At 1200 baud a 1 ms slip is more than one bit, so a slip during a frame almost always breaks its CRC.
- **USB2RIG receiving:** A logged 81 capture errors (`-71`) in the 5-minute run, one every 3.7 s, and 18 frames were lost. About 1 error in 4 cost a frame; the rest fell between frames or didn't disturb the audio enough.
- Lost frames were spread through the whole run, with a few runs of 2–4 lost in a row.
- With USB2RIGs at both ends, expect roughly the two loss rates combined: about 1 frame in 5.

## What this means for other uses

The 1200 baud results are measured above. The rest are my expectations from the measurements; I haven't tested them directly.

- **1200 baud packet and APRS:** expect about 1 frame in 6 lost on transmit, and 1 in 17 on receive. Connected-mode AX.25 retries hide this, at some cost in speed. APRS has no retries, so lost beacons are just gone. A USB2RIG is a poor choice for a digipeater or iGate, which needs to hear and send every frame.
- **Voice, AllStarLink, EchoLink:** a 1 ms slip or a lost packet is at most a faint tick, so units A and B should be fine. For a node that runs 24/7, soak-test it for a few days first. Don't use a unit that resets, like C: every reset drops the audio and turns AGC back on.
- **FT8 and other slow modes:** likely fine, given that MFSK and ROBUST were clean. FT8's 15-second slots and strong error correction should ride over the occasional 1 ms slip.
- **VARA FM and other fast OFDM modes:** expect much lower throughput, as shown above.

## Practical tips for USB2RIG users on Linux

- **Turn AGC off, and set the mic gain again after any reset or replug.** The unit comes back with AGC on and +23 dB mic gain. For example, for card 2:
  ```
  amixer -c 2 sset "Auto Gain Control" off
  amixer -c 2 sset Mic capture 16
  ```
- **Ubuntu: remove brltty.** brltty (the braille display service) claims CH340 serial devices, and the USB2RIG's serial chip is a CH340. When I plugged one in, brltty also took over another interface's serial port and left a radio keyed for about 9 minutes. Remove it with `sudo apt purge brltty`, unless you use a braille display.
- **Watch for ModemManager too.** It can probe new serial ports and briefly toggle RTS/DTR, which keys the radio.
- **Use udev rules to give each interface a fixed name,** so `/dev/ttyUSB0` and card numbers don't swap between boots.
