# build_redsocks
1 script duy nhất, từ A→Z:

Tải Android NDK r26d
-Tạo toolchain ARM64
-Build OpenSSL
-Build libevent
-Build redsocks


#!/bin/bash
set -e

# ====== Cấu hình ======
NDK=~/android-ndk
TOOLCHAIN=~/arm64-toolchain
SYSROOT=~/arm64-sysroot

sudo apt install -y git build-essential unzip wget python3

# ====== Cài NDK ======
echo "[*] Download & setup Android NDK r26d..."
wget -q https://dl.google.com/android/repository/android-ndk-r26d-linux.zip
unzip -q android-ndk-r26d-linux.zip
mv android-ndk-r26d $NDK

# ====== Tạo toolchain ======
echo "[*] Create standalone toolchain..."
python3 $NDK/build/tools/make_standalone_toolchain.py \
  --arch arm64 --api 21 --install-dir $TOOLCHAIN

# ====== Xuất biến môi trường ======
export PATH=$TOOLCHAIN/bin:$PATH
export CC=aarch64-linux-android-gcc
export LD=aarch64-linux-android-ld
export AR=aarch64-linux-android-ar
export RANLIB=aarch64-linux-android-ranlib
export STRIP=aarch64-linux-android-strip
export ANDROID_NDK_HOME=$NDK

# ====== Build OpenSSL ======
echo "[*] Build OpenSSL..."
rm -rf openssl
git clone https://github.com/openssl/openssl.git
cd openssl
git checkout OpenSSL_1_1_1w
./Configure android-arm64 no-shared --prefix=$SYSROOT
make -j$(nproc)
make install_sw
cd ..

# ====== Build libevent ======
echo "[*] Build libevent..."
rm -rf libevent-2.1.12-stable*
wget -q https://github.com/libevent/libevent/releases/download/release-2.1.12-stable/libevent-2.1.12-stable.tar.gz
tar xvf libevent-2.1.12-stable.tar.gz
cd libevent-2.1.12-stable
./configure \
  --host=aarch64-linux-android \
  --prefix=$SYSROOT \
  --disable-shared \
  --enable-static \
  CC=$CC \
  CFLAGS="-I$SYSROOT/include" \
  LDFLAGS="-L$SYSROOT/lib -lssl -lcrypto" \
  OPENSSL_CFLAGS="-I$SYSROOT/include" \
  OPENSSL_LIBS="-L$SYSROOT/lib -lssl -lcrypto"
make -j$(nproc)
make install
cd ..

# ====== Build redsocks ======
echo "[*] Build redsocks..."
rm -rf redsocks
git clone https://github.com/darkk/redsocks.git
cd redsocks
make clean || true
make \
  CFLAGS="-I$SYSROOT/include" \
  LDFLAGS="-L$SYSROOT/lib -levent -lssl -lcrypto"

echo "[*] Done! Redsocks binary is here: $(pwd)/redsocks"
