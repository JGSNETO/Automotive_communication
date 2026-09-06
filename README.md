# Vehicle Network Lab

An interactive, in-browser simulator that demonstrates how the four dominant in-vehicle
network protocols — **CAN**, **CAN FD**, **LIN**, and **Ethernet** — actually move bits
across a wire. It's built for engineering students who know the theory (bus topologies,
frame formats, arbitration) but want to *see* it happen: bit-by-bit arbitration, master
polling, frame construction, real CRC computation, and switched vs. broadcast delivery.

No build step, no dependencies, no backend — it's a single self-contained HTML file.

---

## Running it locally

Because everything (HTML, CSS, JavaScript) lives in one file, there are two ways to run it:

### Option 1 — just open the file
Double-click `vehicle-network-lab.html`, or open it from your browser with
`File → Open`. Everything runs client-side; the only network request the page makes
is loading two Google Fonts (IBM Plex Mono / IBM Plex Sans). If you're offline, the
page still works — it falls back to your system's monospace/sans-serif fonts.

### Option 2 — serve it over a local web server (recommended)
Some browsers apply stricter security rules to pages opened via `file://`. If you see
any font-loading or console warnings, serve the folder instead:

```bash
# Python 3 (already installed on most systems)
python3 -m http.server 8000

# or, with Node.js installed
npx serve .

# or, with PHP installed
php -S localhost:8000
```

Then visit **http://localhost:8000/vehicle-network-lab.html** in any modern browser
(Chrome, Firefox, Safari, Edge — all current versions work; no IE support).

### Requirements
- Any modern browser with JavaScript and SVG support enabled.
- No npm install, no compiler, no server-side runtime. This is intentional — it should
  run the same way on a classroom projector laptop as it does in CI.

---

## What this application is

Vehicle Network Lab is a teaching tool with three tabs — **CAN**, **LIN**, and
**Ethernet** — each modeling a distinct automotive network architecture with its own
topology, its own way of deciding who gets to talk, and its own idea of what "sending a
message" means electrically. CAN also includes a **CAN FD** mode toggle, since FD is a
variant of the same bus rather than a separate network.

Every number the app shows you — CRC values, checksums, parity bits, arbitration
outcomes — is computed for real from the data you enter, not hard-coded or faked for
effect. The goal is that what you see on screen is what a logic analyzer would actually
show on a real bus.

| Tab | ECUs modeled | What it teaches |
|---|---|---|
| **CAN** | Engine ECU, ABS/Brakes, Body Control, Instrument Cluster, Infotainment | Multi-master bus, bit-wise arbitration, broadcast + receiver-side filtering |
| **LIN** | Cabin Master + Door/Seat/Mirror/Climate slaves | Master/slave polling, fixed schedule tables, UART byte framing |
| **Ethernet** | Front Camera, Surround-View Camera, Radar/LiDAR Fusion, ADAS Domain Controller, Diagnostic Gateway | Switched point-to-point links, MAC addressing, no arbitration needed |

---

## The protocols, explained

### CAN (Controller Area Network)

**Topology:** a single twisted pair (CAN_H / CAN_L) shared by every node — a true bus,
not a star. All nodes see every signal.

**Who talks, and when — multi-master, not master/slave.** Any node can start
transmitting the moment it detects the bus is idle. There is no central controller
handing out turns. This is the most common misconception to correct: CAN has no master.

**Broadcast, not addressed.** A CAN frame doesn't carry a destination. Every node
electrically receives every frame; each node's *acceptance filter* decides afterward
whether to keep it or discard it. The identifier field is really a *message* ID
("engine RPM", "door status"), not a node address — multiple nodes can legitimately
listen to the same ID.

**Priority and arbitration.** If two or more nodes start transmitting at the same
instant, they resolve the conflict without any outside referee, using the identifier
itself:
- CAN uses **dominant** (logical 0, actively driven) and **recessive** (logical 1, idle)
  bus states. Dominant always wins if both are driven at once.
- While transmitting, every node also *reads back* the bus. If a node sends a recessive
  bit but reads back a dominant bit, it knows another node with higher priority is also
  transmitting — it immediately stops and retries later.
- Because the comparison runs most-significant-bit first, **the numerically lowest
  identifier always wins** — a lower CAN ID means higher priority. This is why safety-
  and real-time-critical messages (engine, braking) are assigned low IDs.
- Critically, arbitration is **lossless**: the winning frame is transmitted exactly as
  if there had been no contention at all. No bandwidth is wasted on the collision itself
  (contrast this with Ethernet's original CSMA/CD, which detects collisions only *after*
  they happen and must retransmit).

**Frame anatomy (Classical CAN, 11-bit identifier):**

| Field | Size | Purpose |
|---|---|---|
| SOF | 1 bit | Start of Frame — always dominant, signals bus is no longer idle |
| Identifier | 11 bits | Message ID; also the arbitration priority (lower = higher priority) |
| RTR | 1 bit | Remote Transmission Request — dominant for a normal data frame |
| IDE | 1 bit | Identifier Extension — dominant = standard (11-bit) frame |
| r0 | 1 bit | Reserved |
| DLC | 4 bits | Data Length Code — how many data bytes follow (0–8) |
| Data | 0–8 bytes | The payload |
| CRC | 15 bits | Checksum (CRC-15, polynomial 0x4599) over SOF through end of data |
| CRC delimiter | 1 bit | Fixed recessive |
| ACK slot | 1 bit | Driven dominant by **any** healthy node that received the frame without error — this happens regardless of whether that node's filter will actually accept the message; ID-based filtering happens afterward, at the application layer |
| ACK delimiter | 1 bit | Fixed recessive |
| EOF | 7 bits | End of Frame — fixed recessive |

**Speed:** up to 1 Mbit/s (typical automotive buses run 125 kbit/s–500 kbit/s
depending on domain).

### CAN FD (Flexible Data-Rate)

CAN FD is the same bus, the same electrical dominant/recessive scheme, and the *same
arbitration mechanism* as Classical CAN — the identifier-based bit-wise arbitration
described above is untouched. FD only changes what happens **after** arbitration is won:

- **Bigger payloads.** Instead of a linear 0–8 byte DLC, FD uses fixed, nonlinear length
  codes to reach up to 64 bytes per frame: DLC codes 0–8 still mean 0–8 bytes, but codes
  9–15 mean 12, 16, 20, 24, 32, 48, and 64 bytes respectively.
- **Optional bit-rate switch (BRS).** A frame can switch to a *faster* bit clock for the
  data portion of the frame (roughly from the DLC field through the CRC), then switch
  back to the nominal rate before the CRC delimiter. Arbitration itself always happens
  at the slower, nominal rate, because arbitration depends on every node's transceiver
  being able to reliably observe dominant-vs-recessive bit-by-bit — that only works
  reliably at the slower speed.
- **New fields:** `FDF` (marks the frame as FD instead of Classical), `BRS` (requests
  the speed switch), `ESI` (Error State Indicator — reports transmitter health).
- **Bigger CRC.** Because the payload can be much larger, FD uses CRC-17 (for payloads
  up to 16 bytes) or CRC-21 (for payloads above 16 bytes) instead of Classical CAN's
  CRC-15.
- **Speed:** nominal (arbitration) phase is the same as Classical CAN; the data phase
  with BRS enabled commonly runs at 2–8 Mbit/s.

> **Simplification note:** real CAN FD also transmits a 3-bit stuff-count field with
> parity ahead of the CRC, which factors into the actual CRC calculation. This app
> computes CRC-17/21 directly over the header+data bits without that stuff-count field,
> to keep the concept visible without requiring a full bit-stuffing simulation. The
> arbitration mechanism, field order, DLC coding, and dual bit-rate behavior are modeled
> faithfully.

### LIN (Local Interconnect Network)

**Topology:** a single wire, one master, multiple slaves — genuinely master/slave,
unlike CAN.

**Who talks, and when.** The master owns a fixed **schedule table** and cycles through
it forever. For each entry, the master transmits a *header* naming a message ID; the one
node assigned to respond to that ID — usually a slave, but sometimes the master itself —
transmits the *response*. Nobody else is allowed to transmit at that moment. Because the
master decides the order in advance, **there is no arbitration and no contention to
resolve** — this is the core contrast with CAN.

**Frame anatomy:**

| Field | Structure | Purpose |
|---|---|---|
| Break | ≥13 dominant bits + 1 recessive delimiter | Signals a new frame is starting; not byte-framed like the rest |
| Sync byte | UART-framed byte, always `0x55` | Lets every slave's clock re-synchronize to the master's bit rate |
| PID | UART-framed byte: 6-bit identifier + 2 parity bits | Names the message; parity bits are computed from specific ID bits per the LIN spec |
| Data | 1–8 UART-framed bytes | The response payload |
| Checksum | 1 UART-framed byte | Enhanced LIN 2.x checksum: 8-bit sum (with end-around carry) of the PID and all data bytes, then inverted |

Every byte on LIN — sync, PID, data, checksum — is wrapped in a standard **UART frame**:
1 start bit (dominant) + 8 data bits (**least-significant bit first**, unlike CAN's
most-significant-bit-first fields) + 1 stop bit (recessive). This is a real point of
contrast with CAN, which has no such per-byte framing overhead.

**Speed:** typically 19.2 kbit/s (up to 20 kbit/s), far slower than CAN — LIN is used
for low-bandwidth, cost-sensitive body/comfort functions (windows, mirrors, seats,
climate controls), not real-time control loops.

### Ethernet (Automotive Ethernet)

**Topology:** switched, point-to-point — the fundamental architectural break from CAN
and LIN. There is no shared bus at all. Every node has its own dedicated, full-duplex
link to a switch. Two nodes can transmit at the exact same instant with zero
conflict, because their signals never share a wire.

**Who talks, and when.** Whenever a node wants to. Since there's no shared medium,
**there is nothing to arbitrate** — the switch simply receives on one port and forwards
independently on another.

**Addressed, not (usually) broadcast.** Every frame carries a source and destination
**MAC address**. The switch inspects the destination address and forwards the frame
down exactly one port — other nodes never see it electrically, unlike CAN's
receive-everything-then-filter model. Broadcast is still possible (destination
`FF:FF:FF:FF:FF:FF`, used for things like DoIP vehicle-discovery messages), but it's the
exception, not the default delivery mode.

**Frame anatomy (Ethernet II):**

| Field | Size | Purpose |
|---|---|---|
| Preamble + SFD | 8 bytes | Alternating bit pattern (`0x55` × 7) + start-frame delimiter (`0xD5`) for clock sync |
| Destination MAC | 6 bytes | Which node (or broadcast) should receive this frame |
| Source MAC | 6 bytes | Which node sent it |
| EtherType | 2 bytes | Identifies the payload protocol (e.g. `0x0800` for IPv4) |
| Payload | 46–1500 bytes | The actual data; short payloads are zero-padded up to the 46-byte minimum |
| FCS (CRC-32) | 4 bytes | Frame Check Sequence — standard IEEE 802.3 CRC-32 over destination through payload |

Ethernet doesn't have a dominant/recessive voltage model at all — it's a clocked,
line-coded serial link (automotive variants typically use 100BASE-T1 or 1000BASE-T1,
which use differential PAM-based encoding, not a simple two-state voltage). That's why
this app shows Ethernet frames as a byte-by-byte packet layout instead of a voltage
trace: drawing a fake dominant/recessive waveform for Ethernet would teach the wrong
mental model.

**Speed:** 100 Mbit/s (100BASE-T1) to 1 Gbit/s+ (1000BASE-T1) and climbing — orders of
magnitude faster than CAN or LIN, which is why it's used for camera, radar/LiDAR, and
other high-bandwidth ADAS data that CAN/CAN FD can't carry.

---

## Quick comparison

| | CAN | CAN FD | LIN | Ethernet |
|---|---|---|---|---|
| Topology | Shared bus | Shared bus | Shared single wire | Switched, point-to-point |
| Control model | Multi-master | Multi-master | Single master, polled slaves | No arbitration needed |
| Addressing | Broadcast + receiver-side filter | Broadcast + receiver-side filter | Role-assigned by schedule | Switched, addressed by MAC |
| Priority mechanism | Bit-wise arbitration (lowest ID wins) | Same as CAN | Master's schedule order | None (or QoS/priority tagging in TSN variants) |
| Typical speed | Up to 1 Mbit/s | Up to ~8 Mbit/s (data phase) | ~19.2 kbit/s | 100 Mbit/s – 1 Gbit/s+ |
| Max payload | 8 bytes | 64 bytes | 8 bytes | 1500 bytes |
| Typical use | Powertrain, chassis, safety | Same domains, more data (e.g. OTA-capable modules) | Body/comfort (seats, mirrors, HVAC) | ADAS sensors, diagnostics, infotainment backbones |

---

## Using the simulator

- **Bit / byte time** (top right) controls animation speed for every tab — slow it down
  for a lecture, speed it up for a demo.
- **CAN tab:** check 2+ ECUs and hit **Transmit** to watch bit-by-bit arbitration play
  out before the winning frame is broadcast. Toggle **CAN FD frame** / **Bit-rate
  switch** to see the extended payload and the faster data-phase segment (shaded, and
  narrower on the trace).
- **LIN tab:** hit **Step** to advance through the master's schedule one frame at a
  time, or **Auto cycle** to let it run continuously.
- **Ethernet tab:** two independent send slots let you fire frames from two different
  sources at once — try sending both simultaneously to see that neither has to wait for
  the other.

---

## Notes on accuracy vs. simplification

This app prioritizes getting the *concepts* right — arbitration, framing, addressing,
priority — over full protocol-conformance. Specific simplifications:

- CAN/CAN FD: bit-stuffing (inserting a complementary bit after 5 consecutive identical
  bits) is not simulated, so on-wire bit counts shown are pre-stuffing.
- CAN FD CRC omits the stuff-count-and-parity field that real controllers fold in.
- LIN, CAN, and CAN FD CRCs/checksums are otherwise computed with the real, standard
  algorithms (CRC-15 poly `0x4599`, CRC-17 poly `0x3685B`, CRC-21 poly `0x302899`, LIN
  enhanced checksum, LIN PID parity), not placeholders.
- Ethernet FCS is a genuine IEEE 802.3 CRC-32 over the real assembled frame bytes.
- Physical-layer signal encoding (e.g. NRZ vs. Manchester vs. PAM) is not modeled in
  detail — the CAN/LIN traces show the logical dominant/recessive bit sequence, and the
  Ethernet view shows the byte-level frame layout instead of a voltage waveform, since
  Ethernet's actual line coding isn't a two-level signal at all.

---

## License

Add your project's license here.
