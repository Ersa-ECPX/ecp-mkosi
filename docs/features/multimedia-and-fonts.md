# Multimedia, hardware video decode and fonts

**Status:** the image's package list is built; the full-codecs bundle (`@codecs`) was built and checked in a container
against the live repositories. Not tested on a running desktop: video playback in Firefox or GNOME after installing
`@codecs`, hardware decode on a real GPU, and rendering of the added fonts.

## What the image contains

From openSUSE's own repository (no third party):

- **Audio:** MP3 (lame, mpg123), AAC (fdk-aac, faad2), Opus, Vorbis, FLAC, Speex.
- **Video:** VP8/VP9, AV1 (dav1d, aom, SVT-AV1), Theora.
- **Images and subtitles:** WebP, HEIF, JPEG XL, libass.
- **Frameworks:** GStreamer good, good-extra, bad, ugly and libav plugins, and FFmpeg.
- **Hardware video decode and encode:** VA-API (`Mesa-libva`, `intel-media-driver`, `libva-utils`), the VDPAU bridge and
  Intel oneVPL.

openSUSE builds its FFmpeg "limited in the number of codecs supported": no H.264, H.265 or AAC decoding. That is why the
full set is an optional bundle.

## The full codecs: `foundation compat install @codecs`

Installs the **`codecs`** bundle from the catalog: Packman's full FFmpeg libraries (replacing the base's limited ones, for
every FFmpeg major version the base has), x264, x265, de265 and the patent-encumbered GStreamer plugins, as one
[system extension](repositories-and-bundles.md). It shows a notice, asks before turning on the third-party Packman
repository, and downloads the packages straight from Packman to the computer: they are never part of the image. Some of
these codecs are patent-encumbered in some countries; checking that using them is allowed where you are is up to you.

Built in a Tumbleweed container on a base with GStreamer and the limited FFmpeg: 29 packages, 265 files, 87 s; the
extension's `libavcodec.so.62` has the H.264 decoder and the x264, x265, de265 and faac GStreamer plugins are present.

Flatpak applications bring their codecs from the Flatpak runtime and are not affected ([applications](applications.md)).

## Fonts

Noto fonts for every script the installer's language list and greeting use (about 300 MB): Japanese, simplified and
traditional Chinese, Korean (Noto Sans CJK); Thai, Lao, Khmer, Myanmar; Devanagari, Bengali, Gujarati, Gurmukhi, Kannada,
Malayalam, Tamil, Telugu, Odia, Ol Chiki, Meetei Mayek and Sinhala; Arabic (UI, Naskh, Nastaliq, which also covers Persian
and Urdu); Hebrew, Armenian, Georgian, Ethiopic; symbols and math. Before they were added, those scripts showed as boxes
in the welcome greeting. The font cache is built at image build time and rebuilt on first boot of an installed system
(`ersa-var-init.service`), because `/var` starts empty.

## Limits

- Full codecs need network and the Packman repository; offline machines get the limited set.
- Hardware decode is only present as software; whether a given GPU is used depends on its driver and the application.
- Tibetan has no Noto package in the repository, so it is not covered.

See also: [repositories and bundles](repositories-and-bundles.md), [installer](installer-app.md).
