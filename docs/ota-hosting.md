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
check then rejects. They can also offer an older signed version; an install refuses it without `--force`, but the automatic check never offers one.
The rest of this page compares other places to host.

A host needs only plain HTTPS downloads at a stable URL: `latest-<arch>.json` and one archive per version
(`ecp_<version>-x86_64.tar.zst`, about 1.9 GB compressed). Check current prices and limits before choosing.

| Option | Notes |
| --- | --- |
| Local server (`scripts/serve-update`) | testing; no TLS; a QEMU guest reaches it at `10.0.2.2` |
| GitHub Releases | free; 2 GiB per file, which the image will outgrow |
| Object storage with a custom domain (Cloudflare R2, Backblaze B2) | no or low egress fees; the likely real service |
| A small VPS with Caddy or nginx | full control, you run it |
| SourceForge, openSUSE mirrors/OBS | not suitable |

## Whatever you choose

- **Serve it from a domain you control** (for example `updates.<your-domain>`, a CNAME to the
  backend), and use that URL everywhere. Then the storage can change later without
  touching a single installed machine.
- Sign each archive (`scripts/make-update-archive --sign`) with the image-update key kept **outside** this repository; the public key
  ships in the image (`/usr/share/foundation/keys/`). Without it, an install needs `--unsigned`, which is for tests only.
- Keep old versions for a while: a machine that is several versions behind can still reach the
  newest one, because each update is a full image.
