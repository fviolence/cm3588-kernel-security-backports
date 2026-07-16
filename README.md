# Building and Installing a Security-Patched FriendlyElec RK3588 Linux 6.1.141 Kernel

Due to rapid developments in the field of Linux kernel vulnerability research and the extremely infrequent updates to official images, there has been a need to port certain patches to address the most high-profile CVEs affecting the kernel.

This guide builds FriendlyElec's RK3588 Linux 6.1.141 vendor kernel with a small security backport series, packages the result as Debian packages, and installs it without replacing the operating system or root filesystem.

The patch set targets the FriendlyElec `nanopi6-v6.1.y` kernel at version **6.1.141**. Do not assume that it applies unchanged to a later vendor-kernel revision.

> This repository is a tested backport set for a specific FriendlyElec Linux 6.1.141 baseline. It is not a general-purpose Linux security update service. New vendor-kernel revisions and newly disclosed CVEs must be evaluated separately.

## Scope

The supplied patches address:

* **CVE-2026-31431 ("Copy Fail")** in `algif_aead`.
* **CVE-2026-43284 ("Dirty Frag", ESP path)** in IPv4/IPv6 ESP handling.
* **CVE-2026-43500 ("Dirty Frag", RxRPC path)** in encrypted RxRPC DATA handling.
* **CVE-2026-43499 ("GhostLock")** in the RT-mutex waiter-removal path.
* **CVE-2026-53163** in the RT-mutex proxy-lock cleanup path, a NULL-pointer-dereference regression introduced by the GhostLock fix.

This procedure produces:

* `kernel.img`
* `resource.img`
* a complete `/lib/modules/<kernel-release>` tree
* a FriendlyElec-compatible kernel-headers `.deb`
* a local kernel-image `.deb` containing the images and modules

The kernel-image package created here deliberately **does not write raw partitions from a Debian maintainer script**. Installation of files, backup of existing partitions, flashing, and verification are separate explicit steps.

### Not applicable: CVE-2026-46242 ("Bad Epoll")

CVE-2026-46242 does not affect the Linux 6.1.141 baseline used by this guide. The affected Linux 6.1 stable range begins at 6.1.175, so no Bad Epoll backport is included in this patch set.

---

## Tested source revisions

This patch series and build procedure were tested against the following exact FriendlyElec source revisions:

* [`sd-fuse_rk3588` commit `c4fd64218b6f68cb136da6b9d43fc069383e68fe`](https://github.com/friendlyarm/sd-fuse_rk3588/commit/c4fd64218b6f68cb136da6b9d43fc069383e68fe)

  * Branch at the time of testing: `kernel-6.1.y`
  * Commit subject: `Remove ETHTOOL_LEGACY_2500baseX env var from r8125 driver build`

* [`kernel-rockchip` commit `474962f3f7ff699a8e021b77183c667174088154`](https://github.com/friendlyarm/kernel-rockchip/commit/474962f3f7ff699a8e021b77183c667174088154)

  * Branch at the time of testing: `nanopi6-v6.1.y`
  * Kernel version: Linux 6.1.141
  * Commit subject: `ARM64: dts: rockchip: Add dts for nanopi-m6 v2`

The branch names are moving references. The commands below explicitly check out the tested commits so that later changes to the FriendlyElec branches do not silently change the build inputs.

Patches may fail to apply, require adjustment, or no longer be appropriate when used with another source revision.

---

## 1. Requirements

Docker is required on the build host.
The target board must already run a FriendlyElec Linux image with the RK3588 GPT layout containing raw partitions named `resource` and `kernel`.
A serial console or tested rescue/eFlasher medium is strongly recommended before changing raw boot partitions.

---

## 2. Build the FriendlyElec Docker environment

FriendlyElec publishes a Docker-based cross-compilation environment with the required ARM64 toolchains.

Create a persistent work directory beside the Docker repository:

```bash
HOST_ROOT="$PWD/cm3588-kernel-build"
mkdir -p "$HOST_ROOT"/{builder,work}
cd $HOST_ROOT
```

Clone and build it:

```bash
git clone \
  https://github.com/friendlyarm/docker-cross-compiler-novnc.git \
  builder/docker-cross-compiler-novnc

docker build --pull \
  -t friendlyarm/cm3588-builder:local \
  "$HOST_ROOT/builder/docker-cross-compiler-novnc"
```

Start the container:

```bash
docker run -d \
  --privileged \
  --name cm3588-builder \
  -v /dev:/dev \
  -e PUID="$(id -u)" \
  -e PGID="$(id -g)" \
  -e USER=ubuntu \
  -e PASSWORD=ubuntu \
  -e HTTP_PASSWORD=disabled \
  -e VNC_PASSWORD=disabled \
  -v "$HOST_ROOT/work:/home/ubuntu/work" \
  friendlyarm/cm3588-builder:local
```

Open a shell as the unprivileged build user:

```bash
docker exec -it \
  --user ubuntu \
  --workdir /home/ubuntu/work \
  cm3588-builder \
  bash
```

All remaining build commands in this guide are run inside that shell unless stated otherwise.

Configure a Git identity for commits created while applying format patches and changing `EXTRAVERSION`:

```bash
git config --global user.name "Kernel Builder"
git config --global user.email "kernel-builder@example.invalid"
```

---

## 3. Download the build system, kernel source, and patch repository

From `/home/ubuntu/work` clone FriendlyElec's RK3588 build scripts:

```bash
git clone \
  --branch kernel-6.1.y \
  --single-branch \
  https://github.com/friendlyarm/sd-fuse_rk3588.git \
  sd-fuse

git -C /home/ubuntu/work/sd-fuse \
  checkout --detach \
  c4fd64218b6f68cb136da6b9d43fc069383e68fe

test "$(
  git -C /home/ubuntu/work/sd-fuse rev-parse HEAD
)" = "c4fd64218b6f68cb136da6b9d43fc069383e68fe"
```

Clone FriendlyElec's Linux 6.1 vendor kernel:

```bash
cd /home/ubuntu/work/sd-fuse

git clone \
  --branch nanopi6-v6.1.y \
  --single-branch \
  https://github.com/friendlyarm/kernel-rockchip.git \
  kernel

git -C /home/ubuntu/work/sd-fuse/kernel \
  checkout --detach \
  474962f3f7ff699a8e021b77183c667174088154

test "$(
  git -C /home/ubuntu/work/sd-fuse/kernel rev-parse HEAD
)" = "474962f3f7ff699a8e021b77183c667174088154"
```

Clone the repository containing security patches:

```bash
cd /home/ubuntu/work

git clone \
  https://github.com/fviolence/cm3588-kernel-security-backports \
  cm3588-kernel-security-backports
```

Return to the kernel repository:

```bash
cd /home/ubuntu/work/sd-fuse/kernel
```

Record the vendor baseline:

```bash
git rev-parse HEAD
git log -1 --oneline --decorate

grep -E \
  '^(VERSION|PATCHLEVEL|SUBLEVEL|EXTRAVERSION) =' \
  Makefile
```

The source version must be Linux `6.1.141`. Stop and review the patch series if the vendor branch has moved to another version.

Create a local working branch:

```bash
git switch -c cm3588-security-6.1.141
```

---

## 4. Patch-series overview and provenance

The patches must be applied in filename order.

### `0001-scatterwalk-backport-memcpy-sglist.patch`

Backports `memcpy_sglist()` to the Linux 6.1 crypto scatterwalk implementation.

This is the first prerequisite for the Copy Fail repair. It originates from the Linux 6.1 stable backport:

```text
aa2a7743a44be64db6e0318fb4fec702e16a6098
crypto: scatterwalk - Backport memcpy_sglist()
```

### `0002-algif-aead-use-memcpy-sglist.patch`

Changes `algif_aead` to copy associated data through `memcpy_sglist()` instead of using a null skcipher.

This is the second Copy Fail prerequisite. It originates from the Linux 6.1 stable backport:

```text
575dfd125c8e842078a16034c7f542983f087246
crypto: algif_aead - use memcpy_sglist() instead of null skcipher
```

### `0003-copy-fail-operate-out-of-place.patch`

Restores out-of-place operation in `algif_aead`, removing the unsafe in-place complexity that permits Copy Fail page-cache corruption.

It is the main repair for CVE-2026-31431 and originates from the Linux 6.1 stable backport:

```text
961cfa271a918ad4ae452420e7c303149002875b
crypto: algif_aead - Revert to operating out-of-place
```

### `0004-dirty-frag-esp-6.1.patch`

Backports the Dirty Frag ESP repair so ESP decryption does not operate in place on shared skb fragments.

This addresses the ESP path of CVE-2026-43284.

### `0005-dirty-frag-esp-6.1-fixup.patch`

Applies the associated Linux 6.1 IPv4 ESP flags correction required by the preceding ESP backport.

The exact upstream/stable commit metadata and sign-offs are retained in the format-patch header.

### `0006-dirty-frag-rxrpc-shared-paged-frags-6.1-friendlyelec.patch`

This patch is a handwritten backport adapted to the older RxRPC source layout in FriendlyElec’s Linux 6.1.141 tree.

```text
aa54b1d27fe0c2b78e664a34fd0fdf7cd1960d71
rxrpc: Also unshare DATA/RESPONSE packets when paged frags are present
```

The original upstream repair targets a newer RxRPC layout:

* DATA processing in `rxrpc_input_call_event()`
* RESPONSE processing in `rxrpc_verify_response()`

FriendlyElec's older 6.1.141 tree has a different structure:

* encrypted DATA skb preparation occurs in `rxrpc_input_packet()` in `net/rxrpc/input.c`
* RxKAD then maps that DATA skb through `skb_to_sgvec()` and decrypts it in place
* RESPONSE verification copies the response and ticket out of the skb with `skb_copy_bits()` and decrypts those private allocations

Consequently, the vendor tree needs only the DATA-side adaptation. The patch forces `skb_copy()` when `skb_has_frag_list()` or `skb_has_shared_frag()` is true, while retaining `skb_unshare()` for ordinary or cloned skbs. It preserves the ownership behavior expected by the existing RxRPC tracing and out-of-memory path.

This repairs the affected path for CVE-2026-43500 without adding an unnecessary RESPONSE-side copy.

### `0007-ghostlock-6.1.patch`

Backports the RT-mutex correction:

```text
rtmutex: Use waiter::task instead of current in remove_waiter()
```

This addresses CVE-2026-43499 ("GhostLock"). The exact source commit, authorship, and sign-offs are preserved in the format-patch header.

This patch must be followed immediately by `0008`. On its own, the GhostLock fix introduces CVE-2026-53163 when `remove_waiter()` is called for a waiter that was never enqueued.

### `0008-locking-rtmutex-Skip-remove_waiter-when-waiter-is-no.patch`

Fixes CVE-2026-53163, a regression introduced by the GhostLock repair in `0007`. During `FUTEX_CMP_REQUEUE_PI`, deadlock detection can return before a waiter is enqueued, leaving `waiter->task` unset. The GhostLock version of `remove_waiter()` then dereferences that null task pointer and can crash the kernel.

The patch makes two related corrections:

* `remove_waiter()` returns immediately when `waiter->task` is null because there is no enqueued waiter state to remove.
* `rt_mutex_start_proxy_lock()` calls `remove_waiter()` only for negative error returns; a positive return means that the proxy task acquired the mutex successfully.

This is a handwritten backport of upstream commit:

```text
40a25d59e85b3c8709ac2424d44f65610467871e
locking/rtmutex: Skip remove_waiter() when waiter is not enqueued
```

It is adapted to FriendlyElec's older Linux 6.1.141 proxy-lock API, where `__rt_mutex_start_proxy_lock()` does not take a `wake_q` argument and the wrapper lacks the newer wake-queue and preemption handling. Apply it only after `0007-ghostlock-6.1.patch`.

Inspect the metadata embedded in all patches:

```bash
sed -n '1,/^---$/p' /home/ubuntu/work/cm3588-kernel-security-backports/*.patch
```

---

## 5. Apply the patches

Ensure the tree is initially clean:

```bash
cd /home/ubuntu/work/sd-fuse/kernel

git status --short
```

Apply every format patch explicitly and in order:

```bash
git am /home/ubuntu/work/cm3588-kernel-security-backports/0001-scatterwalk-backport-memcpy-sglist.patch
git am /home/ubuntu/work/cm3588-kernel-security-backports/0002-algif-aead-use-memcpy-sglist.patch
git am /home/ubuntu/work/cm3588-kernel-security-backports/0003-copy-fail-operate-out-of-place.patch
git am /home/ubuntu/work/cm3588-kernel-security-backports/0004-dirty-frag-esp-6.1.patch
git am /home/ubuntu/work/cm3588-kernel-security-backports/0005-dirty-frag-esp-6.1-fixup.patch
git am /home/ubuntu/work/cm3588-kernel-security-backports/0006-dirty-frag-rxrpc-shared-paged-frags-6.1-friendlyelec.patch
git am /home/ubuntu/work/cm3588-kernel-security-backports/0007-ghostlock-6.1.patch
git am /home/ubuntu/work/cm3588-kernel-security-backports/0008-locking-rtmutex-Skip-remove_waiter-when-waiter-is-no.patch
```

After successful application:

```bash
git status --short
git log --oneline --decorate -14
```

The working tree should be clean.

### 5.1. Optional: enable Waydroid kernel support

Waydroid requires Android Binder IPC and BinderFS. The FriendlyElec Linux 6.1.141 source provides both, but the CM3588 defconfig disables Binder.

Edit the **source defconfig** before building: FriendlyElec's build script regenerates `kernel/.config` on each clean build.

```bash
cd /home/ubuntu/work/sd-fuse/kernel

scripts/config \
  --file arch/arm64/configs/nanopi6_linux_defconfig \
  --enable ANDROID_BINDER_IPC \
  --enable ANDROID_BINDERFS \
  --set-str ANDROID_BINDER_DEVICES "binder,hwbinder,vndbinder"

git diff --check
git diff -- arch/arm64/configs/nanopi6_linux_defconfig

git add arch/arm64/configs/nanopi6_linux_defconfig
git commit -m 'config: enable Binder IPC and BinderFS for Waydroid'
```

The tested configuration already enables `CONFIG_MEMFD_CREATE=y`; legacy ashmem is not required. Binder support is necessary but does not yet establish Waydroid graphics or application compatibility.

---

## 6. Set a unique kernel release

Choose a unique `KSEC` value for this build; use a new value whenever kernel binaries or configuration change. For example, use `sec3` if `sec2` is already installed. Set it **once on the build machine** here. The target machine sets the matching value once in Section 14.

Update `EXTRAVERSION` only after applying the security patches:

```bash
KSEC=sec1
cd /home/ubuntu/work/sd-fuse/kernel

sed -i \
  "s/^EXTRAVERSION =.*/EXTRAVERSION = -cm3588-${KSEC}/" \
  Makefile
```

Review and commit the change:

```bash
git diff --check
git diff -- Makefile

git add Makefile
git commit -m 'build: identify CM3588 security kernel'
```

Verify:

```bash
grep '^EXTRAVERSION' Makefile
git status --short
git log --oneline --decorate -14
```

---

## 7. Build the kernel, modules, third-party drivers, and headers

Return to the `sd-fuse` directory:

```bash
cd /home/ubuntu/work/sd-fuse
```

The build script runs `make distclean` by default and recreates `out/output_rk3588_kmodules`, so an ordinary rebuild already cleans the previous kernel build and staged modules. Leave `SKIP_DISTCLEAN` unset; **do not delete the entire `sd-fuse/out` directory**, which also contains downloaded external-driver sources and packaging outputs.

Do not add arbitrary filenames to `KCFG`: FriendlyElec treats its words as targets under `arch/arm64/configs/`. The optional Binder settings are stored in the board defconfig; no extra `KCFG` fragment is needed.

Build without downloading or modifying a complete OS image:

```bash
DISABLE_MKIMG=1 \
MK_HEADERS_DEB=1 \
BUILD_THIRD_PARTY_DRIVER=1 \
KERNEL_SRC="/home/ubuntu/work/sd-fuse/kernel" \
./build-kernel.sh debian-trixie-core-arm64
```

Environment-variable purpose:

* `DISABLE_MKIMG=1`: stop after kernel, modules, and headers are built; do not update a full root-filesystem image
* `MK_HEADERS_DEB=1`: produce the FriendlyElec-adjusted headers package
* `BUILD_THIRD_PARTY_DRIVER=1`: include the optional external Realtek Wi-Fi driver set
* `KERNEL_SRC="/home/ubuntu/work/sd-fuse/kernel"`: use the patched local kernel checkout

FriendlyElec's build script also builds `r8125`, `cryptodev-linux`, `nft_fullcone`, and an external `rtw88` tree independently of the optional USB Wi-Fi switch.

Determine the resulting kernel release:

```bash
KREL="$(
  make -s \
    -C kernel \
    ARCH=arm64 \
    kernelrelease
)"

printf 'KREL=%s\n' "$KREL"
```

The release must reflect the `KSEC` selected in Section 6 (for example, `KREL=6.1.141-cm3588-sec3`).

If Section 5.1 was used, verify the **generated** configuration after the build:

```bash
grep -E \
  '^CONFIG_ANDROID_(BINDER_IPC|BINDERFS|BINDER_DEVICES)=' \
  kernel/.config
```

Expected:

```text
CONFIG_ANDROID_BINDER_IPC=y
CONFIG_ANDROID_BINDERFS=y
CONFIG_ANDROID_BINDER_DEVICES="binder,hwbinder,vndbinder"
```

Confirm all principal outputs:

```bash
test -s kernel/kernel.img
test -s kernel/resource.img
test -s "out/linux-headers-${KREL}.deb"
test -d "out/output_rk3588_kmodules/lib/modules/${KREL}"
```

Display them:

```bash
ls -lh \
  kernel/kernel.img \
  kernel/resource.img \
  "out/linux-headers-${KREL}.deb"

du -sh "out/output_rk3588_kmodules/lib/modules/${KREL}"

find \
  "out/output_rk3588_kmodules/lib/modules/${KREL}" \
  -type f -name '*.ko*' |
wc -l
```

Preserve the final kernel configuration and patch history:

```bash
cp kernel/.config "out/config-${KREL}"

git -C kernel log --oneline --decorate -14 > "out/git-log-${KREL}.txt"

git -C kernel rev-parse HEAD > "out/git-commit-${KREL}.txt"
```

---

## 8. Repair the generated ARM64 headers package

FriendlyElec's headers packaging copies prebuilt ARM64 Kbuild helpers from:

```text
files/linux-headers-5.10-bin_arm64/scripts
```

into the generated Linux 6.1 headers package.

Two of those helpers are incompatible with the Linux 6.1 module versioning pipeline:

```text
scripts/genksyms/genksyms
scripts/mod/modpost
```

When the generated headers package is used with `CONFIG_MODVERSIONS=y`, external modules that export symbols may receive zero CRC values. The resulting modules can lack a valid `__kcrctab` section and may be force-loaded without normal exported symbol ABI validation.

The kernel image and in-tree modules are not affected. Repair the generated headers package before transferring it to the target or building any DKMS modules.

### Set the package and tool paths

Run these commands from the `sd-fuse` directory:

```bash
cd /home/ubuntu/work/sd-fuse

KREL="$(
  make -s \
    -C kernel \
    ARCH=arm64 \
    kernelrelease
)"

TOOLS_DIR="/home/ubuntu/work/cm3588-kernel-security-backports/tools/linux-6.1.141-arm64"
HEADERS_DEB="/home/ubuntu/work/sd-fuse/out/linux-headers-${KREL}.deb"
ORIGINAL_HEADERS_DEB="/home/ubuntu/work/sd-fuse/out/linux-headers-${KREL}.friendlyelec-original.deb"
REPACK_ROOT="/home/ubuntu/work/sd-fuse/out/repack-linux-headers-${KREL}"
VERIFY_ROOT="/home/ubuntu/work/sd-fuse/out/verify-linux-headers-${KREL}"
```

Display the selected paths:

```bash
printf 'KREL=%s\n' "$KREL"
printf 'TOOLS_DIR=%s\n' "$TOOLS_DIR"
printf 'HEADERS_DEB=%s\n' "$HEADERS_DEB"
```

Confirm that the generated package and replacement tools exist:

```bash
test -s "$HEADERS_DEB"
test -x "$TOOLS_DIR/scripts/genksyms/genksyms"
test -x "$TOOLS_DIR/scripts/mod/modpost"
```

### Verify the supplied tools

Verify their recorded checksums:

```bash
(
  cd "$TOOLS_DIR"
  sha256sum -c SHA256SUMS
)
```

Expected:

```text
scripts/genksyms/genksyms: OK
scripts/mod/modpost: OK
```

Confirm that both are ARM64 executables:

```bash
file "$TOOLS_DIR/scripts/genksyms/genksyms" "$TOOLS_DIR/scripts/mod/modpost"
```

### Extract the generated headers package

Preserve the unmodified FriendlyElec-generated package:

```bash
cp -a "$HEADERS_DEB" "$ORIGINAL_HEADERS_DEB"
```

Remove any previous unpacking trees:

```bash
rm -rf "$REPACK_ROOT" "$VERIFY_ROOT"
```

Extract both the filesystem payload and Debian control metadata:

```bash
dpkg-deb --raw-extract "$ORIGINAL_HEADERS_DEB" "$REPACK_ROOT"
```

Set the extracted headers-tree path:

```bash
HEADER_ROOT="$REPACK_ROOT/usr/src/linux-headers-${KREL}"
```

Confirm that the expected locations exist:

```bash
test -d "$HEADER_ROOT"
test -d "$HEADER_ROOT/scripts/genksyms"
test -d "$HEADER_ROOT/scripts/mod"
test -f "$REPACK_ROOT/DEBIAN/control"
```

Display the incompatible binaries originally included by
FriendlyElec:

```bash
file "$HEADER_ROOT/scripts/genksyms/genksyms" "$HEADER_ROOT/scripts/mod/modpost"
sha256sum "$HEADER_ROOT/scripts/genksyms/genksyms" "$HEADER_ROOT/scripts/mod/modpost"
```

### Install the matching Linux 6.1 ARM64 helpers

Replace `genksyms`:

```bash
install \
  -m 0755 \
  "$TOOLS_DIR/scripts/genksyms/genksyms" \
  "$HEADER_ROOT/scripts/genksyms/genksyms"
```

Replace `modpost`:

```bash
install \
  -m 0755 \
  "$TOOLS_DIR/scripts/mod/modpost" \
  "$HEADER_ROOT/scripts/mod/modpost"
```

Verify that the extracted package now contains byte-identical copies:

```bash
cmp "$TOOLS_DIR/scripts/genksyms/genksyms" "$HEADER_ROOT/scripts/genksyms/genksyms"
cmp "$TOOLS_DIR/scripts/mod/modpost" "$HEADER_ROOT/scripts/mod/modpost"
```

No output means that the comparisons succeeded.

Verify the installed hashes:

```bash
sha256sum "$HEADER_ROOT/scripts/genksyms/genksyms" "$HEADER_ROOT/scripts/mod/modpost"
```

Expected:

```bash
cat $TOOLS_DIR/SHA256SUMS
```

### Regenerate the package file checksums

The package's existing `DEBIAN/md5sums` file still describes the replaced binaries. Regenerate it from the modified payload:

```bash
(
  cd "$REPACK_ROOT"
  find . -type f ! -path './DEBIAN/*' -printf '%P\0' | sort -z | xargs -r0 md5sum > DEBIAN/md5sums
)
```

Confirm that the two replacement files are listed:

```bash
grep -E \
  'scripts/(genksyms/genksyms|mod/modpost)$' \
  "$REPACK_ROOT/DEBIAN/md5sums"
```

### Rebuild the corrected headers package

Rebuild the package at its original final pathname:

```bash
dpkg-deb \
  --root-owner-group \
  -Zxz \
  -b "$REPACK_ROOT" \
  "$HEADERS_DEB"
```

The unmodified package remains available as:

```text
out/linux-headers-<kernel-release>.friendlyelec-original.deb
```

The corrected package is:

```text
out/linux-headers-<kernel-release>.deb
```

### Verify the rebuilt package

Inspect its package metadata:

```bash
dpkg-deb --info "$HEADERS_DEB"
```

Extract the rebuilt package into a fresh verification directory:

```bash
dpkg-deb --raw-extract "$HEADERS_DEB" "$VERIFY_ROOT"
```

Set the verification headers path:

```bash
VERIFY_HEADERS="$VERIFY_ROOT/usr/src/linux-headers-${KREL}"
```

Confirm the packaged tools are ARM64 executables:

```bash
file \
  "$VERIFY_HEADERS/scripts/genksyms/genksyms" \
  "$VERIFY_HEADERS/scripts/mod/modpost"
```

Verify their hashes:

```bash
sha256sum \
  "$VERIFY_HEADERS/scripts/genksyms/genksyms" \
  "$VERIFY_HEADERS/scripts/mod/modpost"
```

Expected:

```bash
cat $TOOLS_DIR/SHA256SUMS
```

Compare them directly with the repository copies:

```bash
cmp \
  "$TOOLS_DIR/scripts/genksyms/genksyms" \
  "$VERIFY_HEADERS/scripts/genksyms/genksyms"

cmp \
  "$TOOLS_DIR/scripts/mod/modpost" \
  "$VERIFY_HEADERS/scripts/mod/modpost"
```

No output means that both comparisons succeeded.

Verify all package payload checksums:

```bash
(
  cd "$VERIFY_ROOT"
  md5sum -c DEBIAN/md5sums
)
```

Every file must report `OK`.

Record the final package checksum:

```bash
sha256sum "$HEADERS_DEB"
```

The temporary extraction trees may now be removed:

```bash
rm -rf "$REPACK_ROOT" "$VERIFY_ROOT"
```

Keep the `.friendlyelec-original.deb` package until the corrected package has been installed and validated successfully.

---

## 9. Remove the duplicate generic `cryptodev.ko`

FriendlyElec's kernel tree contains an in-tree Rockchip cryptodev implementation:

```text
kernel/drivers/crypto/rockchip/cryptodev_linux/cryptodev.ko
```

`rk_crypto.ko` requires Rockchip-specific exported symbols from that module:

```text
rk_cryptodev_register_dev
rk_cryptodev_unregister_dev
```

The build script also builds the unrelated generic `cryptodev-linux` project and copies another module with the same module name to:

```text
/lib/modules/<release>/cryptodev.ko
```

When both are packaged, module-name resolution can select the generic module. `depmod` then fails to link `rk_crypto.ko` to the Rockchip provider, producing unknown-symbol errors at boot.

FriendlyElec's official kernel package contains only the in-tree Rockchip copy. Reproduce that result before packaging.

Set the staging paths:

```bash
cd /home/ubuntu/work/sd-fuse

KMODROOT="/home/ubuntu/work/sd-fuse/out/output_rk3588_kmodules"
MODULE_DIR="$KMODROOT/lib/modules/$KREL"

GENERIC_CRYPTODEV="$MODULE_DIR/cryptodev.ko"
ROCKCHIP_CRYPTODEV="$MODULE_DIR/kernel/drivers/crypto/rockchip/cryptodev_linux/cryptodev.ko"
RK_CRYPTO="$MODULE_DIR/kernel/drivers/crypto/rockchip/rkcrypto/rk_crypto.ko"
```

Confirm the collision:

```bash
find "$MODULE_DIR" -type f -name 'cryptodev.ko*' -print
```

Confirm the Rockchip copy exports the required symbols:

```bash
readelf -Ws "$ROCKCHIP_CRYPTODEV" | grep -E 'rk_cryptodev_(register|unregister)_dev'
```

Confirm the consumer exists:

```bash
test -f "$RK_CRYPTO"
```

Remove only the root-level generic module:

```bash
rm -v "$GENERIC_CRYPTODEV"
```

Regenerate module dependency metadata against the staging root:

```bash
depmod -a -b "$KMODROOT" "$KREL"
```

Confirm exactly one cryptodev provider remains:

```bash
find "$MODULE_DIR" -type f -name 'cryptodev.ko*' -print
```

Expected:

```text
.../kernel/drivers/crypto/rockchip/cryptodev_linux/cryptodev.ko
```

Confirm `rk_crypto` now depends on it:

```bash
grep '^kernel/drivers/crypto/rockchip/rkcrypto/rk_crypto\.ko:' "$MODULE_DIR/modules.dep"
```

Expected:

```text
kernel/drivers/crypto/rockchip/rkcrypto/rk_crypto.ko: kernel/drivers/crypto/rockchip/cryptodev_linux/cryptodev.ko
```

---

## 10. Build the kernel-image `.deb`

The FriendlyElec build script creates the headers package under `out/`, but it deletes the generic `linux-image` package generated internally by `bindeb-pkg`. A generic Debian kernel image package is also insufficient by itself for this platform because the board boots `kernel.img` and `resource.img` from raw GPT partitions.

The following package contains:

* `kernel.img`
* `resource.img`
* `/lib/modules/<release>`
* kernel configuration
* `System.map`
* `Module.symvers`, when available

It does not flash partitions automatically.

Set package metadata:

```bash
cd /home/ubuntu/work/sd-fuse

PKG_VERSION="6.1.141-16+cm3588${KSEC//-/}"
PKG_NAME="linux-image-${KREL}"
PKG_ROOT="/home/ubuntu/work/sd-fuse/out/pkg-${PKG_NAME}"
IMAGE_DEB="/home/ubuntu/work/sd-fuse/out/${PKG_NAME}_${PKG_VERSION}_arm64.deb"
```

Remove an old packaging tree and create the new directory layout:

```bash
rm -rf "$PKG_ROOT"

mkdir -p \
  "$PKG_ROOT/DEBIAN" \
  "$PKG_ROOT/lib/modules" \
  "$PKG_ROOT/usr/lib/cm3588-kernel/${KREL}" \
  "$PKG_ROOT/boot"
```

Copy the sanitized module tree:

```bash
cp -a "$MODULE_DIR" "$PKG_ROOT/lib/modules/"
```

Install the boot images into the package payload:

```bash
install -m 0644 \
  kernel/kernel.img \
  "$PKG_ROOT/usr/lib/cm3588-kernel/${KREL}/kernel.img"

install -m 0644 \
  kernel/resource.img \
  "$PKG_ROOT/usr/lib/cm3588-kernel/${KREL}/resource.img"
```

Install diagnostic/build metadata:

```bash
install -m 0644 \
  kernel/.config \
  "$PKG_ROOT/boot/config-${KREL}"

install -m 0644 \
  kernel/System.map \
  "$PKG_ROOT/boot/System.map-${KREL}"
```

Copy `Module.symvers` when present:

```bash
if test -f kernel/Module.symvers; then
  install -m 0644 \
    kernel/Module.symvers \
    "$PKG_ROOT/usr/lib/cm3588-kernel/${KREL}/Module.symvers"
fi
```

Create the Debian control file:

```bash
cat > "$PKG_ROOT/DEBIAN/control" <<EOF
Package: ${PKG_NAME}
Version: ${PKG_VERSION}
Section: kernel
Priority: optional
Architecture: arm64
Maintainer: Local Kernel Builder <kernel-builder@example.invalid>
Description: Security-patched FriendlyElec RK3588 Linux ${KREL}
 FriendlyElec RK3588 vendor kernel with selected Linux security
 backports. The package installs kernel modules, kernel.img and
 resource.img. Raw boot partitions must be backed up, written and
 verified explicitly by the administrator.
EOF
```

Build the package:

```bash
dpkg-deb \
  --root-owner-group \
  -Zxz \
  -b "$PKG_ROOT" \
  "$IMAGE_DEB"
```

Inspect it:

```bash
dpkg-deb --info "$IMAGE_DEB"
dpkg-deb --contents "$IMAGE_DEB"
```

Record checksums:

```bash
sha256sum "out/linux-headers-${KREL}.deb" "$IMAGE_DEB"
```

The final package paths, derived from the chosen `KSEC`, are:

```bash
printf '%s\n' $(realpath "out/linux-headers-${KREL}.deb") $(realpath "$IMAGE_DEB")
```

---

## 11. Transfer the packages to the target

Copy both packages to the target by a method appropriate for the environment. For example:

From the `sd-fuse` directory, where `KREL` and `IMAGE_DEB` were set on the build machine:

```bash
scp \
  "out/linux-headers-${KREL}.deb" \
  "$IMAGE_DEB" \
  user@target-host:/path/to/somewhere
```

The remaining commands are run on the target board.

---

## 12. Identify and verify the raw boot partitions

Display the complete block-device layout:

```bash
lsblk -rpo NAME,PATH,SIZE,TYPE,FSTYPE,PARTLABEL,MOUNTPOINTS
```

Display the root filesystem source:

```bash
findmnt -no SOURCE /
```

Locate partitions named exactly `resource` and `kernel`:

```bash
lsblk -rpn -o PATH,PARTLABEL | awk '$2 == "resource" || $2 == "kernel"'
```

Assign the exact devices after verifying the output.

Example only:

```bash
RESOURCE_PART=/dev/mmcblk2p4
KERNEL_PART=/dev/mmcblk2p5
```

Do not copy the example device names blindly. A removable SD card, alternate boot medium, or different partition table can change them.

Verify the selected devices and labels:

```bash
lsblk -no PATH,SIZE,TYPE,PARTLABEL "$RESOURCE_PART" "$KERNEL_PART"
```

Expected labels:

```text
resource
kernel
```

Verify that both variables refer to block devices:

```bash
test -b "$RESOURCE_PART"
test -b "$KERNEL_PART"
```

---

## 13. Back up the currently working kernel

Create a timestamped backup directory:

```bash
BACKUP_DIR="$PWD/cm3588-kernel-backup/$(date -u +%Y%m%dT%H%M%SZ)-$(uname -r)"
sudo install -d -m 0700 "$BACKUP_DIR"
```

Save the block-device layout and running-kernel information:

```bash
sudo sh -c "lsblk -rpo NAME,PATH,SIZE,TYPE,FSTYPE,PARTLABEL,MOUNTPOINTS > '$BACKUP_DIR/lsblk.txt'"
uname -a | sudo tee "$BACKUP_DIR/uname.txt" >/dev/null
```

Back up the complete current `resource` partition:

```bash
sudo dd \
  if="$RESOURCE_PART" \
  of="$BACKUP_DIR/resource.img" \
  bs=4M \
  iflag=fullblock \
  status=progress \
  conv=fsync
```

Back up the complete current `kernel` partition:

```bash
sudo dd \
  if="$KERNEL_PART" \
  of="$BACKUP_DIR/kernel.img" \
  bs=4M \
  iflag=fullblock \
  status=progress \
  conv=fsync
```

Back up the currently running module tree:

```bash
sudo tar -C / -czf "$BACKUP_DIR/modules.tar.gz" "lib/modules/$(uname -r)"
```

Record checksums:

```bash
sudo sha256sum \
  "$BACKUP_DIR/resource.img" \
  "$BACKUP_DIR/kernel.img" \
  "$BACKUP_DIR/modules.tar.gz" |
sudo tee "$BACKUP_DIR/SHA256SUMS"
```

Confirm the files are non-empty:

```bash
sudo ls -lh "$BACKUP_DIR"
```

Keep a copy of this backup somewhere accessible from rescue media.

---

## 14. Install the packages without rebooting

Set the expected new release:

```bash
KSEC=sec1
KREL="6.1.141-cm3588-${KSEC}"
```

Install the headers first:

```bash
sudo dpkg -i "./linux-headers-${KREL}.deb"
```

Install the image/module payload:

```bash
sudo dpkg -i "./linux-image-${KREL}_6.1.141-16+cm3588${KSEC//-/}_arm64.deb"
```

Confirm installation:

```bash
dpkg -s "linux-headers-${KREL}" "linux-image-${KREL}"
```

Confirm the module and image payloads exist:

```bash
test -d "/lib/modules/${KREL}"
test -s "/usr/lib/cm3588-kernel/${KREL}/kernel.img"
test -s "/usr/lib/cm3588-kernel/${KREL}/resource.img"
```

Do not reboot yet.

---

## 15. Rebuild DKMS modules

List current DKMS modules:

```bash
sudo dkms status
```

Build every registered DKMS module for the new kernel:

```bash
sudo dkms autoinstall -k "$KREL"
```

Regenerate module dependencies:

```bash
sudo depmod -a "$KREL"
```

Check DKMS again:

```bash
sudo dkms status

sudo find /var/lib/dkms \
  -type f \
  -path "*${KREL}*/log/make.log" \
  -exec grep -HnE \
    'has no CRC|version generation failed' \
    {} + || true
```

Every out-of-tree module required for boot, networking, storage, or services must show as installed for the new release.

Examples:

```bash
sudo modinfo -k "$KREL" r8125
```

If ZFS is used:

```bash
sudo modinfo -k "$KREL" zfs
```

```bash
find "/lib/modules/${KREL}" \
  -type f \
  \( -name 'zfs.ko*' -o -name 'spl.ko*' \) \
  -print

SPL_MODULE="$(sudo modinfo -k "$KREL" -n spl)"

readelf -SW "$SPL_MODULE" |
grep -E '__versions|__ksymtab|__kcrctab'
```

A correctly versioned SPL module must contain all three relevant sections:

```text
__versions
__ksymtab
__kcrctab
```

Do not reboot if a required DKMS module failed to build.

---

## 16. Pre-reboot module validation

Confirm the module metadata exists:

```bash
test -s "/lib/modules/${KREL}/modules.dep"
test -s "/lib/modules/${KREL}/modules.alias"
```

Look for broken symlinks:

```bash
find "/lib/modules/${KREL}" -xtype l -print
```

An absent `build` or `source` symlink is expected in this package. Unexpected broken links to module files are not.

Verify the Realtek Ethernet module:

```bash
sudo modinfo -k "$KREL" r8125 | grep -E '^(filename|version|vermagic):'
```

Verify the Rockchip cryptodev provider:

```bash
sudo modinfo -k "$KREL" cryptodev | grep -E '^(filename|name|description|vermagic):'
```

The filename must end with:

```text
kernel/drivers/crypto/rockchip/cryptodev_linux/cryptodev.ko
```

Verify the `rk_crypto` dependency:

```bash
grep \
  '^kernel/drivers/crypto/rockchip/rkcrypto/rk_crypto\.ko:' \
  "/lib/modules/${KREL}/modules.dep"
```

Dry-run the dependency resolution for the new kernel:

```bash
sudo modprobe -S "$KREL" --show-depends rk_crypto
```

Expected order:

```text
.../cryptodev_linux/cryptodev.ko
.../rkcrypto/rk_crypto.ko
```

Dry-run other required modules:

```bash
sudo modprobe -S "$KREL" -n -v r8125
```

If ZFS is used:

```bash
sudo modprobe -S "$KREL" -n -v zfs
```

---

## 17. Check image and partition sizes

Set the installed image paths:

```bash
RESOURCE_IMAGE="/usr/lib/cm3588-kernel/${KREL}/resource.img"
KERNEL_IMAGE="/usr/lib/cm3588-kernel/${KREL}/kernel.img"
```

Display sizes:

```bash
stat -c '%n: %s bytes' "$RESOURCE_IMAGE" "$KERNEL_IMAGE"

sudo blockdev --getsize64 "$RESOURCE_PART"
sudo blockdev --getsize64 "$KERNEL_PART"
```

Verify that each image fits its target partition:

```bash
test \
  "$(stat -c %s "$RESOURCE_IMAGE")" \
  -le \
  "$(sudo blockdev --getsize64 "$RESOURCE_PART")"
```

```bash
test \
  "$(stat -c %s "$KERNEL_IMAGE")" \
  -le \
  "$(sudo blockdev --getsize64 "$KERNEL_PART")"
```

Stop if either test fails.

---

## 18. Write and verify the raw partitions

Write `resource.img` first:

```bash
sudo dd \
  if="$RESOURCE_IMAGE" \
  of="$RESOURCE_PART" \
  bs=4M \
  status=progress \
  conv=fsync,notrunc
```

Verify the written bytes:

```bash
sudo cmp \
  -n "$(stat -c %s "$RESOURCE_IMAGE")" \
  "$RESOURCE_IMAGE" \
  "$RESOURCE_PART"
```

No output means the comparison succeeded.

Write `kernel.img`:

```bash
sudo dd \
  if="$KERNEL_IMAGE" \
  of="$KERNEL_PART" \
  bs=4M \
  status=progress \
  conv=fsync,notrunc
```

Verify the written bytes:

```bash
sudo cmp \
  -n "$(stat -c %s "$KERNEL_IMAGE")" \
  "$KERNEL_IMAGE" \
  "$KERNEL_PART"
```

No output means the comparison succeeded.

Flush outstanding writes:

```bash
sudo sync
```

At this point:

* the old raw partitions have been backed up
* the new headers and modules are installed
* required DKMS modules have been built
* module dependency resolution has been checked
* both new raw images have been written and byte-compared

Reboot:

```bash
sudo reboot
```

---

## 19. Validate the new kernel after reboot

Confirm the exact release:

```bash
uname -r
uname -a
cat /proc/version
```

The result must match the kernel release selected in Section 14 (not the previously running kernel).

Check failed services:

```bash
sudo systemctl --failed --no-pager
```

Check networking:

```bash
ip -brief link
ip -brief address
```

Check the Ethernet driver:

```bash
sudo lsmod | grep -E '^r8125\b'
```

```bash
sudo dmesg -T | grep -Ei 'r8125|link up|firmware|unknown symbol|invalid module' | tail -100
```

Check storage:

```bash
findmnt
lsblk -f
```

If ZFS is used:

```bash
sudo lsmod | grep -E '^(zfs|spl)\b'
```

```bash
sudo zpool status
sudo zfs list

SPL_MODULE="$(sudo modinfo -n spl)"

readelf -SW "$SPL_MODULE" |
grep -E '__versions|__ksymtab|__kcrctab'

cat /sys/module/spl/taint
cat /sys/module/zfs/taint
cat /proc/sys/kernel/tainted
```

> A correctly versioned SPL module must contain __kcrctab. O is expected for an out-of-tree module and P is expected for ZFS because of its CDDL licence classification. F must not be present; it indicates that a module was force-loaded.

If containers are used:

```bash
sudo docker ps
```

Check for serious kernel errors:

```bash
sudo dmesg -T | grep -Ei 'Oops:|Kernel panic|Call trace:|BUG:|Unknown symbol|invalid module|failed to load' | tail -200
sudo journalctl -k -b --no-pager | grep -Ei 'spl:|zfs:|no versions for exported symbols|has no CRC'
```

---

## 20. Validate the Rockchip crypto accelerator

Load the accelerator:

```bash
sudo modprobe rk_crypto
```

Confirm both modules:

```bash
sudo lsmod | grep -E '^(rk_crypto|cryptodev)\b'
```

Confirm the device node:

```bash
ls -l /dev/crypto
```

Inspect messages:

```bash
sudo dmesg -T | grep -Ei 'rk.crypto|rk_crypto|cryptodev|unknown symbol' | tail -50
```

A healthy registration includes messages equivalent to:

```text
cryptodev: driver 1.12 loaded
rk-crypto ...: register to cryptodev ok!
rk-crypto ...: Accelerator successfully registered
```

---

## 21. Validate RK3588 hardware-acceleration interfaces

Check the expected device nodes:

```bash
ls -l \
  /dev/mpp_service \
  /dev/rga \
  /dev/mali0 \
  /dev/dri \
  /dev/dma_heap \
  2>/dev/null
```

Inspect the `clinfo` and relevant kernel logs:

```bash
clinfo

sudo dmesg -T | grep -Ei 'mpp|rga|mali|panthor|panfrost|drm|vpu|iommu' | tail -200
```

For a Jellyfin container using Jellyfin FFmpeg, initialize RKMPP and OpenCL explicitly:

```bash
sudo docker exec \
  jellyfin \
  /usr/lib/jellyfin-ffmpeg/ffmpeg \
  -v debug \
  -init_hw_device rkmpp=rk \
  -init_hw_device opencl=ocl@rk
```

A successful result identifies the Mali-G610 OpenCL device and completes device initialization without an RKMPP or OpenCL error.

Confirm that the container sees all required devices:

```bash
sudo docker exec \
  jellyfin \
  sh -c \
  'ls -l /dev/dri /dev/dma_heap /dev/mali0 /dev/rga /dev/mpp_service'
```

An actual transcode remains the definitive end-to-end test.

---

## 22. Rollback

If the new kernel does not boot, use a rescue SD/eFlasher system or another method that provides block-device access.

Identify the correct `resource` and `kernel` partitions again. Do not assume their device paths.

Restore the saved images:

```bash
sudo dd \
  if=<backup-directory>/resource.img \
  of=<resource-partition> \
  bs=4M \
  status=progress \
  conv=fsync,notrunc
```

```bash
sudo dd \
  if=<backup-directory>/kernel.img \
  of=<kernel-partition> \
  bs=4M \
  status=progress \
  conv=fsync,notrunc
```

Flush writes:

```bash
sudo sync
```

The old `/lib/modules/<old-release>` directory is intentionally left installed, so restoring the old raw kernel and resource partitions should restore the previous kernel/module pairing.

---

## 23. Rebuilding a later revision

For the next release:

1. Start from the same verified vendor baseline, or review and rebase against a newer baseline.
2. Apply the patches in the same order.
3. Increment `EXTRAVERSION`.
4. Increment the Debian package version.
5. Build the kernel, modules, and headers.
6. Repair the generated headers `.deb` with the matching ARM64 `genksyms` and `modpost`.
7. Verify the repaired package hashes and payload checksums.
8. Remove the root-level generic `cryptodev.ko`.
9. Rerun staged `depmod`.
10. Verify `rk_crypto` dependency resolution.
11. Create the kernel-image package.
12. Install and validate without removing the known-working prior kernel or module tree.

> Never reuse an existing kernel release string for different module binaries.
> Do not assume the supplied Kbuild helpers remain valid after changing the vendor kernel baseline. Revalidate or replace them when moving to another Kbuild source revision.

### 23.1 Cleanup after rebuild

Only after verifying the new kernel, and only if the old kernel is no longer needed for rollback, identify the obsolete package and DKMS versions:

```bash
dpkg -l 'linux-image-6.1.141-cm3588-*' 'linux-headers-6.1.141-cm3588-*'
sudo dkms status
```

Replace the placeholder with the **obsolete** release, not the active one:

```bash
OLD_KREL='6.1.141-cm3588-<old-release>'
sudo apt purge "linux-image-${OLD_KREL}" "linux-headers-${OLD_KREL}"
```

Remove obsolete DKMS instances only after confirming their exact module versions with `dkms status` and that the current kernel's ZFS module still loads. Do not remove the old kernel while relying on it as your rollback option.

---

## References

### FriendlyElec sources and build environment

* [FriendlyElec RK3588 build system — `kernel-6.1.y`](https://github.com/friendlyarm/sd-fuse_rk3588/tree/kernel-6.1.y)
* [FriendlyElec Rockchip kernel — `nanopi6-v6.1.y`](https://github.com/friendlyarm/kernel-rockchip/tree/nanopi6-v6.1.y)
* [FriendlyElec Docker cross-compilation environment](https://github.com/friendlyarm/docker-cross-compiler-novnc)

### Exact tested source revisions

- [`sd-fuse_rk3588` — `c4fd64218b6f68cb136da6b9d43fc069383e68fe`](https://github.com/friendlyarm/sd-fuse_rk3588/commit/c4fd64218b6f68cb136da6b9d43fc069383e68fe)
- [`kernel-rockchip` — `474962f3f7ff699a8e021b77183c667174088154`](https://github.com/friendlyarm/kernel-rockchip/commit/474962f3f7ff699a8e021b77183c667174088154)

### Debian package manipulation

- [Debian `dpkg-deb(1)` manual](https://manpages.debian.org/bookworm/dpkg/dpkg-deb.1.en.html)

### CVE records

* **CVE-2026-31431 — Copy Fail**

  * [CVE record](https://www.cve.org/CVERecord?id=CVE-2026-31431)
  * [NVD record](https://nvd.nist.gov/vuln/detail/CVE-2026-31431)

* **CVE-2026-43284 — Dirty Frag, ESP path**

  * [CVE record](https://www.cve.org/CVERecord?id=CVE-2026-43284)
  * [NVD record](https://nvd.nist.gov/vuln/detail/CVE-2026-43284)

* **CVE-2026-43500 — Dirty Frag, RxRPC path**

  * [CVE record](https://www.cve.org/CVERecord?id=CVE-2026-43500)
  * [NVD record](https://nvd.nist.gov/vuln/detail/CVE-2026-43500)

* **CVE-2026-43499 — GhostLock**

  * [CVE record](https://www.cve.org/CVERecord?id=CVE-2026-43499)
  * [NVD record](https://nvd.nist.gov/vuln/detail/CVE-2026-43499)

* **CVE-2026-53163 — RT-mutex GhostLock-fix regression**

  * [CVE record](https://www.cve.org/CVERecord?id=CVE-2026-53163)
  * [NVD record](https://nvd.nist.gov/vuln/detail/CVE-2026-53163)

* **CVE-2026-46242 — Bad Epoll**

  * [CVE record](https://www.cve.org/CVERecord?id=CVE-2026-46242)
  * [NVD record](https://nvd.nist.gov/vuln/detail/CVE-2026-46242)

  CVE-2026-46242 is listed for applicability documentation only. It does not affect the Linux 6.1.141 baseline used by this repository and no Bad Epoll patch is included.

### Copy Fail fixes

The Linux 6.1 backport consists of three commits applied in order:

* [`crypto: scatterwalk - Backport memcpy_sglist()`](https://github.com/gregkh/linux/commit/aa2a7743a44be64db6e0318fb4fec702e16a6098)
* [`crypto: algif_aead - use memcpy_sglist() instead of null skcipher`](https://github.com/gregkh/linux/commit/575dfd125c8e842078a16034c7f542983f087246)
* [`crypto: algif_aead - Revert to operating out-of-place`](https://github.com/gregkh/linux/commit/961cfa271a918ad4ae452420e7c303149002875b)

Repository patches:

* [`0001-scatterwalk-backport-memcpy-sglist.patch`](0001-scatterwalk-backport-memcpy-sglist.patch)
* [`0002-algif-aead-use-memcpy-sglist.patch`](0002-algif-aead-use-memcpy-sglist.patch)
* [`0003-copy-fail-operate-out-of-place.patch`](0003-copy-fail-operate-out-of-place.patch)

### Dirty Frag ESP fixes

Repository patches:

* [`0004-dirty-frag-esp-6.1.patch`](0004-dirty-frag-esp-6.1.patch)
* [`0005-dirty-frag-esp-6.1-fixup.patch`](0005-dirty-frag-esp-6.1-fixup.patch)

The original commit identifiers, authorship, stable metadata, and sign-offs are retained in the format-patch headers.

### Dirty Frag RxRPC fix

Upstream repair for the newer RxRPC architecture:

* [`rxrpc: Also unshare DATA/RESPONSE packets when paged frags are present`](https://github.com/torvalds/linux/commit/aa54b1d27fe0c2b78e664a34fd0fdf7cd1960d71)

Repository-specific backport for the older FriendlyElec RxRPC layout:

* [`0006-dirty-frag-rxrpc-shared-paged-frags-6.1-friendlyelec.patch`](0006-dirty-frag-rxrpc-shared-paged-frags-6.1-friendlyelec.patch)

The repository patch adapts the encrypted DATA-path repair to `rxrpc_input_packet()`. A RESPONSE-side change is unnecessary in this tree because its RxKAD response verifier copies encrypted data into private allocations before decryption rather than decrypting the skb fragments in place.

### GhostLock fix and RT-mutex follow-up

Repository patches, in required order:

* [`0007-ghostlock-6.1.patch`](0007-ghostlock-6.1.patch)
* [`0008-locking-rtmutex-Skip-remove_waiter-when-waiter-is-no.patch`](0008-locking-rtmutex-Skip-remove_waiter-when-waiter-is-no.patch)

`0007` fixes CVE-2026-43499 by making `remove_waiter()` operate on `waiter->task` instead of `current`. That correction exposes CVE-2026-53163 in paths where a waiter was never enqueued and `waiter->task` remains null.

`0008` completes the repair by skipping removal for a non-enqueued waiter and by limiting proxy-lock cleanup to negative error returns. It adapts upstream commit [`40a25d59e85b3c8709ac2424d44f65610467871e`](https://github.com/torvalds/linux/commit/40a25d59e85b3c8709ac2424d44f65610467871e) to FriendlyElec's older Linux 6.1.141 RT-mutex API.

The original metadata for `0007` and the adaptation rationale for `0008` are retained in their format-patch headers. The patches are a dependency pair: do not ship `0007` without `0008`, and do not apply `0008` to the unmodified vendor baseline without first applying `0007`.

### Bad Epoll applicability

* [Openwall advisory](https://www.openwall.com/lists/oss-security/2026/07/08/14)
* [CVE-2026-46242 NVD record](https://nvd.nist.gov/vuln/detail/CVE-2026-46242)

Bad Epoll is not patched by this repository because the Linux 6.1.141 baseline predates the affected Linux 6.1 stable range.

The exact source references, commit messages, authorship, review tags, stable tags, and sign-offs for every shipped format patch are retained in the patch headers.

## License

The original documentation in this repository is released under CC0-1.0.

The kernel patches are distributed under GPL-2.0-only and retain their original authorship, attribution, commit metadata, and sign-offs.

The ARM64 Kbuild helper binaries under tools/ are derived from Linux kernel sources and are distributed under GPL-2.0-only. Their source revision and provenance are documented alongside the binaries.