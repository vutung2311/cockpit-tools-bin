# Maintainer: Gemini CLI Agent
pkgname=cockpit-tools-bin
pkgver=1.1.1
pkgrel=1
pkgdesc="Universal AI IDE account manager for Antigravity / Codex / GitHub Copilot / Windsurf / Kiro / Cursor / Gemini-cli / CodeBuddy"
arch=('x86_64')
url="https://github.com/jlcodes99/cockpit-tools"
license=('MIT')
depends=('gtk3' 'webkit2gtk-4.1' 'libnm' 'libayatana-appindicator')
makedepends=('curl' 'binutils')
provides=('cockpit-tools')
conflicts=('cockpit-tools')
source=()
sha256sums=()

pkgver() {
  curl -s "https://api.github.com/repos/jlcodes99/cockpit-tools/releases/latest" | grep -oP '"tag_name": "v\K[^"]+'
}

build() {
  curl -L -o "${srcdir}/Cockpit.Tools_${pkgver}_amd64.deb" "https://github.com/jlcodes99/cockpit-tools/releases/download/v${pkgver}/Cockpit.Tools_${pkgver}_amd64.deb"
  cd "$srcdir"
  ar x "Cockpit.Tools_${pkgver}_amd64.deb" data.tar.gz
}

package() {
  cd "$srcdir"
  tar -xzf data.tar.gz -C "${pkgdir}"
}
