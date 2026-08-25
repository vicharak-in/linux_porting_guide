# Linux Kernel Porting Guide

## 1. Theory & Operation of Each Component

A complete MIPI DSI display stack relies on five essential hardware signals and sub-systems to bring up video, power, backlight, and touch.

---

### Hardware Signal & Interface Breakdown

| Subsystem / Line | Hardware Pins Involved | Function & Role |
| --- | --- | --- |
| **I2C Bus** | `SDA`, `SCL` | Bidirectional control channel. Used for reading capacitive touch coordinates (`0x41`) and sending power/backlight commands to the onboard microcontroller (`0x45`). |
| **Reset Pin (RST)** | `TP_RST_L` / `LCD_RST` | Active-low hardware reset. Puts the touch and display controllers into a known initial state before communication begins. |
| **Interrupt Pin (INT / IRQ)** | `TP_INT_L` / `IRQ` | Active-low edge/level trigger. The touch IC pulls this low to alert the host SoC that a finger event (touch down, motion, release) has occurred. |
| **Power Control (EN / VDD)** | `VCC_3V3`, `LCD_BL_EN` | Supplies power rails (analog, digital, and I/O) and manages the hardware power sequencing (VDD $\rightarrow$ Reset release $\rightarrow$ MIPI DSI stream). |
| **MIPI DSI Differential Pairs** | `CLK_P/N`, `D0_P/N`, `D1_P/N` | High-speed low-voltage differential lines for sending display configuration (LP mode) and pixel streaming (HS mode). |

---

### Operation of Each Component

**I2C Control (Inter-Integrated Circuit)**

* **Touch Controller:** Instead of streaming high-speed data, touch coordinates are stored in registers on the touch IC (Ilitek). The SoC reads $X, Y$ coordinate buffers, pressure, and multitouch points over standard I2C at speeds between 100 kHz and 400 kHz.
* **Bridge / Power Management:** On panels with built-in microcontrollers (like Raspberry Pi Touch Display 2), the I2C bus is also used to send PWM backlight brightness values and trigger internal power sequencing.

**Hardware Reset Pin (`RST`)**

* Silicon controllers inside display panels and touch digitizers must start from a clean power state to prevent register latch-ups or corrupted firmware boot states.
* **Sequence:** The kernel driver asserts the reset line (pulls `RST` **LOW**), waits for a specific duration ($10\text{ ms} - 50\text{ ms}$ defined by the IC datasheet), then releases it (**HIGH**). Only after this delay will the IC acknowledge I2C transactions or accept DSI packets.

**Interrupt Line (`INT` / `IRQ`)**

* Polling a touch controller continuously over I2C wastes CPU cycles and bus bandwidth.
* **Event Loop:**
1. The touch IC remains idle until a capacitive change is detected on the glass.
2. The touch IC pulls the `INT` line **LOW**.
3. The SoC kernel catches the falling edge or low level via GPIO interrupt.
4. The SoC initiates an I2C transaction to read the touch coordinate registers, then clears the interrupt.



**MIPI DSI Video Pipeline (`CLK` + `Data Lanes`)**

* **Initialization (LP Mode):** When the panel powers up, the DSI host sends short packet Display Command Set (DCS) commands over `D0` at low speed to configure internal timing generators, pixel formats, and sleep-out commands.
* **Continuous Streaming (HS Mode):** Once configured, the DSI clock switches to high-speed mode, and active RGB pixels stream synchronously across the data lanes into the display panel's driver IC.

## 2. [Setup](https://docs.vicharak.in/vicharak_sbcs/axon/display/mipi-dsi/) 

**Hardware Connection** : 

Axon's MIPI TX0 Connector <-> MIPI DSI Adapter ( converts 30 Pin to 15 Pin (raspberry pi compatible)) <-> DSI Display

**Pre-requisites** :

- MIPI DSI Display
- Configure Kernel and make overlays according to MIPI-DSI Display
- Vicharak PCB ( Adapter 30 Pin to 15 Pin converter ) For DSI Display
- Vicharak Flex Cable 30 Pin 0.4mm Pitch Cable (Golden Color)

<img width="452" height="633" alt="image" src="https://github.com/user-attachments/assets/a124c0e9-473f-4694-93f1-24f2449933d4" />


## 3. Axon MIPI TX Connector Pinout

**MIPI_DPHY0_TX**

| Package Pin# | Function | Pin | Pin | Function | Package Pin# |
| --- | --- | --- | --- | --- | --- |
| — | GND | **1** | **2** | GND | — |
| — | — | **3** | **4** | MIPI_DPHY0_TX_CLK_N | — |
| — | — | **5** | **6** | MIPI_DPHY0_TX_CLK_P | — |
| — | GND | **7** | **8** | GND | — |
| — | MIPI_DPHY0_TX_D3_N | **9** | **10** | MIPI_DPHY0_TX_D1_N | — |
| — | MIPI_DPHY0_TX_D3_P | **11** | **12** | MIPI_DPHY0_TX_D1_P | — |
| — | GND | **13** | **14** | GND | — |
| — | MIPI_DPHY0_TX_D2_N | **15** | **16** | MIPI_DPHY0_TX_D0_N | — |
| — | MIPI_DPHY0_TX_D2_P | **17** | **18** | MIPI_DPHY0_TX_D0_P | — |
| — | GND | **19** | **20** | GND | — |
| A24(GPIO1_A0) | I2C2_SDA_M4# | **21** | **22** | VCC_3V3_MIPI_TX | — |
| A25(GPIO1_A1) | I2C2_SCL_M4# | **23** | **24** | VCC_3V3_MIPI_TX | — |
| Y7(GPIO3_C1) | TP_RST_L_0 | **25** | **26** | VCC_3V3_MIPI_TX | — |
| Y29(GPIO3_C0) | TP_INT_L_0 | **27** | **28** | VCC_3V3_MIPI_TX | — |
| AE31(GPIO4_C2) | LCD_BL_PWM1 | **29** | **30** | LCD_BL_EN_0 | AD30(GPIO2_C3) |

---

**MIPI_DPHY1_TX**

| Package Pin# | Function | Pin | Pin | Function | Package Pin# |
| --- | --- | --- | --- | --- | --- |
| — | GND | **1** | **2** | GND | — |
| — | — | **3** | **4** | MIPI_DPHY1_TX_CLK_N | — |
| — | — | **5** | **6** | MIPI_DPHY1_TX_CLK_P | — |
| — | GND | **7** | **8** | GND | — |
| — | MIPI_DPHY1_TX_D3_N | **9** | **10** | MIPI_DPHY1_TX_D1_N | — |
| — | MIPI_DPHY1_TX_D3_P | **11** | **12** | MIPI_DPHY1_TX_D1_P | — |
| — | GND | **13** | **14** | GND | — |
| — | MIPI_DPHY1_TX_D2_N | **15** | **16** | MIPI_DPHY1_TX_D0_N | — |
| — | MIPI_DPHY1_TX_D2_P | **17** | **18** | MIPI_DPHY1_TX_D0_P | — |
| — | GND | **19** | **20** | GND | — |
| AH24(GPIO3_D0) | I2C5_SDA_M0 | **21** | **22** | VCC_3V3_MIPI_TX | — |
| AJ24(GPIO3_C7) | I2C5_SCL_M0 | **23** | **24** | VCC_3V3_MIPI_TX | — |
| AE28(GPIO3_B2) | TP_RST_L_1 | **25** | **26** | VCC_3V3_MIPI_TX | — |
| AB28(GPIO3_D5) | TP_INT_L_1 | **27** | **28** | VCC_3V3_MIPI_TX | — |
| AF34(GPIO3_C3) | LCD_BL_PWM2 | **29** | **30** | LCD_BL_EN_1 | AE30(GPIO2_C5) |

**MIPI DSI Vicharak ADAPTER Pinout**

- Will Update soon 

## Porting guide and architecture breakdown to bring up the Raspberry Pi 10.1" Touch Display (ILI79600A) on the Vicharak Axon RK3588 board.

---

### 1. Understanding `rpi-panel-v2-regulator.c`

The Raspberry Pi Touch Display 2 uses an onboard Attiny/Microcontroller bridge on the display PCB that handles power rail sequencing and PWM backlight control over I2C (at address `0x45`).

Instead of routing separate GPIO/PWM lines from the host SoC:

* The host sends I2C commands to address `0x45` to enable power rails (`regulator-rpi-panel-v2`).
* The same microcontroller handles the backlight brightness via the regulator driver's backlight sub-device.
* The ILI79600 DSI panel driver references this regulator node (`power-supply = <&regulator_node>`) to sequence display power-on before issuing DSI commands.

**Axon Compatibility:** You **must port** `rpi-panel-v2-regulator.c`. It is SoC-agnostic and only requires standard Linux I2C and regulator subsystems.

---

### 2. Driver Porting Steps ([Kernel 6.1](https://github.com/vicharak-in/rockchip-linux-kernel/tree/6.1/))

#### A. Copy Driver Source Files

Copy the three drivers from the Raspberry Pi tree into your `vicharak-linux-kernel` [repository](https://github.com/vicharak-in/rockchip-linux-kernel/tree/6.1/):

1. **Panel Driver:**
- [Source](https://github.com/raspberrypi/linux/blob/16f1da3c4e94437449d6aa151589ca0ad4b388bb/drivers/gpu/drm/panel/panel-ilitek-ili79600a.c#L450): `drivers/gpu/drm/panel/panel-ilitek-ili79600a.c`
- Destination: `drivers/gpu/drm/panel/panel-ilitek-ili79600a.c`

2. **Power/Backlight Driver:**
- [Source](https://github.com/raspberrypi/linux/blob/rpi-6.18.y/drivers/regulator/rpi-panel-v2-regulator.c): `drivers/regulator/rpi-panel-v2-regulator.c`
- Destination: `drivers/regulator/rpi-panel-v2-regulator.c`

3. **Touch Driver:**
- [Source](https://github.com/raspberrypi/linux/blob/rpi-6.18.y/drivers/input/touchscreen/ilitek_v3_ts_i2c.c): `drivers/input/touchscreen/ilitek_v3_ts_i2c.c`
- Destination: `drivers/input/touchscreen/ilitek_v3_ts_i2c.c`

#### B. Update Kconfig & Makefiles

* **`drivers/gpu/drm/panel/Kconfig`**:
```kconfig
config DRM_PANEL_ILITEK_ILI79600A
    tristate "Ilitek ILI79600A based DSI panels"
    depends on OF
    depends on DRM_MIPI_DSI
    depends on BACKLIGHT_CLASS_DEVICE
    help
      Say Y here if you want to enable support for Ilitek ILI79600A DSI panels.

```


* **`drivers/gpu/drm/panel/Makefile`**:
```makefile
obj-$(CONFIG_DRM_PANEL_ILITEK_ILI79600A) += panel-ilitek-ili79600a.o

```


* **`drivers/regulator/Kconfig`**:
```kconfig
config REGULATOR_RPI_PANEL_V2
    tristate "Raspberry Pi 10.1inch / 7inch V2 panel regulator"
    depends on I2C
    select REGMAP_I2C
    help
      Regulator and backlight driver for Raspberry Pi Touch Panel V2.

```


* **`drivers/regulator/Makefile`**:
```makefile
obj-$(CONFIG_REGULATOR_RPI_PANEL_V2) += rpi-panel-v2-regulator.o

```


* **`drivers/input/touchscreen/Makefile`**:
```makefile
obj-$(CONFIG_TOUCHSCREEN_ILITEK_V3) += ilitek_v3_ts_i2c.o

```

---

### 3. Pin Mapping Analysis

From your adapter schematic and 30-pin definitions:

| Signal | MIPI DPHY0 TX (Port 0) | MIPI DPHY1 TX (Port 1) | 15-Pin Adapter J4 Pin | Function on Display |
| --- | --- | --- | --- | --- |
| **DSI Data 0** | `MIPI_DPHY0_TX_D0` | `MIPI_DPHY1_TX_D0` | Pins 8, 9 (`DN0`, `DP0`) | DSI Lane 0 |
| **DSI Clock** | `MIPI_DPHY0_TX_CLK` | `MIPI_DPHY1_TX_CLK` | Pins 5, 6 (`CN`, `CP`) | DSI Clock |
| **DSI Data 1** | `MIPI_DPHY0_TX_D1` | `MIPI_DPHY1_TX_D1` | Pins 2, 3 (`DN1`, `DP1`) | DSI Lane 1 |
| **I2C Bus** | `I2C2` (`GPIO1_A0`/`GPIO1_A1`) | `I2C5` (`GPIO3_D0`/`GPIO3_C7`) | Pins 11, 12 (`SCL0`, `SDA0`) | Touch & Attiny Power IC |
| **Touch IRQ** | `GPIO3_C0` (Pin 27) | `GPIO3_D5` (Pin 27) | Routed via I2C/Bridge | Touch Interrupt |
| **Touch Reset** | `GPIO3_C1` (Pin 25) | `GPIO3_B2` (Pin 25) | Routed via I2C/Bridge | Touch Reset |
| **Power 3.3V** | `VCC_3V3_MIPI_TX` | `VCC_3V3_MIPI_TX` | Pin 15 | Power Input |

---

### 4. Device Tree Overlays

Save these files in `arch/arm64/boot/dts/rockchip/overlays/`.

#### Overlay 1: `rk3588-axon-dsi0-ili79600-10inch.dts` (For DPHY0 TX)

```dts
/dts-v1/;
/plugin/;

#include <dt-bindings/gpio/gpio.h>
#include <dt-bindings/interrupt-controller/irq.h>

/ {
    metadata {
        title = "Raspberry Pi 10.1 inch DSI Touch Display on MIPI TX0";
        compatible = "vicharak,axon";
        category = "display";
    };

    fragment@0 {
        target = <&i2c2>;
        __overlay__ {
            status = "okay";
            #address-cells = <1>;
            #size-cells = <0>;

            panel_reg: panel_reg@45 {
                compatible = "raspberrypi,v2-regulator";
                reg = <0x45>;
            };

            touchscreen@41 {
                compatible = "ilitek,ili2117", "ilitek,ili2130", "ilitek,ili2132";
                reg = <0x41>;
                interrupt-parent = <&gpio3>;
                interrupts = <0 IRQ_TYPE_LEVEL_LOW>; /* GPIO3_C0 = pin 0 */
                reset-gpios = <&gpio3 1 GPIO_ACTIVE_LOW>; /* GPIO3_C1 = pin 1 */
                touchscreen-size-x = <1200>;
                touchscreen-size-y = <1920>;
                status = "okay";
            };
        };
    };

    fragment@1 {
        target = <&dsi0>;
        __overlay__ {
            status = "okay";
            #address-cells = <1>;
            #size-cells = <0>;

            panel@0 {
                compatible = "raspberrypi,dsi-10-1inch", "ilitek,ili79600a";
                reg = <0>;
                power-supply = <&panel_reg>;
                backlight = <&panel_reg>;
                dsi-lanes = <2>;
                video-mode = <2>;

                port {
                    panel_in_dsi0: endpoint {
                        remote-endpoint = <&dsi0_out_panel>;
                    };
                };
            };

            ports {
                #address-cells = <1>;
                #size-cells = <0>;
                port@1 {
                    reg = <1>;
                    dsi0_out_panel: endpoint {
                        remote-endpoint = <&panel_in_dsi0>;
                    };
                };
            };
        };
    };

    fragment@2 {
        target = <&dsi0_dphy>;
        __overlay__ {
            status = "okay";
        };
    };
};

```

---

#### Overlay 2: `rk3588-axon-dsi1-ili79600-10inch.dts` (For DPHY1 TX)

```dts
/dts-v1/;
/plugin/;

#include <dt-bindings/gpio/gpio.h>
#include <dt-bindings/interrupt-controller/irq.h>

/ {
    metadata {
        title = "Raspberry Pi 10.1 inch DSI Touch Display on MIPI TX1";
        compatible = "vicharak,axon";
        category = "display";
    };

    fragment@0 {
        target = <&i2c5>;
        __overlay__ {
            status = "okay";
            #address-cells = <1>;
            #size-cells = <0>;

            panel_reg_1: panel_reg@45 {
                compatible = "raspberrypi,v2-regulator";
                reg = <0x45>;
            };

            touchscreen@41 {
                compatible = "ilitek,ili2117", "ilitek,ili2130", "ilitek,ili2132";
                reg = <0x41>;
                interrupt-parent = <&gpio3>;
                interrupts = <5 IRQ_TYPE_LEVEL_LOW>; /* GPIO3_D5 = pin 5 */
                reset-gpios = <&gpio3 2 GPIO_ACTIVE_LOW>; /* GPIO3_B2 = pin 2 */
                touchscreen-size-x = <1200>;
                touchscreen-size-y = <1920>;
                status = "okay";
            };
        };
    };

    fragment@1 {
        target = <&dsi1>;
        __overlay__ {
            status = "okay";
            #address-cells = <1>;
            #size-cells = <0>;

            panel@0 {
                compatible = "raspberrypi,dsi-10-1inch", "ilitek,ili79600a";
                reg = <0>;
                power-supply = <&panel_reg_1>;
                backlight = <&panel_reg_1>;
                dsi-lanes = <2>;
                video-mode = <2>;

                port {
                    panel_in_dsi1: endpoint {
                        remote-endpoint = <&dsi1_out_panel>;
                    };
                };
            };

            ports {
                #address-cells = <1>;
                #size-cells = <0>;
                port@1 {
                    reg = <1>;
                    dsi1_out_panel: endpoint {
                        remote-endpoint = <&panel_in_dsi1>;
                    };
                };
            };
        };
    };

    fragment@2 {
        target = <&dsi1_dphy>;
        __overlay__ {
            status = "okay";
        };
    };
};

```

---

### 5. Build and Verification

* Build the kernel and device tree overlays:
```bash
make ARCH=arm64 rockchip_defconfig
make ARCH=arm64 dtbs

```


* Test I2C detection on boot (ensure `0x41` and `0x45` show as `UU` or detect properly):
```bash
# For TX0 (I2C2)
i2cdetect -y 2

# For TX1 (I2C5)
i2cdetect -y 5

```


* Verify DRM connector initialization via `dmesg | grep -i ili79600` and `modetest -M rockchip`.
