# Installer and Deck translations

The graphical installer and Deck use separate gettext domains, `ersa-installer` and `ersa-deck`. Thirteen languages have catalogs: Hindi (`hi`), German (`de`), French (`fr`), Spanish (`es`), Brazilian Portuguese (`pt_BR`), Italian (`it`), Russian (`ru`), Japanese (`ja`), Simplified Chinese (`zh_CN`), Korean (`ko`), Arabic (`ar`), Bengali (`bn`) and Tamil (`ta`). The installer switches its own pages and controls as soon as a language is chosen; choosing English restores English. Deck follows the session language (`LANGUAGE`, then `LC_ALL`, `LC_MESSAGES`, or `LANG`). Languages without a catalog show English. The installer's language picker is broader than the catalog list; for example, `es_MX` can use an `es` catalog, while `pt_PT` does not use `pt_BR`.

Messages live in `foundation/data/po/<domain>/`. To add a language, create `<lang>.po` in both domains and run `update` to populate the messages in POT order. Translate every `msgstr`, including every required plural form. To correct a translation, edit the `.po` file directly and run from `foundation/`:

```sh
python3 tools/i18n.py check
python3 tools/i18n.py compile --out /tmp/ecp-locale
```

The RPM spec compiles catalogs during `%install`; no gettext command line tools are needed. The installer RPM owns its catalog and the Deck RPM owns its catalog.

All catalogs are initial, machine-assisted translations without native-speaker review; terms such as "slot" (written as "system copy"), "bundle" and the boot-choice buttons need a native speaker's judgement before the wording is final. Time zone and city names come from the system's time zone database and stay English. Catalog compilation, model tests, and an RPM build without `%check` ran locally. GUI tests require `ERSA_GUI_TESTS` and a working display; The installer and Deck were rendered under a virtual display in Hindi, Japanese, Tamil, Arabic (right-to-left layout mirrored correctly) and, for Deck, Hindi, and looked right; the other languages were not looked at. A booted image and the actual fonts present on the installer medium have not been verified.

`mkosi.conf` lists `google-noto-sans-fonts` for Latin and Cyrillic, `google-noto-sans-cjk-fonts` for Japanese, Simplified Chinese, and Korean, `google-noto-sans-arabic-fonts` for Arabic, `google-noto-sans-bengali-fonts` for Bengali, and `google-noto-sans-tamil-fonts` for Tamil. None of these requested scripts lacks a listed Noto family. This is a configuration check, not a font-rendering or image-content test.

The 17 messages of the installer's encryption page are translated into Hindi only; in the other languages they show English until translated (`python3 tools/i18n.py check` lists them, `--strict` makes them errors). A catalog message with no translation is left out of the compiled `.mo`, so the English text shows, never a blank.
