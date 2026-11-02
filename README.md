# marathon-updates

Where Marathon phones get their updates. There are two things here.

- **The OS image feeds.** systemd-sysupdate reads them from this repository's
  releases.
- **Reef's app catalogue.** Reef reads it from this repository's GitHub Pages
  site.

Both are signed on the publisher's machine. Neither signing key is ever given
to GitHub. CI checks every signature with the public keys in [`keys/`](keys).

The code that builds, publishes and consumes all this lives in
[Marathon-Shell](https://github.com/patrickjquinn/Marathon-Shell):
[`docs/OTA.md`](https://github.com/patrickjquinn/Marathon-Shell/blob/main/docs/OTA.md)
and
[`docs/reef/marathon-catalogue.md`](https://github.com/patrickjquinn/Marathon-Shell/blob/main/docs/reef/marathon-catalogue.md).

## OS image feeds

Each device and channel has one rolling release, tagged `<channel>/<image id>`:

| Device | Stable feed | Test feed |
|---|---|---|
| Zinwa Q25 | `stable/zinwa-q25_marathon_edge` | `test/zinwa-q25_marathon_edge` |
| HackberryPi CM5 | `stable/hackberrypi-cm5_marathon_edge` | `test/hackberrypi-cm5_marathon_edge` |
| Raspberry Pi 5 (plain) | `stable/raspberry-pi5_marathon_edge` | `test/raspberry-pi5_marathon_edge` |
| QEMU dev-sim (A/B) | `stable/qemu-aarch64_marathon_edge` | `test/qemu-aarch64_marathon_edge` |

- **Test feeds** are pre-releases. They are for the developers' own phones.
- **Stable feeds** are what everyone else gets.

A release holds every kept version's payload, plus the files sysupdate reads
first:

- `SHA256SUMS`;
- `SHA256SUMS.gpg`, the signature over it by [`keys/marathon-ota.asc`](keys/marathon-ota.asc);
- `marathon-os.metainfo.xml`, each version's download size and release notes.

The payload depends on the device:

| Device | Payload |
|---|---|
| Zinwa Q25 | the root file, `.ext4.zst` |
| Raspberry Pi | the boot and root slots, `.boot.<uuid>.raw.zst` and `.root.<uuid>.raw.zst` |
| QEMU | the UKI plus the `/usr` and verity partitions |

**How a phone finds its feed.** Each image is built with
`MARATHON_OTA_URL=https://github.com/patrickjquinn/marathon-updates/releases/download/<channel>`.
Its sysupdate transfers read `<MARATHON_OTA_URL>/%M_%W_%w/`, and the
specifiers come from the image's own os-release. A Q25 on stable therefore
reads:

```
https://github.com/patrickjquinn/marathon-updates/releases/download/stable/zinwa-q25_marathon_edge/SHA256SUMS
```

GitHub answers with a 302 to its object store, and systemd-pull follows it.

**Limits.**
- Each asset must be under 2 GiB. `marathon-ota` refuses a larger one.
- Replacing `SHA256SUMS` is not atomic. For about a second, a phone may see
  the old manifest, or a manifest that doesn't match its signature. That
  check fails, and the phone retries at its next check. `marathon-ota` waits
  until GitHub serves the new pair.

## Reef catalogue

GitHub Pages serves `site/` at https://patrickjquinn.github.io/marathon-updates/.
Reef reads:

```
https://patrickjquinn.github.io/marathon-updates/reef/<arch>/catalogue-full.json
https://patrickjquinn.github.io/marathon-updates/reef/<arch>/catalogue-full.json.asc
```

`<arch>` is `aarch64` or `x86_64`. Reefs from before the `marathon` install
kind read `catalogue.json` instead. They reject a whole catalogue that holds
a kind they don't know, so `catalogue.json` is `catalogue-full.json` without
those entries. `tools/reef-catalogue sign` writes it and signs both files.

- **The signature** (`<file>.asc`) is a detached ASCII GnuPG signature by
  [`keys/marathon-reef.asc`](keys/marathon-reef.asc). This is the scheme
  Reef's Sailfish OS store uses for its index.
- **Revision.** `revision` must never go backwards, so a replayed older
  catalogue is refused.
- **Deploying.** The `pages` workflow deploys `site/` only when every
  catalogue in it verifies.

Icons and screenshots go in `site/reef/<arch>/assets/`.

**Installable kinds.** `flatpak`, `android` and `marathon`; `catalogue.json`
carries only the first two. A later Reef lists an unknown kind and doesn't
install it, so a new kind breaks nothing from then on.

**Marathon's own apps.** The release `apps/<arch>` holds signed `.marathon`
packages of the apps the image ships but Reef updates. Each package has a
`.minisig` beside it, by Marathon's app key. Each `marathon` entry names the
package's URL there, its SHA-256 and `min_image`, the oldest image it runs
on. Marathon-Shell's `scripts/reef/publish-apps` builds, signs and uploads
them, and adds the entries to `catalogue-full.json`.

## Publishing (on the publisher's machine)

**Keys.** The private keys never leave the publisher's machine:

| Key | Private key | Fingerprint |
|---|---|---|
| OS images | `~/.marathon-secrets/ota/gnupg` | `433D273DC2C27285A9671D4277C177A04B2956BD` |
| Reef catalogue | `~/.marathon-secrets/reef/gnupg` | `F2E11B366BAF957F487B86E64D63ECF4806B0F6E` |

**An OS image.** Run this from a Marathon-Shell checkout. Use the
`--channel test` feed first; the device-tested artifact goes to stable.

```sh
scripts/ota/marathon-ota publish --image-dir <mkosi.output>/<image id> --version <IMAGE_VERSION> \
    --feed <local feed root>/<channel> --gnupghome ~/.marathon-secrets/ota/gnupg --notes notes.txt \
    --github patrickjquinn/marathon-updates --channel <channel>
```

**The local feed directory.** `marathon-ota` keeps the newest two versions
(`--keep`). The local feed directory is the release's mirror: keep it.

- Run `marathon-ota github-sync` to push it again after a failed upload.
- If the directory is lost, `gh release download <channel>/<image id> -D <feed>/<channel>/<image id>`
  restores it.

**The Reef catalogue.**

```sh
$EDITOR site/reef/aarch64/catalogue-full.json
tools/reef-catalogue sign site/reef/aarch64/catalogue-full.json --gnupghome ~/.marathon-secrets/reef/gnupg
git commit -am "reef: ..." && git push
```

## CI

`validate` runs on every push, on every release change, and daily. It runs
`tools/reef-catalogue verify` and `tools/check-feeds`, which check that:

- each feed's signature is good;
- every asset is listed in `SHA256SUMS`, and nothing else is;
- `SHA256SUMS` and the AppStream catalog agree;
- every asset is under the size limit;
- small assets match their checksums.

Payloads are not downloaded: the phone checks them against the signed
manifest.

`test/sysupdate-proof_marathon_edge` is a test feed with tiny payloads, kept
to prove the path. No image has that id, so no phone reads it.
