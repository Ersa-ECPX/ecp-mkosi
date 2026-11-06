# Provisioning and device management

Status: the provisioning *state* is implemented (see `foundation/README.md#provisioning`). Everything
about a management server and a custom first-boot application is a proposal.

## Two different problems

1. **First-boot front end**: the wizard a person sees (language, keyboard, network, account, and
   possibly enrolment). Today GNOME Initial Setup.
2. **Management backend (MDM)**: a server and a device-side agent that enrol the device, apply
   policy and settings, deploy applications, control the update channel, report inventory and
   compliance, and act remotely.

They meet at one fact: *this device is provisioned*. Keep them separate; share the device side.

## The device side (built)

`foundationd` is the on-device management plane: one D-Bus API, polkit on every change, jobs. A
management agent is **one more client of it**. It talks to the remote server and applies what it is
told through the same calls a person uses (`Config`, `Apps`, `Updates` later, `Boot`), so there is a
single authorized, auditable path for every change and no second tool that edits the system.

State that already exists: `system.provisioned` (one-way latch), `ota.*` and other settings with a
schema (stored; the agent would set `ota.url` and the channel), and a job log.

Needed when an agent exists: a device identity created on first boot (never baked into the image),
enrolment secrets held as systemd credentials (not settings), and a place to record who manages the
device. These are not built.

## First-boot front end: options

| Option | What it takes | Good for | Cost |
| --- | --- | --- | --- |
| **Upstream GNOME Initial Setup, unchanged** (today) | nothing | consumer setup; enrolment after the account exists | pages can only be skipped or reordered; no place for an enrolment page |
| **Upstream plus our own second-stage app** | a small libadwaita app, run after setup while `system.provisioned` is false | enrolment after the local account exists | two wizards |
| **Our own first-boot app** replacing Initial Setup | a libadwaita app that does language, keyboard, network, enrolment and account creation (through AccountsService) and then calls `CompleteProvisioning` | enrolment *before* the account (zero-touch, directory sign-in); one consistent flow; shares UI code with the planned installer | we own every page, translation and accessibility fix |
| **Fork GNOME Initial Setup** | a patched copy kept in step with every GNOME release | one extra page in the GNOME flow | rebasing every six months on a rolling base; the extra page is a small delta for a large burden |

If a page needs adding only, carry it as a patch series in the RPM (reviewed at each release), not a
fork. If enrolment must come before the account, write our own first-boot app: it is less work than
maintaining a fork, and it is the same toolkit as the installer.

## The server

Not decided, and not needed yet. Questions that decide it: who manages the devices (a company, a
school, a family, a developer's own fleet), which controls matter (settings and policy, application
deployment, the update channel, remote wipe, compliance and inventory, sign-in through a directory),
and whether devices are reached directly or through a relay. Existing open-source and commercial
tools cover parts of this (inventory and queries, configuration management, OTA fleets, identity);
none has been evaluated for this project. Build a server only for the part nothing existing covers.

## Suggested order

1. (done) The provisioning state and its two ways to latch it.
2. Prove the update path (OTA) and the installer; both depend on the slot layout.
3. Decide the first-boot front end from the enrolment requirement above.
4. Design the device identity and the enrolment protocol with the server choice, then write the agent
   as a `foundationd` client.
