# Provisioning and device management

## Two problems

1. **First-boot front end:** the wizard a person sees (language, keyboard, network, account). Today this is GNOME Initial Setup.
2. **Management backend (MDM):** a server and a device agent that enrol the device, apply policy, deploy applications, control
   the update channel and report inventory. Not built, no server chosen.

They meet at one fact: this device is provisioned.

## What is built

`system.provisioned` (and `system.provisioned_at`) is state kept by `foundationd` in `/var/lib/foundation/state.json`. It
latches one way and creates the marker `/var/lib/foundation/provisioned`, which first-boot units use in
`ConditionPathExists=!`. It is set in two ways:

- `Manager.CheckProvisioning` (no authorization, the caller supplies no evidence): looks for the `~/.config/gnome-initial-setup-done`
  file of an administrator (member of `wheel`, uid at least 1000, regular file owned by that user, not a symlink). A user path
  unit, `foundation-provisioning.path`, calls it after the account's first login.
- `Manager.CompleteProvisioning` (polkit `provision`): for any other front end. A second call returns `AlreadyProvisioned`.

A factory reset that wipes `/etc` and `/var` returns the machine to unprovisioned (not implemented as a command). Identity is
created on first boot and never baked into the image. `foundation deploy install --preseed FILE` can seed settings, language,
time zone and host name on the target; it does not mark the machine provisioned.

## A management agent

An agent would be one more client of `foundationd`, using the same calls a person uses (`Config`, `Apps`, `Updates`, `Boot`), so
there is a single authorized, auditable path for every change. Still needed: a device identity made on first boot, enrolment
secrets as systemd credentials, and a record of who manages the device. None of it exists.

## First-boot front end options

| Option | Good for | Cost |
| --- | --- | --- |
| Upstream GNOME Initial Setup (today) | consumer setup | pages can only be skipped or reordered; no enrolment page |
| Plus our own second-stage app | enrolment after the account exists | two wizards |
| Our own first-boot app | enrolment before the account; one flow shared with the installer | we own every page, translation and accessibility fix |
| Fork Initial Setup | one extra page | rebasing every six months |

If only a page is needed, carry a patch series in the RPM rather than a fork. If enrolment must come before the account, write
our own app.

## Order

1. Done: the provisioning state.
2. Prove the update path and the installer.
3. Pick the first-boot front end from the enrolment requirement.
4. Design identity and the enrolment protocol together with the server choice, then write the agent as a `foundationd` client.

## Tested

The provisioning state (a one-way latch, evidence check, explicit call) is built and unit-tested with fake accounts and a private bus; not run on a booted image. A management server, a device agent and a custom first-boot application are proposals only.
