# Maintainer: Tung Vu
# Builds Cockpit Tools from source (the hardened fork) instead of repackaging
# the upstream release .deb.
#
# Build from the fork on GitHub (default):
#   makepkg -si   (package name kept as cockpit-tools-bin so it upgrades in place)
# Build from local commits instead (committed work only, not the working tree):
#   COCKPIT_TOOLS_REPO=/home/tung/Git/cockpit-tools makepkg -si
pkgname=cockpit-tools-bin
pkgver=1.3.61.r1495.g13ca8d39
pkgrel=1
pkgdesc="Universal AI IDE account manager (built from source, token-safety hardened fork)"
arch=('x86_64')
url="https://github.com/vutung2311/cockpit-tools"
license=('CC-BY-NC-SA-4.0')
depends=('gtk3' 'webkit2gtk-4.1' 'libnm' 'libayatana-appindicator' 'libsecret')
makedepends=('git' 'nodejs' 'npm' 'cargo' 'go')
provides=('cockpit-tools')
conflicts=('cockpit-tools')
options=('!lto' '!debug')

_repo="${COCKPIT_TOOLS_REPO:-https://github.com/vutung2311/cockpit-tools.git}"
_branch="${COCKPIT_TOOLS_BRANCH:-main}"
source=("cockpit-tools::git+${_repo}#branch=${_branch}")
sha256sums=('SKIP')

pkgver() {
  cd cockpit-tools
  printf "%s.r%s.g%s" \
    "$(node -p "require('./package.json').version")" \
    "$(git rev-list --count HEAD)" \
    "$(git rev-parse --short=8 HEAD)"
}

prepare() {
  cd cockpit-tools
  export npm_config_cache="$srcdir/npm-cache"
  # electron is only a devDependency used by an optional runtime helper; skip its binary.
  ELECTRON_SKIP_BINARY_DOWNLOAD=1 npm ci --no-audit --no-fund

  # Upstream's updater points at jlcodes99 releases and would offer to replace this
  # build with the unpatched upstream binary; disable it for this package.
  cat > "$srcdir/tauri.pkgbuild.conf.json" <<'EOF'
{
  "bundle": { "createUpdaterArtifacts": false },
  "plugins": { "updater": { "endpoints": [] } }
}
EOF
}

build() {
  cd cockpit-tools
  export npm_config_cache="$srcdir/npm-cache"
  export CARGO_TARGET_DIR="$srcdir/cockpit-tools/target"
  npx tauri build --bundles deb --config "$srcdir/tauri.pkgbuild.conf.json"
}

package() {
  cd cockpit-tools
  local _deb
  _deb="$(find target/release/bundle/deb -maxdepth 1 -name '*.deb' -print -quit)"
  [[ -n "$_deb" ]] || { echo "no .deb produced by tauri build" >&2; return 1; }
  bsdtar -xOf "$_deb" 'data.tar.*' | bsdtar -xf - -C "$pkgdir"
}
