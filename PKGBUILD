# First-party recipe for the starnix aur-mirror. Auto-tracks Zoom's latest
# release and GPG-verifies the OFFICIAL SIGNED rpm against Zoom's
# fingerprint-pinned signing key (the AUR recipe used the UNSIGNED tarball).
pkgname=zoom
pkgver=0
pkgrel=1
pkgdesc="Zoom video conferencing (first-party, GPG-verified official rpm)"
arch=('x86_64')
url="https://zoom.us/"
license=('LicenseRef-zoom')
depends=('fontconfig' 'glib2' 'libpulse' 'libsm' 'ttf-font' 'libx11' 'libxtst' 'libxcb'
  'libxcomposite' 'libxfixes' 'libxi' 'libxcursor' 'libxkbcommon-x11' 'libxrandr'
  'libxrender' 'libxshmfence' 'libxslt' 'mesa' 'nss' 'xcb-util-image'
  'xcb-util-keysyms' 'xcb-util-cursor' 'dbus' 'libdrm' 'gtk3' 'xcb-util-wm')
makedepends=('rpm-tools')
optdepends=('pulseaudio-alsa: audio via PulseAudio' 'ibus: remote control'
  'noto-fonts-emoji: emojis')
options=('!strip')
# Zoom's signing key fingerprint (pin). A recipe edit cannot swap the key
# without changing this reviewed value. From https://zoom.us/linux/download/pubkey.
_zoom_fpr="84C365D6CC9A4886CA926BCC4F2197399706AC24"
source=('zoom-pubkey.key')
sha256sums=('f4482a48ba574e6a0dcfb0fddae2da8f352abea63f21f6c355b8bca3169c39e8')

pkgver() {
  curl -fsSI "https://zoom.us/client/latest/zoom_x86_64.rpm" \
    | tr -d '\r' | sed -nE 's#^[Ll]ocation:.*/prod/([0-9.]+)/zoom_x86_64\.rpm.*#\1#p' \
    | head -n1 | tr -d '[:space:]'
}

build() {
  local ver rpmdb
  ver="$(pkgver)"
  [ -n "$ver" ] || { echo "could not resolve Zoom version" >&2; return 1; }

  # 1. The committed key must be the pinned fingerprint (show-only, no import).
  gpg --with-colons --import-options show-only --import "${srcdir}/zoom-pubkey.key" \
    | awk -F: '/^fpr:/{print $10}' | grep -qx "${_zoom_fpr}" \
    || { echo "committed key fingerprint != pinned ${_zoom_fpr}" >&2; return 1; }

  # 2. Import the pinned key into an isolated rpm keyring.
  rpmdb="${srcdir}/rpmdb"; mkdir -p "$rpmdb"
  rpmkeys --dbpath "$rpmdb" --import "${srcdir}/zoom-pubkey.key"

  # 3. Download the official signed rpm and verify its GPG signature (fail closed).
  # rpm prints e.g. "... signature, key fingerprint: <fpr>: OK". Require BOTH a
  # good signature line AND that it is signed by our PINNED fingerprint, so this
  # cannot pass on an unsigned rpm or one signed by any other key.
  curl -fsSL -o zoom.rpm "https://cdn.zoom.us/prod/${ver}/zoom_x86_64.rpm"
  local fpr_lc; fpr_lc="$(printf '%s' "${_zoom_fpr}" | tr 'A-F' 'a-f')"
  rpmkeys --dbpath "$rpmdb" -Kv zoom.rpm \
    | grep -qiE "signature,[[:space:]]*key (id|fingerprint):[[:space:]]*${fpr_lc}:[[:space:]]*OK" \
    || { echo "rpm GPG signature verification FAILED (not signed by pinned ${_zoom_fpr})" >&2; return 1; }

  # 4. Extract the payload (bsdtar reads rpm directly).
  mkdir -p extracted && bsdtar -C extracted -xf zoom.rpm
}

package() {
  cp -dpr --no-preserve=ownership "${srcdir}/extracted/opt" "${srcdir}/extracted/usr" "${pkgdir}/"
}
