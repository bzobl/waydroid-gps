# GPS for waydroid containers

Use gpsd's implementation of the android.hardware.gnss service [^1] to
provide the host's GPS data to the waydroid container. This allows apps
inside the container to access location.

The gnss service running in the container will connect via TCP to the
gpsd service running on the host using the gpsd client and reports basic
location data and SV info to Android.

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
- python3-pycparser
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
- Apply this repo's patches. This currently has to be done manually.
    - Apply the patch to device/waydroid/waydroid
      ```
      cd device/waydroid/waydroid \
          && git apply <this-repo>/patches/device_waydroid_waydroid.patch
      ```
- Setup the build environment (unsure whether this actually is needed a second time)
  `. build/envsetup.sh`
- Prepare for the image you want to build
  `lunch lineage_waydroid_x86_64-userdebug`
- Build the system image
  `make systemimage -j$(nproc --all)`
- Build the vendor image
  `make vendorimage -j$(nproc --all)`

## Use waydroid

- Install waydroid
- Copy system.img and vendor.img to `/etc/waydroid-extra/images/`
- Run `waydroid init -f`
- The waydroid container will attempt to connect to the host's gpsd via
  TCP. The lxc container should allow the outbound connection by
  default. The host/port to connect to must be configured during build
  through the `service.gpsd.host` and `service.gpsd.port` properties
  specified in `device.mk`. When unset, these will resolve to
  "localhost" and "2947", respectively. In the patch above the host to
  connect to is configured as 192.168.240.1, which is an IP address on
  one of the host's interfaces.
- Make sure the host's gpsd service is configured to listen on the IP
  address configured for the Android image.
  - If running via systemd, make sure the `gpsd.socket` unit is listening
    on the correct interface, e.g, by overriding the unit like so
    ```
    # /etc/systemd/system/gpsd.socket.d/override.conf
    [Socket]
    ListenStream=
    ListenStream=/run/gpsd.sock
    ListenStream=0.0.0.0:2947
    ```
    and add `GPSD_OPTIONS="--listenany"` to `/etc/default/gpsd`.

[^1]: https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/gnss
