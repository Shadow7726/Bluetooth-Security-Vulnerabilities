# Kali Linux — BLE Bluetooth Setup & Bring-Up Manual (In-Depth)

A repeatable, diagnosable procedure for getting a Bluetooth adapter from "just plugged in" to "scanning BLE devices" on Kali — plus a troubleshooting tree for when a layer fails. Target example throughout: finding the **TK V5 / BlueBox** peripheral.

The guiding principle: **BLE bring-up is a stack.** When something doesn't work, you don't guess — you walk the stack bottom-up and find the first layer that's broken.

```
USB adapter ─► kernel/btusb ─► hci0 ─► BlueZ (bluetoothd) ─► bluetoothctl ─► scan ─► connect ─► GATT
```

Everything below is organized around that stack.

---

## 0. One-shot quick start (if you just want it working)

```bash
sudo rfkill unblock bluetooth
sudo systemctl enable --now bluetooth      # enable (at boot) + start (now), in one command
sudo hciconfig hci0 up                      # or: sudo btmgmt power on
bluetoothctl list                           # expect: Controller <MAC> <name> [default]
bluetoothctl
[bluetooth]# scan le
```

If every line behaved, skip to [§8 Scanning](#8-scanning-for-ble-devices). If *any* line failed, don't retry blindly — go to the matching section below.

> **`enable` vs `start` vs `enable --now`:** `start` runs the service *now* (gone after reboot). `enable` makes it auto-start *at boot* (does nothing right now). `enable --now` does both — the correct choice on a fresh box. A system where Bluetooth "works until reboot" is almost always one that was `start`ed but never `enable`d.

---

## 1. Layer 0 — The USB adapter

### Detect it

```bash
lsusb
```

Look for your dongle, e.g.:

```text
Bus 001 Device 005: ID 0a12:0001 Cambridge Silicon Radio, Ltd Bluetooth Dongle (HCI mode)
```

The `ID vvvv:pppp` pair (vendor:product) is your fingerprint for driver debugging. The CSR8510 (`0a12:0001`) is the classic cheap BLE-capable dongle.

### Confirm the kernel bound a driver

```bash
lsusb -t                        # tree view — shows the driver per device
dmesg | grep -iE 'bluetooth|btusb|hci' | tail -n 30
```

Healthy `dmesg` shows `btusb` claiming the device and a firmware load line. If you see the device in `lsusb` but **no `btusb` binding** and no `hciX` appears, that's a driver problem, not a BlueZ problem — see [§1.3](#13-adapter-visible-but-no-hci-interface).

### 1.1 Adapter choice matters for BLE

Not every "Bluetooth" dongle does BLE well. For pentesting:

| Adapter | BLE? | Notes |
|---|---|---|
| CSR8510 A10 (`0a12:0001`) | ✅ 4.0 | Cheap, ubiquitous, fine for central-side scan/connect. Many clones are flaky. |
| Intel AX200/AX210 (often built-in) | ✅ 5.x | Excellent, well-supported |
| Realtek RTL8761B (common USB 5.0) | ✅ 5.0 | Needs `firmware-realtek`; check dmesg for missing fw |
| nRF52840 dongle + Sniffle fw | 🔎 sniff only | A **sniffer**, not a BlueZ controller — won't show in `hciconfig` |

**Counterfeit CSR dongles** are a real time-sink: they enumerate but fail firmware load or drop connections. If `dmesg` shows repeated firmware errors with a `0a12:0001`, suspect a clone.

### 1.2 VM users — the #1 hidden failure (read this)

Kali is frequently run in a VM, and **USB Bluetooth must be explicitly passed through to the guest**, or the host keeps it and the guest sees nothing.

- **VMware:** VM Settings → USB Controller → enable; then from the VM menu, *Removable Devices → [your dongle] → Connect*. Set USB compatibility to 3.0/3.1 if the device isn't detected.
- **VirtualBox:** Settings → USB → enable the controller (xHCI) + add a USB device filter for the dongle. Requires the **Extension Pack**.
- **Built-in laptop Bluetooth generally will NOT pass through** to a VM — it's on an internal bus, not USB. Use an external USB dongle for VM-based testing.

Symptom of a passthrough problem: `lsusb` inside the guest **doesn't list the dongle at all**. Fix it at the hypervisor before touching anything in Kali.

### 1.3 Adapter visible but no HCI interface

`lsusb` shows it, but `hciconfig -a` lists nothing:

```bash
dmesg | grep -i btusb                 # look for firmware load failure
sudo modprobe btusb                   # ensure the module is loaded
lsmod | grep bt                       # btusb, btrtl, btintel, bluetooth should be present
# Realtek/Intel missing firmware:
sudo apt install -y firmware-realtek firmware-linux firmware-linux-nonfree
# then replug or:
sudo modprobe -r btusb && sudo modprobe btusb
```

Still nothing → bad/clone adapter, or (in a VM) passthrough not actually active.

---

## 2. Layer 1 — The HCI interface (hci0)

```bash
hciconfig -a
```

Healthy output:

```text
hci0:   Type: Primary  Bus: USB
        BD Address: 38:44:BE:1E:EA:0E  ACL MTU: ...
        UP RUNNING
        ...
```

Two things to read here:

- **Does `hci0` exist at all?** If not → it's a Layer-0 problem (§1), not here.
- **UP vs DOWN.** `DOWN` means the controller is present but not activated.

### Bring it up

```bash
sudo hciconfig hci0 up         # classic BlueZ path
# OR the newer management API:
sudo btmgmt power on
```

Verify:

```bash
hciconfig -a                    # want: UP RUNNING
```

> `hciconfig` is technically **deprecated** in modern BlueZ (part of `bluez-hcidump`/legacy tools) but still shipped on Kali and still the fastest readout. The modern equivalents are `btmgmt` and `bluetoothctl`. Keep both in your toolkit; when one gives confusing output, cross-check with the other.

### If `hciconfig hci0 up` fails

```bash
rfkill list                     # is it blocked? (see §3)
sudo btmgmt info                # does the mgmt API see the controller?
dmesg | tail                    # firmware / USB reset errors?
```

A controller that's present but refuses to go UP is usually **rfkill-blocked** (§3) or mid-firmware-failure (§1.3).

---

## 3. rfkill — the soft/hard block gate

`rfkill` can silently block the radio regardless of everything else being correct.

```bash
rfkill list
```

```text
X: hci0: Bluetooth
    Soft blocked: no
    Hard blocked: no
```

| State | Meaning | Fix |
|---|---|---|
| **Soft blocked: yes** | Software disabled it (service, DE toggle, airplane mode) | `sudo rfkill unblock bluetooth` |
| **Hard blocked: yes** | A physical switch / BIOS / laptop Fn-key | Toggle the hardware switch or BIOS; software can't override this |

```bash
sudo rfkill unblock bluetooth
rfkill list                      # confirm both now "no"
```

**Hard block cannot be cleared from software** — if you see `Hard blocked: yes`, it's a physical/BIOS switch (common on laptops with a wireless kill switch or an airplane-mode Fn key). On external USB dongles you'll essentially never see a hard block; if you do, suspect the host's own radio, not the dongle.

---

## 4. Layer 2 — BlueZ / bluetoothd (the daemon)

This is the layer that was missing in the original scenario: adapter ✓, hci0 ✓, `btmgmt` ✓ — but `bluetoothctl` showed **no controller** because the **daemon wasn't running**. `bluetoothctl` is just a client; it talks to `bluetoothd` over D-Bus. No daemon → no controller, even though the hardware is perfectly fine.

### Check and start

```bash
systemctl status bluetooth --no-pager
```

```text
Active: inactive (dead)        ← the problem
```

```bash
sudo systemctl enable --now bluetooth      # start now + at boot
systemctl status bluetooth --no-pager      # want: active (running)
```

### Why this layer is sneaky

Every *hardware* check (`lsusb`, `hciconfig`, `btmgmt`) can pass while this layer is dead, because those tools talk to the kernel/controller directly. Only the BlueZ **clients** (`bluetoothctl`, most GUI tools, and your Bleak scripts via D-Bus) need the daemon. So "the hardware is clearly fine but bluetoothctl sees nothing" is the signature of a dead `bluetoothd`.

### Restarting cleanly (when BlueZ gets into a weird state)

```bash
sudo systemctl restart bluetooth
# nuclear option — reset the controller + daemon together:
sudo hciconfig hci0 down
sudo systemctl restart bluetooth
sudo hciconfig hci0 up
```

### Logs for this layer

```bash
journalctl -u bluetooth -n 50 --no-pager    # daemon errors, plugin failures
```

---

## 5. Layer 3 — Confirm BlueZ sees the controller

```bash
bluetoothctl list
```

```text
Controller 38:44:BE:1E:EA:0E CSR8510 A10 [default]
```

If this is **empty** despite everything above passing:

1. `systemctl status bluetooth` — daemon actually running? (§4)
2. `sudo btmgmt info` — mgmt API sees it but bluetoothctl doesn't → D-Bus/daemon issue → `sudo systemctl restart bluetooth`.
3. Permissions — are you in the right group / using the right session? (§7)

### Cross-check the controller's capabilities

```bash
sudo btmgmt info
```

```text
hci0:   Primary controller
        addr 38:44:BE:1E:EA:0E ...
        supported settings: ... le br/edr ...
        current settings: powered le ...
```

**Make sure `le` appears in `supported settings`.** A controller that only lists `br/edr` (Classic) can't do BLE — wrong/too-old adapter for this work.

---

## 6. Inside bluetoothctl — baseline state

```bash
bluetoothctl
[bluetooth]# show
```

```text
Controller 38:44:BE:1E:EA:0E (public)
    Powered: yes
    Discoverable: no
    Pairable: no
    ...
```

For BLE recon you primarily need **`Powered: yes`**. If not:

```text
[bluetooth]# power on
```

Useful baseline toggles for testing (set deliberately, and note them — they affect your attack surface):

```text
[bluetooth]# power on
[bluetooth]# pairable off        # avoid accidental bonding during recon
[bluetooth]# discoverable off    # you're the central; you don't need to advertise
[bluetooth]# agent on            # needed later for pairing interaction
[bluetooth]# default-agent
```

Leaving `pairable off` during reconnaissance helps ensure you're genuinely testing the device's *unauthenticated* behavior and not quietly bonding.

---

## 7. Permissions, groups, and sudo

Most bring-up commands need privilege. Two common snags:

- **`bluetoothctl` scan returns nothing as a normal user, works under sudo** — your user may lack access to the Bluetooth D-Bus policy. On Kali you're often root already; on a hardened setup add your user to the right group and re-login:
  ```bash
  sudo usermod -aG bluetooth "$USER"    # then log out/in
  ```
- **Raw HCI operations** (`hciconfig up`, `btmgmt`, sniffing) need root or the `cap_net_admin`/`cap_net_raw` capabilities. For scripts, running under `sudo` is simplest; for Bleak (which goes through BlueZ/D-Bus) you usually **don't** need sudo once the daemon and permissions are right.

Quick test of whether it's a permission vs. stack problem: run the failing command with `sudo`. If it suddenly works, it's permissions; if it still fails, it's a lower layer.

---

## 8. Scanning for BLE devices

### BLE-only scan (cuts Classic noise)

```text
[bluetooth]# menu scan
[bluetooth]# transport le
[bluetooth]# back
[bluetooth]# scan on
```

or directly:

```text
[bluetooth]# scan le
```

You'll see devices stream in:

```text
[NEW] Device 84:FD:27:7A:6A:F8 BlueBox
[NEW] Device 5C:...            (unnamed)
```

For TK V5 work, watch for the **`BlueBox`** name. Many BLE devices advertise *without* a name — don't assume "no name = not it"; correlate by MAC, RSSI (walk toward the device and watch signal rise), and advertised service UUIDs.

### Capture the full advertisement (recommended)

`bluetoothctl` summarizes; to see the complete advertising payload (manufacturer data, service UUIDs, address type), run a monitor in a second terminal while scanning:

```bash
sudo btmon        # decoded HCI + full advertising reports
```

### Stop and inspect

```text
[bluetooth]# scan off
[bluetooth]# info 84:FD:27:7A:6A:F8
```

```text
Name: BlueBox
Alias: BlueBox
Paired: no
Bonded: no
Trusted: no
Connected: no
UUID: ...
```

This is your initial GAP picture. **Record it before connecting or pairing** — baseline the unauthenticated state first.

### If scanning finds nothing

```bash
rfkill list                     # re-check blocks (§3)
sudo btmgmt info                # confirm 'le' in settings (§5)
sudo hcitool lescan --duplicates    # legacy raw LE scan as a cross-check
```

If `btmon`/`hcitool lescan` see advertisements but `bluetoothctl` doesn't, restart the daemon (§4). If *nothing anywhere* sees adverts, the target may be non-connectable, out of range, using a rotating private address, or simply not advertising right now — move the device, power-cycle it, and retry.

---

## 9. From scan to GATT (hand-off to the assessment)

Once the target is confirmed:

```text
[bluetooth]# connect 84:FD:27:7A:6A:F8
[bluetooth]# menu gatt
[bluetooth]# list-attributes 84:FD:27:7A:6A:F8
```

That dumps services, characteristics, and descriptors with their handles — the starting point of GATT enumeration (covered in the main field manual). **Don't pair yet**: establish what's reachable unauthenticated first.

---

## 10. The complete, annotated startup sequence

Keep this in your pentest notes as the standard procedure:

```bash
# ── Layer 0: adapter ──────────────────────────────
lsusb                                   # dongle present? (VM: must be passed through)
dmesg | grep -i bluetooth | tail        # driver bound, firmware loaded?
hciconfig -a                            # hci0 exists?

# ── Layer: rfkill ─────────────────────────────────
rfkill list                             # soft/hard blocked?
sudo rfkill unblock bluetooth

# ── Layer 2: BlueZ daemon ─────────────────────────
sudo systemctl enable --now bluetooth   # start now + at boot (the key fix)
systemctl status bluetooth --no-pager   # active (running)?

# ── Layer 1/3: controller up & visible ────────────
sudo hciconfig hci0 up                  # or: sudo btmgmt power on
hciconfig -a                            # UP RUNNING?
btmgmt info                             # 'le' in supported settings?
bluetoothctl list                       # Controller ... [default]?

# ── Interactive ───────────────────────────────────
bluetoothctl
```

Inside `bluetoothctl`:

```text
show            # Powered: yes
power on
agent on
default-agent
scan le         # find the target (BlueBox)
scan off
info <TARGET_MAC>
connect <TARGET_MAC>
menu gatt
list-attributes <TARGET_MAC>
```

---

## 11. Troubleshooting tree (walk it top-down)

| Symptom | First suspect | Command | Fix |
|---|---|---|---|
| Dongle not in `lsusb` | USB / VM passthrough | `lsusb` | Pass USB through to VM; try another port; use external dongle |
| In `lsusb`, no `hci0` | Driver / firmware | `dmesg \| grep btusb` | `modprobe btusb`; install firmware-realtek/linux-nonfree; replug |
| `hci0` DOWN, won't go UP | rfkill | `rfkill list` | `rfkill unblock bluetooth`; check hardware switch for hard block |
| Hardware fine, `bluetoothctl list` empty | **bluetoothd dead** | `systemctl status bluetooth` | `systemctl enable --now bluetooth` |
| Works, breaks every reboot | `start` without `enable` | — | `systemctl enable --now bluetooth` |
| Only `br/edr`, no `le` | Wrong/old adapter | `btmgmt info` | Use a BLE-capable (4.0+) adapter |
| Scan empty but `btmon` sees adverts | D-Bus/daemon state | `btmon` vs `scan on` | `systemctl restart bluetooth` |
| Nothing scans at all | Range / private addr / not advertising | `hcitool lescan` | Move closer; power-cycle target; retry |
| Works with sudo only | Permissions | run with `sudo` | add user to `bluetooth` group, re-login |
| Flaky connects, fw errors in dmesg | Clone CSR dongle | `dmesg` | Replace with known-good adapter |

**Method:** start at the top of the tree, run the "command" column, and only move down once that layer is confirmed healthy. The first failing layer is the real problem — fixing layers above it won't help.

---

## 12. The mental model (keep this in your head)

```text
USB adapter        │ lsusb, dmesg, (VM passthrough!)
     ↓             │
kernel / btusb     │ lsmod, modprobe, firmware
     ↓             │
hci0               │ hciconfig -a  → UP RUNNING
     ↓             │
rfkill gate        │ rfkill list   → not blocked
     ↓             │
BlueZ / bluetoothd │ systemctl status bluetooth → running   ◄─ the usual culprit
     ↓             │
bluetoothctl       │ bluetoothctl list → controller [default]
     ↓             │
BLE scanning       │ scan le / btmon
     ↓             │
BLE connection     │ connect <MAC>
     ↓             │
GATT enumeration   │ menu gatt / list-attributes
     ↓             │
characteristics    │ services → chars → descriptors
     ↓             │
Read / Write / Notify
```

The earlier failure in your notes mapped exactly to this: adapter ✓, hci0 ✓, `hci0 UP` ✓, `btmgmt` ✓, **`bluetoothd` ✗ → `bluetoothctl` saw no controller.** Starting (and enabling) the service filled the missing layer. Whenever BLE "won't work," find your ✗ in this column and fix that layer — not the ones above it.

```bash
# the single command that would have prevented it:
sudo systemctl enable --now bluetooth
```
