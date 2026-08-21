# Patches for building AOSP-like ROMs for marble

Patch collection used to build LineageOS 23.2-based ROMs for Xiaomi marble.

The patches are development material, not a single universal patch set. They
touch several Android repositories and may need manual conflict resolution when
used on a different source revision.

## Credits

- [Chaitanyakm](https://github.com/Chaitanyakm) for the MIUI Camera blobs and
  marble development work.
- [ArianK16a](https://github.com/ArianK16a) for maintaining the marble device
  tree and helping with device-specific issues.
- [crDroid Android](https://github.com/crdroidandroid) for marble fixes and
  implementation references.
- [HMA-OSS](https://github.com/frknkrc44/HMA-OSS) for package visibility and
  LineageOS detection mitigation references.
- [TEESimulator](https://github.com/JingMatrix/TEESimulator) for the open-source
  TEE emulation implementation.
- Paranoid Android and VoltageOS for Paranoid Sense and the Face Unlock
  integration references.

## Current patches

`split_patches/` is the current patch set. There are 13 active feature patches:

1. `01_tee_soft_emulate.patch`: TEE/key-attestation emulation and LineageParts
   controls.
2. `02_face_unlock.patch`: Paranoid Sense Face Unlock framework and Settings
   integration.
3. `03_data_switch_tile.patch`: dual-SIM mobile data switch Quick Settings tile.
4. `04_lineage_visibility_and_vendor_system_migration.patch`: package visibility
   controls, installer spoofing, selected service/resource renames and related
   SELinux changes.
5. `05_launcher_recents_lock.patch`: lock tasks in recents and preserve them when
   clearing recent apps.
6. `06_updater_nekolos.patch`: NekoLOS marble updater endpoints and incremental
   OTA handling.
7. `07_dolby_integration.patch`: Dolby effects, audio routes, codec definitions
   and VINTF configuration.
8. `08_gki_kernel_compat_restore.patch`: GKI/module compatibility, kernel fixes,
   compiler selection and related device configuration.
9. `09_bugfix_platform_stability.patch`: MIUI Camera compatibility and platform,
   clipboard, native and SELinux fixes.
10. `10_erofs_partition_ota_shape.patch`: EROFS/fstab configuration, OTA shape,
    Dolby inheritance and Face Unlock packaging.
11. `11_device_experience_regional.patch`: optional MIUI Camera integration,
    marble tuning, audio fixes and regional defaults.
12. `12_mimalloc_strategy.patch`: mimalloc integration in bionic.
13. `13_scrcpy_desktop_mode_fixes.patch`: desktop/freeform virtual-display and
    taskbar fixes.

`99_unclassified.patch` is currently an empty placeholder and does not need to
be applied.

There is no KernelSU, KSU or SUSFS integration in this repository.

## Historical files

Do not stack these files with `split_patches/`:

- `split_patches_old/` contains superseded versions kept for reference.
- `backup_all_changes_*.patch` files are chronological, monolithic snapshots.
  They overlap the split patches and are not additional features to apply.

## Source setup

Initialize and sync a LineageOS 23.2 source tree first. Clone this repository
into a named directory under the source root, not directly into the non-empty
source root:

```bash
git clone -b lineage-23.2 \
    https://github.com/fwlta/patches_for_build_marble_AOSP.git \
    patches_for_build_marble_AOSP
```

The XML files in `local_manifests/` provide the official marble, sm8450 and
mimalloc projects used by parts of this patch set. Copy only projects that are
not already declared by your ROM manifest; duplicate project paths make
`repo sync` fail.

```bash
mkdir -p .repo/local_manifests
cp patches_for_build_marble_AOSP/local_manifests/mimalloc.xml \
    .repo/local_manifests/mimalloc.xml
repo sync -c -j"$(nproc)"
```

The included `marble.xml` and `xiaomi_sm8450.xml` are available when the ROM
manifest does not already provide those device, kernel and vendor projects.

## Applying split patches

Run `patch_diff.sh` from the Android source root. Apply only the features you
intend to ship and review every resulting diff before building.

```bash
chmod +x patches_for_build_marble_AOSP/patch_diff.sh

./patches_for_build_marble_AOSP/patch_diff.sh \
    patches_for_build_marble_AOSP/split_patches/01_tee_soft_emulate.patch
```

Repeat the command for the selected patches. If using the complete current set,
apply patches `01` through `12` in numeric order and resolve conflicts before
continuing.

### Patch 13 exception

`13_scrcpy_desktop_mode_fixes.patch` retains source-root-relative paths and is
not compatible with `patch_diff.sh`. Apply it from the Android source root after
removing its `project .../` marker lines, and always perform a dry run first:

```bash
grep -v '^project .*/$' \
    patches_for_build_marble_AOSP/split_patches/13_scrcpy_desktop_mode_fixes.patch \
    > /tmp/marble-scrcpy.patch

patch --dry-run -p1 < /tmp/marble-scrcpy.patch
patch -p1 < /tmp/marble-scrcpy.patch
rm /tmp/marble-scrcpy.patch
```

## Feature dependencies

### Face Unlock

Face Unlock needs the framework changes from patch 02, product configuration
from patch 10, Paranoid Sense and the AOSPA biometric face interface.

```bash
git clone -b 16.2 \
    https://gitlab.com/voltageos/packages_apps_paranoidsense \
    packages/apps/ParanoidSense

git clone -b beryl --depth=1 \
    https://github.com/AOSPA/android_vendor_aospa \
    /tmp/android_vendor_aospa

mkdir -p vendor/lineage/interfaces
cp -r /tmp/android_vendor_aospa/interfaces/biometrics \
    vendor/lineage/interfaces/
rm -rf /tmp/android_vendor_aospa
```

### Dolby and MIUI Camera

The Dolby patches require `hardware/dolby`. MIUI Camera is inherited only when
its device and vendor projects exist. These maintained repositories can be used:

```text
https://github.com/Desdec-marble/android_hardware_dolby
https://github.com/Desdec-marble/android_device_xiaomi_miuicamera-marble
https://github.com/Desdec-marble/proprietary_vendor_xiaomi_miuicamera-marble
```

Suggested revisions and paths:

```text
android_hardware_dolby: hardware/dolby, revision 16.0
android_device_xiaomi_miuicamera-marble: device/xiaomi/miuicamera-marble, revision 16.0
proprietary_vendor_xiaomi_miuicamera-marble: vendor/xiaomi/miuicamera-marble, revision fifteen
```

Install Git LFS before syncing the MIUI Camera vendor repository:

```bash
git lfs install
```

### GKI and EROFS

Patches 08 and 10 are coupled for the current GKI/EROFS configuration. Patch 08
also expects `prebuilts/clang/host/linux-x86/clang-r416183b` to exist. Review the
AVB, fstab, module-list and OTA changes before using them on another device tree.

### Mimalloc

Patch 12 requires `external/mimalloc`. The included
`local_manifests/mimalloc.xml` supplies the expected project and revision.

### GApps

The current split patches do not enable GApps. Cloning MindTheGapps alone does
not add it to the product; your product configuration must inherit it explicitly.

## PIF Inject

PIF is separate from `split_patches/` and consists of two patches:

- `pif_inject_LOS23.2.patch` targets `frameworks/base`.
- `pif_inject_LineageParts_LOS23.2.patch` targets
  `packages/apps/LineageParts`.

Check each patch from its target repository before applying it:

```bash
cd frameworks/base
git apply --check ../../patches_for_build_marble_AOSP/pif_inject_LOS23.2.patch
git apply ../../patches_for_build_marble_AOSP/pif_inject_LOS23.2.patch
cd ../..

cd packages/apps/LineageParts
git apply --check ../../../patches_for_build_marble_AOSP/pif_inject_LineageParts_LOS23.2.patch
git apply ../../../patches_for_build_marble_AOSP/pif_inject_LineageParts_LOS23.2.patch
cd ../../..
```

These patches overlap files also modified by the TEE and package-visibility
patches. When combining them, a manual merge may be required instead of direct
`git apply`. PIF also requires a working GMS environment and valid runtime
configuration; passing Play Integrity is not guaranteed by applying the patches.

## Build

After reviewing the changes and resolving all conflicts:

```bash
source build/envsetup.sh
lunch lineage_marble-bp4a-userdebug
mka bacon
```

Test the resulting build on your own device and keep a recovery path available.
