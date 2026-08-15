# MMIO/H2RAM hardening notes

This branch carries additional error handling and state-safety changes for the
IT87 MMIO/H2RAM bridge path. It is based on upstream commit
`490f76f61900c163fac5328506c49969bd716dc6`.

## Motivation

Some recent Gigabyte boards expose their IT87 environment controller through
an MMIO/H2RAM bridge. Programming that bridge changes chipset decode state, so
partial writes, failed probes, or suspend transitions must not leave firmware
state half-restored. PWM policy writes also need to stop whenever controller
access has not passed validation.

The branch therefore:

- writes the computed tachometer-enable mask rather than a stale cached value;
- rejects PWM policy writes unless probe or resume validation succeeded;
- propagates PCI configuration access failures;
- requires a complete snapshot before modifying bridge registers;
- rolls back partial AMD and Intel bridge programming;
- validates both slots and tracks the active slot explicitly;
- serializes global bridge changes;
- restores firmware bridge state for suspend and rebuilds it on resume;
- balances PCI enable/reference handling; and
- unwinds partial platform-device registration when bridge setup fails.

## Hardware validation

The combined result represented by this branch has been used on:

- Motherboard: Gigabyte B760 GAMING X
- Firmware: BIOS F18a
- Super I/O: IT8689E revision 2
- Distribution: Ubuntu 24.04.4 LTS
- Kernels: `6.8.0-136-generic` and `6.8.0-137-generic`
- Driver base: `490f76f61900c163fac5328506c49969bd716dc6`

On that system the driver discovers the environment controller at I/O address
`0xa40` through MMIO base `0xfc000000` and exposes six fan inputs and six PWM
channels. DKMS builds were installed for both listed kernels, with
`6.8.0-137-generic` active during the final source capture.

No userspace `fancontrol` daemon was installed during that capture, so this is
not a claim that an automatic userspace fan curve has been endurance-tested.
It is evidence that the module builds, loads, discovers the controller through
the Gigabyte bridge path, and exposes the expected hwmon interfaces.

## Review and testing guidance

These changes touch low-level chipset decode and fan-control paths. Reviewers
should pay particular attention to PCI reference balance, restoration after
each error path, lock ordering between `mmio_lock` and `update_lock`, and
suspend-failure recovery.

For a new board or kernel, start with a build-only test. Keep a known-working
kernel and module available, confirm sensor readings before attempting writes,
and monitor temperatures and fan RPM independently during initial PWM tests.
