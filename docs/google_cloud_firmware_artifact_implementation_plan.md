# Plan: Create a downloadable firmware artifact for PR testing

## Goal

For each pull request, run the existing automated checks. If they pass, make the firmware from that same build downloadable so a person can flash it to the rover, test the changes, and then decide whether to merge the PR.

**Workflow:** PR opened or updated → compile/check → package the exact build output → download and flash → record hardware result → merge only if accepted.

This is a learning-sized CI workflow, not a production release system. It does not change firmware behavior, automatically flash a rover, or claim that a successful compile proves hardware operation.

## Known facts

- The current Cloud Build config is [.cloudbuild/dev/dev.cloudbuild.yaml](../.cloudbuild/dev/dev.cloudbuild.yaml). It installs tools and libraries, compiles `General_Driver`, then copies `*.bin` from the build directory. Compilation and packaging are currently in one step; there is no separate software test suite.
- Board settings are in [.cloudbuild/dev/version.env](../.cloudbuild/dev/version.env): ESP32 Dev Module, ESP32 core 2.0.14, CPU 240 MHz, flash 80 MHz/QIO/4 MB, default partition scheme, PSRAM disabled, debug level None, Events and Arduino on Core 1. Upload speed is 921600; erase-all-flash is disabled in the IDE setup.
- Therefore, the current automated pass condition is a successful compile. Describe the output as **compile-validated**, not fully tested.
- The current local build folder contains `General_Driver.ino.bin`, `General_Driver.ino.bootloader.bin`, and `General_Driver.ino.partitions.bin`, as well as `.elf` and `.map` files. The `.elf` and `.map` files are build/debug outputs, not expected flash images. The map identifies ESP32 core 2.0.14, consistent with the pinned version. These observations are from a local Arduino IDE build and do not yet prove Cloud Build produces byte-identical output.
- **Provisional standard ESP32 upload recipe (verify before use):** for ESP32 core 2.0.14, ESP32 Dev Module, default 4 MB partition scheme, the expected flash images/offsets are `0x1000` = bootloader, `0x8000` = partition table, `0xE000` = core-supplied `boot_app0.bin`, and `0x10000` = application. The first, second, and fourth images correspond to the three `.bin` files in the local build folder; `boot_app0.bin` is typically supplied by the installed core under its `tools/partitions` directory and is not in that build folder. This is a best guess based on the standard core recipe, not yet confirmed from this board's verbose upload output. Do not package or flash based only on this guess.
- A likely direct-flash invocation uses `esptool.py` with `--chip esp32`, `--baud 921600`, `--before default_reset`, `--after hard_reset`, `write_flash -z`, `--flash_mode qio`, `--flash_freq 80m`, and `--flash_size 4MB`, followed by the four offset/image pairs above. The exact executable name, options, paths, and erase behavior must be confirmed from the selected core's upload recipe or Arduino IDE verbose upload log before this command is documented as the supported procedure.
- The firmware initializes LittleFS with format-on-mount-failure enabled in [files_ctrl.h](../General_Driver/files_ctrl.h), and Wi-Fi configuration is read from `/wifiConfig.json` in [wifi_ctrl.h](../General_Driver/wifi_ctrl.h). A normal write of the standard bootloader, partition table, `boot_app0.bin`, and application ranges would be expected not to erase the separate LittleFS data partition when the same partition layout is retained and chip-wide erase is disabled. However, a changed/incompatible partition table or LittleFS mount failure can cause data loss; configuration preservation therefore remains unverified and must be described as a risk until tested on hardware.

## Implementation tasks

### 1. Confirm the no-rebuild flash path

- **Starting point / best guess:** the local Arduino build currently has `General_Driver.ino.bootloader.bin`, `General_Driver.ino.partitions.bin`, and `General_Driver.ino.bin`. For the pinned ESP32 core 2.0.14 and default 4 MB partition scheme, expect the standard image/offset pairs `0x1000` bootloader, `0x8000` partition table, `0xE000` core-supplied `boot_app0.bin`, and `0x10000` application. Do not include `.elf` or `.map` in the flash bundle. Find the core-supplied `boot_app0.bin` under the installed core's `tools/partitions` directory. Treat this entire list as provisional until confirmed by the actual upload recipe.
- Confirm the recipe with one Arduino IDE upload using **verbose output during upload** (or inspect `platform.txt`/`boards.txt` for the exact installed core version). Save the `esptool` command and check all address/file pairs, chip/flash settings, and whether chip-wide erase is enabled. Check the local core version against the pinned 2.0.14 setting; discard/rebuild local outputs if they were built with a different core or board configuration.
- Use the confirmed image files and offsets to write a direct `esptool` procedure that consumes the exported files without compiling. A likely command uses `write_flash -z`, QIO, 80 MHz, 4 MB, and 921600 baud, but use the exact options and executable shown by the installed core's upload recipe. Verify that every referenced file is included or is deliberately fetched from the pinned core, and that the command succeeds without Arduino IDE compiling the sketch.
- Document configuration impact. The expected normal case is that writing only the standard firmware image ranges with the same partition layout and without chip-wide erase leaves LittleFS intact. This is not a guarantee: the firmware calls `LittleFS.begin(true)`, which can format the filesystem after a mount failure. Back up any needed device configuration and verify preservation/reset behavior on a test device before claiming it is retained.
- Capture the full source commit SHA associated with the build (and ensure the working tree/build inputs match that commit) so the artifact can be matched to the PR revision.

**Gate:** These expected offsets and settings are planning assumptions only. Do not claim the bundle is ready for hardware until the actual upload recipe confirms the image list/offsets and direct-flash command, and configuration impact has been verified or clearly documented as an unresolved risk.

### 2. Store the bundle in Artifact Registry

- Use a Google Cloud Artifact Registry **generic** repository to store the bundle. Keep the project, region, repository, and package name in documented non-secret configuration.
- Confirm the repository exists (or document its creation) and verify the supported `gcloud artifacts generic upload` command and download procedure for the selected repository and installed Cloud CLI version.
- Upload the single validated archive as the final required build step. Any compile, packaging, validation, or upload failure must fail the build.
- Identify each artifact by Cloud Build ID as its Artifact Registry version, and put the full source commit SHA and build ID in the manifest. This allows separate builds of the same commit to be distinguished; the PR link/build details must make the corresponding artifact easy to locate. Never silently replace an existing version.
- Grant the Cloud Build service account write access only to the intended repository, plus minimum logging permissions. Do not create or store service-account keys or other credentials in the repository.
- PR builds execute PR-controlled code. The simple direct-upload approach is appropriate only when PR authors and their changes are trusted to use the narrowly scoped repository-write permission. Do not run untrusted external PR code with this permission. A manual approval prompt alone does not make untrusted code safe. If external contributions must be supported, stop and design a separate trusted handoff/publisher before granting upload access.

**Decision gate:** Confirm who may open PRs and trigger builds, and accept the scoped direct-upload trust model before enabling Artifact Registry upload on the PR trigger. The artifact must come from the exact PR build; do not substitute a rebuild from another commit.

### 3. Package only the verified build output

- Add packaging after successful compilation/checks. Packaging must consume the existing build directory and must not compile again.
- Include the verified flash files, a short README with the source commit and exact flash procedure, and a SHA-256 checksum file. Include only the explicitly verified output files; fail clearly if a required file is missing. Do not copy every `.bin` by wildcard.
- Keep generated files out of Git. Do not include credentials or secrets.
- Validate the archive and its checksums before upload. Log the commit SHA, build ID, Artifact Registry package/version, and archive checksum.
- Link the Artifact Registry package/version from the PR or build result so the tester can find it. A failed compile/check must never produce a success-looking candidate.

### 4. Test and record

- Download the Artifact Registry version associated with the PR build and verify the archive checksum, then verify the checksums inside the archive.
- Flash it using the documented no-rebuild procedure, then perform the relevant manual hardware checks.
- Record the PR commit SHA, artifact checksum, and hardware pass/fail result on the PR. If the PR changes after testing, test the new commit's artifact before merging.
- Merge only after the required automated checks and the human hardware decision pass.

## Acceptance checklist

- [ ] A PR build succeeds only when the existing compile/checks pass.
- [ ] A successful build uploads one downloadable archive from that exact PR build to Artifact Registry; the manifest identifies its full source commit SHA and Cloud Build ID.
- [ ] The artifact contains the verified flash files, direct-flash instructions, and checksums; missing required files fail the build.
- [ ] Flashing the artifact does not rebuild firmware, and its effect on device configuration is documented.
- [ ] Artifact Registry upload access is limited to the intended repository, and the PR-trigger trust/permission policy is documented; no keys or production credentials are stored in source control.
- [ ] Hardware results are recorded against the exact tested commit/artifact before merge.

## Explicitly out of scope for this first version

- Production release promotion or rebuilding firmware during release.
- A separate trusted publisher or cross-build handoff, provided the PR trigger is limited to trusted contributors under the policy above.
- New software tests; compile success is the automated gate until a test suite exists.
- Automatic rover flashing, OTA deployment, or claims that CI validates physical behavior.
- Full reproducible-build metadata or byte-for-byte reproducible archives.