# Migrating services to the `simple_systemd_service` system-user approach

This document is the reference for writing a playbook that migrates a service deployed with an old approach to the role's current (and only) approach:

> A hardened systemd **system service** running as a dedicated **`nologin` system user**.

It covers migrating from both historical modes of this role:

1. **User service** (the old `runtime_user` + user-service path): a per-user systemd instance, a full interactive account (`/bin/zsh`, home directory, dbus session), started via `loginctl enable-linger`.
2. **Root-running system service** (the old system-service path): a unit without `User=`, everything running as root.

If you are migrating a hand-rolled kehrkraft-style playbook to the role, the same steps apply; just skip the user-service cleanup if the old deployment never ran in a per-user instance.

## Why migrate

- The user-service mode created a full interactive account: a shell, a home directory and a dbus session. A compromised service then has a login shell and a home to pivot into. The new mode keeps the process unprivileged (that is the meaningful isolation) but gives the account no shell, no home, nothing else.
- User services only start deterministically with linger + dbus; a system service is supervised by pid 1 and starts at boot, period.
- The old system-service mode ran everything as **root**.

## What a migrated deployment should look like

| Thing | Old (user service) | Old (root system service) | New |
|---|---|---|---|
| Unit | `~<user>/.config/systemd/user/<name>.service` | `/etc/systemd/system/<name>.service`, no `User=` | `/etc/systemd/system/<name>.service` with `User=`/`Group=` |
| Runtime account | interactive: `/bin/zsh`, home, dbus | none (root) | `nologin` system user, no home |
| Environment | `~<user>/.config/systemd/user/<name>.conf` | `/etc/<name>.conf` | `/etc/<name>.conf`, `0640 root:<user>` |
| Binary | `~<user>/bin/<name>` (user-writable) | `/usr/local/bin/<name>` | `/usr/local/bin/<name>`, root-owned |
| Data | wherever the app wrote (often the home) | wherever it wrote | `/var/lib/<name>`, `0750`, owned by the user |
| Boot | needs `loginctl enable-linger` + dbus | `WantedBy=multi-user.target` | `WantedBy=multi-user.target` |
| Extra deps | `acl`, `zsh`, dbus | - | - |

## Strategy

Migrate **one service per host, one service at a time**, in a maintenance window:

0. **Inventory** what the old deployment looks like (see below).
1. **Back up** the old user's data.
2. **Stop** the old service before moving files (it may hold the port and/or open database files).
3. **Move** data, binary, secrets and unit to their new locations.
4. **Start** the new system service and **verify** (health check + reboot test).
5. **Decommission** the old user service, account and packages - *after* verification, so rollback stays possible.

### Reuse the account or create a new one?

Two valid options:

- **Reuse the name** (recommended when data lives in the old user's home and you want zero ownership churn): manage the *same* account as a system user. Ansible's `user` module only applies `system: true` when *creating* the account, so an existing account keeps its UID - the number does not matter for a service user. The role changes the shell to `nologin` and sets the home.
- **New name + move** (cleaner separation): create a fresh system user and `chown -R` the data.

Either way keep a backup, and keep the old account around (locked, `nologin`) until the new service has survived a reboot.

## Step-by-step playbook

A copy-paste-friendly skeleton; fill in the per-service values (`<name>`, `<user>`, the port, the data paths) from the inventory step.

### 0. Inventory the old deployment

```yaml
- name: Old account facts
  ansible.builtin.getent:
    database: passwd
    key: "{{ old_runtime_user }}"
  register: old_account

- name: Read the old environment file to learn data paths
  ansible.builtin.slurp:
    src: "/home/{{ old_runtime_user }}/.config/systemd/user/{{ app_name }}.conf"
  register: old_env_file
```

`old_env_file.content` is base64-encoded; decode it (Jinja: `old_env_file.content | b64decode`) or just read it during the inventory pass. Look for `DATABASE_URL`, `PORT`, `DATA_DIR`, ... - that is where the app keeps its state. Also record:

```command
systemctl --user list-units --type=service         # as the old user
ls -l /var/lib/systemd/linger/                     # which users linger
getent passwd <old_user>                           # old home, shell, UID
```

### 1. Back up

```yaml
- name: Back up the old user's home and data
  ansible.builtin.command: tar -C /home -czf "/root/backup-{{ app_name }}-{{ ansible_date_time.epoch }}.tgz" "{{ old_runtime_user }}"
```

### 2. Stop the old service

User services are controlled through the per-user systemd instance, which needs the user's runtime dir and dbus address:

```yaml
- name: Stop the old user service
  ansible.builtin.command: systemctl --user stop "{{ app_name }}.service"
  become_user: "{{ old_runtime_user }}"
  environment:
    XDG_RUNTIME_DIR: "/run/user/{{ old_uid }}"
    DBUS_SESSION_BUS_ADDRESS: "unix:path=/run/user/{{ old_uid }}/bus"
  failed_when: false          # it may already be stopped
```

For a root-running system service, a plain `systemctl stop <name>.service` suffices.

### 3. Move the data

```yaml
- name: Move state into the new data directory
  ansible.builtin.command: mv "/home/{{ old_runtime_user }}/<old-data-path>" "{{ data_dir }}/"
```

Guard such tasks with a `stat`/`creates` check so re-runs are safe. If you reused the account name, ownership of the moved data still matches; otherwise add a `chown -R`.

Then create the system user shape and data directory:

```yaml
- name: Service user is present (nologin, no home)
  ansible.builtin.user:
    name: "{{ runtime_user }}"
    system: true
    shell: /usr/sbin/nologin
    home: "{{ data_dir }}"
    create_home: false

- name: Data directory is present
  ansible.builtin.file:
    path: "{{ data_dir }}"
    state: directory
    mode: '0750'
    owner: "{{ runtime_user }}"
    group: "{{ runtime_user }}"
```

### 4. Deploy with the role

```yaml
- name: Deploy the service
  ansible.builtin.include_role:
    name: uhlig-it.simple_systemd_service
  vars:
    program:
      name: "{{ app_name }}"
      description: "..."
      runtime_user: "{{ runtime_user }}"
      binary: https://.../<name>-linux-amd64.tar.gz   # or a local path
      data_dir: "{{ data_dir }}"
      environment:
        DATABASE_URL: "sqlite:{{ data_dir }}/app.db"
        PORT: "{{ port }}"
```

The role renders the unit (with `User=`, hardening, `WorkingDirectory=data_dir`), the `0640 root:<user>` environment file, installs the binary into `/usr/local/bin`, and enables/starts the service. Secrets must come from your vault - never copy the old `.conf` from the user's home.

### 5. Verify

```yaml
- name: Wait for the service to answer
  ansible.builtin.uri:
    url: "http://localhost:{{ port }}/"
  register: health
  until: health.status == 200
  retries: 20
  delay: 2

- name: Service is active
  ansible.builtin.systemd:
    name: "{{ app_name }}.service"
    state: started
```

Then, manually, **reboot the host** and confirm the service comes up under pid 1:

```command
sudo reboot
systemctl status <name>.service    # after boot
```

This is mandatory: old user services were enabled into a target the user manager does not pull in (see gotchas) - the reboot test proves the migration actually fixed boot-time startup.

### 6. Decommission (after verification)

```yaml
- name: Disable the old user service
  ansible.builtin.command: systemctl --user disable "{{ app_name }}.service"
  become_user: "{{ old_runtime_user }}"
  environment:
    XDG_RUNTIME_DIR: "/run/user/{{ old_uid }}"
    DBUS_SESSION_BUS_ADDRESS: "unix:path=/run/user/{{ old_uid }}/bus"
  failed_when: false

- name: Disable linger for the old user
  ansible.builtin.command: loginctl disable-linger "{{ old_runtime_user }}"

- name: Remove the old per-user unit directory
  ansible.builtin.file:
    path: "/home/{{ old_runtime_user }}/.config/systemd/user"
    state: absent
```

Lock the account for rollback instead of deleting it right away:

```command
usermod -s /usr/sbin/nologin <old_user>
passwd -l <old_user>
```

Delete it only once you are confident:

```command
userdel -r <old_user>
```

Also remove the now-unused `acl`/`zsh` packages if nothing else needs them, and delete any secret-bearing files left in the old home - the old `.conf` was only `0660` inside the user's home; your vault is the only legitimate copy of those secrets now.

## Gotchas

1. **Old user services likely never started at boot.** The old role rendered `WantedBy=network-online.target` - a target that does *not* exist in the per-user systemd manager (it only has `default.target`, `timers.target`, `sockets.target`, `paths.target`, ...). `systemctl --user enable` happily created the symlink; nothing ever pulled that target, so the service stayed down after a reboot. Timer-only jobs were the exception (`timers.target` does exist in the user manager). Confirm with a reboot test; then migrate.
2. **`/nonexistent` home + systemd = `Failed at step CHDIR`.** A service with `User=` defaults its working directory to the user's home from `/etc/passwd`. With a `nologin` user and no home, the service fails before running the binary. Always set `WorkingDirectory=` explicitly; the role does when it renders the unit.
3. **`ProtectSystem=full` makes `/usr`, `/boot`, `/etc` read-only.** Apps that write runtime state under `/etc` or `/usr` break. Keep all writable state under `data_dir` (e.g. `/var/lib/<name>`). The role's `PrivateTmp=true` also means `/tmp` is a fresh, empty directory per start - do not rely on files persisting there.
4. **Secrets and world-readable files.** The unit file must stay world-readable; put secrets only in the `0640 root:<user>` environment file. Re-render it from vault when migrating; do not copy the old file from the user's home.
5. **Port and file conflicts during cutover.** Stop the old service *before* the switch: if both run, one fails to bind the port and data files may be locked (SQLite!). Use a different test port if you want to run both side by side.
6. **Timer-only services.** Migrate the timer too: in the new setup it lives at `/etc/systemd/system/<name>.timer` (`WantedBy=timers.target`) and the unit becomes `Type=oneshot`. Verify the first trigger after the migration.
7. **UID vs name.** Reusing the account name keeps UIDs stable and avoids `chown` churn, but the account's *shape* must change (shell, home). Fresh system accounts get a UID in the system range - if you move data owned by the old UID, `chown -R`.
8. **Do not rush the cleanup.** Keep linger, the old account and the old unit files until the new service has survived a reboot. Rollback then is: re-enable the old unit, stop the new one.

## The role's variables (reference)

| Variable | Meaning |
|---|---|
| `program.name` | Unit/binary name |
| `program.binary` | Local path or `https://...` tarball URL (archive must contain `<name>` at top level) |
| `program.runtime_user` | Required - dedicated system user the service runs as |
| `program.environment` | Dict, rendered into `/etc/<name>.conf` (`0640 root:<user>`) |
| `program.data_dir` | Optional - created `0750`, owned by the user; also the working directory |
| `program.working_directory` | Optional - overrides `data_dir` as cwd |
| `program.groups` | Optional - supplementary groups for the runtime user |
| `program.parameters` | Optional - command line arguments |
| `program.timer` | Optional - `systemd.timer` directives; enables a `Type=oneshot` unit + timer |
| `systemd.target` | Ordering target (`Wants=`/`After=`), default `network-online.target` |
| `systemd.wanted_by` | Enabling target, default `multi-user.target` |
| `systemd.restart` / `systemd.restart_sec` | Restart policy (daemons only), defaults `on-failure` / `3s` |
| `systemd.start_limit_interval` / `systemd.start_limit_burst` | Start limits |
| `systemd.hardening` | Default `true`: `NoNewPrivileges`, `PrivateTmp`, `ProtectSystem=full` |