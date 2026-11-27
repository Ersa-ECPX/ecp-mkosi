# Identity (sign in with Ersa ID)

## What it is

An OpenID Connect client in `foundation`. The provider is a setting: Ersa ID by default, any other OIDC provider if a company names
one, or none. The device, license and telemetry backends are separate services, and each is an OIDC client of the same provider.
The operating system works fully without an identity.

## Using it

```
foundation identity status      # whether you are signed in
foundation identity check       # does the provider offer what this computer needs (discovery, device sign-in, refresh, scopes)
foundation identity login       # prints an address and a code: approve it on any device
foundation identity logout      # revokes the session at the provider (if it offers that) and forgets it here
```

The provider needs OIDC discovery (`/.well-known/openid-configuration`) and the device authorization grant (RFC 8628). A refresh
token (`offline_access`) keeps you signed in; a revocation endpoint makes sign-out real.

## In Deck

Deck's **Ersa ID** page (sidebar, "Account") shows your photo, name and email from the provider and
**Sign In** / **Sign Out** (not the provider's address or client id). Sign In shows a code and a button that opens the sign-in page; approving it on any device finishes it.
There is no devices list: no devices service exists yet.

**Offered once after the first login.** A user service (`foundation-ersa-id-offer.path`) waits for Initial Setup to finish, then runs
`ersa-deck --offer-sign-in`. If a provider is set up and nobody is signed in, a small window offers **Sign In** (opens Deck on the
Ersa ID page) or **Not Now**. Either answer is recorded in `~/.local/state/ersa-id/offered` and the offer never comes back; nothing
is forced. With `identity.enabled=false` or no provider, it never appears. This is per person, not the machine's `provisioned` flag:
that latches when Initial Setup ends, the same moment the offer is due, and the session is per account anyway.

## Choosing or switching off the provider

Settings (`foundation config list`), and so also keys of a [preseed file](deploy.md):

| Key | Meaning |
| --- | --- |
| `identity.enabled` | `false` turns identity off: no commands work, nothing contacts a provider |
| `identity.issuer` | the provider's issuer address (`https://...`) |
| `identity.client_id` | this computer's client id there (a public client, no secret) |
| `identity.scopes` | default `openid profile email offline_access` |

```
identity.issuer=https://sso.example.com
identity.client_id=ecp-workstations
```

The defaults are Ersa ID's: issuer `https://id.ersa.dev` and the public client "Ersa Compute Platform" registered there. A company
sets both keys (they go together) in its preseed file; `identity.enabled=false` turns the feature off.

## How it works

`foundation/identity.py` uses only the standard library. Discovery refuses a provider whose metadata names another issuer, and
plain `http` is accepted only for this machine or a private network (tests). The device flow polls as the provider asks (`slow_down`
included). The ID token arrives from the token endpoint over TLS, so only issuer, audience and expiry are checked, not the signature
(OpenID Connect Core 3.1.3.7). A backend that receives a token from someone else must verify the signature against the provider's keys.
The session (access and refresh token, account claims) is a 0600 file in the person's state directory
(`~/.local/state/ersa-id/`); the access token is refreshed when it nears expiry and the provider's rotated refresh token is kept.

## Not in the installer or Initial Setup (by design)

Signing in is not part of installing or of GNOME Initial Setup. Only the provider settings travel with an install (a preseed writes
them into the installed system's configuration). Initial Setup creates the local account as usual; the person signs in afterwards, as
that user, and the session belongs to that account.

## Not built

Linking the session to the local account, a keyring for the tokens (the file is the stopgap), enrolling
the device with the backend, entitlements (licenses) and telemetry, YubiKey and passkey sign-in on this side, a shared auth library
for the web apps.

## Tested

The client against a fake provider on the loopback address (discovery, the device flow including pending, refusal and expiry checks of
the ID token, refresh with rotation, a revoked grant, sign-out with revocation, the session file's mode) and the preseed keys: unit
tests; Deck's page and the offer window under a virtual display (their logic, not a real sign-in). Against the real Ersa ID (Prism) only discovery, the provider check and the start of a device sign-in were run (all accepted); an approved sign-in, refresh, sign-out and the profile photo were not, and nothing ran on a booted image.
