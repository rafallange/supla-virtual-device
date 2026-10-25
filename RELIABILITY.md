# Reconnection fixes and deployment

The installer repository is `rafallange/supla-virtual-device`. The C/C++ changes
belong to its separate nested source checkout, `src`, whose origin is
`rafallange/supla-core`, branch `supla-mqtt-dev`. Commit/transfer the source
changes in that repository as well as any installer documentation changes.

The repair keeps MQTT running across SUPLA cloud outages. Incoming readings
update the cache immediately; only the SUPLA connection thread sends queued
notifications. Registration replays current readings, and a full SUPLA send
queue causes a retry instead of losing the update. MQTT connection failures,
broker restarts, partial socket writes and missing acknowledgments recover
without an intentional process restart. Passwords are no longer logged.

## Test the source checkout on Linux

Install a C/C++ compiler and OpenSSL development headers, then run from the
installer directory:

```sh
sh src/supla-dev/tests/run_channel_lifecycle_tests.sh
sh src/supla-dev/tests/run_mqtt_transport_tests.sh
make -C src/supla-dev/Release -j2 all
```

For memory/undefined-behavior checks, prefix either test command with
`SANITIZER=address,undefined`. Test runners use temporary build directories;
they do not contact SUPLA or change a running service.

## Update the Raspberry Pi

Transfer the repaired `supla-core` source checkout before rebuilding. The
current locally edited `install.sh` clones/pulls your fork but has its build
and executable-link steps commented out; those existing edits were preserved.
Use the explicit build below to produce the updated executable.

From the deployed installer directory:

```sh
sudo systemctl stop supla-virtual-device
make -C src/supla-dev/Release -j2 all
```

Only after the build succeeds, verify that your unit's `ExecStart` points to
the newly built `src/supla-dev/Release/supla-virtual-device`, or that the
installer's `supla-virtual-device` symlink points there. If the service uses a
separately copied executable, replace that copy before restarting it. Keep the
existing configuration, GUID and authentication-key files unchanged.

```sh
sudo systemctl start supla-virtual-device
sudo journalctl -u supla-virtual-device -f
```

The journal now records successful SUPLA registration/disconnection at INFO
level and MQTT reconnects/timeouts at WARNING level. A normal cloud/broker
interruption should recover without rebooting the Pi.

Update the `tech-check` Python package separately. Its fixes isolate unavailable
rooms, allow nullable metadata, retry missing modules, preserve authentication,
and leave automatic MQTT reconnection enabled after a publish error. When
testing recovery, keep MQTT publishing while SUPLA is unavailable, then restore
cloud connectivity and verify fresh app values and a clean service stop.

Retaining old sensor messages is not required by this repair. Existing sensor
topics/payloads and retain settings are preserved.
