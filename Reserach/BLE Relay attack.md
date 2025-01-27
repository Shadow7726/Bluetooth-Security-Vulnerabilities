

### BLE Relay Attack Overview
A **Bluetooth Low Energy (BLE) relay attack** occurs when an attacker bypasses the security of devices, vehicles, or appliances using BLE’s proximity authentication. The attack exploits the extended use of BLE beyond its original design, including applications like smart locks, vehicles, and keyless entry systems.

#### Attack Path:
- **Relay attack:** The attacker forwards encrypted link-layer data packets between two devices.
- **Result:** The BLE system is tricked into thinking an attacker’s device is nearby, enabling unauthorized access or operation.

#### Key Components:
- **BLE Proximity Authentication:** Used for proximity-based authentication in devices like smart locks and vehicles.
- **Link-layer Encryption:** Encryption used to secure communication, but can be bypassed in relay attacks.
- **Round-trip Latency:** The latency between request and response in BLE communications, used to detect relay attacks.

#### Demonstration:
- Tesla Model 3 and Kwikset/Weiser Kevo smart locks were successfully attacked.
- Relay attack was carried out from up to **25 meters** away, beyond the normal range of BLE.

---

### Tree Structure Representation

```
BLE Relay Attack
├── Vulnerable Devices
│   ├── Tesla Model 3
│   ├── Kwikset/Weiser Kevo Smart Locks
│   └── Other BLE-based systems (locks, vehicles, mobile phones)
├── Attack Mechanism
│   ├── BLE Proximity Authentication
│   ├── Forwarding Encrypted Data Packets (Link-layer PDUs)
│   ├── High Round-trip Latency (8 ms, undetectable)
│   └── Breach with Low Latency Relay
├── Attacker's Goal
│   ├── Unauthorized Access (Vehicle, Home, Device)
│   └── Operate Vulnerable Systems Remotely
└── Defensive Challenges
    ├── BLE Encryption (AES-128) 
    ├── Link-layer Protocol Changes
    ├── Vendor Countermeasures (Encryption, Latency Bounding)
    └── Short Range Defense (Limited to ~100 meters)
```

---

### Recommendations for Mitigation

| **Mitigation Measure**                                | **Description**                                                                                                                                 |
|-------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------|
| **Disable Proximity Key Functionality (Stationary State)**  | Disable the proximity key when the phone or key fob is stationary for a set period (detected by accelerometer).                                    |
| **Use a Second Factor for Authentication**           | Implement a second authentication factor such as a tap on an unlock button in an app to verify user presence.                                     |
| **Disable Passive Unlock Functionality**             | Prevent automatic unlocking or require explicit user approval before unlocking. Disable Bluetooth when not needed.                             |
| **Geofencing with GPS**                               | Use GPS data to confirm the user’s location and prevent the BLE system from being tricked into thinking the user is nearby.                     |
| **Link-layer Encryption**                             | Strengthen link-layer encryption (AES-128) to protect against relayed data being forwarded without detection.                                   |
| **Use Triangulation**                                 | Implement triangulation techniques for localization of devices to prevent remote relay attacks.                                                 |

---

### Key Points of Concern:
- **Limitations of BLE:** BLE was originally designed for short-range communication but is now being used for security-critical applications.
- **Security vs. Convenience Trade-off:** While BLE offers convenience, its security vulnerabilities (like relay attacks) expose users to risks.
- **Countermeasures are Limited:** Defenses such as encryption and latency bounding may not be effective against advanced relay attacks.

---

