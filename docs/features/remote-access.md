# Remote access

## What it is

The image ships the SSH server turned off. `foundation remote` turns it on safely and manages the keys that may log in.

```
foundation remote status                            # SSH on or off, login methods, firewall, key count
foundation remote key add ~/.ssh/id_ed25519.pub     # a .pub file, the key itself, or - for stdin
foundation remote key list [--user NAME]
foundation remote key remove NAME SHA256:...        # by fingerprint, as `key list` shows
foundation remote ssh on [--allow-password]         # keys only unless passwords are allowed
foundation remote ssh off
```

Turning SSH on:

- refuses if password login is off and no user has a key;
- always keeps root login off;
- opens the firewall only if the default zone does not already allow SSH;
- checks the settings with `sshd -t` before touching the service and rolls back if the server rejects them.

Changing who may log in needs an administrator (polkit action `org.ersa.foundation.remote`).

## How it works

foundation writes `/etc/ssh/sshd_config.d/50-foundation.conf`; keys live in each user's `~/.ssh/authorized_keys`; the service
is `sshd.service`. `sshd` uses the first value it finds and reads drop-ins in name order, so to override foundation add a file
that sorts earlier (such as `10-local.conf`).

Keys are written through directory handles that never follow links, because the service works as root in your home directory.
Plain keys are added; keys with options (`command=`, `from=`) are refused, and shown as "has options" if you added them
yourself.

## Limits

- SSH only; no remote desktop.
- No per-key options, certificates or port change.
- It cannot see whether a router or cloud firewall blocks the port.
- It manages the local machine only.

See also: [management service](management-service.md), [foundation CLI](foundation-cli.md).

## Tested

The logic is tested with a fake systemd and firewall and the file handling with real files. Tried once on a virtual machine: turning SSH on, then logging in from the host with a key, worked. Not checked there: that password and root login are refused, that it survives a reboot, the firewall step on a zone that blocks SSH, and the SELinux label of a new `authorized_keys`.
