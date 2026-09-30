---
title: "Modbus RTT Jitter at 30–80 ms: Gateway Measured 1–4 ms — the Root Cause Was the Test Bench"
date: 2026-09-21T12:00:00+08:00
draft: false
tags: ["Modbus", "RS-485", "Linux", "Performance", "Windows"]
categories: ["embedded-linux"]
summary: "An industrial gateway (TI AM64x, Linux PREEMPT_RT) measured 30–60 ms Modbus-TCP RTT and 50–80 ms RTU jitter against a Windows simulator. After correcting the measurement semantics and running a three-step orthogonal isolation loopback (localhost → dual-NIC netns direct link → dual RS-485 crossover), the jitter is attributed to the test peer under these test conditions: board-side pure software stack 0.69 ms, physical NIC 1.02 ms, serial 4.15 ms (wire time about 70%); peer-side interventions such as the FTDI Latency Timer pinned the Windows bench's measured lower bound at 45–50 ms, from which a reference timeout baseline is derived; a single gateway transaction consumes roughly 1%–15% of the end-to-end link budget — not a blocker."
url: "/en-us/blog/tech/modbus-rtt-loopback-benchmark/"
---

> Environment and outputs are sanitized: peer device types are abstracted as numbered "bus devices" with no industry vertical disclosed; the scenario and scheduling parameters are representative of this class of gateways and do not refer to any specific project; device models and hostnames are placeholders; the SoC model is public datasheet information, retained for orientation.

> **TL;DR**: The gateway measured Modbus against a Windows simulator: TCP RTT 30–60 ms, RTU 50–80 ms, jittering. A three-step orthogonal loopback with the peer replaced confirmed the jitter came from the test peer, not the board (under these test conditions) — board-side pure software stack 0.69 ms, physical NIC 1.02 ms, serial 4.15 ms (wire time about 70%) — and a single transaction consumes roughly 1%–15% of the end-to-end link budget (upper bound under the large-frame scope), not a blocker. On the peer side, controlled interventions pinned the bench's measured lower bound at 45–50 ms: dropping the FTDI Latency Timer from 16 to 1 shaved only 6.4 ms (USB scheduling cadence has a floor), and the simulator's roughly 15 ms response cadence remains a black box because Windows timer semantics are per-process. **When performance numbers look wrong, audit the measurement semantics and the peer first; suspect the device under test last.**

---

## An RTT That Blew the Scheduling Budget

The gateway under test is based on TI AM64x (2× Cortex-A53 at the 1.0 GHz class, Linux 6.12 `PREEMPT_RT`). The scenario is a common industrial protocol-conversion deployment: the end-to-end chain is "upstream scheduling system → (northbound protocol) → gateway → (southbound Modbus) → multiple bus devices". The upstream layer polls on a sub-second cycle, and a command must take effect at a bus device within the low hundreds of milliseconds — about half a polling cycle. The gateway is the middle node of that chain; both of its protocol segments must stay a small enough share of the latency budget that the gateway alone does not consume most of it. The goal of this investigation was therefore to confirm whether gateway-side latency is low enough not to block that budget. This post uses a 200 ms polling cycle and a 1-master/4-slave topology (two RS-485 buses with 2 devices each on the RTU side) as representative parameters — chosen as this post's test baseline, not referring to any specific project. A first-round latency test, run by a probe tool against the ModbusTools Modbus Slave simulator on Windows 11 Pro (default parameters, no response delay configured), collided head-on with the budget:

- Modbus-TCP: RTT jittering between 30–60 ms
- Modbus-RTU: RTT jittering between 50–80 ms

At 50 ms per device, polling two devices on one bus consumes half the cycle, and four devices on a single bus fills it entirely; the chain view is even more direct — a single command at 50–80 ms eats most of the low-hundred-ms budget. If those numbers reflected the gateway's real performance, the link latency requirement could not be met and the communication architecture would need re-evaluation. Before drawing that conclusion, one question first: what exactly does that 30 ms measure?

## First, Dissect the Measurement Semantics: What Is Inside the RTT?

The probe source wraps `Connect()` entirely inside the timing window:

```cpp
const auto t_start = std::chrono::steady_clock::now();
const auto conn_res = transport.Connect();  // socket creation + TCP three-way handshake
// ... Send / Receive ...
const auto t_end = std::chrono::steady_clock::now();
```

In other words, every "RTT" includes socket creation and a full TCP three-way handshake; the RTU side likewise does `open()` on the serial node, `tcsetattr` to reset termios, and `close()` after each round. Production communication engines, in contrast, establish the connection once at startup and reuse it for the whole run, with the serial handle held open — each cycle is a pure `Send → Receive`.

Fix the semantics first, then re-measure: move the timing start to the line immediately before `Send()` and the end to the line after receive completes; make the TCP connection persistent and open the RTU serial port once; emit min/max/avg statistics in loop mode. The rework also produced measured values for connection setup: `[CONNECT]` 1.29 ms, `[OPEN]` 0.88 ms — earlier estimates of handshake and serial-open costs (on the order of 20–30 ms and 5–10 ms respectively) were falsified by measurement, ruling out connection setup as a suspect. Re-test results:

- TCP: mean 17.89 ms (14.76–37.36)
- RTU: mean 52.18 ms (34.40–67.27)

With connection setup removed, the numbers were still high, meaning the bulk of the latency sat on the response path. Next, two moves: replace the peer first to prove the board's innocence, then come back to explain why the peer is slow.

## A Three-Step Orthogonal Isolation Loopback

Build a peer of our own: a minimal 0-sleep Modbus slave (replies immediately to FC03/FC06, single in-memory business operation in the microsecond range, `TCP_NODELAY` on), cross-compiled onto the board. Then run three loopback steps ordered "inside out, one more medium per step", each introducing exactly one new variable:

| Step | Topology | New variable | Expected |
| --- | --- | --- | --- |
| 1 | `127.0.0.1` localhost loopback | none | < 0.3 ms |
| 2 | `net5` ↔ `net6` cable cross-plug | NIC controller + bus | 1–2.5 ms |
| 3 | RS485_0 ↔ RS485_1 jumper crossover | UART + transceiver turnaround | 8–15 ms |

Only Step 2 hit its expectation; the two misses are equally informative: Step 1's `<0.3 ms` was estimated from a desktop-class CPU — on a 1 GHz low-power core, a few context switches plus syscalls land in this range; Step 3's 8–15 ms assumed a conventional slave that inserts a t3.5 wait, which the 0-sleep mock eliminates — whichever layer the expectation was wrong about is where the increment gets attributed.

### Step 1: Localhost Loopback — 0.69 ms

Mean 0.69 ms (0.47–1.28). The peer responds with zero sleep; its processing path is a pure in-memory PDU parse plus one `send` syscall, microseconds in cost. The RTT body is the two kernels' TCP/IP stack paths. Software stack: cleared.

### Step 2: Dual-NIC Cable Cross-Plug — 1.02 ms

Mean 1.02 ms (0.78–1.64). Subtracting Step 1's software base, the AX88796B NIC over the GPMC bus contributes only 0.33 ms of hardware-path increment. This step hit four pitfalls, all worth recording:

1. **Weak-host model**: two NICs of the same Linux host sending to each other — the kernel notices the destination is a local IP and shortcuts through `lo`, so the electrical signal never crosses the cable. The standard fix is to lock the peer NIC into its own network namespace, forcing the physical port:
   ```bash
   ip netns add ns_eth6
   ip link set net6 netns ns_eth6
   ip netns exec ns_eth6 ip addr add 192.168.66.2/24 dev net6
   ip netns exec ns_eth6 ip link set net6 up
   ```
2. **Initiator not up + mismatched subnets**: `net5` defaults to DOWN, and once the two sides were configured in different /24 subnets, the SYN found no direct route and leaked out the debug NIC to a switch via the default route — `Connection timed out` (the probe's 1 s budget exhausted). A single cable provides no L3 routing: both sides must share a subnet, and both NICs must be up.
3. **PHY autonegotiation transient**: after link-up, the first two pings were lost and the third read 2049 ms — consistent with PHY autonegotiation taking 1.5–2 s plus ping counting ARP queuing time into the RTT (inference, not separately verified); steady-state retest fell back to 0.4 ms. First-packet anomalies do not imply link failure.
4. **The NIC that "vanished"**: after `ip netns del`, `net6` was nowhere in the main namespace — a process still residing in the deleted namespace keeps it alive as an anonymous namespace holding the NIC. Locate the holder by comparing namespace inodes under `/proc`:
   ```bash
   for pid in $(ls /proc | grep '^[0-9]'); do
     if [ -e "/proc/$pid/ns/net" ] && \
        [ "$(readlink /proc/1/ns/net)" != "$(readlink /proc/$pid/ns/net)" ]; then
       echo "holder: PID $pid ($(cat /proc/$pid/comm))"; kill -9 $pid
     fi
   done
   ```

### Step 3: Dual RS-485 Jumper Crossover — 4.15 ms

Both RS-485 ports are native on-board UARTs (handled by the kernel 8250 driver family), contrasting with the Windows bench's USB adapter path. Mean 4.15 ms (3.97–4.44), jitter 0.47 ms. Physical arithmetic (115200 8N1, 10 bits per byte on the wire):

- Master request frame, 8 bytes: 80 bits ÷ 115200 ≈ 0.69 ms
- Slave response frame, 25 bytes: 250 bits ÷ 115200 ≈ 2.17 ms
- Wire time subtotal: 2.86 ms

Measured 4.15 ms minus wire time leaves 1.29 ms of overhead. This accounting contains no inter-frame gap: the peer's source is 0-sleep immediate — it inserts no t3.5 wait after collecting a full request frame and replies at once (for baud rates above 19200 the spec recommends a conservative fixed value of 1.75 ms — were it present, "wire + t3.5" alone would reach 4.61 ms, already above the measured mean, which independently corroborates its absence). The 1.29 ms is the combined send/receive path cost of both ends' UART FIFO interrupts, drivers, transceiver turnaround and process scheduling (obtained by subtraction, not itemized). Wire time (2.86 ms) is about 70% of the measured mean; this post made no attempt to compress the 1.29 ms (FIFO watermarks, threaded IRQs, etc.) and draws no conclusion about its compressibility.

## Data Summary Table

| Scenario | Medium | Samples | Mean RTT | Range | Jitter (peak-to-peak) |
| --- | --- | --- | --- | --- | --- |
| Control: Windows simulator | TCP | 95 | 17.89 ms | 14.76–37.36 | 22.60 ms |
| Control: Windows simulator | USB-485 | 62 | 52.18 ms | 34.40–67.27 | 32.87 ms |
| Step 1 | localhost | 78 | 0.69 ms | 0.47–1.28 | 0.81 ms |
| Step 2 | net5 ↔ net6 | 28 | 1.02 ms | 0.78–1.64 | 0.86 ms |
| Step 3 | RS485_0 ↔ RS485_1 | 70 | 4.15 ms | 3.97–4.44 | 0.47 ms |

Sample sizes 28–95, statistics as min/max/avg, no percentiles collected; conclusions in this post are therefore limited to means and lower bounds, with tail latency left for on-site integration to re-verify with larger samples.

## 15.625 and 16: From Mechanistic Hypothesis to Controlled Intervention

The board is cleared; now explain the first two rows of the control group: why the peer is slow, and why the slowness takes exactly that shape. Matching two minimum values against known system constants does line up dimensionally. But between "it fits" and "it is proven" stands the intervention — this section records how the hypotheses were constructed, what was intervened, and how the results fed back to revise them.

Constructing the hypotheses:

- **TCP side**: the default timer resolution on desktop Windows is about 15.625 ms (1000 ms / 64 Hz); without raising it, waits such as `Sleep` and `WM_TIMER` quantize to that granularity. If the GUI simulator's response path contains such a wait (e.g. a timer-driven receive poll), the RTT floor gets lifted to around one tick. Measured minimum 14.76 ms, mean 17.89 ms, maximum 37.36 ms (about two ticks plus processing) is compatible with a "response stage with a roughly 15.6 ms period" model. Conversely, blocking sockets, IOCP and event objects are all woken immediately by network completion; had the simulator taken such a path, the floor would sit in the milliseconds (link ping `<1 ms`) — the measured floor of 14.76 ms already excludes the "immediate wake" class of paths; there is indeed a stage with a roughly 15 ms period on the response path.
- **RTU side**: the adapter chip is confirmed as an FTDI FT231X (VID `0x0403` / PID `0x6015`, driver `ftser2k.sys` v2.12.36.20), whose driver defaults the Latency Timer to 16 ms — short frames wait out the timeout in the chip, counted from the first byte, before being reported to the host (a timeout flush, not "filling a packet"; a buffer that fills early is reported at once, and this test's frames are far below that threshold). Stacked with the peer's roughly 15.6 ms scheduling and this test's pure wire time (FC03 read of 10 registers: request 8 B + response 25 B ≈ 2.86 ms), the total is ≈ 34.5 ms — 0.2 ms away from the measured minimum of 34.40 ms.

The interventions that followed fall into three groups:

1. **FTDI Latency Timer from 16 to 1** (Device Manager → port Properties → Port Settings → Advanced): RTU mean dropped from 51.89 ms to 45.51 ms — only 6.38 ms, far below the roughly 15 ms the single-factor model predicted. The mechanism is confirmed present but with a smaller effective coefficient: setting the Latency Timer to 1 ms does not make the chip actually report at 1 ms — USB bulk-transfer scheduling cadence draws a floor under the effective flush interval. The single-factor model is thereby falsified, and the bench's measured lower bound is pinned at 45–50 ms — under this bench configuration (FT231X adapter, ftser2k v2.12.36.20 driver, default-parameter simulator), further Windows-side tuning showed no marginal gain.
2. **Disabling USB LPM and selective suspend**: re-test 51.42 ms, within run-to-run variance of the 51.89 ms pre-intervention baseline — no material effect; RTT still fluctuates in the 45–50 ms bench band.
3. **Topology ruled out**: the adapter sits behind a Genesys GL850G Multi-TT hub with an independent Transaction Translator per downstream port, ruling out same-port full-/low-speed contention.

The system clock granularity cannot be settled this cleanly. The measured system timer period is already 1.0 ms (programmable floor 0.5 ms), which looks like it would falsify the 15.625 ms quantization hypothesis; but since Windows 10 2004, `timeBeginPeriod` applies per process, so a system-wide reading does not guarantee that the simulator process's wait primitives schedule at 1 ms too. The discriminating experiment would raise the frequency inside the simulator's process context and see whether the floor drops to milliseconds — not implementable from outside; the specific origin of the roughly 15 ms cadence (the simulator's own timer choice, or a built-in response delay) remains an application-layer black box.

The reach of the ping corroboration also needs bounding: Windows pinging the gateway at `<1 ms` proves the physical link healthy and the latency generated inside the peer host; the reverse direction's 100% loss is just the Windows firewall disabling ICMP echo by default, unrelated to port 502. It supports "the bottleneck is on the peer" — it supports no specific clock mechanism.

One more independent sample cross-checks the bench model: an FC06 control write against the same peer measured RTT 63 ms with 0 retries. Breakdown:

- Wire time: request and echo frames of 8 bytes each, ≈ 1.38 ms
- t3.5 inter-frame gap and transceiver turnaround: estimated 1.5–2.0 ms (that peer's implementation is unpublished; the spec's conservative value)
- Board-side cost: half of Step 3's measured 1.29 ms two-end overhead, ≈ 0.5–0.7 ms
- Remaining ≈ 59 ms: on the Windows + USB adapter side

The latter three items are estimates or subtraction leftovers, not independent measurements; the t3.5 and board items are order-of-magnitude closure, not precise attribution — fitting one observation with several adjustable parameters is not causation, exactly the scenario item 2 of "Reusable Methods" below warns about. The Windows/USB side accounts for over 90%, in the same order as the 45–50 ms read-transaction band.

The section's conclusion is therefore two-layered: the bulk of the probe readings comes from the test peer — established by the substitution experiments above; within the peer, the USB adapter buffering is intervention-confirmed present but its benefit has a floor, with the measured bench bound at 45–50 ms; the simulator's application-layer roughly 15 ms cadence remains a black box.

Two side notes: real bus devices are mostly MCU bare-metal firmware answering on hard interrupts, with protocol handling typically 1–3 ms (industry experience, not measured here); a simulator-class 15 ms floor normally does not bind real devices, though slow real slaves exist (SCADA-configured PLCs, VFDs with protocol gateways) — unverified here. Moving the simulator into WSL2 or a VM usually brings no fundamental improvement either — Hyper-V virtual switching, VM-Exits and host clock scheduling add new latency layers.

## Full-System Re-Verification: From One Transaction to One Scheduling Cycle

Steps 1–3 answer "what does one transaction cost"; scheduling cares about "does one cycle suffice". A bounded 10-second full-system run of the in-house scheduling engine under representative parameters: 4 logical slaves, 200 ms polling cycle, 100 ms timeout, a typical telemetry frame profile (137-byte single-transaction response), atomic commit once all 4 channels are present. The peer is the on-board mock slave. RTU runs 4 Slave IDs time-division on a single bus — the representative deployment is two RS-485 buses with 2 devices each, so a single-bus 4-slave topology is twice as harsh, and margin conclusions only relax for the deployment; TCP, with only two on-board test NICs, carries 4 Unit IDs on 2 physical endpoints (still one master, logical topology unchanged).

Results: TCP 50/50 rounds fully committed, RTU 47–48/50 rounds; `ALL_READY` (the engine's slave-online status flag — a different scope from per-round full commit) held throughout. Both numbers are mechanism-consistent: TCP's four concurrent channels do not interfere; on RTU's single time-division bus, one timeout-retry (a 100 ms timeout wait plus one ≈ 14 ms retry transaction — see the next section for the per-transaction accounting) already consumes more than half of a 200 ms cycle, and two consecutive timeouts exceed the entire cycle — occasional dropped rounds match the signature of one-off events such as startup alignment, write injection, or a sporadic retry; the exact trigger points were not annotated per-round. Judging by the result shape, steady-state throughput shortfall is excluded: were every round overrunning, the score would sit stably low and never reach 50, rather than mostly full with the occasional 47–48.

The physical floor of one RTU round under this frame profile: single-transaction wire time 0.69 + 11.9 ≈ 12.6 ms (request 8 B + response 137 B), so 4 slaves ≈ 50.3 ms — 25% of the 200 ms cycle. That is the pure wire floor; transposing Step 3's measured 1.29 ms per-transaction overhead gives ≈ 14 ms per transaction and ≈ 56 ms per round — that overhead includes per-transaction costs such as FIFO interrupts and turnaround, possibly slightly higher for large frames; the exact value awaits single-packet measurement against the real point table. Even at the estimated value, cycle margin stays above 70%. And this is the harsh-topology accounting: the representative deployment (two RS-485 buses, 2 devices each) runs one bus at ≈ 25.2 ms wire time per round (≈ 28 ms transposed) — only 13–14% of the 200 ms cycle — Step 3's "wire-time-dominated" conclusion holds at the system level, with even wider margin for that form.

Back to the original question — is the 30–80 ms high latency an ARM RT-Linux gateway problem — under these test conditions, the evidence says no: component level (three-step loopback: dual-NIC TCP at ≈ 1 ms, dual RS-485 at ≈ 4 ms) and system level (full rounds in the 200 ms-cycle run) both show no gateway-side bottleneck; the high latency comes from the bench peer (Windows host, USB adapter, simulator software). Against the investigation goal — does gateway-side latency block the end-to-end link budget — closure comes in two scopes: chain latency per transaction, measured 1.02 ms (TCP) / 4.15 ms (RTU small frame), ≈ 14 ms estimated for large frames — about 1%–15% of the low-hundred-ms link budget; scheduling throughput per round, ≈ 56 ms for 4 slaves with over 70% cycle margin. Under neither scope does the gateway constitute a blocker. Boundary conditions to note: the peers in both isolation and full-system tests were mocks, the tests were idle single-transaction scope over a bounded 10-second window, and northbound-concurrent, long-duration, more field-like load profiles were not covered; the northbound protocol segment (upstream system to gateway) is outside this post's measurement scope. The (southbound-segment) communication infrastructure is no longer a blocker under this baseline.

(A counting-semantics lesson: an early version of these full-system statistics mistook 1 ms scheduling ticks for transaction rounds and produced throughput numbers inflated by about two orders of magnitude; the corrected scope is "atomic commit after all 4 channels present". Define what "one" is before talking about numbers — isomorphic to the measurement-semantics problem.)

## Conclusions and Engineering Outputs

- Attribution: the first-round 30–60 ms (TCP) / 50–80 ms (RTU) jitter and the post-semantics-fix re-test are both attributable to the test peer — with the peer replaced, jitter at every layer collapses to under 1 ms (peak-to-peak); the board showed no bottleneck in any layer's experiments, and the serial link is about 70% wire time. Within the peer, interventions confirmed: the USB adapter Latency Timer (16→1, mean −6.4 ms) has a benefit with a measured floor; the bench's measured lower bound is 45–50 ms; the simulator's application-layer roughly 15 ms response cadence remains a black box.
- Reference timeout baseline: Modbus-TCP 15–25 ms, Modbus-RTU 30–50 ms. The margin semantics must be stated — against the gateway's own cost (TCP 1.02 ms / RTU 4.15 ms) those are 15–25× / 7–12×; but a timeout must cover the peer's response, and after including real-slave processing of 1–3 ms and the RTU t3.5 (≈ 1.75 ms), the effective end-to-end margin roughly halves (TCP ≈ 4–12×, RTU ≈ 3–7×). Timeouts should not be stretched to hundreds of milliseconds — a single device dropping off must be recognized and cut out by the polling loop within tens of milliseconds.
- Baseline usage constraints: first, it presumes real slaves respond markedly faster than the Windows simulator; second, RTU's 30–50 ms sits below the Windows bench's measured RTT (45–80 ms) — during integration against a Windows simulator, keep the rig's 100 ms configuration. The baseline tightens the 100 ms conservative configuration and awaits on-site integration validation before deployment.
- Scheduling budget: reckoned against a 100 ms scope twice as harsh as the representative 200 ms cycle (the full-system run actually used 200 ms), a single transaction costs TCP 1.02% / RTU 4.15%; for multiple slaves, see the per-round physical floor above — the harsh stress topology (single bus, 4 slaves) runs ≈ 50.3 ms per round (over 70% margin on a 200 ms cycle); the representative deployment (two buses, 2 devices each) runs ≈ 25.2 ms per bus.
- If you must use a Windows simulator for functional integration (this bench's adapter is a confirmed FTDI FT231X; CH340/CP2102 have no such setting): Device Manager → port Properties → Port Settings → Advanced → Latency Timer from 16 to 1. On this bench the mean dropped ≈ 6.4 ms (51.89→45.51); the benefit has a USB-scheduling floor, so do not expect the theoretical 15 ms. Fine for functional integration, not a peer for performance evaluation — at least under this bench's measured conditions (default parameters, no response delay).

## Reusable Methods

1. **Measurement semantics before measurement data.** Given a performance number, first ask what the timing window contains: connection handshake? device open/close? the peer's scheduling period? Undefined semantics make numbers incomparable.
2. **Dimensional matches produce candidate explanations; only interventions produce verdicts.** System constants like 15.625 ms and 16 ms upgrade a hypothesis from guess to candidate — this post's minimum-value comparison excluded the "immediate wake" class of paths; but fitting one observation with several adjustable parameters is not causation. The discriminating method is to change one variable (raise timer frequency, change the Latency Timer, swap the adapter chip) and watch whether the minimum moves with it. Intervention results can be non-ideal — this post's 16→1 Latency Timer change bought only 6.4 ms, falsifying the single-factor model; that too is information: it points the search at the next physical floor instead of licensing more parameters on the same hypothesis.
3. **Orthogonal isolation of variables.** When an entire chain is suspect, add media layer by layer from pure software to physical: localhost sets the software base, a physical link measures the hardware increment, a bus crossover separates wire time from overhead. Each layer's increment is separately attributable.
4. **A simulator is not the device under test.** GUI simulators carry message pumps and system-clock jitter: good for functional checks, bad for performance. Use a minimal 0-sleep peer for performance work, pressing peer cost to negligibility.
5. **For a true physical link between two NICs of one host, netns is the standard tool.** Otherwise the weak-host model shortcuts packets through the kernel and the physical link is never exercised.

When doing the same class of bus-latency verification on embedded Linux, this "semantics fix → dimensional match and intervention → three-step loopback" procedure and the commands in this post can be applied directly.
