# Maintainer: BrLi <brli [at] chakralinux [dot] org>

pkgname=vscodium
pkgver=1.74.2.22355
pkgrel=3
pkgdesc="Free/Libre Open Source Software Binaries of VSCode"
arch=('x86_64' 'aarch64' 'armv7h')
url='https://vscodium.com'
license=('MIT')

# Important: Remember to check https://github.com/microsoft/vscode/blob/master/.yarnrc (choose correct tag) for target electron version
_electron=electron19

depends=($_electron 'libsecret' 'libx11' 'libxkbfile' 'ripgrep')
optdepends=('bash-completion: Bash completions'
            'zsh-completions: ZSH completitons'
            'x11-ssh-askpass: SSH authentication')
makedepends=('git' 'gulp' 'npm' 'python' 'yarn' 'nodejs-lts-gallium' # dependencies for vscode
             'jq') # explictly required for generating customized product.json
provides=('code')
conflicts=('code')
options=('!strip')
source=("git+https://github.com/VSCodium/vscodium.git#tag=${pkgver}"
        "git+https://github.com/microsoft/vscode.git#tag=${pkgver:0:6}"
        "git+https://github.com/flathub/com.vscodium.codium.git"
        'product_json.diff'
        'codium.js')
sha256sums=('SKIP'
            'SKIP'
            'SKIP'
            '3f147cc835dd53ad3697a0234b9e4c74187220d6a73479bd0685011194457555'
            '44c252c08fe9c76dc0351c88bc76c3bcf5e32f5c2286cc82cd2a52cca0217fbc')

###############################################################################

# Even though we don't officially support other archs, let's allow the
# user to use this PKGBUILD to compile the package for their architecture.
case "$CARCH" in
  x86_64)
    _vscode_arch=x64
    ;;
  aarch64)
    _vscode_arch=arm64
    ;;
  armv7h)
    _vscode_arch=arm
    ;;
  *)
    # Needed for mksrcinfo
    _vscode_arch=DUMMY
    ;;
esac

prepare() {
    cd 'vscodium'

    # ./get_repo.sh
    # Fetched in source=()
    rm -rf 'vscode'
    mv "${srcdir}/vscode" 'vscode'

    # Configuration from Arch community/code
    cd 'vscode'

    # Change electron binary name to the target electron
    sed -i "s|#!/usr/bin/electron|#!/usr/bin/$_electron|" "${srcdir}/codium.js"

    # This patch no longer contains proprietary modifications.
    # See https://github.com/Microsoft/vscode/issues/31168 for details.
    patch -p0 -i "${srcdir}/product_json.diff"

    # Set the commit and build date
    local _commit=$(git rev-parse HEAD)
    local _datestamp=$(date -u -Is | sed 's/\+00:00/Z/')
    sed -i "s/@COMMIT@/$_commit/
            s/@DATE@/$_datestamp/
            s,\.vscode-oss,\.config/VSCodium," product.json

    # Build native modules for system electron
    local _target=$(</usr/lib/$_electron/version)
    sed -i "s/^target .*/target \"${_target//v/}\"/" .yarnrc

    # Patch appdata and desktop file
    sed -i 's|/usr/share/@@NAME@@/@@NAME@@|@@NAME@@|g
            s|@@NAME_SHORT@@|VSCodium|g
            s|@@NAME_LONG@@|VSCodium|g
            s|@@NAME@@|codium|g
            s|@@ICON@@|vscodium|g
            s|@@EXEC@@|/usr/bin/codium|g
            s|@@LICENSE@@|MIT|g
            s|@@URLPROTOCOL@@|vscodium|g' resources/linux/code{.desktop,-url-handler.desktop}

    sed -i 's|MimeType=.*|MimeType=x-scheme-handler/vscodium;|' resources/linux/code-url-handler.desktop


    # Add completitions for vscodium
    cp resources/completions/bash/code resources/completions/bash/codium
    cp resources/completions/zsh/_code resources/completions/zsh/_codium

    # Patch completitions with correct names
    sed -i 's|@@APPNAME@@|codium|g' resources/completions/{bash/codium,zsh/_codium}
    sed -i 's|@@APPNAME@@|code|g' resources/completions/{bash/code,zsh/_code}

    # Fix bin path
    sed -i "s|return path.join(path.dirname(execPath), 'bin', \`\${product.applicationName}\`);|return '/usr/bin/codium';|g
            s|return path.join(appRoot, 'scripts', 'code-cli.sh');|return '/usr/bin/codium';|g" \
            src/vs/platform/environment/node/environmentService.ts

    # Prepare VScode Script Does Following Jobs
    # ../update_settings.sh
    # Apply patches, update product.json
    # Undo Telemetry
    # Sed around to be VSCodium
    cd ..
    # Patch VSCodium patches for Arch
    sed -i "s:\.\./:../:" prepare_vscode.sh update_settings.sh undo_telemetry.sh
    sed -i 's:\grep -rl --exclude-dir=\.git -E "${TELEMETRY_URLS}" \.:rg --no-ignore --iglob "!*.map" -l "${TELEMETRY_URLS}" .:' undo_telemetry.sh
    # We don't build in prepare()
    sed 's,CHILD_.*,echo done,g' -i prepare_vscode.sh
    # fix undo_telemetry.sh
    sed 's,./node_modules/@vscode/ripgrep/bin/rg,/usr/bin/rg,g' -i undo_telemetry.sh
    local OS_NAME=linux
    local RELEASE_VERSION="${pkgver}"
    local BUILD_SOURCEVERSION=$( echo "${RELEASE_VERSION/-*/}" | sha1sum | cut -d' ' -f1 )
    ./prepare_vscode.sh
    cat 'vscode/package.json' | jq --arg 'path' 'version' --arg 'value' "$( echo "${RELEASE_VERSION}" | sed -n -E "s/^(.*)\.([0-9]+)(-insider)?$/\1/p" )" 'setpath([$path]; $value)' > 'vscode/package.json'
    cat 'vscode/package.json' | jq --arg 'path' 'release' --arg 'value' "$( echo "${RELEASE_VERSION}" | sed -n -E "s/^(.*)\.([0-9]+)(-insider)?$/\2/p" )" 'setpath([$path]; $value)' > 'vscode/package.json'
}

build() {
    cd 'vscodium/vscode'
    yarn install --arch=$_vscode_arch
    gulp --openssl-legacy-provider --max_old_space_size=9999 vscode-linux-$_vscode_arch-min
}

package() {
    cd 'vscodium'

    # Install resource files
    install -dm 755 "$pkgdir/usr/lib/$pkgname"
    cp -r --no-preserve=ownership --preserve=mode VSCode-linux-$_vscode_arch/resources/app/* "$pkgdir"/usr/lib/$pkgname/

    # Replace statically included binary with system copy
    ln -sf /usr/bin/rg "$pkgdir/usr/lib/$pkgname/node_modules.asar.unpacked/@vscode/ripgrep/bin/rg"

    # Install binary
    install -Dm 755 /dev/stdin "$pkgdir"/usr/bin/codium<<END
#!/bin/bash

set -euo pipefail

flags_file="\${XDG_CONFIG_HOME:-\$HOME/.config}/codium-flags.conf"

declare -a flags

if [[ -f "\${flags_file}" ]]; then
    mapfile -t < "\${flags_file}"
fi

for line in "\${MAPFILE[@]}"; do
    if [[ ! "\${line}" =~ ^[[:space:]]*#.* ]]; then
        flags+=("\${line}")
    fi
done

exec $_electron /usr/lib/vscodium/out/cli.js /usr/lib/vscodium/codium.js "\${flags[@]}" "\$@"
END
    install -Dm 755 "$srcdir"/codium.js "$pkgdir"/usr/lib/$pkgname/codium.js

    # Code compatible symlinks
    ln -sf /usr/bin/codium "$pkgdir"/usr/bin/vscode
    ln -sf /usr/bin/codium "$pkgdir"/usr/bin/code
    # ln -sf /usr/bin/codium "$pkgdir"/usr/bin/vscodium

    # Install appdata and desktop file
    install -Dm 644 $srcdir/com.vscodium.codium/com.vscodium.codium.metainfo.xml "$pkgdir"/usr/share/metainfo/com.vscodium.codium.metainfo.xml
    install -Dm 644 vscode/resources/linux/code.desktop "$pkgdir"/usr/share/applications/codium.desktop
    install -Dm 644 vscode/resources/linux/code-url-handler.desktop "$pkgdir"/usr/share/applications/codium-url-handler.desktop
    install -Dm 644 VSCode-linux-$_vscode_arch/resources/app/resources/linux/code.png "$pkgdir"/usr/share/pixmaps/vscodium.png

    # Install bash and zsh completions
    install -Dm 644 vscode/resources/completions/bash/code "$pkgdir"/usr/share/bash-completion/completions/code
    install -Dm 644 vscode/resources/completions/bash/codium "$pkgdir"/usr/share/bash-completion/completions/codium
    install -Dm 644 vscode/resources/completions/zsh/_code "$pkgdir"/usr/share/zsh/site-functions/_code
    install -Dm 644 vscode/resources/completions/zsh/_codium "$pkgdir"/usr/share/zsh/site-functions/_codium

    # Install license files
    install -Dm 644 VSCode-linux-$_vscode_arch/resources/app/LICENSE.txt "$pkgdir"/usr/share/licenses/$pkgname/LICENSE
    install -Dm 644 VSCode-linux-$_vscode_arch/resources/app/ThirdPartyNotices.txt "$pkgdir"/usr/share/licenses/$pkgname/ThirdPartyNotices.txt
}
