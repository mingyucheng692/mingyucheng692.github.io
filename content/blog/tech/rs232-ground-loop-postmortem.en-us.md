---
title: "Industrial PC RS-232 Garbled Data: Software Correct, Hardware Fine — Ground Loop Postmortem"
date: 2026-09-02T12:00:00+08:00
draft: false
tags: ["RS232", "Serial", "Linux", "Hardware", "EMC", "Industrial", "Postmortem"]
categories: ["embedded-linux"]
summary: "An industrial PC (TI AM64x heterogeneous SoC, A53 Linux + R5F RTOS, 24V supply) garbles continuously on a 3-wire RS-232 port at correct 115200 8N1 settings. Via SSH, the tx/rx/fe counters in /proc/tty/driver/serial verified both directions of the channel, ruling out software and unit defects; garbling persisted with only GND connected and vanished the moment the laptop's power adapter was unplugged — a ground loop between the 24V industrial supply and mains. Dual-isolation converter verified in practice: no interference with the laptop plugged in."
url: "/en-us/blog/tech/rs232-ground-loop-postmortem/"
---

> Environment and outputs are desensitized: device models and hostnames are abstracted to placeholders; the SoC model and MMIO addresses are public datasheet data, kept as locating references.

Env: industrial PC (IPC; TI AM64x heterogeneous SoC: Cortex-A53 running Linux, Cortex-R5F running an RTOS), powered by a 24V industrial supply, exposing a 3-wire RS-232 port (TXD/RXD/GND). Debug side: laptop + USB-to-RS-232 adapter.

## Level Identification (Step One)

Multimeter on DC, probes on signal vs GND:

```text
IPC TXD — GND: -5.7V      ← negative idle level, definitely RS-232
IPC RXD — GND: 0V         ← floating input, normal
Adapter TXD — GND: -5.6V  ← adapter is RS-232 level too
```

RS-232 idle level is -3V ~ -15V; TTL idles at +3.3V and never goes negative. **Never connect a 3.3V TTL module directly to this port** — the -5.7V negative level will punch through an ordinary TTL chip's GPIO.

Wiring follows the crossover rule: adapter TXD → IPC RXD, adapter RXD → IPC TXD, GND common.

## Symptom

Serial terminal at 115200 8N1, all three wires connected: a continuous garbled stream, no interaction possible. The garbage is patterned (lots of `F7 F7 ... FF 00` repeats — not random characters).

Anomalies surfaced one by one during troubleshooting:

1. Two units of the same IPC model behave identically
2. The IPC side receives plaintext sent from the PC (visible via `cat /dev/ttyS2` over SSH)
3. **Later wire-by-wire teardown: two-wire combos (RXD+GND / TXD+GND) still garble; down to a single wire, any one of them keeps garbling, and disconnecting it stops the noise**. GND-only garbling was the turning point — the ground wire carries no signal and should never "receive" anything

## Key Evidence: Counter Observation on the Linux Side

The IPC is reachable over SSH — no blind guessing needed. The kernel serial driver ships free observation points.

**1. Console binding:**

```bash
cat /proc/cmdline | grep -o 'console=[^ ]*'
# console=ttyS2,115200n8
```

**2. UART enumeration and power-domain analysis (dmesg):**

```text
4a00000.serial: ttyS0  ← MCU domain, owned by the R5F/M4 real-time cores
4a10000.serial: ttyS1  ← MCU domain
2800000.serial: ttyS2  ← MAIN domain, A53 Linux primary console
2810000~2860000: ttyS3~ttyS8  ← MAIN domain extensions, tx:0 rx:0 unused
```

On a heterogeneous SoC, UARTs belong to different power domains: MCU-domain UARTs are owned by the real-time cores and inaccessible from Linux. Always map the power domain before attributing a physical port.

**3. Driver-level TX/RX counters:**

```bash
cat /proc/tty/driver/serial | grep '2:'
# 2: uart:8250 mmio:0x02800000 irq:314 tx:2132 rx:296 fe:55 RTS|DTR
```

- `tx:2132` — the kernel has output 2132 bytes of boot logs

- `rx:296` — 296 bytes received (including PC keystrokes and floating-input noise)

- `fe:55` — **55 framing errors**, the physical-layer signature of garbling (no valid stop bit at sample time)

**4. Bidirectional channel verification:**

```bash
# Device → PC: send one line of plaintext from SSH
stty -F /dev/ttyS2 115200 raw -echo
echo "=== HELLO FROM DEVICE ===" > /dev/ttyS2
cat /proc/tty/driver/serial | grep '2:'
# tx: 2132 → 2158 → 2184, exactly +26 bytes each time
```

tx grows exactly with each echo → the device TX path is hardware-healthy (oscilloscope cross-check: TXD shows evenly spaced level transitions while sending, gone once idle).

```text
# PC → device: serial terminal sends HELLO_PC\n, SSH runs cat /dev/ttyS2
ELLO_PC
HELLO_PC
ELLO_PC
```

rx 296 → 348 (+52), fe stops growing, the device receives plaintext (occasional first-byte loss, see takeaways) → **115200 baud is definitely correct** (any clock offset would make clean plaintext impossible).

**Contradiction locked**: correct software config + healthy bidirectional hardware + adapter loopback OK + two units reproducing — yet the PC side stays garbled. The problem converges on the electrical layer between the two ground systems.

## Hypotheses & Verification

| # | Hypothesis | Verification | Result | Conclusion |
|---|---|---|---|---|
| A | Baud rate / frame format mismatch | Exhausted 9600~230400 × 8N1/8N2/8E1/7E1 | All garbled | Ruled out |
| B | Adapter defect | Adapter TXD-RXD loopback short | Echoes perfectly | Ruled out |
| C | Defective IPC unit | Retest on a second unit + oscilloscope | Identical symptoms, clean waveform | Ruled out |
| D | Wiring / pinout error | Multimeter in-circuit on both TXD-vs-GND | Both ≈ -5.7V negative | Ruled out |
| E | Common-ground interference / ground loop | **Unplug the laptop's power adapter, run on battery** | Garbling vanishes instantly, bidirectional plaintext | **Root cause confirmed** |

E costs the least to verify (10 seconds) yet came last — the "garbled = baud rate" instinct is strong, so a full parameter sweep came first. Single-wire garbling had shown up early on and was waved away as "floating-input noise": a floating signal line picks up noise, nothing to see. Only after the software layer was fully ruled out did the wire-by-wire teardown happen — and garbling with only GND left broke that explanation. The ground wire carries no signal, so "connect the ground and data appears" cannot be floating noise; the interference must be conducted along that ground wire. Anomaly data must either be falsified or promoted to prime suspect — never buried under a plausible-sounding explanation.

## Root Cause

**A ground loop (common-mode potential difference) between the 24V industrial switching supply and the laptop's mains adapter.** With the two ground potentials unequal, mains-frequency leakage current continuously equalizes through the serial cable; the RX sampling reference (GND) on the PC adapter side drifts along with it, and the perfectly normal -5.7V level gets misread as garbage.

Two corroborating phenomena:

- **Asymmetry**: the IPC-side RS-232 chip (industrial grade) has stable receive thresholds and the 24V supply has low internal impedance — it rides out common-mode wobble; the PC's non-isolated adapter has a fragile sampling circuit and breaks first — hence "the device receives, the PC doesn't"

- **Human antenna effect**: after switching to battery, touching the adapter's metal shell or the USB cable brings the garbling back; release, and it's gone. A battery-powered laptop is a floating ground (high impedance); mains-frequency interference carried by the human body couples capacitively into the floating ground — GND twitches, the potential difference corrupts the levels. Touching the IPC shell has no effect — the 24V industrial supply forms a "strong ground", and microamp-level body charge is a drop in the ocean

## Remediation

Option 1 is the permanent fix (recommended); 2 and 3 are stopgaps for when you cannot wait for procurement.

### Option 1: Dual-Isolation USB-to-RS-232 Converter (Permanent · Recommended)

Three non-negotiable selection criteria:

1. **Speed**: must support 115200 bps and above. Low-speed optocouplers (PC817-class) top out around 38400 — pick a high-speed optocoupler, magnetic-coupler, or digital isolator
2. **True dual isolation**: signal isolation (isolator chip) + power isolation (dedicated DC-DC module, rated 2500Vrms). Signal isolation without power isolation is fake isolation — the ground stays connected
3. **Bridge chip**: prefer FT232RL / CP2102 — accurate clocks at high baud, good Linux support; avoid PL2303/CH340 (counterfeit-prone, large clock deviation at high baud)

The isolation barrier physically severs the two ground systems — in principle immune to every common-ground scenario: laptop on mains, desktop, human-body coupling.

**Verified in practice**: per the three criteria above, I bought a dual-isolation converter (opto signal isolation + DC-DC power isolation, 2500Vrms isolation rating, 300–460800bps, 600W surge + ±15kV ESD protection on the RS-232 side, USB bus-powered). Laptop **plugged into mains**, direct to the IPC: clean bidirectional plaintext, zero interference — the old "plug in and garble" scenario is gone for good.

### Option 2: Run the Laptop on Battery (Emergency Verification)

Unplug the adapter and communication recovers. Fine for short debugging sessions, but the floating ground is vulnerable to the human-antenna effect (touch metal → garble), and battery life is finite.

### Option 3: Equipotential Bonding (Emergency · High Risk, Measure First, Strict Order)

**Step 0 (mandatory): measure the potential difference between the two grounds with a multimeter on AC** — probes on the IPC's metal shell (or the 24V supply's V-) and the laptop/desktop's metal chassis (or the USB shell):

```text
Potential diff < a few volts   → low leakage, proceed with bonding in the order below
Potential diff of tens of volts → abandon common ground, use Option 2/Option 1
                                (shorting = massive instantaneous current, board-frying risk)
```

The potential difference determines the instantaneous short-circuit current — measuring is the only go/no-go safety valve; skipping it is bonding blind.

Once confirmed safe, use a **thick wire** (never DuPont jumpers) to bond the two chassis/grounds first, then attach the signal lines:

```text
1. Thick wire: IPC metal shell ←→ laptop/desktop metal chassis (screw hole)
2. Confirm the bond is solid, then plug in TXD/RXD/GND last
3. Teardown in reverse: signal lines first, thick ground wire last
```

Risk: with heavy field leakage (a big motor starting, supply leakage), all current flows through this wire; the moment it loosens or its resistance rises, the current re-routes through the signal lines and can punch through the USB bridge chip or even the motherboard. **Emergency only, never permanent**.

## Verification

```text
# Adapter unplugged (battery), loop-send from SSH
while true; do echo "1234567890" > /dev/ttyS2; sleep 1; done

# PC serial terminal HEX mode, one line per second
31 32 33 34 35 36 37 38 39 30 0A

# PC sends HELLO_PC\n → cat /dev/ttyS2 over SSH
HELLO_PC
```

Bidirectional plaintext restored, fe counter flat. With the dual-isolation converter, the plugged-in scenario is equally stable — closed loop.

## Key Takeaways

- Multimeter voltage is the fastest way to identify a serial level: steady negative TXD-vs-GND (-3 ~ -15V) = RS-232, +3.3V = TTL; never connect a TTL module to an RS-232 negative-level port

- Serial garbling ≠ wrong baud rate. Before sweeping parameters, do three things: **wire teardown** (garbling with only GND left = interference conducted along the ground, points straight at a common-ground problem), check the garbage pattern (`F7 FF`-style regular repeats = noise pickup, not baud mismatch), check the kernel counters (rx growth = physically delivered)

- Don't let plausible explanations bury anomalies: single-signal-wire garbling was dismissed as "floating noise"; only garbling with GND alone — a wire that carries no signal — broke the misread

- The tx/rx/fe counters in `/proc/tty/driver/serial` are a free kernel-level observation point: rx growth proves physical delivery, flat fe proves frame integrity — more reliable than any serial terminal

- On heterogeneous SoCs (A53 + R5F), UARTs sit in different power domains; matching `dmesg` MMIO addresses against the datasheet locates ownership — MCU-domain UARTs are off-limits from Linux

- When industrial equipment (24V switching supply) meets consumer electronics (mains adapter), **the ground loop should be the default suspect**; "unplug and run on battery" is the 10-second test

- Floating-ground systems are extremely sensitive to human capacitive coupling: touch metal → garble is the hallmark of a high-impedance floating ground

- First-byte loss (`HELLO_PC` → `ELLO_PC`) is UART start-bit sync jitter; it recovers with continuous traffic and is not a fault

- Measure the ground potential difference (AC) before equipotential bonding; tens of volts means abandon common ground — emergency-only, board-frying risk

- The only long-term fix is a dual-isolation converter (signal + power isolation, 115200+ speed) — verified in practice

