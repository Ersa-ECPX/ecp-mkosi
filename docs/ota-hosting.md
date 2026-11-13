# Where to host updates

**Today: GitHub Releases.** The default `ota.url` is `https://github.com/Ersa-ECPX/ecp-mkosi/releases/download/updates`. The feed `latest-x86_64.json` is a file on a fixed
release named `updates`; the archive it names is the update of the versioned release `v<version>`. After publishing a release (not a
draft), publish the feed for it:

```sh
scripts/release 0.1.2 --no-build --sign --draft     # then publish the draft on GitHub
scripts/publish-feed 0.1.2                          # creates the release "updates" the first time; --dry-run shows the feed
```

Until the first `publish-feed`, the address answers 404 and the update window says it cannot reach the update source. The feed is
not signed (the archive is). Anyone who can edit the release `updates` can point machines at a different archive, which the signature
check then rejects. They can also offer an older signed version: `foundationd` does not refuse downgrades yet.
The rest of this page compares other places to host.

What `foundation update install` needs from a server: plain HTTPS file downloads at a stable URL, one update archive per version
(`ecp_<version>-x86_64.tar.zst`, built by `scripts/make-update-archive`; the manifest and its signature are inside the
archive). Not a web page, not a download wizard. There is no index and no automatic check yet: a client is given the URL.

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
  backend), and use that URL everywhere. Then the storage can change later without
  touching a single installed machine.
- Sign each archive (`scripts/make-update-archive --sign`) with the image-update key kept **outside** this repository; the public key
  ships in the image (`/usr/share/foundation/keys/`). Without it, an install needs `--unsigned`, which is for tests only.
- Keep old versions for a while: a machine that is several versions behind can still reach the
  newest one, because each update is a full image.
