| Title               | Transient Denial of Service Vulnerabilities in Unisoc Tanggula T760's 5G Baseband |
| ------------------- | --------------------------------------------------------------------------------- |
| Affected Product    | Unisoc Tanggula T760 system-on-chip (SoC) with integrated 5G baseband             |
| Affected Firmware   | `5G_MODEM_V2_23B_W24.33.5_P16.1` (older versions may also be affected)            |
| CVE IDs             | [CVE-2025-31717](https://www.cve.org/CVERecord?id=CVE-2025-31717), [CVE-2025-31718](https://www.cve.org/CVERecord?id=CVE-2025-31718) |
| Vendor Website      | https://www.unisoc.com                                                            |
| Identified in       | June 2025                                                                         |
| Identified by       | IoT Lab, University of Applied Sciences Upper Austria, Campus Hagenberg           |
| Website             | https://www.fh-ooe.at/si/                                                         |
| Team                | Denis Krämer B.Sc.                                                                |
|                     | Dieter Vymazal M.Sc.                                                              |
|                     | DI Markus Zeilinger                                                               |
| Contact Information | Denis Krämer <br> <denis.kraemer@students.fh-hagenberg.at> <br> 7B5B 5FCD 14A2 4A20 C43A A6D0 8B78 FC61 E205 BE39 |

## Vendor Description

*"UNISOC is a globally leading chip design company specializing in the communication semiconductor industry for over 20 years. It possesses comprehensive capabilities in chip design, wireless communication, and the integration of hardware and software systems [...]."*

Source: https://www.unisoc.com/en/about/company-info

## Overview

During the 5G NR initial attachment procedure, unauthenticated radio resource control (RRC) messages are used to set up a connection between the baseband and a base station (gNB) of the 5G mobile network. By modifying downlink RRC setup messages in a specific manner, several reachable assertions in the baseband firmware of the Unisoc Tanggula T760 can be triggered, each of which result in a silent panic and restart of the modem. Consequently, the mobile network connection of the Motorola Moto G35 5G smartphone (which uses this SoC) becomes temporarily unavailable, and the modem keeps restarting until the transmission of the modified RRC packets via the gNB is stopped. This classifies each of the vulnerabilities as transient DoS.

## Impact

An attacker can use a malicious gNB to send modified RRC setup messages to nearby Unisoc basebands and continuously trigger a modem reset, which results in a loss of mobile network connectivity for these subscribers until the attack is stopped. The attack requires no knowledge of any confidential information stored on the universal integrated circuit cards (UICCs) of the targets, because the RRC setup message is sent by the network to the baseband before any mutual authentication (5G-AKA) is performed. Only the mobile country code (MCC) and mobile network code (MNC) of the target's network is required to successfully execute such an attack. Since the [ITU-T assigns and publishes the MCC/MNC of public networks](http://handle.itu.int/11.1002/pub/82153a48-en), and only a handful of them exist per country, we argue that this requirement adds little complexity to the attack. Moreover, while the attack requires physical proximity to the target, it requires no interaction on behalf of the user.

## Proof of Concept

In the proof of concept (PoC) video for vulnerability V1, a modified RRC setup message is sent to the baseband upon connecting to a malicious nearby gNB. A reachable assertion in the firmware is triggered and the modem restarts, which can be verified by the Android Debug Bridge (adb) logs of the smartphone. The adb logs contain further information about the location of the exception. Moreover, a pop-up notification `No SIM card found` on the display of the Android smartphone and a temporary loss of connectivity can be observed.

<video controls muted preload><source src="/unisoc_t760_baseband_vulnerabilities/assets/videos/V1_poc_exploit_2x.mp4" type="video/mp4"></video> <br> *Video 1: Proof of concept exploitation of vulnerability V1.*

## Reproduction

We used the [5Ghoul packet interception API](https://github.com/asset-group/5ghoul-5g-nr-attacks#4--create-your-own-5g-exploits-test-cases) to write test cases that modify downlink traffic and trigger the vulnerabilities, but any toolset capable of performing such modifications is suitable to execute the attacks. We then used an RF shielded test environment comprised of a software-defined radio (SDR) and the affected user equipment (UE) to transmit modified RRC setup messages to the baseband. The traffic captured on the NR-Uu interface while triggering the vulnerabilities is shown in the provided Wireshark screenshots, which each depict the original RRC message on the left-hand side (not sent to the baseband) and the modified RRC message on the right-hand side (sent to the baseband). Finally, we observed several modem panics by inspecting the adb logs of the smartphone.

## Vulnerability V1: invalid contention resolution ([CVE-2025-31717](https://www.cve.org/CVERecord?id=CVE-2025-31717))

Triggering vulnerability V1 results in the following adb log entry:

```
MODEM_SILENT_PANIC: Error message: Modem Assert: NR_PAL_L_TASK Task  PHY CP assert in file nr_fsm_handle.c line 12999 exp=0 info=[PAL:ASSERT:bwpIdx is not found.1], [dfs=5], [abort_pc=0 abort_r13=0 DFSR=0 FAR=0 svc_r13=0 irq_r13=0]
```

To reproduce this issue, starting from the MAC-NR layer, change the first four bytes of a valid RRC setup message to `{0xf0, 0x1d, 0x66, 0xb1}`, as shown in figure 1. According to the interpretation by the Wireshark dissector, this results in a different semantic interpretation of the `Contention Resolution` subheader.

<img src="assets/images/V1_rrc_setup_original_vs_modified.png" alt=""> <br> *Figure 1: Comparison of the original and modified RRC setup message for vulnerability V1 in Wireshark.*

## Vulnerability V2: invalid monitoringSymbolsWithinSlot ([CVE-2025-31718](https://www.cve.org/CVERecord?id=CVE-2025-31718))

Triggering vulnerability V2 results in the following adb log entry:

```
MODEM_SILENT_PANIC: Error message: Modem Assert: NR_CTRL_SLOT_TASK Task  PHY CP assert in file nr_sym_buf_v2.c line 351 exp=0 info=[pdcerrinfo:0x19,pdc_trig_type:0x00000000, sbuf(0x10051006,0x10071008,0x1009100a,0x100b100c,0x100d1100,0x11011102,0x11031104,0x11051106,0x11071108),curslot:0x1
```

Note: this message is logged without a closing bracket.

To reproduce this issue, starting from the MAC-NR layer, change the byte at offset 51 of a valid RRC setup message to `{0x5b}`, as shown in figure 2. According to the interpretation by the Wireshark dissector, this corresponds to changing the value of the `monitoringSymbolsWithinSlot` field inside the RRC payload to `{0x82, 0xd8}`.

<img src="assets/images/V2_rrc_setup_original_vs_modified.png" alt=""> <br> *Figure 2: Comparison of the original and modified RRC setup message for vulnerability V2 in Wireshark.*

Note: the payload that triggers vulnerability V2 is identical to [CVE-2023-32841](https://www.cve.org/CVERecord?id=CVE-2023-32841), which affects another baseband vendor.

## Affected Product and Firmware

The following product and firmware were tested and found to be affected by the vulnerabilities:

* Smartphone: Motorola Moto G35 5G (Unisoc Tanggula T760 with integrated 5G baseband)
* Firmware: Android 14, Build: `UOA34.216-174-12`, Patch level: 2025-04-05, Baseband version: `5G_MODEM_V2_23B_W24.33.5_P16.1` (older versions may also be affected)

SHA-256 hash of the affected baseband firmware image (bundled by Motorola):
```
ffc8ece7229e30cc530d45fa1257c63c57370e6e36503aa39b315f8f7eede126  SC9600_QogirN6Pro_PHY_SECU_modem.bin
```

## Solution

Unisoc publicly disclosed the vulnerabilities V1 and V2 as part of their [October 2025 Security Bulletin](https://www.unisoc.com/en/support/announcement/1976557615080263681). Moreover, Unisoc stated that they have released updated baseband firmware which fixes these vulnerabilities. According to Unisoc, the baseband firmware version `5G_MODEM_V2_23B_W24.26.4_P21` for the Motorola Moto G35 5G includes patches for the vulnerabilities V1 and V2.

## Communication Timeline

| Date       | Sender       | Description |
| ---------- | ------------ | ----------- |
| 2025-06-18 | Denis Krämer | Contacted Unisoc using their PGP public key and attached the vulnerability report |
| 2025-06-19 | Unisoc       | Acknowledges receipt of the report and requests further information on the vulnerabilities |
| 2025-06-19 | Denis Krämer | Provided the requested information to Unisoc |
| 2025-07-24 | Unisoc       | Acknowledges the vulnerabilities as valid, plans to disclose them in October 2025 |
| 2025-07-24 | Unisoc       | Asks for the information to be published on their acknowledgements page |
| 2025-07-29 | Unisoc       | Asks for the acknowledgement information again |
| 2025-07-29 | Denis Krämer | Provided the information for the acknowledgements page |
| 2025-07-29 | Unisoc       | Confirms receipt of the information |
| 2025-10-11 | -            | Unisoc releases their October 2025 security bulletin |
| 2025-10-14 | Denis Krämer | Inquired about the mapping of vulnerabilities to CVE IDs, pointed out an inconsistency in the CVSS scores |
| 2025-10-15 | Unisoc       | Clarifies the mapping and updates the CVSS scores |
| 2025-11-04 | Denis Krämer | Asked which baseband firmware version fixes the vulnerabilities V1 and V2 |
| 2025-11-06 | Unisoc       | States that firmware version `5G_MODEM_V2_23B_W24.26.4_P21` fixes these issues |

## Version History

| Date       | Version | Changes         |
| ---------- | ------- | ----------------|
| 2025-11-25 | v1.0    | Initial release |