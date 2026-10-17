# Maintainer: Tung Vu
# Builds Cockpit Tools from source (the hardened fork) instead of repackaging
# the upstream release .deb.
#
# Build from the fork on GitHub (default):
#   makepkg -si   (package name kept as cockpit-tools-bin so it upgrades in place)
# AUR also has a cockpit-tools-bin (upstream's unpatched binary). epoch=999 keeps this
# build newer than any AUR version, so paru -Syu never offers to replace it.
# Build from local commits instead (committed work only, not the working tree):
#   COCKPIT_TOOLS_REPO=/home/tung/Git/cockpit-tools makepkg -si
# prepare() refuses to build if a token-safety patch is missing; check() runs the
# hardening tests (skipped by --nocheck, so don't use it).
pkgname=cockpit-tools-bin
epoch=999
pkgver=1.3.70.r1545.gfa72cec8
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

# The fork's token-safety patches must survive every upstream merge. A clean merge only
# proves the patches applied, so check that each guard is still present, still set to
# the safe value, and still the only path in. Runs in prepare() to fail before the build.
_hardening_checks() {
  local fail=0

  _require() { # file, fixed string, description
    grep -qF -- "$2" "$1" || { echo "MISSING: $3 ($1)" >&2; fail=1; }
  }
  _forbid() { # file, fixed string, description
    ! grep -qF -- "$2" "$1" || { echo "FORBIDDEN: $3 ($1)" >&2; fail=1; }
  }

  _require src-tauri/src/lib.rs '.invoke_handler(ipc_guard::guard(tauri::generate_handler![' "IPC origin guard around invoke_handler"
  _require src-tauri/src/modules/announcement.rs 'const REMOTE_SPONSOR_ROUTE_REWRITE_ENABLED: bool = false;' "remote sponsor base-URL rewrite disabled"
  _require src-tauri/src/modules/announcement.rs 'if !REMOTE_SPONSOR_ROUTE_REWRITE_ENABLED' "sponsor rewrite gated on the flag"
  _require src-tauri/src/modules/external_import.rs 'payload.auto_import = false;' "deep links cannot auto-import"
  _require src-tauri/src/modules/external_import.rs 'payload.activate = false;' "deep links cannot auto-activate"
  _require src-tauri/src/lib.rs '?<redacted>' "deep-link query redacted in logs"
  _require src-tauri/src/models/codex_local_access.rs 'CodexLocalAccessScope::Localhost' "legacy local-access configs default to localhost"
  _require src-tauri/src/modules/web_report.rs 'token == "change-this-token" || token.len() < 16' "web report refuses default or short token"
  _require src-tauri/src/modules/websocket.rs 'accept_hdr_async(stream, check_ws_origin)' "WebSocket Origin check"
  _require src-tauri/src/modules/config.rs 'write_secret_string_atomic(&status_path' "server.json written 0600"
  _require src-tauri/src/modules/account.rs 'restrict_data_dir_permissions(&data_dir)' "data dir 0700"
  _require src-tauri/src/modules/kiro_oauth.rs 'normalize_idc_region(Some(region.as_str()))' "Kiro refresh region validated"
  _require src-tauri/src/modules/webdav_sync.rs 'ensure_secure_transport(&normalized_base_url)' "WebDAV https-only"
  _require src-tauri/src/modules/webdav_domain.rs 'url::Url::parse(trimmed)' "WebDAV allowlist host parsed with url crate"
  _require src-tauri/src/modules/diagnostics.rs 'BEARER_SCHEME_RE' "Sentry bearer redaction"
  _forbid src-tauri/src/modules/diagnostics.rs 'env::var("SENTRY_DSN")' "generic SENTRY_DSN enables reporting"
  _forbid src-tauri/src/modules/diagnostics.rs 'option_env!("SENTRY_DSN")' "generic SENTRY_DSN baked in at build time"
  _require src/pages/ApiKeyFunPage.tsx 'isApiKeyFunHostUrl(remoteBaseUrl)' "APIKEY.FUN base URL pinned to apikey.fan"

  # A guard only helps if nothing goes around it: exactly one command handler, and no
  # WebSocket accept without the Origin callback (test files excluded).
  local handlers unguarded_ws
  handlers="$(grep -rF --include='*.rs' '.invoke_handler(' src-tauri/src | wc -l)"
  if [[ "$handlers" != 1 ]]; then
    echo "FORBIDDEN: expected exactly one .invoke_handler( in src-tauri/src" >&2; fail=1
  fi
  unguarded_ws="$(grep -rlF --include='*.rs' 'accept_async(' src-tauri/src | grep -v '_tests' || true)"
  if [[ -n "$unguarded_ws" ]]; then
    echo "FORBIDDEN: WebSocket accept without Origin check in: $unguarded_ws" >&2; fail=1
  fi

  unset -f _require _forbid
  if (( fail )); then
    echo "error: token-safety hardening is incomplete; fix the fork before building" >&2
    return 1
  fi
  echo "All token-safety hardening checks passed."
}

prepare() {
  cd cockpit-tools
  _hardening_checks
  export npm_config_cache="$srcdir/npm-cache"
  # electron is only a devDependency used by an optional runtime helper; skip its binary.
  ELECTRON_SKIP_BINARY_DOWNLOAD=1 npm ci --no-audit --no-fund --allow-remote=all

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
  # Dependencies are reused across builds, but the app crate itself is recompiled from
  # scratch on every version. Compile it incrementally (and in more codegen units) so a
  # few new commits only rebuild what changed; dependencies stay fully optimized.
  npx tauri build --bundles deb --config "$srcdir/tauri.pkgbuild.conf.json" -- \
    --config 'profile.release.package.cockpit-tools.incremental=true' \
    --config 'profile.release.package.cockpit-tools.codegen-units=256'
}

# The hardening tests. cargo test exits 0 when a filter matches nothing, so also
# require each named test to have run and passed.
# Reuses the release dependencies from build() but compiles the app crate itself
# unoptimized and incrementally: optimizing it for a test run is most of the cost.
check() {
  cd cockpit-tools
  export CARGO_TARGET_DIR="$srcdir/cockpit-tools/target"
  local log="$srcdir/hardening-tests.log"
  cargo test --release \
    --config 'profile.release.package.cockpit-tools.opt-level=0' \
    --config 'profile.release.package.cockpit-tools.incremental=true' \
    -p cockpit-tools --lib -- \
    ipc_guard origin_tests webdav diagnostics external_import >"$log" 2>&1 \
    || { tail -n 60 "$log" >&2; return 1; }
  grep -E '^test result:' "$log"

  local t missing=0
  for t in \
    ipc_guard::tests::trusts_bundled_app_origins \
    ipc_guard::tests::rejects_remote_origins \
    modules::websocket::origin_tests::allows_local_clients_and_rejects_websites \
    modules::webdav_sync::tests::connection_requires_https_for_remote_hosts \
    modules::webdav_domain::tests::parses_hosts_and_allowlist \
    modules::diagnostics::tests::sanitize_text_redacts_short_bearer_and_json_secrets
  do
    grep -qF "test $t ... ok" "$log" || { echo "MISSING TEST: $t" >&2; missing=1; }
  done
  (( ! missing )) || { echo "error: a hardening test did not run; see $log" >&2; return 1; }
}

package() {
  cd cockpit-tools
  local _deb
  _deb="$(find target/release/bundle/deb -maxdepth 1 -name '*.deb' -print -quit)"
  [[ -n "$_deb" ]] || { echo "no .deb produced by tauri build" >&2; return 1; }
  bsdtar -xOf "$_deb" 'data.tar.*' | bsdtar -xf - -C "$pkgdir"
}
