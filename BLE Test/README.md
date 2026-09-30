# BLE Penetration Testing — In-Depth Field Manual

**Running example:** `TK Connect` (mobile app, BLE **central**) ↔ `TK V5` (telematics device, BLE **peripheral**).

This manual assumes you already have the conceptual grounding (GAP/GATT, services/characteristics, pairing/bonding). It focuses on the part that's usually left as "we'll do a hands-on lab later" — the actual commands, scripts, captures, and decision-making of a real assessment. Theory is kept as tight reference; the weight is on execution.

---

## 0. Scope, authorization, and safety (read first)

BLE testing touches physical hardware you can brick, and radio you can legally get in trouble for. Before any command:

- **Written authorization** covering the specific device(s), MAC/identifiers, physical location, and time window. GPS/telematics devices are often installed in vehicles you don't own — get explicit permission for the vehicle too.
- **Test on a dedicated unit**, never a customer's live-installed device. Firmware, reset, and fuzzing tests can permanently disable hardware.
- **RF is regulated.** Passive sniffing is generally fine on your own gear; *active* jamming/MITM (btlejack, mirage) can be illegal outside a controlled environment or Faraday setup. Know your jurisdiction.
- **Don't leak location/PII.** Telematics devices emit real GPS. Handle captured data like the sensitive data it is; scrub it from reports.
- **Change one thing at a time** and keep a lab journal (timestamp, action, device state, result). Half of BLE RE is correlation, and you can't correlate what you didn't log.

Destructive tests (firmware downgrade, reset loops, oversized-write fuzzing) go **last**, on an expendable unit, with a recovery plan.

---

## 1. Reference model (condensed)

```
                  BLE
                   │
          ┌────────┴────────┐
       Central           Peripheral
      (TK Connect)         (TK V5)
          │                 │
          └────── link ─────┘
                   │
        GAP  ──────┴──────  GATT / ATT
     (discover/connect)   (data + operations)
                             │
                    Service → Characteristic → Descriptor
                                  │
                     props: READ WRITE WWR NOTIFY INDICATE
```

**The two questions that structure the whole engagement:**

1. **Authentication** — *who are you?* (can you connect / pair at all)
2. **Authorization** — *what are you allowed to do?* (can a merely-connected or merely-paired peer hit privileged characteristics)

Most real BLE IoT findings live in #2: encryption is present, pairing works, and yet a paired-but-unprivileged peer can write the "command" characteristic. **Encryption ≠ authorization.**

### Layer cheat-sheet

| Term | One-liner | Where you see it |
|---|---|---|
| GAP | Discovery, roles, connection setup | Advertising, scan results |
| GATT | How data is *organized* (services/chars) | Enumeration output |
| ATT | How attributes are *accessed* (read/write/notify) | Wireshark packets, handles |
| Handle | Numeric ID of one attribute (`0x0012`) | Packet captures, gatttool |
| UUID | Identity of a service/char (16-bit standard or 128-bit custom) | Enumeration |
| CCCD (`0x2902`) | Descriptor that enables notify/indicate | Subscribing to NOTIFY |

**Custom 128-bit UUIDs are where the interesting stuff lives.** Standard services (`0x180A` Device Info, `0x180F` Battery) are documented; vendor UUIDs like `6E400001-B5A3-F393-E0A9-E50E24DCCA9E` (that's the common Nordic UART Service — worth recognizing) hide proprietary command channels.

---

## 2. Lab setup

### 2.1 Hardware

| Role | Options | Notes |
|---|---|---|
| Host adapter (connect/read/write) | Built-in BT, or a known-good USB dongle (CSR8510, Intel AX200) | For interacting *as a central* |
| Sniffer | Nordic **nRF52840 dongle** (with Sniffle fw), Ubertooth One, TI CC2540 | Passive capture of *others'* traffic |
| Active/MITM | Two nRF adapters, or **btlejack** on 3× micro:bit | Only in controlled RF space |
| Phone | Android with **nRF Connect** app | Fast triage GUI, HCI snoop log |

The nRF52840 + **Sniffle** (by NCC Group) is the current sweet spot for connection-following BLE 5 sniffing and is far more reliable than the legacy Ubertooth for modern devices.

### 2.2 Linux tooling install (Debian/Kali/Ubuntu)

```bash
# Core BlueZ stack + utilities
sudo apt update
sudo apt install -y bluez bluez-tools bluez-hcidump wireshark

# Python interaction library (the workhorse)
python3 -m venv ~/ble && source ~/ble/bin/activate
pip install bleak

# Sniffle (nRF52840) — flash firmware then use the host script
git clone https://github.com/nccgroup/Sniffle
# (flash sniffle_cc1352_1M.hex per repo instructions to the dongle)

# Optional advanced
pip install mirage        # framework: recon/MITM/fuzz (or apt on Kali)
# btlejack: pip install btlejack  (needs micro:bit hardware)
```

### 2.3 Sanity checks

```bash
hciconfig -a                 # is the controller up?
sudo rfkill list             # is BT soft/hard blocked?
sudo rfkill unblock bluetooth
sudo hciconfig hci0 up
bluetoothctl list            # confirm controller MAC
```

If `hci0` is missing after plugging a dongle: `dmesg | tail`, check firmware, `sudo systemctl restart bluetooth`.

---

## 3. Phase 1 — Reconnaissance (change nothing)

Goal: build a complete profile of the device *before* touching it.

### 3.1 Live monitor + scan

Open two terminals. First, watch the raw HCI:

```bash
sudo btmon        # decoded HCI/advertising firehose — leave running, log everything
```

Second, scan:

```bash
bluetoothctl
[bluetooth]# power on
[bluetooth]# menu scan
[bluetooth]# transport le      # LE-only, ignore Classic noise
[bluetooth]# back
[bluetooth]# scan on
```

You'll get e.g. `[NEW] Device AA:BB:CC:DD:EE:FF TK_V5`. Capture from `btmon` (not just bluetoothctl) because btmon shows the *full* advertising payload.

### 3.2 What to record

| Field | Why it matters |
|---|---|
| Address + **address type** | Public vs Random changes tracking & privacy analysis |
| RSSI over distance | Proximity attack feasibility |
| Device name | Fingerprinting, sometimes leaks serial |
| **Manufacturer data** (AD type `0xFF`) | Often encodes serial/state/company ID unencrypted |
| Service UUIDs advertised | Tells you the API surface pre-connection |
| TX power, flags, appearance | Fingerprinting |
| Connectable? Undirected? | Whether anyone can connect |

### 3.3 Address types — the "device disappeared" trap

```
Public            → stable, tied to OUI (vendor identifiable)
Random Static     → stable per boot, changes on reboot
Resolvable (RPA)  → rotates (privacy); resolvable only with the IRK
Non-resolvable    → rotates, not linkable
```

If TK V5 "vanishes" mid-test, suspect RPA rotation, not disconnection. In `btmon`/nRF Connect the address type is labeled explicitly. A telematics tracker that uses a **fixed public address** is itself a privacy finding (passive long-term tracking).

### 3.4 Decode manufacturer data

Manufacturer data starts with a little-endian 16-bit **Company ID** (Bluetooth SIG assigned), then vendor bytes. Example payload `4C00 02 15 ...` → Company `0x004C` = Apple (iBeacon). For an unknown vendor, log the raw bytes across multiple device states (idle, moving, GPS-locked) and diff them — bytes that change with state are decodable telemetry leaking without any connection.

---

## 4. Phase 2 — Connection & GATT enumeration

**Test the unauthenticated state first.** Do *not* pair yet. The most valuable finding is often "no pairing required to do X."

### 4.1 Quick enumeration with bluetoothctl / gatttool

```bash
bluetoothctl
[bluetooth]# connect AA:BB:CC:DD:EE:FF
[bluetooth]# menu gatt
[bluetooth]# list-attributes AA:BB:CC:DD:EE:FF   # services, chars, descriptors + handles
```

Legacy but fast (note: `gatttool` is deprecated in newer BlueZ but still ships on Kali):

```bash
gatttool -b AA:BB:CC:DD:EE:FF --primary          # services
gatttool -b AA:BB:CC:DD:EE:FF --characteristics   # chars + handles + properties
gatttool -b AA:BB:CC:DD:EE:FF --char-read -a 0x0012
```

### 4.2 Full enumeration script (Bleak) — the real workhorse

This is the script you'll actually run. It connects, walks the entire GATT tree, prints properties, and *safely* reads every readable characteristic.

```python
#!/usr/bin/env python3
import asyncio
from bleak import BleakClient, BleakScanner

ADDR = "AA:BB:CC:DD:EE:FF"   # or scan by name below

async def find_by_name(name="TK_V5"):
    dev = await BleakScanner.find_device_by_name(name, timeout=15)
    return dev.address if dev else None

async def main():
    async with BleakClient(ADDR, timeout=20) as c:
        print(f"Connected: {c.is_connected}\n")
        for svc in c.services:
            print(f"[Service] {svc.uuid}  ({svc.description})")
            for ch in svc.characteristics:
                props = ",".join(ch.properties)
                print(f"  [Char] {ch.uuid}  handle=0x{ch.handle:04x}  props={props}")
                # SAFE read only — never auto-write
                if "read" in ch.properties:
                    try:
                        val = await c.read_gatt_char(ch.uuid)
                        print(f"        READ = {val.hex(' ')}  ascii={val!r}")
                    except Exception as e:
                        print(f"        READ failed: {e}")
                for d in ch.descriptors:
                    print(f"    [Desc] {d.uuid}  handle=0x{d.handle:04x}")

asyncio.run(main())
```

### 4.3 Reading the map

Turn the raw enumeration into a hypothesis table immediately:

| UUID (short) | Props | Hypothesis | Priority |
|---|---|---|---|
| `...1235` | READ | Device ID / firmware string | Info-disclosure |
| `...1236` | WRITE, WWR | **Command channel** | ★ highest |
| `...1237` | NOTIFY | Telemetry / command responses | RE target |
| `...1238` | READ, WRITE | Config | Auth check |

**WRITE / WRITE-WITHOUT-RESPONSE characteristics are your primary targets.** They're where commands enter the device.

---

## 5. Phase 3 — Read / Write / Notify testing

### 5.1 Read testing

For every readable characteristic already dumped in §4.2, ask:

- Is the value **sensitive** (serial, IMEI, GPS, keys, config)?
- Was it readable **without pairing**? → information disclosure finding.
- Does it identify/track the device?

Telematics devices frequently expose IMEI or ICCID on a readable characteristic with no auth — a standard, reportable finding.

### 5.2 Write testing — start boring, escalate slowly

Never fire "restart" as your first write. Establish the protocol's shape with inert values and watch the NOTIFY channel for responses.

```python
#!/usr/bin/env python3
import asyncio
from bleak import BleakClient

ADDR   = "AA:BB:CC:DD:EE:FF"
CMD    = "0000xxxx-...."   # WRITE characteristic
RESP   = "0000yyyy-...."   # NOTIFY characteristic (responses land here)

async def main():
    async with BleakClient(ADDR) as c:
        # subscribe first so we see any reply to our write
        async def on_notify(_, data: bytearray):
            print(f"  <- NOTIFY {data.hex(' ')}")
        if any("notify" in ch.properties for s in c.services for ch in s.characteristics):
            await c.start_notify(RESP, on_notify)

        for probe in [b"\x00", b"\x01", b"\xff", b"\x00\x00"]:
            print(f"-> WRITE {probe.hex(' ')}")
            try:
                await c.write_gatt_char(CMD, probe, response=True)
            except Exception as e:
                print(f"   write err: {e}")
            await asyncio.sleep(0.8)     # let the device respond / settle

asyncio.run(main())
```

Watch for: a response byte pattern, a disconnect, an error code, or a state change. Map the opcode space methodically (`00, 01, 02, ...`) but **only execute actions inside your authorized, non-destructive scope.** If `02` looks like "reboot," note it and move on — don't loop it.

### 5.3 Notify/Indicate testing

Subscribing writes `01 00` to the char's CCCD (`0x2902`) automatically via Bleak's `start_notify`. Log every notification with a timestamp and the correlated device state:

```python
import asyncio, time
from bleak import BleakClient
ADDR, NOTIFY = "AA:BB:CC:DD:EE:FF", "0000yyyy-...."

async def main():
    def cb(_, d):
        print(f"{time.time():.3f}  {d.hex(' ')}")
    async with BleakClient(ADDR) as c:
        await c.start_notify(NOTIFY, cb)
        await asyncio.sleep(120)          # move the device, change GPS, etc.
        await c.stop_notify(NOTIFY)

asyncio.run(main())
```

Now change the device's real-world state (move it, cover GPS) and diff the notification stream — this is your entry to protocol RE (§7).

---

## 6. Phase 4 — Pairing & bonding security

Now compare the **unauthenticated** results above against the **authenticated** state, and evaluate the pairing itself.

### 6.1 Identify the pairing method & security level

```bash
sudo btmgmt info                       # controller capabilities
bluetoothctl
[bluetooth]# pair AA:BB:CC:DD:EE:FF    # observe: passkey prompt? numeric compare? nothing?
```

Watch `btmon` during pairing for the **SMP** exchange (Pairing Request/Response). Record:

| Attribute | Where | Finding if... |
|---|---|---|
| Method: Just Works / Passkey / Numeric Comparison | SMP pairing feature exchange | Just Works on a device that issues vehicle commands = weak MITM protection |
| Legacy Pairing vs **LE Secure Connections** | SC flag in pairing req | Legacy = vulnerable key exchange (crackable, e.g. crackle for legacy) |
| IO capabilities | pairing req | `NoInputNoOutput` forces Just Works |
| MITM flag set? | pairing req | Absent = no MITM protection required |

### 6.2 Pairing-method downgrade

Devices that support strong pairing but *accept* weaker are a classic finding. Probe by advertising constrained IO capabilities from your side:

```bash
# Force Just Works by presenting NoInputNoOutput
sudo btmgmt io-cap 3     # 3 = NoInputNoOutput
# then attempt pairing and see whether the device still bonds
```

If the peripheral downgrades to Just Works when the central claims no IO, MITM protection is bypassable.

### 6.3 Legacy pairing key recovery

If (and only if) the device uses **LE Legacy** pairing, its Long-Term Key derivation is weak. Capture the pairing exchange with a sniffer (§8) and run `crackle` against the pcap to recover the TK/STK/LTK, then decrypt the session offline. **LE Secure Connections (ECDH-based) defeats this** — confirming SC is present is the mitigation you'll recommend.

### 6.4 Pairing vs bonding

- **Pairing** = keys generated this session.
- **Bonding** = keys *stored* for reconnection.

Check whether bonding keys are stored securely on the peripheral and whether a bonded-but-unauthorized peer retains privileged access indefinitely.

---

## 7. Phase 5 — Authorization matrix (the core deliverable)

Run every sensitive operation in each auth state and fill this in. This table is what makes the report credible.

| Function | Unpaired | Paired (Just Works) | Paired (authorized user) | Expected? |
|---|---|---|---|---|
| Read device info / IMEI | ✅/❌ | | | |
| Read GPS position | | | | |
| Read config | | | | |
| **Write config** | | | | |
| **Diagnostic / vehicle command** | | | | |
| **Firmware update** | | | | |
| Reset / reboot | | | | |
| Disable device | | | | |

**The headline finding pattern:** any ✅ in the *Unpaired* or *Just-Works* column for a privileged row (command/config/firmware) = broken authorization. That a legitimate app also does it doesn't make it authorized — the *device* must enforce it, not the app.

---

## 8. Phase 6 — Sniffing (observe the legitimate app)

To reverse the real protocol, capture what `TK Connect` actually sends. Two routes:

### 8.1 Android HCI snoop log (easiest, no extra hardware)

1. On the phone: **Developer Options → Enable Bluetooth HCI snoop log**.
2. Reproduce actions in TK Connect (unlock, request GPS, etc.).
3. Pull the log and open in Wireshark:

```bash
adb bugreport bugreport.zip     # snoop log is inside, or:
adb pull /data/misc/bluetooth/logs/btsnoop_hci.log
wireshark btsnoop_hci.log
```

This gives you the plaintext ATT operations the app performs, mapped to UI actions — the fastest way to learn the command protocol.

### 8.2 Over-the-air sniffing with Sniffle (nRF52840)

```bash
# From the Sniffle python_cli directory:
python3 sniff_receiver.py -l -o capture.pcap        # -l follows connections
# then trigger app actions; Sniffle follows the connection and logs to pcap
wireshark capture.pcap
```

In Wireshark, filter and read the ATT layer:

```
btatt                                   # all ATT
btatt.opcode.method == 0x12             # Write Request
btatt.opcode.method == 0x1b             # Handle Value Notification
btatt.handle == 0x0012                  # one attribute
```

Map **handles → characteristics** (from §4) so a packet like `Write Request, Handle 0x0012, Value: aa0105` becomes "wrote `AA 01 05` to the command characteristic."

### 8.3 Protocol reverse engineering

Correlate captured writes/notifications with device state:

```
Notification stream while accelerating:
  01 00 46      → 70
  01 00 47      → 71     ⇒  byte[2] = speed
  01 00 48      → 72

Command captured on "get GPS" tap:
  AA 03 00      ⇒  opcode 0x03 = GPS request; byte[0]=AA framing
```

Build a decode table (framing byte, opcode, length, payload, checksum?). Test your model by *constructing* a valid frame yourself and writing it (§5.2) within scope.

---

## 9. Phase 7 — Replay & manipulation

### 9.1 Replay

Capture a legitimate command (e.g. `AA 01 05`), then re-send the identical bytes with your own script (§5.2). If it re-executes, there's **no anti-replay**. Look for whether the protocol includes any of:

```
sequence counter · nonce · timestamp · challenge-response · MAC/signature over the payload
```

Their absence on a state-changing command is a real finding: an attacker who sniffs one "unlock"/"command" can replay it forever.

### 9.2 Manipulation

Flip bytes in a known-good command (`AA 01 05` → `AA 01 06`) and observe. If the device acts on modified frames with no integrity check, commands are forgeable. Escalate toward reaching opcodes the app never exposes (undocumented/privileged commands) — again, strictly within authorized, non-destructive scope.

---

## 10. Phase 8 — Fuzzing

Only *after* you understand the framing (fuzzing blind is mostly noise). Mutate systematically and monitor for crash/reset/hang.

```python
import asyncio, os
from bleak import BleakClient
ADDR, CMD = "AA:BB:CC:DD:EE:FF", "0000xxxx-...."

cases = [
    b"", b"\x00", b"\xff",
    b"\x00"*20, b"\xff"*512,          # length extremes
    b"\xaa\x01" + b"\x41"*250,        # oversized payload on known opcode
    os.urandom(64),                    # random
]

async def main():
    for i, c in enumerate(cases):
        try:
            async with BleakClient(ADDR, timeout=10) as cl:
                await cl.write_gatt_char(CMD, c, response=False)
                print(f"[{i}] sent {len(c)}B ok")
        except Exception as e:
            print(f"[{i}] {len(c)}B -> {e}   (possible crash/disconnect — log & re-scan)")
        await asyncio.sleep(1)

asyncio.run(main())
```

Keep `btmon` running alongside. Record which input caused: crash, disconnect, spontaneous reboot, error code, or silent hang. A reproducible crash from a malformed BLE write is a denial-of-service (and possibly memory-corruption) finding. **mirage** and Defensics have proper BLE fuzzing modules if you need coverage-guided depth.

---

## 11. Phase 9 — Firmware / OTA update security

If a firmware/OTA service exists (`Start`, `Data`, `Verify`, `Commit` characteristics, or Nordic DFU `0xFE59`):

| Question | Test |
|---|---|
| Is auth required to start an update? | Attempt OTA start unpaired / Just-Works |
| Is the image **signed** and signature **verified on-device**? | Upload a flipped-bit image; does it reject? |
| Is the image encrypted? | Inspect captured chunks |
| Is **downgrade** prevented? | Attempt to flash an older valid image |
| Can arbitrary firmware be flashed? | Attempt a modified valid-format image |

**Do destructive OTA tests only on an expendable unit with a recovery/reflash path (SWD/JTAG).** Signature-bypass on an OTA channel is the most severe class of BLE finding — it's remote code execution on the device.

---

## 12. MITM (controlled environment only)

For man-in-the-middle you clone the peripheral's advertising and relay GATT while sitting between app and device. Tools: **mirage** (`ble_mitm` module), **GATTacker**, **btlejack** (connection hijack on legacy). Requirements and cautions:

- Two adapters (one impersonates the peripheral to the phone, one is central to the real device), or specialized hardware.
- Strong pairing with MITM protection (Numeric Comparison / Passkey under LE SC) is *designed* to defeat this — a successful MITM against SC is itself the finding.
- Legally sensitive: do it in a Faraday bag/room, never over the air near third parties.

```bash
# mirage example (conceptual — confirm interfaces on your rig)
sudo mirage "ble_mitm|TARGET=AA:BB:CC:DD:EE:FF|INTERFACE1=hci0|INTERFACE2=hci1"
```

---

## 13. Common BLE IoT vulnerability checklist

Run down this list against TK V5:

- [ ] Sensitive data (IMEI/GPS/serial) readable **without pairing**
- [ ] Privileged characteristic writable **without pairing** (broken authz)
- [ ] **Just Works** pairing on a command-issuing device (no MITM protection)
- [ ] **LE Legacy** pairing (key-recovery via crackle)
- [ ] Pairing **downgrade** accepted
- [ ] **No anti-replay** on state-changing commands
- [ ] **No integrity** on commands (byte-flip forgery)
- [ ] Undocumented/hidden privileged opcodes reachable
- [ ] Static/predictable/reused **passkey**
- [ ] Fixed public address → long-term **tracking**
- [ ] Manufacturer data leaks state/identity unencrypted
- [ ] **Unsigned/unverified OTA** or downgrade allowed
- [ ] Crash/DoS from malformed writes
- [ ] Bonded-but-unauthorized peer retains privileged access

---

## 14. Reporting template

For each finding:

```
Title:            e.g. "Vehicle command characteristic writable without pairing"
Severity:         CVSS + business impact (physical/vehicle impact raises it)
Affected:         TK V5, firmware X.Y.Z, characteristic UUID + handle
Preconditions:    proximity / unpaired / Just-Works paired
Reproduction:     exact steps + the script/command + captured bytes
Evidence:         btmon / Wireshark snippet, notification log (PII scrubbed)
Root cause:       missing authorization check on ATT write
Impact:           what an attacker in BLE range achieves
Remediation:      enforce LE Secure Connections + per-command authorization;
                  add replay protection (counter+MAC); sign+verify OTA
```

**Deliverables:** the filled **authorization matrix** (§7), the **decoded protocol table** (§8.3), the **vulnerability checklist** (§13), and per-finding write-ups. The matrix and protocol table are what distinguish a real BLE assessment from "I ran nRF Connect once."

---

## 15. Command quick-reference

```bash
# Recon
sudo btmon                                  # full HCI/advertising monitor
bluetoothctl → scan on                      # discover
sudo hcitool lescan --duplicates            # (legacy) raw scan

# Enumerate
bluetoothctl → connect <MAC> → list-attributes
gatttool -b <MAC> --characteristics
python enum.py                              # Bleak full-tree dump (§4.2)

# Interact
gatttool -b <MAC> --char-read -a 0x0012
gatttool -b <MAC> --char-write-req -a 0x0012 -n aa0105

# Pairing analysis
sudo btmgmt info ; sudo btmgmt io-cap 3     # force NoInputNoOutput (downgrade test)
bluetoothctl → pair <MAC>                   # watch SMP in btmon

# Sniff
adb pull /data/misc/bluetooth/logs/btsnoop_hci.log     # phone-side
python3 sniff_receiver.py -l -o cap.pcap                # Sniffle OTA
wireshark cap.pcap                                       # filter: btatt

# Offline
crackle -i legacy_pairing.pcap              # legacy LTK recovery
```

---

### Learning order (if you're building the skill, not just running the checklist)

1. **Fundamentals** — central/peripheral, advertising, GAP, connection.
2. **GATT** — services/UUIDs/characteristics/descriptors + properties.
3. **Security** — pairing methods, bonding, LE SC vs legacy, auth vs authz.
4. **Tooling** — `bluetoothctl`, `btmon`, `btmgmt`, Bleak, Wireshark.
5. **Advanced** — Sniffle sniffing, protocol RE, replay/manipulation, fuzzing, MITM, OTA analysis.

Everything above becomes concrete the first time you run the §4.2 enumeration script against a real device and watch the §5 notify stream change as you move the hardware. Start there.
