# riscv-cove

## CoVE KVM RISCV64 on QEMU with device filtering

### Dependency

```bash
sudo apt install binutils-riscv64-linux-gnu libfdt-dev bazel-bootstrap
```

### 1. Build QEMU

```bash
cd ./qemu
mkdir build && cd ./build
export CFLAGS="-Wno-error=dangling-pointer"
../configure --target-list="riscv64-softmmu"
make -j$(nproc)
cd ..
```

The above commands will create `./qemu/build/riscv64-softmmu/qemu-system-riscv64` which will be our QEMU system emulator.

### 2. Build Common Host & Guest Linux Kernel Image

```bash
set RUSTUP_DIST_SERVER=https://rsproxy.cn
set RUSTUP_UPDATE_ROOT=https://rsporxy.cn/rustup
set RUSTUP_DIST_SERVER=https://mirrors.ustc.edu.cn/rust-static
set RUSTUP_UPDATE_ROOT=https://mirrors.ustc.edu.cn/rust-static/rustup rustup update
unset http_proxy
unset https_proxy
rustup default stable
cd linux
export ARCH=riscv
export CROSS_COMPILE=riscv64-linux-gnu-
mkdir build
make O=./build defconfig
scripts/config --file build/.config --enable CONFIG_RISCV_COVE_HOST
scripts/config --file build/.config --enable CONFIG_RISCV_COVE_GUEST
scripts/config --file build/.config --enable CONFIG_RISCV_SBI_V01
scripts/config --file build/.config --enable CONFIG_KVM
scripts/config --file build/.config --enable CONFIG_HVC_RISCV_SBI
scripts/config --file build/.config --enable CONFIG_FW_LOADER

# build dev auth firmware.
cd ./tools/device-authorize/
python3 build-firmware.py --input riscv-cove.json --output riscv-cove.bin
cd -

scripts/config --file build/.config --set-val CONFIG_EXTRA_FIRMWARE \"riscv-cove.bin\"
scripts/config --file build/.config --set-val CONFIG_EXTRA_FIRMWARE_DIR \""$PWD"/tools/device-authorize/\"

make O=./build -j$(nproc)
cd ..
```

The above commands will create `linux/build/arch/riscv/boot/Image` which will be our Guest and Host kernel.

### 3. Add libfdt library to CROSS_COMPILE SYSROOT directory 

```bash
cd dtc
export ARCH=riscv
export CROSS_COMPILE=riscv64-linux-gnu-
export CC="${CROSS_COMPILE}gcc -mabi=lp64d -march=rv64gc"
export TRIPLET=$($CC -dumpmachine)
export SYSROOT=$($CC -print-sysroot)
sudo -E make NO_PYTHON=1 NO_YAML=1 DESTDIR=$SYSROOT PREFIX=/usr LIBDIR=/usr/lib/$TRIPLET install-lib install-includes
cd ..
```

The above commands will install cross-compiled libfdt library at `$SYSROOT/usr/lib/riscv64-linux-gnu` directory of cross-compile toolchain.

### 4. Build KVMTOOL

```bash
cd kvmtool
export ARCH=riscv
export CROSS_COMPILE=riscv64-linux-gnu-
make lkvm-static -j$(nproc)
${CROSS_COMPILE}strip lkvm-static
cd ..
```

The above commands will create `kvmtool/lkvm-static` which will be our user-space tool for KVM RISC-V.

### 5. Build Host RootFS containing KVM RISC-V module, KVMTOOL and Guest Linux

```bash
cd buildroot
make qemu_riscv64_virt_min_defconfig
make -j$(nproc)
cd ..
# Copy guest image and lkvm-tool to rootfs.
mkdir -p buildroot/output/images/mount
sudo mount buildroot/output/images/rootfs.ext2 buildroot/output/images/mount
sudo cp linux/build/arch/riscv/boot/Image \
       kvmtool/lkvm-static \
       buildroot/output/images/mount/root/
sudo umount buildroot/output/images/mount
```

The above commands will create `output/images/rootfs.ext2` which will be our host rootfs containing kvmtool and Guest Kernel.

### 6. Build Salus and run KVM on QEMU

```
# Set up environment variables required by salus.
export QEMU=$PWD/qemu/
export LINUX=$PWD/linux/build/
export BUILDROOT=$PWD/buildroot/

cd salus
make run_buildroot
```

Run Guest Linux using KVMTOOL. You will need to ssh into host with root:root at localhost:7722. HVC console doesn't support input for now.

```bash
ssh -p 7722  root@localhost
root@localhost's password: root

./lkvm-static run -c2 --console virtio --cove-vm -p "earlycon=sbi console=hvc1 dev.authorize.all=false device.authorize.firmware=riscv-cove.bin" -k ./Image
```
