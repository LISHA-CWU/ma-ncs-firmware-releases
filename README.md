# ma-ncs-firmware-releases

Public OTA release mirror for [MA-NCS firmware](https://github.com/LISHA-CWU/ma-firmware)
(board `rma`, ISP1807-LR / nRF52840). This repo holds only **build outputs** —
signed DFU packages, per-release test specs, and a version manifest. It is
published automatically by CI in the source repo on every merge to `main`
that bumps `firmware/VERSION`. No firmware source code lives here.

Same manifest schema and CI scripts as
[`champs-firmware-releases`](https://github.com/LISHA-CWU/champs-firmware-releases)
(the CHAMPS/HRPC equivalent of this repo) — a client polling either repo
parses one manifest format regardless of which device family it's talking to.

## manifest.json

The registry every update client polls, keyed by board:

```json
{
  "schema": 1,
  "boards": {
    "rma": {
      "latest": "0.1.3",
      "versions": {
        "0.1.3": {
          "zip_url": "https://github.com/LISHA-CWU/ma-ncs-firmware-releases/releases/download/v0.1.3/dfu_rma_v0.1.3.zip",
          "zip_sha256": "...",
          "image_hash": "...",
          "tests_url": "https://github.com/LISHA-CWU/ma-ncs-firmware-releases/releases/download/v0.1.3/tests_v0.1.3.json",
          "released_at": "2026-08-27T12:00:00Z",
          "notes": "..."
        }
      }
    }
  }
}
```

Fetch unauthenticated at:

```
https://raw.githubusercontent.com/LISHA-CWU/ma-ncs-firmware-releases/main/manifest.json
```

- `zip_sha256` — SHA-256 of the `.zip` asset itself, for download integrity.
- `image_hash` — the hash MCUboot's `imgtool` embedded in the signed image's
  TLV (read from `imgtool dumpinfo`), *not* the same as `zip_sha256`. A DFU
  client can verify the image actually running post-update (e.g. via MCUmgr's
  image-state command) against this value, independent of how the zip was
  transported.
- `tests_url` — the post-update sampling-rate/conformance test spec for that
  release (see below).

## Releases

Each GitHub Release (tagged `vX.Y.Z`) carries three assets:

- `dfu_rma_vX.Y.Z.zip` — signed, nRF Connect Device Manager / MCUmgr SMP
  compatible DFU package.
- `merged_rma_vX.Y.Z.hex` — full image (MCUboot + signed app), for the
  first, SWD-only flash before a device is DFU-capable. (`champs-firmware-releases`
  does not publish this — every MA-RMA unit needs one first SWD flash since
  there is no factory-programmed bootloader yet.)
- `tests_vX.Y.Z.json` — the post-update stream conformance spec for that
  release: which BLE streams to subscribe to, for how long, and the
  expected rate/tolerance per stream. Schema:

  ```json
  {
    "schema": 1,
    "streams": [
      {
        "key": "signal",
        "nominal_hz": 3333,
        "method": "count",
        "duration_s": 10,
        "tolerance_pct": 3
      }
    ],
    "inter_stream_quiesce_s": 3
  }
  ```

  `method` is `"count"` (count notifications over `duration_s`, compare to
  `nominal_hz * duration_s` within `tolerance_pct`) or `"timestamp_delta"`
  (measure the interval between consecutive notifications instead — used for
  streams too sparse for packet-counting to resolve within `tolerance_pct`,
  gated additionally by `min_packets`). `inter_stream_quiesce_s` is the pause
  between subscribing to consecutive streams in the sequence.

Anyone can install the DFU package directly with Nordic's **nRF Connect
Device Manager** app, independent of any custom client.
