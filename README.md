## hp-wmi manual fan + keyboard RGB control (junamat fork)

An out-of-tree build of the kernel's `drivers/platform/x86/hp/hp-wmi.c` driver that adds
manual fan control and zoned keyboard RGB control for HP Omen/Victus laptops. It builds on
kernels both older and newer than 6.14.

This is a fork of [TUXOV/hp-wmi-fan-and-backlight-control](https://github.com/TUXOV/hp-wmi-fan-and-backlight-control).
Everything upstream supports still works here. The fork adds support for the
**Victus 16-e0xxx (board 8934)**, a board whose BIOS claims it has no software fan control,
plus a fan-curve daemon and a test helper. See [What this fork changes](#what-this-fork-changes).

### Supported hardware

| Laptop | Board | Fans | Platform profiles | Keyboard | Status |
|---|---|---|---|---|---|
| Victus 16-e0xxx (Ryzen 5 5600H) | `8934` | Manual, max, auto, **zero RPM**, 1000 RPM floor | quiet / balanced / performance (sets AMD power limits) | single-colour (no RGB) | Tested in this fork, see [docs/boards/8934.md](docs/boards/8934.md) |
| Victus 16-s1 (9Z791EA) | Victus S | Manual, max, auto | yes | 4-zone RGB | Tested upstream |
| Other Omen / Victus boards known to upstream `hp-wmi` | various | depends on board | depends on board | single-zone or 4-zone RGB | Untested here, reports welcome |

Boards are picked up by their DMI board name (`cat /sys/class/dmi/id/board_name`). If yours
is not listed, check the upstream tester thread:
https://github.com/TUXOV/hp-wmi-fan-and-backlight-control/issues/1

### What this fork changes

Driver (`hp-wmi.c`):
- **Board 8934 fan control.** The BIOS reports no fan support, but its ACPI implements the
  same GM fan commands as Victus S. The driver arms user-defined fan mode (GM `0x10`) before
  setting targets and keeps it alive.
- **Zero RPM on 8934.** On this board a fan target of 0 stops the fan instead of meaning
  "auto". Writing `pwmN = 0` in manual mode stops that fan. Going back to auto never sends 0;
  the driver stops its keep-alive and lets the EC return to its own curve. If whatever set
  zero RPM dies, the EC takes over again within about 2 minutes.
- **1000 RPM fan floor on 8934.** The BIOS fan table starts at 2300 RPM, but both fans hold
  900–1900 RPM steadily, so manual targets on this board go down to 1000 RPM. Boards can set
  their own floor through the `min_rpm` fan-profile field.
- **Platform profiles on 8934.** Uses the Victus GM `0x1A` path, which applies the real AMD
  power limits. The generic WMI `0x4C` call does nothing on this board.
- **Fans back to auto on unload (all boards).** Unloading the driver in manual or max mode
  now hands the fans back to the EC. Before, max mode stayed on until reboot.
- **Fan mode survives suspend (all boards).** The keep-alive stops before sleep, and manual
  or max mode is re-applied on resume and after hibernation.
- **pwm ↔ RPM rounding (all boards).** Conversions round to nearest instead of truncating,
  so a `pwmN` write no longer loses up to 100 RPM.

Userspace (`tools/`):
- [`hp-fancurve`](tools/hp-fancurve/README.md): daemon that drives `pwm1`/`pwm2` from CPU
  and GPU temperature, with a curve per platform profile and optional zero RPM.
- [`fanlab`](tools/README.md): small root helper for testing fan control without a general
  sudo rule.

Packaging: DKMS version `0.0.4.2`.

### Installation

DKMS:
```bash
git clone https://github.com/junamat/hp-wmi-fan-and-backlight-control
cd hp-wmi-fan-and-backlight-control
make
sudo make install-dkms
```

Arch package:
```bash
git clone https://github.com/junamat/hp-wmi-fan-and-backlight-control
cd hp-wmi-fan-and-backlight-control
make install-arch
```

Fedora (akmods):
```bash
sudo dnf install akmods kmodtool kernel-devel gcc make elfutils-libelf-devel rpm-build
git clone https://github.com/junamat/hp-wmi-fan-and-backlight-control
cd hp-wmi-fan-and-backlight-control
make akmod
sudo dnf install ./akmod-build/RPMS/*/*.rpm
```
Install all the generated RPMs in one transaction: the akmod package depends on the common
package built next to it. akmods builds the module for the running kernel and rebuilds it
after kernel updates. To remove it:
```bash
sudo make uninstall-akmod
```

NixOS: add the repo as a flake input and enable the module.
```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    hp-wmi-control.url = "github:junamat/hp-wmi-fan-and-backlight-control"; # or a local path
  };
  outputs = { self, nixpkgs, hp-wmi-control, ... }: {
    nixosConfigurations.myhost = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        ./configuration.nix
        hp-wmi-control.nixosModules.default
        { hardware.hp-wmi-control.enable = true; }
      ];
    };
  };
}
```
Supported boards are detected by DMI name; the old `victus-15-support` option is gone.

To try the module without installing it (lost on reboot):
```bash
git clone https://github.com/junamat/hp-wmi-fan-and-backlight-control
cd hp-wmi-fan-and-backlight-control
make
sudo rmmod hp-wmi
sudo modprobe led_class_multicolor
sudo insmod hp-wmi.ko
```

### Usage

#### Fans

Standard hwmon interface under `/sys/devices/platform/hp-wmi/hwmon/hwmon*/`
(see `Documentation/ABI/testing/sysfs-class-hwmon`):

| File | Meaning |
|---|---|
| `pwm1_enable` | `2` = automatic (EC), `1` = manual, `0` = max |
| `pwm1`, `pwm2` | Fan 1 (CPU) / fan 2 (GPU) target, 0–255, linear to 0–max RPM. Manual mode only. |
| `fan1_input`, `fan2_input` | Current speed in RPM |

```bash
cd /sys/devices/platform/hp-wmi/hwmon/hwmon*
echo 1   | sudo tee pwm1_enable   # manual
echo 180 | sudo tee pwm1          # set fan 1
cat fan1_input                    # read fan 1 RPM
echo 2   | sudo tee pwm1_enable   # back to automatic
```

- Targets are clamped to the board's minimum (the BIOS fan table minimum, or the board's
  `min_rpm` floor) and maximum.
- On zero RPM boards (8934), `pwmN = 0` stops that fan.
- The driver refreshes the firmware every 90 s so the EC does not drop manual mode.

For temperature-based curves, use [`hp-fancurve`](tools/hp-fancurve/README.md).
For board 8934 there is a ready-made zero RPM curve:
[`examples/8934-zero-rpm.toml`](tools/hp-fancurve/examples/8934-zero-rpm.toml).

#### Platform profiles

`/sys/firmware/acpi/platform_profile`, with the choices in `platform_profile_choices`.
On laptops that support it, Fn+P switches profiles.

#### Keyboard RGB

4-zone keyboards expose one multicolor LED per zone under `/sys/class/leds/` (named per
`Documentation/leds/leds-class.rst`): WASD plus three vertical thirds of the rest.
- `hp::kbd_zoned_backlight-left`
- `hp::kbd_zoned_backlight-center`
- `hp::kbd_zoned_backlight-right`
- `hp::kbd_zoned_backlight-wasd`

Single-zone RGB keyboards use `hp::kbd_backlight`. Both take `multi_intensity` (`R G B`,
0–255 each) and `brightness` (0–255):
```bash
echo "255 0 0" | sudo tee /sys/class/leds/hp::kbd_zoned_backlight-left/multi_intensity
echo 128       | sudo tee /sys/class/leds/hp::kbd_zoned_backlight-left/brightness
```
Brightness changes from the backlight hotkey (Fn+F4) are reported through the LED class
`brightness_hw_changed` uevent.

### GUI

Upstream's GUI is [victus-control](https://github.com/Vilez0/victus-control). It is not
tested with this fork's 8934 additions.

### Repository layout

| Path | What |
|---|---|
| `hp-wmi.c` | The driver |
| `docs/boards/` | Per-board notes: ACPI/EC maps, hardware test logs |
| `tools/hp-fancurve/` | Fan-curve daemon, systemd unit, example configs |
| `tools/fanlab` | Root test helper and sudoers example |
| `dkms.conf`, `PKGBUILD`, `*.spec`, `flake.nix` | DKMS, Arch, Fedora akmod and NixOS packaging |

### Known issues

- Unloading the driver while in max mode leaves the EC in max fan mode. Inherited from
  upstream, affects all boards.

### Disclaimer

USE IT AT YOUR OWN RISK. NO RESPONSIBILITY IS ACCEPTED FOR ANY DAMAGE.
