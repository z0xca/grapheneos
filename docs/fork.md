# Operate the fork

This fork of [schnatterer/rooted-graphene](https://github.com/schnatterer/rooted-graphene) publishes rootless GrapheneOS updates for one Pixel 8 Pro (`husky`). The phone uses the fork's signing keys and receives updates through Custota. It does not use upstream's public OTA server.

## Source of truth

| Topic | File |
|---|---|
| Release channel, tool versions, patcher commit, and build defaults | [`rooted-ota.sh`](../rooted-ota.sh) |
| Schedule and rootless Husky inputs | [`release-husky.yaml`](../.github/workflows/release-husky.yaml) |
| Release job, manual inputs, signing secrets, and checkout pin | [`release-single.yaml`](../.github/workflows/release-single.yaml) |
| Sensor rewrite and patcher registration | [`norththermrewire.py`](../modules/norththermrewire.py) and the [registration patch](../patches/my-avbroot-setup-norththermrewire.patch) |

The thermal module uses `cam_therm` in place of `north_therm` in the affected formulas. It does not repair the sensor or remove the thermal shutdown thresholds. `NORTH_THERM_REWIRE=false` turns off the rewrite for a local build. See the [fault report](north-therm-fault.md) for the diagnosis and remaining risks.

## Release setup

- Store the signing keys, OTA certificate, and passphrases in the repository secrets named by the release job.
- Back up both signing keys outside GitHub.
- Enable **Release husky** and **Release single device**. Leave **Renovate** disabled.
- Set GitHub Pages to serve the root of `gh-pages`. This branch holds the update feed, not the build source.
- Use the `rootless/` directory on this repository's Pages site as the Custota server URL. Custota reads `rootless/husky.json` for this phone.
- Keep the stock GrapheneOS system updater disabled on this installation. Use Custota for the fork's updates.

## Normal updates

1. The Husky release job reads the latest version from the configured GrapheneOS channel.
2. The patcher verifies the input OTA against the pinned GrapheneOS keys. Then it applies the thermal module, Custota, and OEMUnlockOnBoot and signs the OTA.
3. The build verifies the new or cached OTA ZIP against the fork's public keys.
4. The release job uploads the ZIP and its `.csig`. It then updates the Custota feed on `gh-pages`.

The scheduled run normally skips a device, version, and flavor with a matching release asset. A fork code change usually reaches the phone with the next GrapheneOS release.

### Check for a partial release

A published ZIP does not mean that Custota can see the update. The next scheduled run can skip that version instead of repairing the feed.

If a release run fails:

1. Read the Actions log to find the failed step.
2. Check that the release contains the ZIP and its matching `.csig`.
3. Check that the live `rootless/husky.json` points to both files.

## Publish a change before the next GrapheneOS release

`force-build` does not overwrite release assets. If the release already has an asset with the same name, the upload can fail.

1. In Actions, open **Release single device**.
2. Set the device to `husky`.
3. Select `skip-rooted`, `force-build`, and `force-ota-server-upload`.
4. Leave `skip-rootless` and `skip-release` clear.
5. Run the workflow.
6. Check that the published `rootless/husky.json` points to the intended ZIP and `.csig`.
7. If the feed points to both files, install the update in Custota.

## Maintenance and local tests

- Sync from upstream regularly. If the patcher pin changes, verify that it still registers the module and produces the intended vendor edits.
- Check that scheduled runs still occur. GitHub can [delay, drop, or disable schedules](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule).
- If symptoms return, use the [sensor checks](north-therm-fault.md#how-to-diagnose-it-again).

Push and pull request jobs do not publish releases. Their build-only command puts `createRootedOta` on the left of `&&`. Bash can ignore `errexit` inside that function, so a green run does not prove that every verification passed.

### Test a build locally

Use a disposable checkout with temporary signing keys. Do not use release credentials or publish test assets from that checkout.

1. Replace `trust/signing-avb_pkmd.bin` and `trust/signing-ota.crt` with the test public files. Leave the GrapheneOS input trust anchors unchanged.
2. Leave `GITHUB_REPO` unset to skip the release lookup.
3. Leave `MAGISK_PREINIT_DEVICE` unset for a rootless build.
4. If you need to inspect the output after the build, set `SKIP_CLEANUP=true`.
5. Run `DEVICE_ID=husky bash -c 'source ./rooted-ota.sh; createRootedOta'`. Do not put `createRootedOta` on the left of `&&`.
6. Remove the test checkout after the test.

## Trust and recovery

The [`trust/`](../trust/) directory contains public files only. The build uses the GrapheneOS pins for the input OTA and the fork's pins for the output.

- **Stock AVB key:** Compare the SHA-256 of `husky-avb_pkmd.bin` with the Pixel 8 Pro fingerprint in the [GrapheneOS install guide](https://grapheneos.org/install/cli).
- **Stock OTA certificate:** Verify `grapheneos-ota.crt` against the certificate in `/system/etc/security/otacerts.zip` from a stock build whose `vbmeta` verifies against that AVB key.
- **Fork keys:** Compare the SHA-256 of `signing-avb_pkmd.bin` with the fingerprint on the phone's yellow boot screen. Compare `signing-ota.crt` with the OTA certificate in the release secret `CERT_OTA_BASE64`.

If GrapheneOS rotates a key, verify the replacement independently before changing the pin. Do not trust a new key only because it came with the download.

A normal OTA keeps user data and uses the other slot. For a fresh install or recovery:

1. Before flashing, check that the target build does not violate rollback protection. Compare the actual builds, not just their stable or security preview labels.
2. Verify that the flashed `vendor` image contains the thermal rewrite. A successful boot does not prove that the patch is present.
3. Before relocking, verify that both slots contain bootable images signed with the fork's AVB key. A successful boot while unlocked does not verify the signing key.
4. Before relocking, verify that OEM unlocking remains allowed.

**CAUTION:** Keep backups of `avb.key`, `ota.key`, and their passphrases. Without the original keys, this build path cannot keep signing updates for the installation. A migration to new keys requires an unlock and a reflash, which erase user data.

Keep OEM unlocking allowed. If neither slot boots and OEM unlocking is disabled, the phone can be unrecoverable. [OEMUnlockOnBoot](https://github.com/chenxiaolong/OEMUnlockOnBoot) reduces this risk after a successful boot. It cannot help if the OS does not start.

With OEM unlocking allowed, someone with physical access can unlock and reflash the phone. Unlocking erases user data. It does not grant access to the existing encrypted data.

On the first patched installation, Auditor pairing failed with `invalid verified boot key fingerprint`.

## Rules for AI assistants in this repository

- Do not commit or push. A human reviews every change and writes every commit message.
- Build and verify each change, then show the diff.
- Write the smallest possible diff. Prefer community-maintained tools and upstream mechanisms. The thermal module is the only accepted AI-written logic.
- Pin GitHub Actions in the release workflow to commit SHAs. Keep the disabled Renovate workflow unchanged from upstream.
- Write prose in Simplified Technical English.
- Give plain commands, not wrapper scripts.
- Base suggestions on what the community ships or on a stated problem.
