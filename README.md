# `simple_systemd_service`

An Ansible collection for deploying a simple binary that runs as a systemd service; either as long-running job or invoked by an accompanying systemd timer.

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
  # if left empty, no timer will be installed
  timer:
    - OnUnitActiveSec=1m
```

This role can also send an event to an InfluxDB instance when a program has been deployed. Set the following variable in order to enable this:

* `influxdb_url` - where to write to, incl. user, password and database name
