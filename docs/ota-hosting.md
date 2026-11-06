# Where to host updates

What `systemd-sysupdate` needs from a server (a `url-file` source): plain HTTPS file downloads at a
stable URL, with a `SHA256SUMS` manifest (and a detached `SHA256SUMS.gpg`) listing every file at
that base URL. Not a web page, not a download wizard. Upload the payloads first and `SHA256SUMS` last,
so a client never sees a manifest naming a file that is not there yet.

Payload size today: the root image is about 1.4 GiB compressed, the kernel image about 0.1 GiB.
Prices and limits below are from memory and change: check them before choosing.

| Option | Good for | Notes |
| --- | --- | --- |
| **Local server** (`scripts/serve-update`) | testing | no TLS; QEMU guest reaches it at `10.0.2.2` |
| **GitHub Releases** | early public builds | free; one file may not exceed 2 GiB (the image will grow toward it); URLs are `releases/download/<tag>/<file>`; there is no directory, so a manifest listing only the newest release can use `releases/latest/download/` |
| **Cloudflare R2** | a real update service | no egress fees; a free tier of about 10 GB of *storage* (three versions of about 1.5 GiB each fit); S3-compatible; use a custom domain |
| **Backblaze B2** | similar | cheap storage, egress rules differ; check them |
| **A small VPS** (for example Hetzner) with Caddy or nginx | full control | a few euros a month with generous traffic; range requests, logs and atomic publishing under your control; you run it |
| **SourceForge** | not recommended | built for people choosing a mirror in a browser: downloads go through redirects, and automated clients can be served pages instead of files, with throttling. Nothing in it is designed for a machine fetching a manifest and files by name |
| **openSUSE mirrors / OBS** | not applicable | would mean building the image on OBS, which this project does not do |

## Whatever you choose

- **Serve it from a domain you control** (for example `updates.<your-domain>`, a CNAME to the
  backend), and put *that* in the transfer files. Then the storage can change later without
  touching a single installed machine.
- Sign `SHA256SUMS` with a key kept **outside** this repository, and ship the public key in the image
  (`/usr/lib/systemd/import-pubring.pgp`). Without that, the transfer files must say `Verify=no`.
- Keep old versions for a while: a machine that is several versions behind can still reach the
  newest one, because each update is a full image.
