# Maintainer: BrLi <brli [at] chakralinux [dot] org>

pkgname=vscodium
pkgver=1.56.2
pkgrel=1
pkgdesc="Free/Libre Open Source Software Binaries of VSCode"
arch=('x86_64' 'aarch64' 'armv7h')
url='https://vscodium.com'
license=('MIT')

# Important: Remember to check https://github.com/microsoft/vscode/blob/master/.yarnrc (choose correct tag) for target electron version
_electron=electron

depends=($_electron 'libsecret' 'libx11' 'libxkbfile' 'ripgrep')
optdepends=('bash-completion: Bash completions'
            'zsh-completions: ZSH completitons'
            'x11-ssh-askpass: SSH authentication')
makedepends=('git' 'gulp' 'npm' 'python2' 'yarn' 'nodejs-lts-erbium' 'jq')
source=("git+https://github.com/VSCodium/vscodium.git#tag=${pkgver}"
        "git+https://github.com/microsoft/vscode.git#tag=${pkgver}"
        'code.js')
sha256sums=('SKIP'
            'SKIP'
            '44c252c08fe9c76dc0351c88bc76c3bcf5e32f5c2286cc82cd2a52cca0217fbc')
provides=('code')
conflicts=('code')

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
    mv '../vscode' 'vscode'

    # Configuration from Arch community/code
    cd 'vscode'
    # Set the commit and build date
    local _commit=$(git rev-parse HEAD)
    local _datestamp=$(date -u -Is | sed 's/\+00:00/Z/')
    sed -e "s/@COMMIT@/$_commit/" -e "s/@DATE@/$_datestamp/" -i product.json

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
            s|@@URLPROTOCOL@@|vscodium|g
            s|inode/directory;||' resources/linux/code{.desktop,-url-handler.desktop}

    sed -i 's|MimeType=.*|MimeType=x-scheme-handler/codium;|' resources/linux/code-url-handler.desktop


    # Add completitions for vscodium
    cp resources/completions/bash/code resources/completions/bash/codium
    cp resources/completions/zsh/_code resources/completions/zsh/_codium

    # Patch completitions with correct names
    sed -i 's|@@APPNAME@@|codium|g' resources/completions/{bash/code,zsh/_codium}
    sed -i 's|@@APPNAME@@|codium|g' resources/completions/{bash/code,zsh/_codium}

    # Fix bin path
    sed -i "s|return path.join(path.dirname(execPath), 'bin', \`\${product.applicationName}\`);|return '/usr/bin/codium';|g
            s|return path.join(appRoot, 'scripts', 'code-cli.sh');|return '/usr/bin/codium';|g" \
            src/vs/platform/environment/node/environmentService.ts

    # Prepare VScode
    # ../update_settings.sh
    # Apply patches, update product.json
    # Undo Telemetry
    # Sed around to be VSCodium
    cd ..
    # We don't build in prepare()
    sed 's/yarn/#yarn/g' -i prepare_vscode.sh
    ./prepare_vscode.sh
}

build() {
    # https://github.com/mapbox/node-sqlite3/issues/1044
    mkdir -p path
    ln -sf /usr/bin/python2 path/python
    export PATH="$PWD/path:$PATH"

    cd 'vscodium/vscode'

    # Command in community/code
    CHILD_CONCURRENCY=1 yarn install --arch=$_vscode_arch --frozen-lockfile

    # The default memory limit may be too low for current versions of node
    # to successfully build vscode. Change it if this number still doesn't
    # work for your system.
    mem_limit="--max_old_space_size=6144"

    if ! /usr/bin/node $mem_limit /usr/bin/gulp vscode-linux-$_vscode_arch-min
    then
        echo
        echo "*** NOTE: If the build failed due to running out of file handles (EMFILE),"
        echo "*** you will need to raise your max open file limit."
        echo "*** You can check this for more information on how to increase this limit:"
        echo "***    https://ro-che.info/articles/2017-03-26-increase-open-files-limit"
        exit 1
    fi
}

package() {
    cd 'vscodium'

    # Install resource files
    install -dm 755 "$pkgdir"/usr/lib/$pkgname
    cp -r --no-preserve=ownership --preserve=mode VSCode-linux-$_vscode_arch/resources/app/* "$pkgdir"/usr/lib/$pkgname/

    # Replace statically included binary with system copy
    ln -sf /usr/bin/rg "$pkgdir/usr/lib/$pkgname/node_modules.asar.unpacked/vscode-ripgrep/bin/rg"

    # Install binary
    install -Dm 755 /dev/stdin "$pkgdir"/usr/bin/codium<<END
#!/bin/bash

ELECTRON_RUN_AS_NODE=1 exec $_electron /usr/lib/vscodium/out/cli.js /usr/lib/vscodium/code.js "\$@"
END
    install -Dm 755 "$srcdir"/code.js "$pkgdir"/usr/lib/$pkgname/code.js

    # Code compatible symlinks
    ln -sf /usr/bin/codium "$pkgdir"/usr/bin/vscode
    ln -sf /usr/bin/codium "$pkgdir"/usr/bin/code
    ln -sf /usr/bin/codium "$pkgdir"/usr/bin/vscode
    ln -sf /usr/bin/codium "$pkgdir"/usr/bin/vscodium

    # Install appdata and desktop file
    install -Dm 644 vscode/resources/linux/code.appdata.xml "$pkgdir"/usr/share/metainfo/codium.appdata.xml
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
