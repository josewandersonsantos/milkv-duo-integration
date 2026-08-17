# Milk-V Duo CV1800B — Linux ↔ FreeRTOS IPC Demo

Experimental embedded Linux project for the **Milk-V Duo (CV1800B)** demonstrating communication between the Linux and FreeRTOS cores using the CVITEK mailbox mechanism.

The initial goal is intentionally simple:

* Run Linux on the main C906 core.
* Run FreeRTOS on the secondary C906 core.
* Send commands from Linux to FreeRTOS.
* Control the onboard LED from FreeRTOS.
* Exchange acknowledgements between the two cores.
* Print diagnostic messages through the serial console.
* Build a reproducible SD-card image.

The project is intended as a starting point for more complex heterogeneous Linux/RTOS applications.

---

## Hardware

### Milk-V Duo

The original Milk-V Duo is based on the **CVITEK CV1800B** SoC.

| Core | Frequency | Software |
| ---- | --------: | -------- |
| C906 |    ~1 GHz | Linux    |
| C906 |  ~700 MHz | FreeRTOS |

The Linux core is responsible for high-level applications, while the secondary core runs the real-time firmware.

---

## Architecture

```text
                         Milk-V Duo
                    CVITEK CV1800B
┌───────────────────────────────────────────────────┐
│                                                   │
│                    C906 @ 1 GHz                   │
│                      Linux                        │
│                                                   │
│               ┌─────────────────┐                 │
│               │   ipc-demo      │                 │
│               │                 │                 │
│               │ LED_ON          │                 │
│               │ LED_OFF         │                 │
│               │ LED_TOGGLE      │                 │
│               │ PING            │                 │
│               └────────┬────────┘                 │
│                        │                          │
│                        │ rtos_cmdqu / mailbox     │
│                        ▼                          │
│               ┌─────────────────┐                 │
│               │ Linux mailbox   │                 │
│               │ driver          │                 │
│               └────────┬────────┘                 │
│                        │                          │
│════════════════════════╪══════════════════════════│
│                        │                          │
│                        ▼                          │
│               ┌─────────────────┐                 │
│               │ FreeRTOS        │                 │
│               │ communication   │                 │
│               │ task            │                 │
│               └────────┬────────┘                 │
│                        │                          │
│              ┌─────────┴─────────┐                │
│              │                   │                │
│              ▼                   ▼                │
│             LED                UART               │
│                                                   │
│                    C906 @ 700 MHz                 │
│                     FreeRTOS                      │
│                                                   │
└───────────────────────────────────────────────────┘
```

---

## Inter-Core Communication

The CV1800B uses a mailbox-based communication mechanism between the Linux and FreeRTOS cores.

This project intentionally uses the platform's native mailbox mechanism instead of introducing RPMsg/OpenAMP at this stage.

The Linux-side mailbox driver is located in the Linux kernel source tree:

```text
linux_5.10/drivers/soc/cvitek/rtos_cmdqu/
```

The corresponding FreeRTOS implementation is located under:

```text
freertos/cvitek/driver/rtos_cmdqu/
freertos/cvitek/task/comm/src/riscv64/
```

The communication flow is:

```text
Linux application
       │
       │ command
       ▼
Linux mailbox driver
       │
       │ mailbox
       ▼
FreeRTOS communication task
       │
       ├── LED control
       │
       └── response
       │
       ▼
Linux mailbox driver
       │
       ▼
Linux application
```

---

## Initial Protocol

The first version of the protocol contains only a few commands.

```text
Command             Value
--------------------------------
LED_ON              0x01
LED_OFF             0x02
LED_TOGGLE          0x03
PING                0x04
```

Responses:

```text
Status              Value
--------------------------------
OK                  0x00
ERROR               0x01
INVALID_COMMAND     0x02
```

The protocol is deliberately small. The goal of this project is to validate the communication mechanism before introducing a more complex application protocol.

---

## Example

The Linux application sends:

```text
LED_ON
```

The FreeRTOS core receives the command:

```text
RT: command received: LED_ON
RT: LED -> ON
RT: response -> OK
```

Linux receives:

```text
IPC: TX LED_ON
IPC: RX OK
```

The same process is used for `LED_OFF`, `LED_TOGGLE`, and `PING`.

---

## Repository Structure

```text
milkv-duo-ipc/
│
├── README.md
├── LICENSE
├── .gitignore
├── Makefile
│
├── buildroot/
├── linux_5.10/
├── freertos/
├── u-boot-2021.10/
├── opensbi/
├── fsbl/
├── device/
│
├── app/
│   │
│   ├── linux/
│   │   └── ipc-demo/
│   │       ├── main.c
│   │       ├── mailbox.c
│   │       └── mailbox.h
│   │
│   └── freertos/
│       ├── include/
│       │   ├── app.h
│       │   ├── ipc.h
│       │   ├── led.h
│       │   └── serial.h
│       │
│       └── src/
│           ├── app.c
│           ├── ipc.c
│           ├── led.c
│           └── serial.c
│
├── common/
│   └── ipc_protocol.h
│
├── build/
│   ├── build.sh
│   └── package.sh
│
└── scripts/
    ├── flash.sh
    └── serial.sh
```

---

## Directory Responsibilities

### `buildroot/`

Buildroot source tree.

It is responsible for generating the Linux userspace, toolchain, root filesystem and Linux-side application packages.

---

### `linux_5.10/`

Linux kernel source used by the Milk-V Duo platform.

The mailbox driver used for communication with the FreeRTOS core is located here.

---

### `freertos/`

FreeRTOS source and platform integration.

The application-specific FreeRTOS code should remain as small and isolated as possible.

---

### `app/linux/`

Linux applications running on the main C906 core.

Example:

```text
app/linux/ipc-demo/
```

This application sends commands to the FreeRTOS core.

---

### `app/freertos/`

Application code running on the secondary C906 core.

Responsibilities include:

* Receiving IPC commands.
* Controlling the LED.
* Sending responses.
* Printing diagnostic information.
* Running real-time tasks.

---

### `common/`

Code shared between Linux and FreeRTOS.

The IPC protocol should be defined here.

Example:

```c
typedef enum
{
    IPC_CMD_LED_ON = 0x01,
    IPC_CMD_LED_OFF = 0x02,
    IPC_CMD_LED_TOGGLE = 0x03,
    IPC_CMD_PING = 0x04,
} ipc_command_t;
```

Keeping the protocol definition in one place prevents Linux and FreeRTOS from developing incompatible message definitions.

---

## Boot Flow

The intended boot flow is:

```text
Power On
   │
   ▼
FSBL
   │
   ▼
OpenSBI / Bootloader
   │
   ├───────────────┐
   │               │
   ▼               ▼
Linux           FreeRTOS
C906 @ 1 GHz    C906 @ 700 MHz
   │               │
   │               │
   └───────┬───────┘
           │
           ▼
       IPC ready
```

The exact firmware loading and startup mechanism is controlled by the Milk-V Duo SDK and boot chain.

The project should therefore avoid treating the FreeRTOS firmware as an ordinary Linux executable.

---

## Build Environment

The project is based on the official Milk-V Duo Buildroot SDK.

The official SDK contains the major platform components:

```text
buildroot
linux_5.10
freertos
u-boot
opensbi
fsbl
device
```

The repository should pin a known SDK revision to make builds reproducible.

---

## Building

The first development stage should use the official SDK build system.

Typical configuration:

```bash
source device/milkv-duo/boardconfig.sh
source build/envsetup_milkv.sh
defconfig cv1800b_milkv_duo_sd
```

Build the complete firmware:

```bash
build_all
```

Generate the SD-card image:

```bash
pack_sd_image
```

The generated image is produced by the SDK under the corresponding `install` output directory.

---

## Development Strategy

The project is intentionally developed in stages.

### Stage 1 — Linux boot

Verify that the board boots Linux correctly.

```text
Boot
 │
 ▼
Linux
 │
 ▼
Serial console
```

---

### Stage 2 — FreeRTOS boot

Verify that the secondary core starts FreeRTOS.

```text
Linux
  │
  │
  └──────────────► FreeRTOS
                       │
                       ▼
                     UART
```

The FreeRTOS serial output should be visible through the board's serial console.

---

### Stage 3 — LED control

Run a minimal FreeRTOS application that controls the onboard LED.

The default Linux LED blinking script must be disabled during this test so that Linux and FreeRTOS do not simultaneously control the same LED.

---

### Stage 4 — Linux → FreeRTOS IPC

Send commands from Linux:

```text
ipc-demo led on
ipc-demo led off
ipc-demo led toggle
ipc-demo ping
```

Expected flow:

```text
Linux                  FreeRTOS
  │                       │
  │ LED_ON                │
  ├──────────────────────►│
  │                       │
  │                       ├── LED ON
  │                       │
  │ OK                    │
  │◄──────────────────────┤
```

---

### Stage 5 — Bidirectional communication

Add structured responses:

```text
Linux
 │
 │ command
 ▼
FreeRTOS
 │
 │ status/data
 ▼
Linux
```

At this point the IPC layer becomes reusable by the rest of the application.

---

## Future Protocol

Once the basic communication works, the protocol can evolve into something like:

```c
typedef struct
{
    uint32_t command;
    uint32_t sequence;
    uint32_t length;
    uint8_t payload[64];
} ipc_message_t;
```

Possible future commands:

```text
GET_STATUS
GET_VERSION
LED_SET
GPIO_SET
SENSOR_READ
SENSOR_CONFIG
UART_SEND
TIME_SYNC
RESET
PING
```

The final application architecture could become:

```text
                       Linux
                         │
                  High-level app
                         │
                    IPC service
                         │
                         ▼
                  ┌─────────────┐
                  │   Mailbox   │
                  └──────┬──────┘
                         │
                         ▼
                      FreeRTOS
                         │
             ┌───────────┼───────────┐
             │           │           │
             ▼           ▼           ▼
          Sensors     Control      UART
             │           │           │
             └───────────┴───────────┘
```

---

## Goals

The project is intended to eventually provide:

* [ ] Reproducible Milk-V Duo SD-card image.
* [ ] Linux application for IPC.
* [ ] FreeRTOS application for IPC.
* [ ] Linux ↔ FreeRTOS mailbox communication.
* [ ] LED control from FreeRTOS.
* [ ] Serial diagnostics.
* [ ] Bidirectional messages.
* [ ] Structured IPC protocol.
* [ ] Error handling.
* [ ] IPC timeout handling.
* [ ] FreeRTOS watchdog.
* [ ] Linux-side monitoring of the RTOS core.

---

## Why Mailbox?

The CV1800B platform already provides a mailbox-based mechanism for communication between the Linux and real-time cores.

Using the native mechanism first has several advantages:

1. It matches the platform's official architecture.
2. It avoids introducing an additional IPC framework.
3. It keeps the first prototype small.
4. It allows the communication mechanism to be validated before designing a larger protocol.

RPMsg/OpenAMP can be evaluated later if a more generic message-oriented IPC layer becomes necessary.

---

## References

* Milk-V Duo official documentation
* Milk-V Duo Buildroot SDK
* CV1800B Linux kernel
* CVITEK RTOS command queue / mailbox implementation

This project is for experimentation and embedded systems development on the Milk-V Duo CV1800B.

## Cloning

git clone --recursive <https://github.com/josewandersonsantos/milkv-duo-integration.git>
