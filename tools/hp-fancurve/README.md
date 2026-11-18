# hp-fancurve

Some HP boards keep their fans near idle speed under sustained load while the
EC is in automatic mode. On board 8934 the CPU reached 89 °C at about 45 W with
both fans flat at about 2000 / 2200 RPM. `hp-fancurve` puts the `hp-wmi` fans in
manual mode and drives `pwm1` / `pwm2` from CPU and GPU temperatures, with a
curve set for each platform profile (quiet, balanced, performance).

It needs a driver build that exposes `pwm1_enable`, `pwm1` and `pwm2` (this repo).
Python 3.11 or newer, no other dependencies.

## Behaviour

- Uses the higher of the CPU and `amdgpu` temperatures. A powered-off dGPU
  (`EBUSY`) counts as no reading.
- Ramps up immediately. Ramps down only after the lower target has held for
  `down_delay`, and then by at most `down_step` % per step.
- Follows `/sys/firmware/acpi/platform_profile`.
- At or above `critical` it switches to max fan mode. If the CPU sensor is
  unreadable it also switches to max fan mode.
- On stop it hands the fans back to EC auto (`pwm1_enable=2`). On board 8934
  the EC keeps the last targets for up to about 2 minutes before its own curve
  resumes.

## Configuration

`/etc/hp-fancurve.toml` is optional; every key falls back to a built-in default. See
[`hp-fancurve.toml.example`](hp-fancurve.toml.example) for all keys. Curves are
`[temperature °C, pwm %]` points per fan and profile, interpolated linearly.

### Zero RPM (opt-in)

On boards where the driver supports it (board 8934), `zero_rpm = true` stops both fans
while the temperature stays below `zero_rpm_below` for `zero_rpm_hold` seconds, and spins
them up again at `zero_rpm_resume`. The stop is re-sent every `zero_rpm_refresh` seconds.
If the daemon dies with the fans stopped, the EC resumes its own curve within about 2
minutes.

Ready-made config for board 8934:
[`examples/8934-zero-rpm.toml`](examples/8934-zero-rpm.toml) (fans off below 45–50 °C,
then 1000 RPM rising to max at 80 °C).

## Install

```bash
sudo install -m 0755 tools/hp-fancurve/hp-fancurve /usr/local/bin/hp-fancurve
sudo install -m 0644 tools/hp-fancurve/hp-fancurve.service /etc/systemd/system/
sudo install -m 0644 tools/hp-fancurve/hp-fancurve.toml.example /etc/hp-fancurve.toml  # optional
sudo systemctl daemon-reload && sudo systemctl enable --now hp-fancurve
```

To try it without writing anything:

```bash
hp-fancurve --dry-run
```
