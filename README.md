
# Bluetooth Security Vulnerabilities and Associated Tools:

1. **Resource Draining (DoS)**
   - *Description:* Drains battery or crashes devices by flooding with packets.
   - *Tools:*
      - [ubertooth](https://github.com/greatscottgadgets/ubertooth): Hardware tool for Bluetooth sniffing, eavesdropping, and packet capturing.
      - [gattacker](https://github.com/securing/gattacker): Bluetooth protocol fuzzer and MITM attack tool.

2. **DoS through Connection**
   - *Description:* Floods the target device with data to block communications.
   - *Tools:*
      - [gattacker](https://github.com/securing/gattacker): Bluetooth protocol fuzzer and MITM attack tool.
      - [blueranger](https://github.com/Fullmetal5/blueranger): Bluetooth MITM attack framework.

3. **Jamming (DoS)**
   - *Description:* Jams Bluetooth signals to block communications.
   - *Tools:*
      - Signal generator: [Hardware tool to generate radio frequency signals](https://www.jammer-store.com/bluetooth-jammers.html).
      - RF amplifier: [Hardware tool to amplify radio frequency signals](https://www.minicircuits.com/solutions/bluetooth.html).

4. **Device Spoofing**
   - *Description:* Impersonates legitimate devices by spoofing identifiers.
   - *Tools:*
      - [gattacker](https://github.com/securing/gattacker): Bluetooth protocol fuzzer and MITM attack tool.
      - [btlejack](https://github.com/virtualabs/btlejack): Bluetooth Low Energy MITM attack framework.

5. **Eavesdropping**
   - *Description:* Intercepts and listens to Bluetooth traffic.
   - *Tools:*
      - [ubertooth](https://github.com/greatscottgadgets/ubertooth): Hardware tool for Bluetooth sniffing, eavesdropping, and packet capturing.
      - [bluelog](https://github.com/MS3FGX/Bluelog): Bluetooth scanner and sniffer tool for Linux.

6. **Man-in-the-Middle (MitM)**
   - *Description:* Intercepts and alters Bluetooth communications.
   - *Tools:*
      - [gattacker](https://github.com/securing/gattacker): Bluetooth protocol fuzzer and MITM attack tool.
      - [btlejack](https://github.com/virtualabs/btlejack): Bluetooth Low Energy MITM attack framework.

7. **Legacy Pairing**
   - *Description:* Exploits weaknesses in outdated pairing methods.
   - *Tools:*
      - [crackle](https://github.com/mikeryan/crackle/): Bluetooth brute force PIN cracking tool.
      - [BTCrack](https://www.autosec.org/btcrack.html): Bluetooth PIN and key cracking tool.

8. **Replay Attacks**
   - *Description:* Records and replays Bluetooth messages.
   - *Tools:*
      - [gattacker](https://github.com/securing/gattacker): Bluetooth protocol fuzzer and MITM attack tool.

9. **Relay Attacks**
   - *Description:* Forwards Bluetooth messages between devices.
   - *Tools:*
      - [gattacker](https://github.com/securing/gattacker): Bluetooth protocol fuzzer and MITM attack tool.

10. **Malicious Apps**
    - *Description:* Apps that exploit Bluetooth vulnerabilities.
    - *Tools:*
       - [cara](https://github.com/projectcara/cara): Bluetooth security auditing and exploitation tool.
       - [gattacker](https://github.com/securing/gattacker): Bluetooth protocol fuzzer and MITM attack tool.

11. **Weaken Pairing Algorithm**
    - *Description:* Exploits weaknesses in the pairing algorithm.
    - *Tools:*
       - [crackle](https://github.com/mikeryan/crackle/): Bluetooth brute force PIN cracking tool.
       - [BTCrack](https://www.autosec.org/btcrack.html): Bluetooth PIN and key cracking tool.

12. **Debug Mode**
    - *Description:* Enabled debug mode leaves vulnerabilities.
    - *Tools:*
       - [hcitool](https://manpages.debian.org/testing/bluez-hcidump/hcitool.1.en.html): Bluetooth configuration and testing tool for Linux.

13. **Characteristic Permissions**
    - *Description:* Incorrect permissions enable unauthorized access.
    - *Tools:*
       - [gattacker](https://github.com/securing/gattacker): Bluetooth protocol fuzzer and MITM attack tool.
       - [btlejack](https://github.com/virtualabs/btlejack): Bluetooth Low Energy MITM attack framework.

14. **Identity Tracking**
    - *Description:* Tracks devices via Bluetooth identifiers.
    - *Tools:*
       - [btscanner](https://github.com/ghostop14/btscanner): Bluetooth device scanner and reconnaissance tool.
       - [bluelog](https://github.com/MS3FGX/Bluelog): Bluetooth scanner and sniffer tool for Linux.

15. **BLEEDINGBIT**
    - *Description:* Sniffs traffic of Bluetooth Low Energy devices.
    - *Tools:*
       - [ubertooth](https://github.com/greatscottgadgets/ubertooth): Hardware tool for Bluetooth sniffing, eavesdropping, and packet capturing.

16. **BleedingTooth**
    - *Description:* Exploits firmware vulnerabilities to gain unauthorized access.
    - *Tools:*
       - [cara](https://github.com/projectcara/cara): Bluetooth security auditing and exploitation tool.
       - [gattacker](https://github.com/securing/gattacker): Bluetooth protocol fuzzer and MITM attack tool.

17. **SweynTooth**
    - *Description:* Exploits firmware vulnerabilities to intercept connections and cause DoS.
    - *Tools:*
       - [blueranger](https://github.com/Fullmetal5/blueranger): Bluetooth MITM attack framework.

18. **KNOB**
    - *Description:* Manipulates key length negotiations to weaken encryption.
    - *Tools:*
       - [crackle](https://github.com/mikeryan/crackle/): Bluetooth brute force PIN cracking tool.
       - [BTCrack](https://www.autosec.org/btcrack.html): Bluetooth PIN and key cracking tool.

