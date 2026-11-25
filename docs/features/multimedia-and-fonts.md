# Multimedia, hardware video decode and fonts

## What the image contains

From openSUSE's own repository (nothing third-party):

- **Audio:** MP3 (lame, mpg123), AAC (fdk-aac, faad2), Opus, Vorbis, FLAC, Speex.
- **Video:** VP8/VP9, AV1 (dav1d, aom, SVT-AV1), Theora.
- **Images and subtitles:** WebP, HEIF, JPEG XL, libass.
- **Frameworks:** GStreamer good, good-extra, bad, ugly and libav plugins, and FFmpeg.
- **Hardware decode and encode:** VA-API (`Mesa-libva`, `intel-media-driver`, `libva-utils`), the VDPAU bridge, Intel oneVPL.

openSUSE's FFmpeg is limited in codecs (no H.264, H.265 or AAC decoding), so the full set is an optional bundle.

## Full codecs

```
foundation compat install @codecs
```

Installs the `codecs` bundle ([repositories and bundles](repositories-and-bundles.md)): Packman's full FFmpeg libraries
(replacing the base's, for every FFmpeg major version the base has), x264, x265, de265 and the patent-encumbered GStreamer
plugins, as one system extension. It shows a notice, asks before enabling the Packman repository and downloads packages
straight from Packman; they are never part of the image. Checking that these codecs are allowed where you live is up to you.

Built in a Tumbleweed container (29 packages, 265 files, 87 s); the extension's `libavcodec.so.62` has the H.264 decoder and
the x264, x265, de265 and faac GStreamer plugins are present. Flatpak applications are unaffected
([applications](applications.md)).

## Fonts

Noto fonts cover every script the installer's language list and greeting use (about 300 MB): CJK (Noto Sans CJK), Thai, Lao,
Khmer, Myanmar, the Indic scripts, Arabic (UI, Naskh, Nastaliq; covers Persian and Urdu), Hebrew, Armenian, Georgian,
Ethiopic, symbols and math. Before they were added those scripts showed as boxes in the welcome greeting. The font cache is
built at image build and rebuilt on first boot of an installed system (`ersa-var-init.service`), because `/var` starts empty.
The interface and monospace fonts are described in [branding](branding.md).

## Limits

- Full codecs need network and Packman; offline machines get the limited set.
- Which GPU does hardware decode depends on its driver and the application.
- Tibetan has no Noto package in the repository and is not covered.

See also: [repositories and bundles](repositories-and-bundles.md), [installer](installer-app.md).

## Tested

The image's package list is built and the `@codecs` bundle was built and checked in a container against the live repositories. Not tested on a running desktop: video playback in Firefox or GNOME after `@codecs`, hardware decode on a real GPU, and rendering of the added fonts.
