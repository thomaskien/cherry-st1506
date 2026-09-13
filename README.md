# CHERRY ST-1506 / Cobra V2.2 — hardware notes and Linux repurposing

> **Status:** work in progress / reverse-engineering notes from one defective CHERRY ST-1506 unit.  
> **Last updated:** 2026-09-13

This repository documents observations from opening and investigating a defective **CHERRY eHealth Terminal ST-1506**, with the aim of reusing the enclosure, touchscreen/display and possibly the original **PX30 / Cobra** mainboard as a general-purpose Linux device. A second goal is to fit a normal USB smart-card reader into the enclosure and use it through a custom SICCT proxy.

These notes deliberately distinguish between **observed on the actual unit**, **documented upstream**, and **not yet verified**.

**Video:** [Watch on YouTube](https://www.youtube.com/watch?v=uRO1DDZfvUY)

## Important warning

Opening an ST-1506 is **not a repair procedure for a terminal that is to remain in TI service**. The ST-1506 is a security-certified device with tamper protection and security seals. Opening the enclosure can trigger the permanent tamper state and invalidates the assumptions under which it is used as a certified eHealth card terminal.

The work described here is intended only for **defective/decommissioned hardware that will be repurposed as an ordinary Linux/electronics project**. A modified device must not be presented or used as a certified gematik/BSI eHealth terminal.

---

## 1. Device identification

Official CHERRY specifications for the ST-1506:

| Property | Value |
|---|---:|
| Product | CHERRY eHealth Terminal ST-1506 |
| External dimensions | **195 × 104 × 76 mm** |
| Display | **5 inch, 720p** |
| External connectivity | Ethernet and USB; CHERRY documents RNDIS/ECM support for USB operation |

Source: CHERRY product page:  
https://www.cherry.de/de-de/produkt/ehealth-terminal-st-1506

CHERRY download page (manuals, USB-LAN proxy, firmware/release notes):  
https://www.cherry.de/de-de/produkt/ehealth-terminal-st-1506/downloads

### Unit investigated here

The opened PCB is clearly marked:

```text
Cobra V2.2
```

The application processor visible on the board is a **Rockchip PX30**.

This matches the upstream Linux **PX30 Cobra** platform very closely. Mainline Linux now contains `px30-cobra.dtsi` plus several Cobra display variants.

Upstream description of the Cobra family states that these are PX30 touchscreen devices with eMMC, Ethernet, USB host + OTG and 720×1280 displays.

Linux sources:

- Base DTS:  
  https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/rockchip/px30-cobra.dtsi
- Rockchip DT Makefile, showing the Cobra DTBs:  
  https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/rockchip/Makefile

Known upstream DTB variants include:

```text
px30-cobra-ltk050h3146w.dtb
px30-cobra-ltk050h3146w-a2.dtb
px30-cobra-ltk050h3148w.dtb
px30-cobra-ltk500hd1829.dtb
```

This is strong evidence that retaining the original mainboard and running a current mainline Linux kernel is a realistic route.

---

## 2. Opening the enclosure

### What CHERRY support advised

A CHERRY technician advised removing the **front display/glass assembly from the front using a plastic plectrum**, because the front is adhesively bonded.

### Heating

A display-separator heating plate with vacuum was used for experimentation.

Observed on this unit:

- **55 °C was not enough** to produce appreciable movement of the bonded front.
- The exact temperature/time at which the unit was ultimately opened was **not recorded here**, so there is deliberately no claim that a specific higher temperature is proven safe for the ST-1506.
- General display-repair practice is to heat evenly and use a thin plastic pick to cut softened adhesive rather than bending the glass.

A vacuum separator plate is useful because it can hold the glass flat while the housing is worked away from it.

### Security seals / tamper protection

The visible BSI/security stickers are part of the security concept, not ordinary cosmetic labels. The ST-1506 administrator documentation and Common Criteria material describe security seals and an active manipulation-protection mechanism.

A visible side sticker on the investigated unit appears to bridge a case seam; there is **no evidence that it hides a service screw**. Treat this as an observation, not a factory service instruction.

Relevant public material:

- Common Criteria certificate/report material:  
  https://www.commoncriteriaportal.org/files/epfiles/1124V3a_pdf.pdf
- CHERRY/ST-1506 security target:  
  https://www.commoncriteriaportal.org/files/epfiles/1124V3b_pdf.pdf
- CHERRY administrator/user manuals are available from the CHERRY download page above.

The documented tamper condition can be caused by opening/manipulation, transport/fall damage, device defects or a depleted security battery. Once the device has been opened for this project, it should be regarded as **repurposing hardware only**.

---

## 3. Mainboard / Cobra platform

### Confirmed from the unit

- PCB marking: **Cobra V2.2**
- SoC: **Rockchip PX30**
- Display connector on PCB: **P10**, 31 contacts; silkscreen marks the ends as `1/2` and `30/31`.
- USB-C connection can enumerate the unit as a USB device (see RNDIS section below).

### Useful information already present in upstream Linux

The current upstream `px30-cobra.dtsi` exposes a surprisingly large amount of the hardware design.

#### Serial console

The upstream DTS sets:

```dts
chosen {
    stdout-path = "serial5:115200n8";
};
```

So the most promising passive debug interface is **UART5 at 115200 baud, 8N1**.

The DTS also enables `uart5` and assigns `uart5_xfer` pins.

**Next useful hardware task:** identify the UART5 test pads/header on the Cobra V2.2 PCB and capture the boot log with a **3.3 V UART adapter**, initially RX + GND only.

Do not assume a test pad voltage without measuring it.

#### eMMC

The mainline DTS enables a non-removable, 8-bit eMMC device. This makes an eMMC backup a high-priority step before any experimentation that writes to the original storage.

#### Ethernet

The current Cobra DTS describes a **TI DP83825** Ethernet PHY at MDIO address 0.

#### USB

The upstream platform enables:

```text
USB2 PHY
USB2 host PHY
USB2 OTG PHY
USB20 OTG controller
USB host EHCI
USB host OHCI
```

The board DTS also contains GPIO controls for USB-hub reset, USB-A 5 V enable and USB-A data enable.

#### Other useful peripherals already described upstream

The DTS contains definitions for:

- PWM backlight (`pwm0`, period corresponding to 25 kHz)
- PWM beeper (`pwm1`)
- RGB ring LEDs on PWM5/PWM6/PWM7
- watchdog
- SAR ADC / thermal ADC
- display subsystem / DSI D-PHY
- eMMC reset/power sequencing

Source:  
https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/rockchip/px30-cobra.dtsi

---

## 4. Touchscreen

### Controller identified from the actual flex PCB

The touchscreen controller is marked **Goodix GT911**.

The upstream Cobra DTS independently confirms exactly this controller:

```dts
touchscreen@14 {
    compatible = "goodix,gt911";
    reg = <0x14>;
    AVDD28-supply = <&vcc_2v8>;
    VDDIO-supply = <&vcc_3v3>;
    ...
};
```

For this Cobra design, upstream Linux describes:

- I²C address: **0x14**
- analog supply: **2.8 V**
- I/O supply: **3.3 V**
- interrupt: GPIO0 A1
- reset: GPIO0 B5
- `touchscreen-inverted-x`

The generic upstream Goodix binding supports the GT911 and documents the possible I²C addresses **0x14 and 0x5d**; the address can depend on the INT/reset sequence. The Cobra DTS specifically selects `0x14`.

Sources:

- Cobra DTS:  
  https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/rockchip/px30-cobra.dtsi
- Goodix Device Tree binding:  
  https://github.com/torvalds/linux/blob/master/Documentation/devicetree/bindings/input/touchscreen/goodix.yaml

### Touch flex markings observed

The touch/display assembly carries markings resembling:

```text
LTK50365A0
FPC50365A0-V0
```

These markings appear to identify the flex/assembly and are **not yet sufficient to prove the exact LCD module variant**.

---

## 5. LCD / MIPI-DSI display

### What is known

The Cobra platform uses MIPI-DSI display variants. Mainline Linux supports the following Leadtek panels on Cobra:

- `LTK050H3146W`
- `LTK050H3146W-A2`
- `LTK050H3148W`
- a prototype variant with `LTK500HD1829`

The Leadtek driver in mainline Linux supports the first three variants and includes their initialization sequences:

https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/panel/panel-leadtek-ltk050h3146w.c

Kernel Kconfig describes the LTK050H3146W family as **720×1280, 24-bit RGB over MIPI DSI**.

### Exact module variant on this unit

**Not yet confirmed.**

Before applying external power to the panel, look for a module label identifying one of the Leadtek variants above. The flex marking alone should not be treated as the LCD model number.

### 31-pin P10 connector

The PCB connector P10 has 31 positions, and the physical routing visible in the photographs is consistent with a MIPI-DSI interface (differential pairs separated by grounds).

A public LTK050H3146W drawing/datasheet gives the following **31-pin interface**. This is highly relevant to the Cobra/P10 investigation, but until the exact panel variant is confirmed it should be treated as a **candidate pinout, not a license to wire power blindly**.

| Pin | LTK050H3146W datasheet signal |
|---:|---|
| 1 | GND |
| 2 | VDDC_3V3 |
| 3 | GND |
| 4 | TOUCH_INT |
| 5 | TOUCH_RSTN |
| 6 | TOUCH_SDA |
| 7 | TOUCH_SCL |
| 8 | GND |
| 9 | LED K |
| 10 | LED A |
| 11 | LED K |
| 12 | RSTN |
| 13 | VCC |
| 14 | VCC_1.8 |
| 15 | VCC_2.8 |
| 16 | GND |
| 17 | D3P |
| 18 | D3N |
| 19 | GND |
| 20 | D2P |
| 21 | D2N |
| 22 | GND |
| 23 | CKP |
| 24 | CKN |
| 25 | GND |
| 26 | D1P |
| 27 | D1N |
| 28 | GND |
| 29 | D0P |
| 30 | D0N |
| 31 | GND |

Public drawing/datasheet mirror:  
https://www.texim-europe.com/getfile.ashx?id=138898

The same document specifies a typical backlight operating point around:

```text
IF = 40 mA
VF = 19.2 V typ.
```

Therefore **do not connect 5 V directly to LEDA**. The original Cobra board uses a dedicated backlight supply/control path; the upstream DTS models the backlight as PWM-controlled and powered from the system 5 V rail, implying a separate LED boost/current-driver stage on the board.

### Separate six-pin touch connector (from the same LTK050H3146W document)

The Leadtek document also lists:

| TP pin | Signal |
|---:|---|
| 1 | VDD 3.3 V |
| 2 | GND |
| 3 | INT |
| 4 | RST 3.3 V |
| 5 | SDA 3.3 V |
| 6 | SCL 3.3 V |

Again: confirm the actual assembly before wiring it independently.

---

## 6. USB-C / RNDIS findings on the original board

The opened/defective terminal was connected over USB-C to a Linux laptop.

### Enumeration

It appeared as:

```text
ID 046a:0084 CHERRY eHealth Terminal ST1506
```

Kernel messages from a successful enumeration included:

```text
usb 3-2: new high-speed USB device number 2 using xhci_hcd
usb 3-2: New USB device found, idVendor=046a, idProduct=0084, bcdDevice= 4.00
usb 3-2: Product: eHealth Terminal ST1506
usb 3-2: Manufacturer: CHERRY
rndis_host 3-2:1.0 eth0: register 'rndis_host' ... RNDIS device, 00:00:00:a8:ff:fb
rndis_host 3-2:1.0 enx000000a8fffb: renamed from eth0
```

This confirms that the original firmware reaches a state where its USB gadget enumerates as an **RNDIS network adapter**.

CHERRY officially documents USB operation requiring RNDIS/ECM support. IGEL's ST-1506 documentation describes a typical static setup with:

```text
terminal: 192.168.42.42
host:     192.168.42.1
SICCT:    TCP/UDP 4742
```

Important: `192.168.42.42` is a **configurable/default example**, not proof that every ST-1506 is currently configured to that address.

Source:  
https://kb.igel.com/en/igel-os/11.20.430/using-cherry-ehealth-card-terminal-st-1506-in-usb-

### Data-path problem observed on this unit

The Linux host subsequently logged transmit-queue timeouts:

```text
rndis_host ... NETDEV WATCHDOG: ... transmit queue 0 timed out
```

In one run the USB device then disconnected. In another run the RNDIS interface remained present long enough for testing.

The host interface was manually configured:

```bash
sudo ip link set enx000000a8fffb up
sudo ip addr flush dev enx000000a8fffb
sudo ip addr add 192.168.42.1/24 dev enx000000a8fffb
ping -I enx000000a8fffb -c 5 192.168.42.42
```

Result:

```text
5 packets transmitted, 0 received, 100% packet loss
```

and:

```text
192.168.42.42 FAILED
```

from `ip neigh show dev enx000000a8fffb`.

So far we know:

- USB enumeration works.
- The RNDIS driver binds successfully.
- The interface can reach `UP,LOWER_UP` on the Linux host.
- No ARP reply was received from the tested `192.168.42.42` address.
- The host's RNDIS transmit queue timed out.

We **do not yet know the cause**. Possibilities include the ST-1506 firmware/tamper state, a different configured device IP, an RNDIS/USB interaction problem, or a host-controller issue. Do not treat the current evidence as proof of one specific cause.

### Rockchip loader / MaskROM

No USB device with Rockchip VID `2207` was observed during the captured tests. In other words, the tested boots stayed in the CHERRY USB-gadget path; **MaskROM/Loader mode has not yet been entered or tested**.

Before attempting any loader/MaskROM write operation, back up the original eMMC if possible.

---

## 7. Replacement USB smart-card reader

The project also tested a small USB smart-card reader intended for installation inside the empty/repurposed ST-1506 enclosure.

Linux identifies it as:

```text
058f:9540 Alcor Micro Corp. AU9540 Smartcard Reader
```

`pcsc`/pyscard enumerates it as:

```text
Alcor Micro AU9540 00 00
```

This is a normal CCID/PC-SC style USB reader. Whether it completes the full intended eGK workflow in this project is **not yet documented after the software-selection fix below**.

### SICCT proxy issue found

The existing Python SICCT proxy was not actually selecting this reader. Its configuration still contained the previous reader:

```python
READER_INDEX = 1
EXPECTED_READER_SUBSTR = "Alcor Link AK9563"
```

With only the new reader present, it enumerated as reader `[0]`, so neither selector matched. The server therefore printed:

```text
Verfügbare Reader:
  [0] Alcor Micro AU9540 00 00
Kein passender Reader ausgewählt/gefunden. Server startet trotzdem.
```

and every SICCT `REQUEST ICC` ended in:

```text
request_icc: kein passender Reader gefunden
```

with SICCT status response `64 A1`.

The minimal configuration change is:

```python
READER_INDEX = 0
EXPECTED_READER_SUBSTR = "Alcor Micro AU9540"
```

For a generic single-reader installation, the existing selector can also be made to fall back to index 0 by using:

```python
READER_INDEX = 0
EXPECTED_READER_SUBSTR = ""
```

A more robust future change would automatically select the only available PC/SC reader if exactly one reader is present.

The current proxy connects to physical cards explicitly with:

```python
conn.connect(CardConnection.T1_protocol)
```

A future compatibility improvement may be to test automatic protocol negotiation as well, but this has not yet been necessary to explain the failure above: the failure happened **before any card connection attempt**, because no reader was selected.

---

## 8. SICCT traffic already observed

The custom proxy successfully handled parts of the SICCT session before reaching the reader-selection problem:

- `INIT CT SESSION` succeeded.
- Session IDs were returned.
- `CLOSE CT SESSION` worked.
- `OUTPUT` requests were decoded; for example the client sent display text:

```text
-- TEST --
SICCTx1
```

- `REQUEST ICC` failed only because `find_reader()` returned no matching reader.
- `EJECT ICC` reset local reader state and generated the proxy's slot-removed event.

This is useful because it separates **SICCT protocol/session handling** from the independent **PC/SC reader-selection problem**.

---

## 9. Suggested repository photo layout

A useful structure for publishing the board photographs would be:

```text
images/
  enclosure-front.jpg
  enclosure-bottom.jpg
  security-seal.jpg
  cobra-board-overview-front.jpg
  cobra-board-overview-back.jpg
  cobra-v2.2-marking.jpg
  px30-closeup.jpg
  display-p10-connector.jpg
  display-fpc-front.jpg
  display-fpc-back.jpg
  gt911-touch-controller.jpg
```

For reverse-engineering work, retain the original full-resolution images as well as web-sized copies. Macro shots should include a ruler or known-pitch connector where possible.

---

## 10. Recommended next steps

1. **Find UART5** on the Cobra V2.2 board and capture a completely passive 115200-8N1 boot log.
2. **Back up the eMMC** before modifying any bootloader, partition table or filesystem.
3. Determine the **exact LCD module variant** from a physical label.
4. Confirm the **P10 pitch and connector part/family**; it visually appears consistent with a fine-pitch FFC, but the pitch has not yet been measured/documented.
5. Compare measured P10 continuity/voltages with the candidate 31-pin Leadtek pinout before attaching an external display host.
6. Test the **GT911** directly under mainline Linux using the upstream Cobra DTS.
7. Investigate whether current mainline `px30-cobra-*.dtb` plus a suitable U-Boot can boot from the existing board without touching the original eMMC initially.
8. Investigate Rockchip **Loader/MaskROM** only after a backup strategy is in place.
9. Re-test the **AU9540** after fixing the SICCT reader selector and document ATR/APDU behavior with an eGK.
10. If the original board proves inconvenient, use the documented MIPI-DSI + GT911 interfaces to drive the original front assembly from another Linux SBC.

---

## 11. Confidence / open questions

| Finding | Confidence |
|---|---|
| Board is marked Cobra V2.2 | **Confirmed on hardware** |
| Main SoC is Rockchip PX30 | **Confirmed on hardware** |
| Mainline Linux contains a PX30 Cobra platform | **Confirmed upstream** |
| UART console is configured as UART5 / 115200n8 in upstream DTS | **Confirmed upstream** |
| Touch controller is Goodix GT911 | **Confirmed on hardware and upstream** |
| Cobra GT911 address is 0x14 | **Confirmed upstream** |
| P10 has 31 contacts | **Confirmed on hardware** |
| Display uses MIPI DSI | **Strongly supported by upstream Cobra variants and routing** |
| Exact display model in this ST-1506 | **Not yet confirmed** |
| Candidate 31-pin Leadtek pinout applies unchanged to this exact module | **Plausible, must be verified** |
| USB-C can enumerate the original ST-1506 as 046a:0084 | **Confirmed experimentally** |
| USB gadget is RNDIS | **Confirmed experimentally** |
| Current RNDIS data path works correctly | **No — TX timeout / no ARP response observed** |
| Cause of RNDIS timeout | **Unknown** |
| Device entered Rockchip MaskROM/Loader | **Not observed** |
| AU9540 is selected by the original SICCT config | **No; config mismatch identified** |
| AU9540 completes eGK read after config fix | **Not yet documented** |

---

## 12. Public references

### CHERRY / ST-1506

- Product page:  
  https://www.cherry.de/de-de/produkt/ehealth-terminal-st-1506
- Downloads/manuals/USB-LAN proxy:  
  https://www.cherry.de/de-de/produkt/ehealth-terminal-st-1506/downloads
- Common Criteria certification report:  
  https://www.commoncriteriaportal.org/files/epfiles/1124V3a_pdf.pdf
- Security Target:  
  https://www.commoncriteriaportal.org/files/epfiles/1124V3b_pdf.pdf
- IGEL RNDIS integration notes:  
  https://kb.igel.com/en/igel-os/11.20.430/using-cherry-ehealth-card-terminal-st-1506-in-usb-

### Linux / Cobra

- `px30-cobra.dtsi`:  
  https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/rockchip/px30-cobra.dtsi
- Cobra DTB entries:  
  https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/rockchip/Makefile
- Original Cobra DT patch discussion/description:  
  https://lists.infradead.org/pipermail/linux-arm-kernel/2025-May/1026461.html

### Display / touch

- Leadtek panel driver:  
  https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/panel/panel-leadtek-ltk050h3146w.c
- Leadtek LTK050H3146W drawing / pinout mirror:  
  https://www.texim-europe.com/getfile.ashx?id=138898
- Goodix GT9xx Device Tree binding:  
  https://github.com/torvalds/linux/blob/master/Documentation/devicetree/bindings/input/touchscreen/goodix.yaml

---

## 13. Contributions wanted

Useful contributions would include:

- annotated Cobra V2.2 board photographs,
- UART5 test-pad identification,
- complete boot logs,
- eMMC partition maps / read-only backups where legally distributable,
- exact ST-1506 display-module identification,
- P10 connector part number and verified pinout,
- mainline Linux/U-Boot boot instructions,
- GT911 test results,
- Rockchip Loader/MaskROM observations,
- AU9540/eGK PC/SC compatibility results,
- a small adapter PCB for reusing the display/touch assembly.

Please clearly label measurements as **measured**, **derived from upstream DTS**, or **assumed from a similar panel**. That distinction is especially important around the display power rails and MIPI connector.

---

## License / provenance note

This README contains original observations from the investigated unit plus references to publicly available vendor documentation and upstream Linux sources. Before publishing photographs, firmware dumps or copied source material, choose appropriate repository licensing and verify that redistribution is permitted. Linking to upstream/vendor material is preferable to copying proprietary documents into the repository.
