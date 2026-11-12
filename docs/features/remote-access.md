# Remote access

**Status:** built; the logic is tested with a fake systemd and firewall, and the file handling with real files. Tried once on
a virtual machine: turning SSH on and then logging in from the host with a key worked (host keys had to be made by hand
that time, which is fixed since). Not yet checked there: that password login and root login are refused, that it comes back
after a reboot, the firewall step on a default zone that blocks SSH, and the SELinux label of a new `authorized_keys`.

## What it does for you

The Ersa Compute Platform ships the SSH server but leaves it **off**. `foundation remote` turns it on safely, so you can
log in from another computer, and manages the keys that are allowed to.

```
foundation remote status                     # is SSH on, how people log in, is the firewall open, how many keys
foundation remote key add ~/.ssh/id_ed25519.pub     # allow a key (a .pub file, the key itself, or - for standard input)
foundation remote key list [--user NAME]
foundation remote key remove NAME SHA256:...        # by fingerprint, as `key list` shows
foundation remote ssh on [--allow-password]         # keys only, unless you allow passwords
foundation remote ssh off
```

Turning SSH on:

- **will not lock you out of a useless server:** with password login off and no key for any user, it refuses and says so;
- **keeps root out:** root login is always off;
- **opens the firewall only when needed:** if the default zone already allows SSH it leaves it alone;
- **checks its own work:** the settings are tested with `sshd -t` before the service is touched, and rolled back if the
  server rejects them.

Changing who may log in needs an administrator (polkit action `org.ersa.foundation.remote`, asked again after a while).

## Where things are

| What | Where |
| --- | --- |
| The settings foundation writes | `/etc/ssh/sshd_config.d/50-foundation.conf` |
| A user's allowed keys | `~/.ssh/authorized_keys` |
| The service | `sshd.service` |

`sshd` uses the **first** value it finds for a setting, and reads the drop-ins in name order. To override what foundation
wrote, put a file that sorts before it (for example `10-local.conf`). Your own edits to `/etc/ssh/sshd_config` live in the
writable `/etc` and survive updates; a drop-in is cleaner because the shipped file can change without clashing.

Keys are written through directory handles that never follow links, because the service works as root inside your home
directory. A plain key is added; keys with options (`command=`, `from=`) are refused here, and are shown (with "has
options") if you added them yourself.

## Limits

- SSH only. Remote desktop is not part of this.
- No per-key options, no certificates, no changing the port yet.
- It cannot see whether something else (a router, a cloud firewall) blocks the port.
- It manages the machine you run it on; managing many machines is a separate, later piece of work.

See also: [management service](management-service.md), [foundation CLI](foundation-cli.md).
