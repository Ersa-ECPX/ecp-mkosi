# Devices and diagnostics (design)

Status: a design, nothing is built. It describes what to build and what it may collect, so that it can be reviewed before code.
It extends [identity](features/identity.md) (sign in with Ersa ID) and the "management backend" half of
[provisioning](provisioning.md).

## Two separate things

| | **Devices** | **Diagnostics** |
| --- | --- | --- |
| What it is | Linking this computer to your Ersa ID so you can see and manage it from your account | Anonymous reports that help us find and fix problems |
| Needs sign-in | Yes: the person links the computer | No: works without an account |
| Identifier | A device id, tied to the account | A random install id, tied to nothing, rotated every 30 days |
| Default | Off until the person links it | **On**, one visible switch, one click to turn off |
| Setting | `devices.enabled` (turns the feature off for a company) | `telemetry.enabled` |

They never share an identifier. A diagnostics report cannot be matched to an account or to a linked device, and switching one off
does not affect the other. The operating system works fully with both off.

## What a check-in may contain

Only these fields. Anything else needs a change to this page first.

| Field | Example | Why |
| --- | --- | --- |
| `schema` | `1` | so the server can read old reports |
| `version`, `arch` | `1.0.2`, `x86_64` | which image is running |
| `slot`, `update_state` | `a`, `confirmed` / `waiting` / `rolled_back` | whether updates land safely |
| `boot_ok` | `true` | did the last boot reach the desktop |
| `encrypted`, `tpm` | `true`, `true` | is the data volume encrypted, is there a TPM |
| `hardware` | `{"cpu_family": "amd-zen4", "ram_gb": 16}` | what to test on; coarse, no serial numbers |

Never sent: file names or contents, the list of apps or bundles, browsing, user names, host name, MAC addresses, serial numbers, the
recovery key or any key, or the computer's IP address (the server must not store it; see below). A linked device also sends its
name (chosen by the person) and nothing else beyond the table.

Diagnostics adds one more thing: **failure events**, a short fixed reason code and a version. Examples: `update_rolled_back:boot`,
`boot_failed:initrd`. No logs.

`foundation devices show` prints the exact report that would be sent next (that command exists; `foundation telemetry` does not, because
diagnostics is not built), and `foundation devices status` shows the link, how the key is kept and the last attempt. What the software
sends is what the person can read.

## Devices: how a computer is linked

The account service is Prism (Ersa ID). It has the device endpoints, built and tested in its own repository (`Ersa-ECPX/prism`, the
Devices page and `/api/oauth/me/devices`); the first design below this line assumed a separate backend, which is no longer needed.

1. The computer makes a **device key** (Ed25519) the first time it is linked, never in the image. With a TPM the private key is sealed
   to it (`systemd-creds`, `/var/lib/foundation/device/key.cred`, no PCR policy so an update or a firmware change does not lose it):
   a copy of the file is useless on any other computer. Without a TPM, or if sealing fails, it is a file only root can read
   (`key.pem`), and `Devices.Status` says which (`key_protection`). A file key is sealed the first time a TPM is usable. Sealing ties
   the key to the TPM at rest; the key is unsealed into memory to sign, so root on the running computer can still use it. Only the
   public half leaves.
2. The person signs in with Ersa ID (the device flow that already exists), then runs **Link this computer** in Deck or
   `foundation devices link`. The person's own client asks foundationd for the public key (`Devices.PrepareLink`), registers it with
   Prism using the person's token (scope `devices:write`; the token never reaches foundationd), and tells foundationd the id Prism
   returned (`Devices.CompleteLink`). Prism first hands out a challenge, and the client has foundationd sign it (`Devices.SignLink`):
   Prism links the key only if the signature over `prism-device-link.<challenge>.<public key>` checks, so someone who merely knows a
   public key cannot claim it. foundationd signs only that one form and refuses any other input, so it is no signing oracle. The address of the account service is the setting `devices.url`, not an argument, so a call
   cannot send reports somewhere else.
3. From then on the computer reports with a check-in **signed by the device key** over its id, the time and the exact body. Prism
   refuses a changed body, a time more than five minutes off or not newer than the last accepted check-in (no replays), an unknown or
   unlinked device, and fields it does not know, and rate-limits the route. No account token is kept on the computer for this. The
   request is sent by a child process that has dropped root (the key is used first, then the network is touched without privilege),
   does not follow redirects and reads at most a few kilobytes of any answer.
4. **Unlinking** works from the website (the person removes the computer; its next check-in is refused) and from the computer
   (`foundation devices unlink`, Deck), which forgets the link and the key. A refusal never deletes the link by itself: Prism answers
   the same way for an unlinked computer and for other signature problems, so the person decides.

A computer belongs to one account. A computer shared by several people is linked by one person; each user's own settings are a
separate matter (settings sync, later).

## Diagnostics: how reports are sent

- **No timer of its own.** The update check already runs about every six hours (`foundation-autoupdate.timer`, with the network up).
  After it, the same run calls `Devices.CheckinIfDue`, and foundationd sends only if the computer is linked, `devices.enabled` is on
  and about five hours have passed since the last attempt, so every run reports. That is one small HTTPS request (about 1 KB) and no extra
  wake-up. A failed attempt is recorded (`Devices.Status` shows it) and not retried before that time is over. It runs even when
  `ota.auto_check` is off. Diagnostics (anonymous, no link needed) has no service yet; it would use the same ride-along.
- **When the network comes up.** A NetworkManager hook (`/usr/lib/NetworkManager/dispatcher.d/90-foundation-devices`) starts
  `foundation-devices-online.service` when a connection comes up or the internet becomes reachable (not for a captive portal), and only
  on a linked computer; the service calls `Devices.CheckinOnline`. foundationd sends if fifteen minutes have passed since the last
  attempt, and a service that cannot be reached yet (the name servers may not answer in the first second) does not use up that time, so
  the next event tries again. This is how a rename made on the website reaches the computer within seconds of its coming online, with no
  polling and no extra wake-up. It is a poll on an event, not a push: a computer that stays online all day learns of a rename at the next
  scheduled report (at most about six hours). Instant delivery, for commands such as wipe, needs a push channel and belongs to a device
  management daemon of its own. The hook has not been run against a real NetworkManager or under SELinux: if it does nothing, look for
  denials (`ausearch -m avc -ts recent`) and run `journalctl -u foundation-devices-online`.
- Without a linked device the request is anonymous: the random install id and the diagnostics fields only (not built).
- Reports go to `telemetry.url` and `devices.url` (settings, so a company can point them at its own backend or leave them empty).
  Nothing is sent when the URL is empty or the switch is off.
- Nothing runs from the initrd or before the first boot is complete.

## Where the switch lives

- **Installer:** one line on the encryption/sign-in step area says diagnostics are on and why, with the switch next to it. It writes
  `telemetry.enabled` into the installed system. A preseed file can set it.
- **Deck:** a Privacy page (diagnostics switch and "Show what is sent") and a Devices page (this computer's link, and the list of
  the person's computers from the backend).
- **Command line:** `foundation telemetry status|show|on|off`, `foundation devices status|link|unlink|list`.
- **foundationd:** `Devices` and `Telemetry` D-Bus interfaces, polkit actions `org.ersa.foundation.devices` and `.telemetry`. Reads
  need no authorization; changing the switch or the link does.

## The backend (a separate service)

An OIDC resource server, a client of the same provider as everything else. It is not part of this repository's image.

- **Devices API (built, in Prism):** `GET /api/user/devices`, `PATCH` and `DELETE /api/user/devices/{id}` for the person (session);
  `POST`, `GET` and `DELETE /api/oauth/me/devices` with the scopes `devices:write` and `devices:read`; and
  `POST /api/oauth/me/devices/{id}/checkin`, signed by the device. JSON, size limits, a fixed set of fields. Diagnostics
  (`POST /v1/diagnostics` in the first draft) is not built, and does not belong in Prism: it is anonymous and not part of an account.
- **Data:** accounts (`sub`), devices (id, public key, name, owner, linked at, revoked at), the latest check-in per device and 30
  days of history, diagnostics as counters by version and reason.
- **Retention:** raw diagnostics 30 days, then only counts. A deleted account loses its devices and check-ins. An account owner
  can export the data held about their computers.
- **No IP storage:** requests are handled without keeping the client address; access logs drop it. This must be checked on the
  deployed service before the privacy page says it.
- **Verification:** device assertions (signature, nonce not seen before, short expiry); tokens' signatures against the provider's keys.

## Not in this design (later)

Fleet management for teams (groups, staged rollouts, policy) needs an organisation model in Ersa ID and a policy channel to the
device; it builds on the device key and check-in above but is its own design. Settings sync, a TPM-bound device key, and
entitlements (licenses) are also later.

## Plan, with how each step is tested

1. **Prism: the account side. Done.** The Devices page, the endpoints and the scopes, with tests (signature, replay window, other
   people's devices, unknown fields). Live in the `Ersa-ECPX/prism` repository; not deployed to id.ersa.dev yet.
2. **foundationd backend and interface. Done** (`foundationd/backends/devices.py`, the Devices D-Bus interface, the settings
   `devices.enabled` and `devices.url`, the polkit action `org.ersa.foundation.devices`, the ride-along in `foundation-autoupdate`).
   Tested with unit tests (key as file and sealed, the link challenge, report, signature, once a day, failures, the sender against a
   local server, the privilege drop) and against a real foundationd on a private bus. The TPM seal and unseal also round-trip on a real
   TPM 2.0 (`ECP_TEST_REAL_TPM=1`, off by default). Not tested against a real Prism, as root, or on a booted image. The package
   requires `python3-cryptography`: the Tumbleweed index provides it (python313-cryptography, the same way as python3-gobject).
3. **The person's side. Done:** `foundation devices status|show|link|unlink|checkin` (`foundation/devices.py`) and a "This computer" part
   on Deck's Ersa ID page. There is one sign-in for everything: signing in to Ersa ID (Deck or `foundation identity login`) asks for
   `devices:write` along with the rest (the default of `identity.scopes`), and linking and unlinking use that stored sign-in. If the
   account service does not grant the permission the sign-in still works and linking says to sign in again. Linking reports at once, so the
   computer shows on the account with its report. **Unlinking removes the computer from the account on the website first** (DELETE with the
   stored sign-in) and then forgets the key here; if the account cannot be told nothing changes, and `foundation devices unlink --local`
   forgets it on this computer only. Tested with fakes for the account service and foundationd; never against a real Prism or on a booted
   image. The app this computer signs in with (`identity.client_id`) must be owned by a site admin and allowed `devices:write`, or Prism
   refuses with "insufficient scope".
4. **The name.** The account's name for the computer (renamed on the website, or given at linking) comes back in every check-in answer,
   and the host name follows it when it changes (`devices.sync_hostname`, on by default): spaces become hyphens, the ends are trimmed, only
   letters, digits and hyphens stay, at most 63 characters ("Ersa Workstation" is `Ersa-Workstation`). A host name set by hand afterwards
   stays until the name changes again. Checking in sends a `User-Agent`: a CDN in front of the account service refuses the bare Python one
   with an HTTP 403.
5. **Deck and installer:** the diagnostics switch, with screenshots in English and Hindi.
6. **Diagnostics service,** when its home is decided.

## Open questions

- Where the backend is hosted, and its stack. It needs to verify JWTs and keep a small database; this design does not depend on it.
- Whether the install id rotates every 30 days or per release.
- Whether an organisation can require diagnostics for its own computers (a policy), or only the person decides.
- What the privacy page promises about the server, once the deployed service has been checked.
