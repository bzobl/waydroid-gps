# GPS serial driver for waydroid

## Building

### Setup the build environment

Choose a distribution to build with, this doc assumes Debian trixie.

Install all dependencies listed on
<https://wiki.lineageos.org/emulator#install-the-build-packages>. In
addition install

- libncurses5 (which is no longer available in trixie, so take it from boowkworm)
- meson
- glslang-tools (unsure whether we need libglvnd-dev)
- python3-mako
- python3-yaml

### Setup ccache

To improve build speeds it might be advisable to use `ccache`. You might
want to set the following variables
```
export USE_CCACHE=1
export CCACHE_DIR=/mnt/ccache/
export CCACHE_EXEC=/usr/bin/ccache
```
and set the cache size to some value, e.g., 10G
```
ccache -M 50G
```
and enabling compression
```
ccache -o compression=true
```

### Setup LVM with caching

The build takes up some disk space, around 280GB. If you're using an
HDD, you might want to consider setting up LVM caching using a quicker
drive. See `man lvmcache`.

### Building system and vendor images

Follow the instructions on
<https://docs.waydro.id/development/compile-waydroid-lineage-os-based-images>. In short:

- Initialize the repository
  `repo init -u https://github.com/LineageOS/android.git -b lineage-20.0 --git-lfs`
- Sync it
  `repo sync build/make`
- Grab waydroid's local manifests and put them into the
  `.repo/local_manifests/` directory
  `wget -O - https://raw.githubusercontent.com/waydroid/android_vendor_waydroid/lineage-20/manifest_scripts/generate-manifest.sh | bash`
- Copy this repo's `90-waydroid-gps.xml` file to the
  `.repo/local_manifests/` directory
- Sync all repositories
  `repo sync`
- Setup the build environment
  `. build/envsetup.sh`
- Apply the waydroid patches
  `apply-waydroid-patches`
- Setup the build environment (unsure whether this actually is needed
  a second time)
  `. build/envsetup.sh`
- Prepare for the image you want to build
  `lunch lineage_waydroid_arm64-userdebug`
- Build the system image
  `make systemimage -j$(nproc --all)`
- Build the vendor image
  `make vendorimage -j$(nproc --all)`
