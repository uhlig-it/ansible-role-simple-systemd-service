# `simple_systemd_service`

Ansible role for deploying a simple binary that runs as a systemd service; either as long-running job or invoked by an accompanying systemd timer.

# Variables

```yaml
program:
  binary: /tmp/hello.amd64
  name: hello
  description: Says hello in a random language
  parameters: [
    - --one
    - --two
  ]
  # zero or more of OnActiveSec, OnBootSec, OnStartupSec, OnUnitActiveSec, OnUnitInactiveSec
  # see https://www.freedesktop.org/software/systemd/man/systemd.timer.html#OnActiveSec=
  # if left empty, no timer will be installed and the binary is assumed to run as a daemon.
  timer:
    - OnUnitActiveSec=1m
```

This role can also send an event to an InfluxDB instance when a program has been deployed. Set the variable `deployment_event_url` in order to enable write the event, incl. user, password and database name (as URL path).

# TODO

* support [readyness and liveness](https://vincent.bernat.ch/en/blog/2017-systemd-golang)
