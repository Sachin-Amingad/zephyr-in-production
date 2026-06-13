# Agent Notes

This repository is a Zephyr in Production workshop project. It starts as a
simple blinky application and now contains the first production-oriented pieces:
a custom board definition, devicetree overlays, and sysbuild configuration for
MCUboot.

## Project Layout

```text
.
├── app/
│   ├── CMakeLists.txt
│   ├── prj.conf
│   ├── src/main.c
│   ├── boards/
│   │   ├── frdm_mcxa156.overlay
│   │   └── training.overlay
│   ├── sysbuild.conf
│   └── sysbuild/mcuboot.conf
├── boards/training/
│   ├── board.c
│   ├── board.cmake
│   ├── board.yml
│   ├── Kconfig
│   ├── Kconfig.training
│   ├── production.overlay
│   ├── training.dts
│   ├── training-pinctrl.dtsi
│   ├── training.yaml
│   └── training_defconfig
├── zephyr/module.yml
├── west.yml
├── README.md
└── TROUBLESHOOTING.md
```

## West Manifest

`west.yml` pins the project to Zephyr `v4.4.0` and imports only the dependencies
needed for this workshop:

- `cmsis_6`
- `hal_nxp`
- `mcuboot`
- `mbedtls`
- `tf-psa-crypto`
- `zcbor`

The dependencies are checked out under `deps/` because the manifest uses:

```yaml
path-prefix: deps
```

That keeps the workspace smaller than a full Zephyr checkout.

## Zephyr Module Registration

`zephyr/module.yml` tells Zephyr that this repository contributes extra build
content:

```yaml
build:
  settings:
    snippet_root: .
    board_root: .
```

The important part for the current code is `board_root: .`. This lets Zephyr
discover the custom board under `boards/training` when building from this repo.

Without this, `west build -b training app` would not know that `training` is a
valid board.

## Application

The application lives in `app/`.

### `app/CMakeLists.txt`

This is the minimal Zephyr application CMake file:

```cmake
cmake_minimum_required(VERSION 3.20.0)

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(zephyr_in_prod)

target_sources(app PRIVATE src/main.c)
```

It finds Zephyr, declares the project, and adds `app/src/main.c` to the firmware
image.

### `app/prj.conf`

The app enables the features used by `main.c` and by the workshop:

```conf
CONFIG_GPIO=y
CONFIG_LOG=y

CONFIG_SHELL=y
CONFIG_ASSERT=y
```

- `CONFIG_GPIO=y` enables the GPIO driver API used to control the LED.
- `CONFIG_LOG=y` enables Zephyr logging, used by `LOG_INF()`.
- `CONFIG_SHELL=y` enables Zephyr shell support.
- `CONFIG_ASSERT=y` enables runtime assertions.

For a release build, shell/log/assert settings are often reduced or disabled to
save flash and RAM.

### `app/src/main.c`

The app is a configurable LED blinky:

```c
#define LED_NODE DT_ALIAS(led0)
#define ZEPHYR_USER_NODE DT_PATH(zephyr_user)
#define BLINK_PERIOD_MS DT_PROP_OR(ZEPHYR_USER_NODE, blink_period_ms, 1000)
```

These lines are the key production-training idea:

- `LED_NODE` reads the devicetree alias named `led0`.
- `ZEPHYR_USER_NODE` points at `/zephyr,user`.
- `BLINK_PERIOD_MS` reads the `blink-period-ms` devicetree property.
- If `blink-period-ms` is missing, it falls back to `1000`.

The LED is then converted into a Zephyr GPIO descriptor:

```c
static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(LED_NODE, gpios);
```

At runtime, `main()`:

1. Checks whether the LED GPIO device is ready.
2. Configures the LED pin as an active output.
3. Toggles the LED forever.
4. Logs the logical LED state.
5. Sleeps for `BLINK_PERIOD_MS`.

The important design point is that the C code does not hard-code the LED pin or
blink period. Those are controlled by devicetree and overlays.

## Application Overlays

There are two application-level overlays:

- `app/boards/training.overlay`
- `app/boards/frdm_mcxa156.overlay`

Both currently contain the same content:

```dts
/ {
	aliases {
		led0 = &green_led;
	};

	zephyr,user {
		blink-period-ms = <2000>;
	};
};
```

This changes the app from using the board default `led0` to using the green LED,
and it changes the blink period to 2000 ms.

Because `main.c` reads `DT_ALIAS(led0)` and `blink-period-ms`, no C code change
is needed to move from one LED or blink rate to another.

## Custom Board: `training`

The custom board is under `boards/training`. It is based on the NXP
FRDM-MCXA156-style board definition, but renamed as a workshop product board
called `training`.

### `boards/training/board.yml`

This registers the board metadata:

```yaml
board:
  name: training
  full_name: Training
  vendor: nxp
  socs:
  - name: mcxa156
```

This is why the board can be built with:

```bash
west build -b training app
```

### `boards/training/Kconfig.training`

This selects the SoC and part number:

```conf
config BOARD_TRAINING
	select SOC_MCXA156 if BOARD_TRAINING
	select SOC_PART_NUMBER_MCXA156VLL
```

The board is therefore treated as an MCXA156-based target.

### `boards/training/Kconfig`

This enables the board early init hook:

```conf
config BOARD_TRAINING
	select BOARD_EARLY_INIT_HOOK
```

That hook is implemented in `board.c`.

### `boards/training/training_defconfig`

The board default configuration enables core platform features:

```conf
CONFIG_CONSOLE=y
CONFIG_UART_CONSOLE=y
CONFIG_SERIAL=y
CONFIG_UART_INTERRUPT_DRIVEN=y
CONFIG_GPIO=y
CONFIG_LPADC_DO_OFFSET_CALIBRATION=y
CONFIG_TEST_RANDOM_GENERATOR=y
```

These are defaults for the board itself. The application can still add more
settings in `app/prj.conf`.

### `boards/training/board.cmake`

This declares the supported flash/debug runners:

```cmake
board_runner_args(jlink "--device=MCXA156")
board_runner_args(linkserver "--device=MCXA156:FRDM-MCXA156")
board_runner_args(pyocd "--target=mcxA156")

include(${ZEPHYR_BASE}/boards/common/linkserver.board.cmake)
include(${ZEPHYR_BASE}/boards/common/jlink.board.cmake)
include(${ZEPHYR_BASE}/boards/common/pyocd.board.cmake)
```

The project can therefore flash using runners such as LinkServer, J-Link, or
pyOCD, depending on what is installed and connected.

For the current hardware setup, LinkServer has been used with:

```bash
west flash -d build/mcuboot --runner linkserver
```

### `boards/training/training.dts`

This is the main devicetree description for the custom board.

At the top it includes the MCXA156 SoC description and the board pin control
file:

```dts
#include <nxp/mcx/nxp_mcxa156.dtsi>
#include "training-pinctrl.dtsi"
```

The root node identifies the board:

```dts
model = "NXP Training board";
compatible = "nxp,mcxa156", "nxp,mcx";
```

The aliases define friendly names used by applications and subsystems:

```dts
aliases {
	led0 = &red_led;
	led1 = &green_led;
	led2 = &blue_led;
	sw0 = &user_button_2;
	sw1 = &user_button_3;
	pwm-0 = &flexpwm0_pwm0;
	mcuboot-button0 = &user_button_2;
	watchdog0 = &wwdt0;
	ambient-temp0 = &p3t1755;
	die-temp0 = &temp0;
};
```

By default, the board says `led0` is the red LED. The app overlay changes
`led0` to the green LED.

The `chosen` node tells Zephyr which devices to use for core system roles:

```dts
chosen {
	zephyr,sram = &sram0;
	zephyr,flash = &flash;
	zephyr,flash-controller = &fmu;
	zephyr,code-partition = &slot0_partition;
	zephyr,console = &lpuart0;
	zephyr,shell-uart = &lpuart0;
	zephyr,uart-mcumgr = &lpuart0;
	zephyr,canbus = &flexcan0;
	zephyr,system-timer-companion = &lptmr0;
};
```

For MCUboot, the most important line is:

```dts
zephyr,code-partition = &slot0_partition;
```

That tells Zephyr that the application image should be linked into the primary
application slot, not at address zero.

The LED nodes define the physical GPIO pins:

```dts
red_led: led_0 {
	gpios = <&gpio3 12 GPIO_ACTIVE_LOW>;
	label = "Red LED";
};

green_led: led_1 {
	gpios = <&gpio3 13 GPIO_ACTIVE_LOW>;
	label = "Green LED";
};

blue_led: led_2 {
	gpios = <&gpio3 0 GPIO_ACTIVE_LOW>;
	label = "Blue LED";
};
```

They are active-low LEDs, so GPIO active/inactive logic is inverted relative to
the electrical level.

The UART used for console and shell is `lpuart0` at 115200 baud:

```dts
&lpuart0 {
	status = "okay";
	current-speed = <115200>;
	pinctrl-0 = <&pinmux_lpuart0>;
	pinctrl-names = "default";
};
```

The flash partitions are defined at the bottom of the file:

```dts
&flash {
	partitions {
		compatible = "fixed-partitions";
		#address-cells = <1>;
		#size-cells = <1>;

		boot_partition: partition@0 {
			label = "mcuboot";
			reg = <0x00000000 DT_SIZE_K(64)>;
			read-only;
		};

		slot0_partition: partition@10000 {
			label = "image-0";
			reg = <0x00010000 DT_SIZE_K(424)>;
		};

		slot1_partition: partition@7a000 {
			label = "image-1";
			reg = <0x0007a000 DT_SIZE_K(424)>;
		};

		storage_partition: partition@e4000 {
			label = "storage";
			reg = <0x000e4000 DT_SIZE_K(112)>;
		};
	};
};
```

This layout is for MCUboot:

- `boot_partition`: MCUboot itself, starting at address `0x0`, size 64 KiB.
- `slot0_partition`: primary app slot, starting at `0x10000`, size 424 KiB.
- `slot1_partition`: secondary app/update slot, starting at `0x7a000`, size 424 KiB.
- `storage_partition`: remaining storage area, starting at `0xe4000`, size 112 KiB.

With the normal MCUboot flow, the app runs from `slot0_partition`. A new signed
image can be placed in `slot1_partition`, and MCUboot can validate/swap it into
slot 0.

### `boards/training/training-pinctrl.dtsi`

This file defines the pin multiplexing for board peripherals. Examples:

- `pinmux_lpuart0` maps UART0 RX/TX to `P0_2` and `P0_3`.
- `pinmux_lpuart1` maps UART1 RX/TX to `P3_20` and `P3_21`.
- `pinmux_lpspi0` maps SPI signals to `P1_0` through `P1_3`.
- `pinmux_flexcan0` maps CAN TX/RX pins.
- `pinmux_i3c0`, `pinmux_lpi2c0`, `pinmux_lpi2c2`, and `pinmux_lpi2c3`
  configure I3C/I2C pins.

The devicetree enables peripherals by referencing these pinctrl nodes.

### `boards/training/board.c`

This file implements `board_early_init_hook()`. It performs low-level SoC setup
before the main application starts.

Important jobs in this file:

- Sets the core clock target to 96 MHz.
- Configures regulator and SRAM voltage settings for the selected frequency.
- Configures flash wait states.
- Sets up FRO high-frequency and 12 MHz clocks.
- Routes clocks to enabled peripherals.
- Releases resets and enables clock gates for GPIO ports.
- Sets up UART, timers, ADC, CAN, FlexIO, I2C, SPI, USB, watchdog, and other
  peripheral clocks when their devicetree nodes are enabled.
- Updates `SystemCoreClock`.

This is board support code rather than application logic. The app should not
need to call this directly.

### `boards/training/production.overlay`

This is a board-level overlay for the workshop production configuration:

```dts
/ {
	aliases {
		led0 = &green_led;
	};

	zephyr,user {
		blink-period-ms = <2000>;
	};
};
```

It does the same functional thing as the app overlays: use the green LED and set
the blink period to 2000 ms.

## Sysbuild and MCUboot

The repository has been prepared for Zephyr sysbuild.

Sysbuild lets one `west build --sysbuild` command build multiple coordinated
images. In this project, the two images are:

- MCUboot bootloader
- Application firmware

### `app/sysbuild.conf`

```conf
# Build MCUboot and the application as one coordinated sysbuild.
SB_CONFIG_BOOTLOADER_MCUBOOT=y
SB_CONFIG_MCUBOOT_MODE_SWAP_USING_MOVE=y
SB_CONFIG_BOOT_SIGNATURE_TYPE_RSA=y
SB_CONFIG_BOOT_SIGNATURE_KEY_FILE="${APP_DIR}/../keys/my-key.pem"
```

This enables MCUboot in sysbuild mode, selects swap-using-move mode, and enables
RSA signing with the project-local private key. The key lives under `keys/`,
which is ignored by git and must not be committed.

### `app/sysbuild/mcuboot.conf`

```conf
CONFIG_BOOT_SWAP_USING_MOVE=y
CONFIG_BOOT_SIGNATURE_TYPE_RSA=y
CONFIG_BOOT_SIGNATURE_TYPE_RSA_LEN=2048
```

This config is applied specifically to the MCUboot image.

- `CONFIG_BOOT_SWAP_USING_MOVE=y` selects MCUboot's move-based swap algorithm.
- `CONFIG_BOOT_SIGNATURE_TYPE_RSA=y` enables RSA signature verification.
- `CONFIG_BOOT_SIGNATURE_TYPE_RSA_LEN=2048` uses RSA-2048 signatures.

With sysbuild, Zephyr also signs the app image during the build using the
configured MCUboot key. That avoids needing a separate manual `west sign`
command for the normal workflow.

Block 3 generated the workshop key with:

```bash
mkdir -p keys
imgtool keygen -k keys/my-key.pem -t rsa-2048
```

The signed image can be checked with:

```bash
imgtool verify -k keys/my-key.pem build/signed/app/zephyr/zephyr.signed.bin
```

Verifying the same image with the MCUboot demo key should fail. That proves the
bootloader is no longer trusting the public demo key from the MCUboot tree.

Typical build command:

```bash
west build -b training app --sysbuild -d build/mcuboot --pristine
```

Typical flash command:

```bash
west flash -d build/mcuboot --runner linkserver
```

Useful generated artifacts after a successful sysbuild include:

- `build/mcuboot/mcuboot/zephyr/zephyr.bin`: MCUboot image.
- `build/mcuboot/app/zephyr/zephyr.bin`: unsigned/plain app image.
- `build/mcuboot/app/zephyr/zephyr.signed.bin`: signed app image.

## CI Build and Signing

Block 4 adds a GitHub Actions workflow at `.github/workflows/build.yml`.
After Block 5, that workflow builds the hardened release variant.

The workflow does the same production build as the local Block 3 flow, but in a
clean GitHub runner:

1. Checks out this repository into an `app/` directory.
2. Sets up Python 3.12.
3. Uses `zephyrproject-rtos/action-zephyr-setup@v1` to install Zephyr tooling and
   the `arm-zephyr-eabi` toolchain.
4. Installs the project Python tools from `app/requirements.txt`.
5. Writes the private signing key from the GitHub secret
   `MCUBOOT_SIGNING_KEY` to `app/keys/my-key.pem`.
6. Builds the signed release sysbuild image:

   ```bash
   west build -b training app/app --sysbuild -d build/prod --pristine -S release
   ```

7. Verifies signing happened:

   ```bash
   imgtool verify -k app/keys/my-key.pem build/prod/app/zephyr/zephyr.signed.bin
   ```

8. Uploads the generated firmware artifacts.

The workflow expects this GitHub Actions secret:

```text
MCUBOOT_SIGNING_KEY
```

Its value must be the full contents of `keys/my-key.pem`, including the PEM
header and footer. The key itself must not be committed to git.

The uploaded artifact is named `training-firmware` and includes:

- `build/prod/mcuboot/zephyr/zephyr.bin`
- `build/prod/app/zephyr/zephyr.signed.bin`
- `build/prod/app/zephyr/zephyr.elf`

## Release Hardening

Block 5 adds a Zephyr snippet named `release` under `snippets/release`.

The snippet is registered automatically because `zephyr/module.yml` has:

```yaml
snippet_root: .
```

The snippet metadata is:

```yaml
name: release
append:
  EXTRA_CONF_FILE: release.conf
  EXTRA_DTC_OVERLAY_FILE: release.overlay
```

### `snippets/release/release.conf`

The release Kconfig turns off development affordances:

```conf
CONFIG_SHELL=n
CONFIG_LOG=n
CONFIG_ASSERT=n
CONFIG_BOOT_BANNER=n
CONFIG_PRINTK=n
CONFIG_TEST_RANDOM_GENERATOR=n
```

It also folds in the hardenconfig recommendations accepted for this workshop
app:

```conf
CONFIG_BUILD_OUTPUT_STRIPPED=y
CONFIG_BUILTIN_STACK_GUARD=y
CONFIG_FAULT_DUMP=0
CONFIG_HW_STACK_PROTECTION=y
CONFIG_OVERRIDE_FRAME_POINTER_DEFAULT=y
CONFIG_STACK_SENTINEL=y
```

### `snippets/release/release.overlay`

The release devicetree overlay changes the status LED to blue:

```dts
/ {
	aliases {
		led0 = &blue_led;
	};
};
```

The normal training overlay maps `led0` to green. The release snippet is applied
after it, so the release build resolves `led0` to `blue_led`.

Build the release image with:

```bash
west build -b training app --sysbuild -d build/release --pristine -S release
```

Run the app hardening audit with:

```bash
west build -d build/release/app -t hardenconfig
```

The current release app passes the local hardenconfig check without a remaining
failure table.

The measured app binary size changed from:

```text
build/signed/app/zephyr/zephyr.bin    74528 bytes
build/release/app/zephyr/zephyr.bin   16364 bytes
```

The signed release app is:

```text
build/release/app/zephyr/zephyr.signed.bin
```

## MCUboot Image Acceptance and Rollback

MCUboot decides whether to keep or roll back an updated image using metadata in
the image trailer.

The common flow is:

1. A new image is written to `slot1_partition`.
2. MCUboot validates the image signature.
3. MCUboot swaps the image into `slot0_partition`.
4. The new app boots in test mode.
5. If the app decides it is healthy, it marks itself confirmed.
6. On the next reset, MCUboot keeps the image.
7. If the app never confirms itself, MCUboot rolls back to the previous image.

So yes, a production application normally needs a health decision. After the app
has checked enough of itself to be trusted, it should mark the image as OK using
the MCUboot/bootutil API, commonly through `boot_write_img_confirmed()`.

The workshop app currently blinks an LED and does not yet implement image
confirmation logic.

## Notes for Small Flash Parts

The current training board partition layout assumes enough internal flash for:

- MCUboot
- primary image slot
- secondary image slot
- storage

On a smaller part such as an MCXA153 with only 128 KiB flash, the normal
two-internal-slot MCUboot layout is usually too tight for Zephyr. Options for
small flash parts include:

- Use no bootloader for very small demos.
- Use overwrite-only MCUboot mode.
- Put `slot1_partition` on external flash.
- Aggressively reduce logging, shell, assertions, drivers, and other features.

For the normal rollback-capable two-slot layout, 256 KiB is a more realistic
starting point, and larger flash is more comfortable.

## Useful Commands

Build the app for the custom board:

```bash
west build -b training app --pristine
```

Build app plus MCUboot through sysbuild:

```bash
west build -b training app --sysbuild -d build/mcuboot --pristine
```

Flash the sysbuild output:

```bash
west flash -d build/mcuboot --runner linkserver
```

Watch logs:

```bash
minicom -D /dev/ttyACM0 -b 115200
```

Alternative serial tools:

```bash
picocom -b 115200 /dev/ttyACM0
tio /dev/ttyACM0
```

List available boards and confirm `training` is discovered:

```bash
west boards | rg training
```

Inspect final build configuration:

```bash
rg "CONFIG_BOOT|CONFIG_MCUBOOT|FLASH_LOAD_OFFSET" build/mcuboot/app/zephyr/.config
rg "CONFIG_BOOT|CONFIG_MCUBOOT|FLASH_LOAD_OFFSET" build/mcuboot/mcuboot/zephyr/.config
```
