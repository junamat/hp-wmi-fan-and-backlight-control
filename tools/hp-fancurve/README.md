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
