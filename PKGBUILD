# Maintainer: Walter Souto <357932+wsouto@users.noreply.github.com>
# Contributor: Maxime Gauduin <alucryd@archlinux.org>
# Contributor: Christian Heusel <gromit@archlinux.org>
# Contributor: George Sofianos <george@sofianos.dev>
# Contributor: Michele Balistreri <michele@bitgamma.com>
# Contributor: Caleb Maclennan <caleb@alerque.com>

pkgbase=lemonade
pkgname=(
  lemonade-server
  lemonade-desktop
)
pkgdesc='Local AI server and app that runs optimized LLMs on your own GPUs and NPUs'
pkgver=2026.43.3
pkgrel=2
_tag=v${pkgver}
arch=(x86_64)
url=https://github.com/lemonade-sdk/lemonade
license=(Apache-2.0)
options=(!debug)
makedepends=(
  cargo
  cargo-tauri
  cli11
  cmake
  cpp-httplib
  gdk-pixbuf2
  git
  gtk3
  libcap
  libdrm
  libwebsockets
  mbedtls
  ninja
  nlohmann-json
  nodejs-lts-krypton
  npm
  python
  systemd-libs
  unzip
  webkit2gtk-4.1
  zstd
)
source=(
  git+${url}.git#tag=${_tag}
  lemonade-sysusers.patch
  lemonade-web-app.patch
)
b2sums=('f808f739d07065924f5e5adcb4075d0c42b591fdc01c859bc11dc31e2fd3d46e4d0262c9b2f59542c91c935661882c2ec72d920bd6eda710bd78307a5bfc1cc8'
        '5b8174e0d88a825eb69e96f82140bae27a081665b472efaac0920829963f7790b9a1b9c9f5d787901ff4278eb31dc389e3e09dfad61fda490c7ceea355c51e14'
        '247f81584ba72065b4dd8fa9dae40c4bd4be86faad7f72bd69ca50d3caee595f19ea74079db00bd621eeedf3a135959453898815fa152599a0be466cdb5e9121')

prepare() {
  cd lemonade
  patch -Np1 --fuzz=0 -i ../lemonade-sysusers.patch
  patch -Np1 --fuzz=0 -i ../lemonade-web-app.patch

  printf '%s\n' "${pkgver}" > .version

  cd src/app
  export RUSTUP_TOOLCHAIN=stable
  npm ci --ignore-scripts
}

build() {
  local cmake_options=(
    -S lemonade
    -B build
    -G Ninja
    -D CMAKE_BUILD_TYPE=None
    -D CMAKE_INSTALL_PREFIX=/usr
    -D BUILD_TESTING=OFF
    -W no-dev
  )
  cmake "${cmake_options[@]}"
  cmake --build build

  cd lemonade/src/app
  export RUSTUP_TOOLCHAIN=stable
  cargo tauri build --no-bundle
}

package_lemonade-server() {
  depends=(
    curl
    glibc
    jq
    libcap
    libdrm
    libgcc
    libstdc++
    libwebsockets
    mbedtls
    systemd-libs
    unzip
    zstd
  )
  optdepends=(
    'fastflowlm: FLM support'
    'llama-cpp: Use system llama.cpp'
    'xdg-utils: Open the web app when no Chromium-based browser is available'
  )
  backup=(etc/default/lemond)

  DESTDIR="${pkgdir}" cmake --install build
}

package_lemonade-desktop() {
  depends=(
    cairo
    gdk-pixbuf2
    glib2
    glibc
    gtk3
    libgcc
    libsoup3
    webkit2gtk-4.1
  )

  cd lemonade
  install -Dm 755 src/app/src-tauri/target/release/lemonade-app -t "${pkgdir}/usr/bin/"
  install -Dm 644 data/lemonade-app.desktop -t "${pkgdir}/usr/share/applications/"
  install -Dm 644 src/app/assets/logo.svg "${pkgdir}/usr/share/pixmaps/lemonade-app.svg"
}
