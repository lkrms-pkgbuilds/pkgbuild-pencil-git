# Maintainer: Luke Arms <luke@arms.to>
# Contributor: Lukas Jirkovsky <l.jirkovsky@gmail.com>
# Contributor: Elia s. <elia.defcon7@gmail.com>
# Contributor: Caleb Maclennan <caleb@alerque.com>
# Contributor: Pavan Rikhi <pavan.rikhi@gmail.com>
# Contributor: BrLi <brli at chakralinux dot org>

_electronver=15
_nodever=14
pkgname=pencil-git
_pkgname=${pkgname%-git}
pkgver=3.1.0.r1553.47b2389
pkgrel=1
pkgdesc="An open-source GUI prototyping tool"
arch=('i686' 'x86_64')
url="http://pencil.evolus.vn/"
license=('GPL2')
depends=("electron$_electronver")
makedepends=('git' 'nvm' 'jq' 'libxcrypt-compat')
provides=("${_pkgname}=${pkgver}")
conflicts=("$_pkgname")
replaces=('evolus-pencil-git' 'evolus-pencil-svn' 'evolus-pencil-devel-git' 'evolus-pencil-git-dev-branch')
source=("git+https://github.com/evolus/pencil.git#branch=development"
    'pencil.desktop'
    'pencil.xml')
sha256sums=('SKIP'
    'c19b8aad227b712e363b1443efb7ab34616f321023d0e8cf311bfe168a2858c1'
    '7b9ea81e48f5c3967758b9d56788adfdfc89041f4a2d67ae261281763392c787')

pkgver() {
    cd "${srcdir}/${_pkgname}"
    printf '%s.r%s.%s\n' \
        "$(jq -r '.version' app/package.json)" \
        "$(git rev-list --count HEAD)" \
        "$(git rev-parse --short HEAD)"
}

_ensure_local_nvm() {
    if type nvm &>/dev/null; then
        nvm deactivate
        nvm unload
    fi
    unset npm_config_prefix
    export NVM_DIR=${srcdir}/.nvm
    . /usr/share/nvm/init-nvm.sh
}

prepare() {
    local temp
    cd "${srcdir}/${_pkgname}"
    temp=$(mktemp)
    jq '{
  "name":         null,
  "description":  (.build.linux.description // null),
  "author":       (.build.linux.vendor // null)
} + . |
  .build.linux |= del(.description, .synopsis, .vendor) |
  del(.build.win)' package.json >"$temp"
    cp "$temp" package.json
    rm "$temp"
    _ensure_local_nvm
    # ` || false` is a workaround until this upstream fix is released:
    # https://github.com/nvm-sh/nvm/pull/2698
    nvm ls "$_nodever" &>/dev/null ||
        nvm install "$_nodever" || false
}

build() {
    cd "${srcdir}/${_pkgname}"
    _ensure_local_nvm
    nvm use "$_nodever"
    npm install --global yarn
    yarn install --frozen-lockfile --no-default-rc \
        --cache-folder "${srcdir}/.yarn" --no-progress
    # electron-builder only generates /usr/share/* assets for target package
    # types 'apk', 'deb', 'freebsd', 'p5p', 'pacman', 'rpm' and 'sh', so build a
    # pacman package and unpack it
    local _appname _appver _outfile _unpackdir=${srcdir}/${_pkgname}-${pkgver}.unpacked
    _appname=$(jq -r '.name' app/package.json)
    _appver=$(jq -r '.version' app/package.json)
    _outfile=dist/${_appname}-${_appver}.pacman
    rm -rf "${_unpackdir}"
    mkdir -p "${_unpackdir}"

    local i686=ia32 x86_64=x64
    # Add -c.asar=false to suppress creation of an app archive
    ./node_modules/.bin/electron-builder build \
        --linux pacman \
        --"${!CARCH}" \
        -c.electronDist="/usr/lib/electron$_electronver" \
        -c.electronVersion="$(<"/usr/lib/electron$_electronver/version")"
    tar -C "${_unpackdir}" -xf "${_outfile}"
    diff --unified --color=always "${_unpackdir}/usr/share/applications/pencil.desktop" "${srcdir}/pencil.desktop" ||
        install -Dm644 "${srcdir}/pencil.desktop" "${_unpackdir}/usr/share/applications/pencil.desktop"
    install -Dm644 "${srcdir}/pencil.xml" "${_unpackdir}/usr/share/mime/packages/pencil.xml"
}

package() {
    local _appname _appdir _appfile=
    _appname=$(jq -r '.name' "${srcdir}/${_pkgname}/app/package.json")
    _appdir=opt/${_appname}/resources/app
    cd "${srcdir}/${_pkgname}-${pkgver}.unpacked"
    mv "usr" "${pkgdir}/"
    install -d "${pkgdir}/usr/lib"
    [[ -d $_appdir ]] || {
        _appdir=${_appdir%/*}
        _appfile=/app.asar
    }
    mv "$_appdir" "${pkgdir}/usr/lib/${_pkgname}"
    install -Dm644 "${srcdir}/${_pkgname}/LICENSE.md" "${pkgdir}/usr/share/licenses/${_pkgname}/LICENSE.md"
    install -Dm755 /dev/stdin "${pkgdir}/usr/bin/pencil" <<END
#!/bin/sh
exec /usr/lib/electron${_electronver}/electron /usr/lib/$_pkgname$_appfile "\$@"
END
}
