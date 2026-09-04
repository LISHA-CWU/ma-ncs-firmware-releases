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

## How a release gets here

**Nothing in this repo is edited by hand, and nothing is contributed here
directly.** Every file and every GitHub Release is written by the
`release.yml` workflow in [`ma-firmware`](https://github.com/LISHA-CWU/ma-firmware).
All changes — firmware, tests spec, release notes — go through that repo's
**branch → pull request → CI checks + hardware test → merge to `main` →
automatic release** flow; see its README's *Contributing* section.

End to end, a release is produced like this:

1. A developer opens a PR against `main` in `ma-firmware` that bumps
   `firmware/VERSION` (`MAJOR.MINOR.PATCHLEVEL`). The PR's `build.yml` refuses to pass if
   firmware paths changed without a version bump, builds the firmware in the
   NCS v3.1.1 container, and uploads the DFU package for on-device testing.
2. The PR is reviewed, tested on hardware, and merged. **Merging to `main`
   is the release action** — there is no manual tag or upload step.
3. `release.yml` in the source repo runs on the push to `main`:
   - `check-version` reads `firmware/VERSION` and checks whether release `vX.Y.Z`
     already exists here. If it does (doc-only merge, or a re-run), the
     workflow stops and nothing changes in this repo.
   - `build` compiles `rma` from a clean `west` workspace pinned to
     NCS v3.1.1, and records the zip's SHA-256 plus the `imgtool` image hash
     of the signed application.
   - `publish` stamps `ci/tests.json` as `tests_vX.Y.Z.json`, adds the version to
     `manifest.json` (URLs, checksums, `released_at`, source commit) and
     sets it as `latest` for the board, commits that to `main` here, creates
     GitHub Release `vX.Y.Z` with the assets, and pushes the same tag to the
     source repo for traceability.
4. Within moments of the `manifest.json` commit, update clients polling the
   raw manifest URL above see the new `latest` and start offering it to
   devices running an older version.

Because step 4 is immediate, a merge to `main` in the source repo is a
deployment to every device in the field that checks for updates. That is the
reason the branch → check → merge discipline exists.

### Versioning and re-publishing

- A version is published at most once. To ship a fix, bump
  `firmware/VERSION` again in a new PR — never delete and re-create a release to reuse
  a number that clients may already have cached.
- If a publish run fails half-way (release created, manifest not updated, or
  vice versa), the operator deletes the partial release/tag here and the
  matching tag on the source repo, then re-runs the workflow. Only in that
  situation is anything in this repo touched by hand.
- `manifest.json` only ever grows; older versions stay listed so a client can
  pin or roll back to a specific release if a newer one misbehaves.

## Installing a release

### Over the air (normal path)

Any device already running an MCUboot image can take a release without a
debug probe:

1. Install Nordic's **nRF Connect Device Manager** app (iOS/Android), or use
   the NexGen Dashboard iOS app which drives the same MCUmgr flow.
2. Download the `dfu_rma_vX.Y.Z.zip` asset for the board you have (check the device's
   Device Information Service model string first) and get it onto the phone.
3. Stop streaming, make sure the device is charging and above ~50% battery.
4. In Device Manager: connect → **Image** tab → select the zip → **Test and
   Confirm** (or **Test** + **Reset** to keep MCUboot's automatic revert).
5. Verify the running image's hash against `image_hash` in `manifest.json`
   via MCUmgr image state.

### Over SWD (first flash or recovery)

Every MA unit needs one SWD flash before it is DFU-capable. Download `merged_rma_vX.Y.Z.hex` from the release, then:

- Install the **SEGGER J-Link Software and Documentation Pack**
  (<https://www.segger.com/downloads/jlink/>) and **nRF Connect for
  Desktop** (<https://www.nordicsemi.com/Products/Development-tools/nRF-Connect-for-Desktop>).
- Wire a J-Link (or an nRF52840 DK) to the board's SWDIO, SWDCLK, GND and
  VTref pads with the board powered.
- In nRF Connect for Desktop open **Programmer**, select the J-Link, **Add
  file** the merged hex, and **Erase & write**. Or from a shell:

  ```sh
  nrfutil install device
  nrfutil device program --firmware merged_<board>_vX.Y.Z.hex --options chip_erase_mode=ERASE_ALL
  nrfutil device reset
  ```

## Building from source

Source lives in [`ma-firmware`](https://github.com/LISHA-CWU/ma-firmware). Building it
requires **nRF Connect SDK v3.1.1** and its toolchain:

1. Install [VS Code](https://code.visualstudio.com/) and the **nRF Connect
   for VS Code Extension Pack** (`nordic-semiconductor.nrf-connect-extension-pack`).
2. From the extension's Welcome page, **Install Toolchain → v3.1.1** and
   **Install SDK → v3.1.1** (or `nrfutil sdk-manager toolchain install
   --ncs-version v3.1.1` and `nrfutil sdk-manager install v3.1.1`).
3. Follow the **Getting started**, **Building** and **Flashing** sections of
   that repo's README for the board target (`rma/nrf52840`) and the exact `west` /
   VS Code steps.
