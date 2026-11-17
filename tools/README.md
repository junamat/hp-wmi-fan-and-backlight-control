# tools

## fanlab

`fanlab` is a small root helper for testing fan control on real hardware without a
general sudo rule. It only runs fixed actions and checks every argument:

| Command | Effect |
|---|---|
| `reload [force]` | Reload the installed `hp_wmi` module |
| `mode 0\|1\|2` | `pwm1_enable`: max, manual, auto |
| `pwm 1\|2 0-255` | `pwmN` (manual mode only) |
| `target 1\|2 RPM` | `fanN_target` (older builds; capped at 8000) |
| `status` | Print the hwmon fan and pwm values and the platform profile |
| `dmesg` | Show the last `hp_wmi` kernel messages |
| `acpi-dump` | Copy DSDT and SSDTs to a new readable directory under `/tmp` |

Install:

```bash
sudo install -o root -g root -m 0755 tools/fanlab /usr/local/sbin/fanlab
sed "s/<user>/$USER/" tools/fanlab.sudoers.example > /tmp/fanlab.sudoers
sudo install -o root -g root -m 0440 /tmp/fanlab.sudoers /etc/sudoers.d/fanlab
sudo visudo -cf /etc/sudoers.d/fanlab
```

Remove:

```bash
sudo rm /etc/sudoers.d/fanlab /usr/local/sbin/fanlab
```

Loading a test build (`insmod hp-wmi.ko`) is deliberately not part of `fanlab`: a rule that
allows it gives full root. Run that step yourself.
