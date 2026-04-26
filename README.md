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
  `lunch lineage_waydroid_x86_64-userdebug`
- Build the system image
  `make systemimage -j$(nproc --all)`
- Build the vendor image
  `make vendorimage -j$(nproc --all)`

#### Changes

- add gps HAL as repo
- modify device/waydroid/waydroid

**TODO:** The following patch currently has to be applied manually, add
a script for that.

```
diff --git a/BoardConfig.mk b/BoardConfig.mk
index 9f7565c..cbb7acc 100644
--- a/BoardConfig.mk
+++ b/BoardConfig.mk
@@ -50,6 +50,9 @@ endif
 TARGET_USERIMAGES_USE_F2FS := true
 TARGET_USERIMAGES_USE_EXT4 := true

+# GPS
+BOARD_HAS_GPS := true
+
 # HIDL
 DEVICE_MANIFEST_FILE := $(DEVICE_PATH)/manifest.xml

diff --git a/device.mk b/device.mk
index 7c17ece..bd3ff66 100644
--- a/device.mk
+++ b/device.mk
@@ -246,3 +246,8 @@ endif
 # Updater
 PRODUCT_PACKAGES += \
     WaydroidUpdater
+
+# GPS
+PRODUCT_PACKAGES += \
+    android.hardware.gnss@1.0-impl \
+    android.hardware.gnss@1.0-service
diff --git a/manifest.xml b/manifest.xml
index 77d4ef4..cdc9873 100644
--- a/manifest.xml
+++ b/manifest.xml
@@ -193,4 +193,13 @@ IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
             <instance>default</instance>
         </interface>
     </hal>
+    <hal format="hidl">
+        <name>android.hardware.gnss</name>
+        <transport>hwbinder</transport>
+        <version>1.0</version>
+        <interface>
+            <name>IGnss</name>
+            <instance>default</instance>
+        </interface>
+    </hal>
 </manifest>
diff --git a/system.prop b/system.prop
index de8cf1d..8cf5d0e 100644
--- a/system.prop
+++ b/system.prop
@@ -34,3 +34,7 @@ ro.lmk.downgrade_pressure=100
 ro.lmk.kill_heaviest_task=true
 ro.lmk.kill_timeout_ms=100
 ro.lmk.use_minfree_levels=true
+
+# GPS
+ro.kernel.android.gps=ttyGPSD
+ro.kernel.android.gpsttybaud=115200
```

## Use waydroid

- Install waydroid
- Copy system.img and vendor.img to `/etc/waydroid-extra/images/`
- run `waydroid init -f`
- add the following line to `/var/lib/waydroid/lxc/waydroid/config_nodes`
  ```
  lxc.mount.entry = /dev/ttyGPSD dev/ttyGPSD none bind,create=file,optional 0 0
  ```
- See <https://github.com/bzobl/gpsdtty> how to provide `/dev/ttyGPSD`.
