# `simple_systemd_service`

Ansible role for deploying a simple binary as a hardened systemd **system service** that runs as a dedicated, unprivileged **system user**.

# Approach

Every service deployed by this role runs as its own system user:

- `program.runtime_user` is created as a `nologin` system account (no shell, no home directory - nothing to pivot into if the service is compromised).
- The unit lives in `/etc/systemd/system` and is supervised by pid 1: deterministic boot, standard `systemctl` / `journalctl` handling. No `loginctl enable-linger`, no dbus session.
- By default the unit is hardened with `NoNewPrivileges=true`, `PrivateTmp=true` and `ProtectSystem=full` (only `/usr`, `/boot`, `/etc` are made read-only; a `program.data_dir` stays writable).
- Secrets go into `/etc/<name>.conf` with mode `0640 root:<runtime_user>`: not world-readable, and the world-readable unit file itself contains no secrets.

This replaces the role's previous two modes (a root-running system service and a per-user service with a full interactive account). See `MIGRATION.markdown` for how to migrate services deployed with those.

# Variables

```yaml
program:
  binary: /tmp/hello.amd64            # binary to deploy: local path, or https:// URL to a tarball
  name: hello                         # service name: /etc/systemd/system/hello.service
  description: Says hello in a random language
  runtime_user: hello                 # REQUIRED: dedicated system user the service runs as
  parameters:
    - --one
    - --two
  environment:                        # dict, rendered into /etc/hello.conf (0640 root:hello)
    foo: some
    bar: thing
  data_dir: /var/lib/hello            # optional; created with mode 0750, owned by runtime_user;
                                      # also the service's working directory
  working_directory: /var/lib/hello   # optional; defaults to data_dir, or / if unset
  groups: []                          # optional supplementary groups for runtime_user
  timer:                              # zero or more of OnActiveSec, OnBootSec, OnStartupSec,
                                      # OnUnitActiveSec, OnUnitInactiveSec (see systemd.timer(5));
                                      # if left empty, the binary is assumed to run as a daemon.
    - OnUnitActiveSec=1m
systemd:
  target: network-online.target       # ordering target (Wants= / After=)
  wanted_by: multi-user.target        # enabling target ([Install])
  start_limit_interval: 5m
  start_limit_burst: 20
  restart: on-failure
  restart_sec: 3s
  hardening: true                     # NoNewPrivileges=true, PrivateTmp=true, ProtectSystem=full
```

If `program.binary` is a URL, the archive is downloaded and **only** the executable named `{{ program.name }}` is extracted from it (it may live anywhere inside the archive) and installed as `/usr/local/bin/{{ program.name }}`; any other content of the archive is discarded.

# TODO

* Support [readyness and liveness](https://vincent.bernat.ch/en/blog/2017-systemd-golang)
* Checksum verification for remote binaries