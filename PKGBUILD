# Maintainer: dh
# Fork of Proton Pass (ProtonMail/WebClients proton-pass@1.42.0) adding Linux
# fingerprint unlock through fprintd/polkit. Patches carry the fork delta.

pkgname=proton-pass-biometrics
pkgver=1.42.0
pkgrel=1
pkgdesc='Proton Pass desktop app with Linux fingerprint (fprintd) unlock — source-built fork'
arch=('x86_64')
url='https://proton.me/pass'
license=('GPL-3.0-or-later')
depends=('alsa-lib' 'at-spi2-core' 'cairo' 'dbus' 'expat' 'fprintd' 'glib2' 'gnome-keyring'
         'glibc' 'gtk3' 'libcups' 'libdrm' 'libx11' 'libxcb' 'libxcomposite' 'libxdamage'
         'libxext' 'libxfixes' 'libxkbcommon' 'libxrandr' 'libxshmfence' 'mesa' 'nspr' 'nss'
         'pango' 'polkit')
makedepends=('nodejs-lts-jod' 'rust')
provides=('proton-pass=1.42.0' 'protonpass')
conflicts=('proton-pass' 'protonpass' 'proton-pass-bin' 'proton-pass-fingerprint')
source=("${pkgname}-${pkgver}.tar.gz::https://github.com/ProtonMail/WebClients/archive/refs/tags/proton-pass@${pkgver}.tar.gz"
        'proton-pass.desktop'
        'ch.proton.pass.unlock.policy'
        '0001-rust-linux-biometrics-backend.patch'
        '0002-ts-enable-linux-biometrics.patch')
sha512sums=('eb7c66df88dcca995b2fccb8433beca0d2efa9e92d7a7b65b33a9ee41a33299d49b95233aa9467c90ee44d1fa5ad4844d0a6461a81f87e7b72f895667dc45bc2'
            '72f941b649c86228f5df950de50f33f346c4ed3ebb36809e14d2ad099e78752edcb1fb8a85bb8314b2c673928397177424eeb3f8f39fc1c6047c430ed93bf541'
            'd14942d312304aa5fc76337161684ba34663da2ad9a000991ae49c11af5b750d82b1f69dddb13b1afcd6c72fae79c79beef9f11ba4201cc4b46c47f5ee6ff4c0'
            'e160127e618311cb31b29698f186a75a0cc93628eb9513519853b69c6310085739eb55ed4bf6476b96d5fb17b4dbb689fc84c7422953d2d8b3d49738411c7153'
            'fa9f36aca13d80b5cdf73f1e7ff60bdbd5bdce213269e39745bf479c1eab521f36cabbb55807c210e20f8fd059be395a43eba4b3ffed9a8f7a164f8fabaaf8ec')

prepare() {
    cd "WebClients-proton-pass-${pkgver}"

    # Linux biometrics backend (fprintd presence check, polkit-gated
    # verification, Secret Service storage) and the TS/renderer gates.
    patch -p1 -i "${srcdir}/0001-rust-linux-biometrics-backend.patch"
    patch -p1 -i "${srcdir}/0002-ts-enable-linux-biometrics.patch"

    # Only the desktop app builds here; the tag tarball ships no
    # utilities/tests/vendor trees that the other globs reference.
    sed -i 's@- applications/\*@- applications/pass-desktop@' pnpm-workspace.yaml

    # Bypass native/build.js (rustup target downloads + npm): build the napi
    # module and host binary directly with cargo against the patched `shared`.
    sed -i 's@"build": "node build.js"@"build": "pnpm run build:napi \&\& pnpm run build:host"@' \
        applications/pass-desktop/native/package.json

    # Forge is run manually in build() so the cargo target dir can be moved
    # out of the tree while the asar is packed (keeps it out of the bundle).
    sed -i 's@ && electron-forge package@@' applications/pass-desktop/package.json
}

build() {
    cd "WebClients-proton-pass-${pkgver}/applications/pass-desktop"

    export COREPACK_ENABLE_DOWNLOAD_PROMPT=0
    # corepack pins pnpm to the tree's packageManager field (12.8.1), so no
    # system pnpm is needed. nodejs-lts-jod (Node 24) is required: Node ≥25
    # hangs silently inside electron-forge's zip extraction and the build
    # completes without ever producing out/.
    # --trust-lockfile: pnpm 12's supply-chain scan fetches publish metadata
    # for every lockfile entry, including nexus-only packages owned by other
    # apps (mail, drive, lumo) the desktop build never touches; the frozen
    # lockfile already pins every resolution.
    corepack pnpm install --frozen-lockfile --trust-lockfile \
        --filter proton-pass-desktop... --filter @proton-pass-desktop/native

    corepack pnpm run build:desktop

    # build.js stages the native messaging host into assets/ on darwin/win
    # only; the packaged app looks for it there on every platform
    # (getHostLocation → ../assets). Stage it from the cargo output.
    cp native/target/release/proton_pass_nm_host assets/

    {
        mv native/target ../rust-target
        NODE_ENV=production corepack pnpm exec electron-forge package
        mv ../rust-target native/target
    }
}

package() {
    cd "WebClients-proton-pass-${pkgver}/applications/pass-desktop"

    install -dm755 "${pkgdir}/opt/${pkgname}"
    # forge's output dir carries mode 0700; cp -a would preserve it and
    # block the user from the app entirely
    cp -a "out/Proton Pass-linux-x64/." "${pkgdir}/opt/${pkgname}/"
    chmod 0755 "${pkgdir}/opt/${pkgname}"
    install -dm755 "${pkgdir}/usr/bin"
    ln -s "/opt/${pkgname}/Proton Pass" "${pkgdir}/usr/bin/proton-pass"

    install -Dm644 "${srcdir}/proton-pass.desktop" \
        "${pkgdir}/usr/share/applications/proton-pass.desktop"
    install -Dm644 assets/logo.svg "${pkgdir}/usr/share/pixmaps/proton-pass.svg"
    install -Dm644 "${srcdir}/ch.proton.pass.unlock.policy" \
        "${pkgdir}/usr/share/polkit-1/actions/ch.proton.pass.unlock.policy"

    cd "${srcdir}/WebClients-proton-pass-${pkgver}"
    install -Dm644 LICENSE "${pkgdir}/usr/share/licenses/${pkgname}/LICENSE"
}
