# ma-ncs-firmware-releases

Public OTA release mirror for [MA-NCS firmware](https://github.com/LISHA-CWU/ma-firmware).
Published automatically by that repo's `release.yml` on every merge to `main`
that bumps `firmware/VERSION`.

## manifest.json

```json
{
  "schema": 1,
  "boards": {
    "<board>": {
      "version": "<semver>",
      "url": "<stable download URL for dfu_application.zip>",
      "sha256": "<imgtool-embedded image hash>",
      "released_at": "<ISO-8601 UTC timestamp>"
    }
  }
}
```

Fetch unauthenticated at:

```
https://raw.githubusercontent.com/LISHA-CWU/ma-ncs-firmware-releases/main/manifest.json
```

`sha256` is the hash MCUboot's `imgtool` embedded in the signed image's TLV
(read from `imgtool dumpinfo`), not an independent hash of the zip — a DFU
client can verify the flashed image against this same value post-update.

## Releases

Each GitHub Release here is tagged `v<semver>` matching a `firmware/VERSION`
bump on the source repo, and carries two assets:

- `dfu_application.zip` — MCUmgr/SMP DFU package
- `merged.hex` — full image (MCUboot + signed app) for the first, SWD-only flash
