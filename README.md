# Optical Transceiver CLI Cheatsheet & DDM Troubleshooting Guide

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Standards](https://img.shields.io/badge/Standards-SFF--8472%20%7C%20OIF--CMIS%20%7C%20IEEE%20802.3-0f766e.svg)](#optical-power-reference--attenuator-sizing)

A concise, production-ready CLI reference for inspecting optical transceivers, reading **DDM/DOM** telemetry, checking **FEC** counters, and troubleshooting physical-layer links across **Cisco**, **Arista**, **Huawei**, **MikroTik**, and **NVIDIA Networking**.

---

## Quick Jump: Primary DDM Command by Platform

| Vendor / OS | Primary DDM / Optical Telemetry Command |
| :--- | :--- |
| **Cisco IOS-XE** (Catalyst) | `show interfaces <intf> transceiver detail` |
| **Cisco NX-OS** (Nexus 9000) | `show interface ethernet <slot/port> transceiver details` |
| **Arista EOS** (4.21.5F+) | `show interfaces <intf> transceiver dom thresholds` |
| **Huawei VRP5** (Campus S-Series) | `display transceiver diagnosis interface <intf>` |
| **Huawei VRP8** (NE40E / NE8000 / CE) | `display optical-module extend information interface <intf>` |
| **Huawei SmartAX OLT** (MA5600T / MA5800) | `display port ddm-info <port>` *(in board interface view)* |
| **MikroTik RouterOS** (v6 / v7) | `/interface ethernet monitor <intf>` |
| **NVIDIA Cumulus Linux** (NVUE 5.11+) | `nv show platform transceiver <port> channel` |
| **NVIDIA Onyx / MLNX-OS** | `show interfaces ethernet <slot/port> transceiver diagnostics` |
| **NVIDIA MFT (`mlxlink`)** | `sudo mlxlink -d <dev> -p <port> --cable --ddm` |

### Contents

1. [Cisco Systems (IOS-XE & NX-OS)](#1-cisco-systems-ios-xe--nx-os)
2. [Arista Networks (EOS)](#2-arista-networks-eos)
3. [Huawei Technologies (VRP5, VRP8 & SmartAX OLT)](#3-huawei-technologies-vrp5-vrp8--smartax-olt)
4. [MikroTik (RouterOS & SwOS)](#4-mikrotik-routeros--swos)
5. [NVIDIA Networking (Cumulus, Linux `ethtool`, Onyx & `mlxlink`)](#5-nvidia-networking-cumulus-linux-ethtool-onyx--mlxlink)
6. [Optical Power Reference & Attenuator Sizing](#6-optical-power-reference--attenuator-sizing)
7. [DDM Link Troubleshooting Matrix](#7-ddm-link-troubleshooting-matrix)

> **Operational Note:** All `show`, `display`, `monitor`, `nv show`, and `ethtool -m` commands below are read-only. Test configuration changes (such as third-party optic overrides or FEC adjustments) inside a maintenance window.

---

## 1. Cisco Systems (IOS-XE & NX-OS)

> **Syntax Tip:** Catalyst IOS-XE uses plural `interfaces` + singular `detail`. Nexus NX-OS uses singular `interface` + plural `details`.

### Cisco IOS-XE (Catalyst 3850 / 9200 / 9300 / 9500 / 9600)

```bash
# Hardware inventory (PID, VID, Serial Number)
show inventory

# DDM/DOM summary across all ports
show interfaces transceiver

# Per-port DDM/DOM values with High/Low Alarm & Warning thresholds
show interfaces TenGigabitEthernet1/1/1 transceiver detail

# Transceiver properties, DOM support list, and raw IDPROM/EEPROM
show interfaces TenGigabitEthernet1/1/1 transceiver properties
show interfaces transceiver dom-supported-list
show idprom interface TenGigabitEthernet1/1/1
```

**Third-Party Optics Override (Catalyst IOS-XE):**

`service unsupported-transceiver` is a hidden global command on Catalyst switches (it does not appear in `?` help or Tab completion). Pair it with disabling `gbic-invalid` err-disable detection, then reseat the optic or bounce the port:

```bash
configure terminal
service unsupported-transceiver
no errdisable detect cause gbic-invalid
end
write memory
```

### Cisco NX-OS (Nexus 9000 Series)

```bash
# Hardware inventory & err-disabled port checks
show inventory
show interface status err-disabled

# Transceiver summary, full DDM details, and calibrations
show interface ethernet 1/1 transceiver
show interface ethernet 1/1 transceiver details
show interface ethernet 1/1 transceiver calibrations

# FEC counters (25G / 100G / 400G)
show interface fec

# NX-OS 10.4(3)F+: dedicated alarms & performance monitoring history
show interface transceiver alarms
show interface transceiver performance-monitoring history optics
show interface transceiver performance-monitoring history fec
```

* **Firmware & Policy Notes:**
  * NX-OS `10.5(3)F+` includes 400G DCO firmware versions inside `show interface <intf> transceiver details`.
  * On standalone Nexus 9000 NX-OS, `service unsupported-transceiver` is enabled by default.
  * On **Cisco ACI (APIC)**, `service unsupported-transceiver` is not supported.
  * Cisco TAC does not support unvalidated third-party optics and may require swapping to genuine modules during troubleshooting.

---

## 2. Arista Networks (EOS)

```bash
# Hardware inventory & transceiver summary
show inventory
show interfaces transceiver

# Classic detailed view
show interfaces Ethernet1/1 transceiver detail

# Modern DOM view with thresholds & unclipped low-power readouts (EOS 4.21.5F+)
show interfaces Ethernet1/1 transceiver dom
show interfaces Ethernet1/1 transceiver dom thresholds
show interfaces Ethernet1/1 transceiver dom thresholds unclipped

# Parsed EEPROM / CMIS pages & configured vs. operational laser parameters
show interface Ethernet1/1 transceiver eeprom
show interfaces Ethernet1/1 transceiver hardware

# Module state (Ready), data-path state (Activated), and active alarms
show transceiver status interface Ethernet1/1

# Pre-FEC BER & Uncorrected BER monitoring (requires PM enabled)
show interfaces Ethernet1/1 transceiver performance-monitoring
show interfaces Ethernet1/1 transceiver performance-monitoring thresholds
```

* **Third-Party & Layer 1 Notes:**
  * On newer EOS releases, `service unsupported-transceiver <CUSTOMER_NAME> <LICENSE_KEY>` is a hidden command requiring an account-specific 8-digit hex key issued by Arista.
  * For Layer 1 speed or FEC policy handling, EOS supports `system l1` → `unsupported [error-correction | speed] action [error | warn]`.

---

## 3. Huawei Technologies (VRP5, VRP8 & SmartAX OLT)

Huawei uses three distinct command sets depending on the hardware family.

### Campus Switches (S5700 / S5735 / S6720 / S6730 / S7700 / S9700 — VRP5)

```bash
# Hardware & electronic label info
display device
display version
display elabel

# Transceiver summary, real-time DDM diagnostics, and alarms
display transceiver
display transceiver diagnosis interface XGigabitEthernet0/0/1
display transceiver alarm interface XGigabitEthernet0/0/1

# Suppress non-Huawei-certified optic alarms
system-view
transceiver phony-alarm-disable
```

### NetEngine Routers (NE40E / NE8000) & CloudEngine Switches (CE5800–CE9800 — VRP8)

```bash
# Transceiver summary across chassis
display optical-module brief

# Basic info (vendor, rate, wavelength, optical power)
display optical-module base information interface 400GE1/0/0

# Extended per-lane DDM telemetry & High/Low Alarm/Warn thresholds
display optical-module extend information interface GigabitEthernet1/0/1

# E-label, firmware version, and inventory
display elabel optical-module brief
display optical-module version interface GigabitEthernet1/0/1 verbose
display inventory

# Suppress non-certified optic alarms
system-view
transceiver non-certified-alarm disable
```

### SmartAX OLT Uplinks (MA5600T / MA5800)

```bash
# Enter uplink board view first (e.g., GIU 0/18 or MPU 0/8)
config
interface giu 0/18

# Inspect optical state and real-time DDM telemetry on port 0
display port opticstate all
display port ddm-info 0
```

---

## 4. MikroTik (RouterOS & SwOS)

All SFP, SFP+, SFP28, QSFP+, QSFP28, and QSFP-DD diagnostics live under `/interface ethernet` (there is no separate `/sfp` submenu).

```routeros
# Live DDM & EEPROM monitor (RouterOS v6 / v7)
/interface ethernet monitor sfp-sfpplus1

# RouterOS v7 slash-path syntax (e.g., 100G QSFP28)
/interface/ethernet/monitor qsfp28-1-1

# System thermal sensors (CPU, PCB, SFP)
/system/health/print
```

* **Key Output Fields:** `sfp-module-present`, `sfp-rx-lose`, `sfp-tx-fault`, `sfp-vendor-name`, `sfp-vendor-part-number`, `sfp-vendor-serial`, `sfp-wavelength`, `sfp-temperature`, `sfp-supply-voltage`, `sfp-tx-bias-current`, `sfp-tx-power`, `sfp-rx-power`, `eeprom`, and `eeprom-checksum`.
* **Field Notes:**
  * **No Vendor Lock:** MikroTik accepts MSA-compliant third-party optics. However, if `eeprom-checksum` reads `bad`, RouterOS suppresses all EEPROM and DDM output.
  * **25G/100G FEC:** RouterOS disables FEC by default on SFP28/QSFP28 ports. Match `fec-mode` (`auto | fec74 | fec91 | off`) on **both ends** when interoperating with other vendors to avoid link failures or false link-ups.
  * **Thermal Protection:** `sfp-shutdown-temperature` defaults to `95°C` (SFP/SFP+/SFP28) and `80°C` (QSFP+/QSFP28). For hot-running 10GBASE-T modules like `S+RJ10` (~`2.7 W` avg, up to `90°C`), use at most one module per 4-port cage block and avoid adjacent placement.
  * **SwOS:** Check the Web UI **SFP** tab or poll via SNMP (`MIKROTIK-MIB`).

---

## 5. NVIDIA Networking (Cumulus, Linux `ethtool`, Onyx & `mlxlink`)

### Cumulus Linux (NVUE — Spectrum Switches)

```bash
# Recommended on Cumulus Linux 5.11+
nv show platform transceiver
nv show platform transceiver brief
nv show platform transceiver detail

# Per-port capabilities, per-channel power (mW & dBm), and thermal readings
nv show platform transceiver swp1
nv show platform transceiver swp1 channel
nv show platform transceiver swp1 temperature
```

### Standard Linux / Cumulus Shell (`ethtool`)

```bash
# Parsed SFP / QSFP / QSFP-DD EEPROM & DDM diagnostics
sudo ethtool -m swp1

# Hex dump or targeted CMIS page/bank inspection
sudo ethtool -m swp1 hex on
sudo ethtool -m swp1 page 0x11 bank 0x00 hex on
```

### NVIDIA Onyx / MLNX-OS (Spectrum Switches)

```bash
show interfaces ethernet transceiver brief
show interfaces ethernet 1/1 transceiver
show interfaces ethernet 1/1 transceiver diagnostics
```

### NVIDIA MFT `mlxlink` (ConnectX NICs, Quantum-2 & Spectrum)

*(In open-source `mstflint`, substitute `mstlink` for `mlxlink`.)*

```bash
# Module info, physical counters/BER, FEC, and read-only eye opening
sudo mlxlink -d /dev/mst/mt53120_pci_cr0 -p 1 --show_module
sudo mlxlink -d /dev/mst/mt53120_pci_cr0 -p 1 -m -c --show_fec
sudo mlxlink -d /dev/mst/mt53120_pci_cr0 -p 1 --show_eye
sudo mlxlink -d /dev/mst/mt53120_pci_cr0 -p 1 --show_module --json

# Cable/transceiver DDM telemetry and raw EEPROM dump
sudo mlxlink -d /dev/mst/mt53120_pci_cr0 -p 1 --cable --ddm
sudo mlxlink -d /dev/mst/mt53120_pci_cr0 -p 1 --cable --dump
```

* **Safety & Bug Notes:**
  * Avoid destructive PRBS tests or `--show_prr` on live production links.
  * On MFT `v4.34–v4.36`, `mlxlink -m` may report identical Rx/Tx power across lanes; use `mlxreg` (`PDDR`, `page_select=3`) or upgrade MFT to read per-lane power accurately.

---

## 6. Optical Power Reference & Attenuator Sizing

### dBm ↔ mW Quick Conversion (`SFF-8472` & `OIF-CMIS`)

Both **SFF-8472** (SFP/SFP+/SFP28) and **OIF-CMIS 4.0–5.3** (QSFP-DD/OSFP) encode optical power as a **16-bit unsigned integer** with a `0.1 µW` (`0.0001 mW`) LSB (`±3 dB` accuracy per SFF-8472):

* `0xFFFF` (`6.5535 mW`) = **`+8.1647 dBm`** (register ceiling)
* `0x0001` (`0.0001 mW`) = **`-40.0000 dBm`** (minimum non-zero reading — switches typically clamp `0x0000` / no light to `-40.0 dBm` or `N/A`)

| Optical Power (dBm) | Linear Power (mW) | Linear Power (µW) | Practical Context |
| :---: | :---: | :---: | :--- |
| **+8.16 dBm** | `6.5535 mW` | `6553.5 µW` | SFF-8472 / CMIS upper encoding limit (`0xFFFF`) |
| **+5.00 dBm** | `3.1623 mW` | `3162.3 µW` | Receiver **damage threshold** (10GBASE-ZR & 400GBASE-DR4) |
| **+4.00 dBm** | `2.5119 mW` | `2511.9 µW` | Max Tx launch power (10GBASE-ZR & 400GBASE-DR4 per lane) |
| **0.00 dBm** | `1.0000 mW` | `1000.0 µW` | `1 mW` reference; min Tx launch for 10GBASE-ZR |
| **-3.00 dBm** | `0.5012 mW` | `501.2 µW` | Half power; healthy mid-range for SR / LR / DR4 |
| **-5.90 dBm** | `0.2570 mW` | `257.0 µW` | Average Rx lower limit per lane for **400GBASE-DR4** |
| **-7.00 dBm** | `0.1995 mW` | `199.5 µW` | **Receiver overload** for 10GBASE-ZR & 80 km 10G BiDi (APD) |
| **-10.00 dBm** | `0.1000 mW` | `100.0 µW` | Near min Rx for 10GBASE-SR (`-9.9 dBm`); healthy for LR / ER / ZR |
| **-20.00 dBm** | `0.0100 mW` | `10.0 µW` | Below sensitivity for SR/LR/ER; valid for 10G-ZR (watch dispersion) |
| **-24.00 dBm** | `0.0040 mW` | `3.98 µW` | 10GE LAN/WAN sensitivity floor for 10GBASE-ZR (`-27 dBm` w/ OTU2 FEC) |
| **-40.00 dBm** | `0.0001 mW` | `0.10 µW` | Register floor (`0x0001`); indicates **LOS / No Light** |

### Reference Optical Budgets & Short-Span Attenuators

Never judge all optics by a single dBm cutoff. Always check the module's specific class:

| Standard | Typical Tx Launch | Rx Operating / Sensitivity | Overload / Damage | Inline Attenuator Rule |
| :--- | :--- | :--- | :--- | :--- |
| **10GBASE-SR** (850 nm MMF) | `-7.3` to `-1.2 dBm` | `-9.9` to `-1.0 dBm` | Min `2 m` patch (802.3ae) | **Not required** |
| **10GBASE-ER** (1550 nm, 40 km) | EML/DFB class | `-15` to `-18 dBm` typical | `-1.0` to `+0.5 dBm` *(by spec)* | **`5 dB`** for spans `< 20 km` |
| **10GBASE-ZR** (1550 nm, 80 km) | `0.0` to `+4.0 dBm` | `-24 dBm` (10GE) / `-27 dBm` (FEC) | **`-7.0 dBm`** / `+5.0 dBm` | **`15 dB`** (`<5 km`), **`10 dB`** (`5–25 km`), **`5 dB`** (`25–45 km`) |
| **10G BiDi 80 km** (1490/1550 nm) | `0.0` to `+4.0 dBm` | `-23` to `-24 dBm` typical | **`-7.0 dBm`** (APD) | **`10–15 dB`** for short bench tests (`<5 km`) |
| **400GBASE-DR4** (1310 nm, 500 m) | `-2.9` to `+4.0 dBm` / lane | `-5.9` to `+4.0 dBm` avg / lane | `+5.0 dBm` / `+5.0 dBm` | **Not required** (requires **RS(544,514) FEC**) |

**Bench-Test Attenuator Sizing Formula** *(for a 2 m patch cord with ~`0.5 dB` loss and `3 dB` safety margin)*:

```text
Recommended Attenuation (dB) = Max_Tx_Power - Rx_Overload - 0.5 + 3.0
```

*Example (10GBASE-ZR):* `+4.0 - (-7.0) - 0.5 + 3.0 = 13.5 dB` → use a **`15 dB`** fixed LC attenuator on the RX side.

---

## 7. DDM Link Troubleshooting Matrix

Capture DDM output from **both ends** of the link, then follow the matching branch:

| Observed Symptom | Likely Root Causes | Verification & Resolution |
| :--- | :--- | :--- |
| **RX = `-40.0 dBm` / `-inf` / `LOS`** | • Broken fiber, unplugged jumper, or reversed polarity<br>• Far-end port is `shutdown`, `err-disabled`, or `TX Fault`<br>• **BiDi mismatch:** same TX wavelength on both ends | 1. Verify far-end TX power and port admin/oper state.<br>2. Flip Duplex LC polarity or verify MPO-12 polarity.<br>3. For BiDi, confirm complementary A/B pairs (e.g., `T1270/R1330` ↔ `T1330/R1270`). |
| **Low RX Power or High Pre-FEC / Post-FEC BER** | • Dirty LC or MPO-12 ferrule end-face<br>• Tight fiber bend, bad splice, or SMF/MMF mismatch<br>• Chromatic dispersion on 80 km 1550 nm ZR links (`~2–3 dB` penalty near `1600 ps/nm`)<br>• FEC mismatch on 25G/100G/400G links | 1. Inspect and clean all connectors.<br>2. Measure total span loss with a power meter / OTDR.<br>3. Align FEC mode on both ends (`RS-FEC` / `fec91`) and check `show interface fec` or `mlxlink -c`. |
| **RX > Overload** *(e.g., `>-7 dBm` on 10G-ZR / 80 km BiDi)* | • Long-reach APD optic (ER, ZR, 40–120 km BiDi) plugged into a short patch cord without attenuation | 1. **Unplug immediately** to avoid permanent APD damage (`+5 dBm` threshold).<br>2. Insert a `5–15 dB` inline attenuator on the **RX** port. |
| **Unrecognized Module / `N/A` DDM / `err-disabled`** | • Bad EEPROM checksum or mismatched vendor coding<br>• Strict host optic check (`gbic-invalid`)<br>• CMIS 4.0 vs. 5.0 mismatch on 400G/800G modules | 1. Reseat module and check `show idprom` / `eeprom-checksum`.<br>2. Confirm switch OS & coding match; apply override only if permitted.<br>3. Check switch firmware & CMIS revision compatibility. |
| **High Temp Alarm / Sudden Port Shutdown** | • High-power 10GBASE-T (`S+RJ10`) or 400G/800G optics exceeding cage airflow<br>• Reversed fan airflow or blocked rack intake | 1. Verify rack ambient temperature and fan airflow direction.<br>2. Stagger 10GBASE-T copper modules (do not populate side-by-side).<br>3. Use industrial-temp (`-40°C` to `+85°C`) optics in uncontrolled environments. |

---

## Maintainer

Maintained by the engineering team at **[Fibertrixa](https://fibertrixa.com/)** (*Shenzhen Maifeixun Technology Co., Ltd.*) — B2B third-party compatible optical transceivers (`100M–800G`), EEPROM-coded and switch-verified before shipment.

* **Website:** [https://fibertrixa.com/](https://fibertrixa.com/)
* **Switch Compatibility Matrices:** [https://fibertrixa.com/en/compatible/](https://fibertrixa.com/en/compatible/)
* **License:** [MIT](LICENSE)
