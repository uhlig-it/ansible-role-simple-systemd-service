# `simple_systemd_service`

Ansible role for deploying a simple binary that runs as a systemd service; either as long-running job or invoked by an accompanying systemd timer.

# Variables

```yaml
systemd:
  target: network.target
  start_limit_interval: 5m
  start_limit_burst: 10
program:
  binary: /tmp/hello.amd64
  name: hello
  description: Says hello in a random language
  parameters:
    - --one
    - --two
  environment:
    foo: some
    bar: thing
  # zero or more of OnActiveSec, OnBootSec, OnStartupSec, OnUnitActiveSec, OnUnitInactiveSec
  # see https://www.freedesktop.org/software/systemd/man/systemd.timer.html#OnActiveSec=
  # if left empty, no timer will be installed and the binary is assumed to run as a daemon.
  timer:
    - OnUnitActiveSec=1m
```

# TODO

* Restart the service when the binary has changed (did not work when plaintweet was updated)
* Support [readyness and liveness](https://vincent.bernat.ch/en/blog/2017-systemd-golang)
