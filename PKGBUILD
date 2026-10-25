# Maintainer: Joshua Warren <816217+joshuaswarren@users.noreply.github.com>
#
# Meta/helper package for the Omarchy Apple bridge stack:
# Apple Reminders over CalDAV (Radicale + vdirsyncer + todoman).
#
# Regenerate the .SRCINFO after every edit of this file:
#   makepkg --printsrcinfo > .SRCINFO

pkgname=omarchy-apple-bridge
pkgver=0.2.0
pkgrel=1
pkgdesc='One-command Apple Reminders setup for Omarchy: Radicale CalDAV plus vdirsyncer, todoman, and a preconfigured Planify account'
arch=(any)
url='https://github.com/joshuaswarren/omarchy-apple-bridge'
license=(MIT)
depends=(todoman vdirsyncer)
optdepends=(
    'flatpak: required for the Planify GUI and the BlueBubbles client'
    'python: seeds the Planify CalDAV account (already pulled in by todoman)'
)
source=(
    apple-bridge-setup
    apple-bridge-planify-seed
    vdirsyncer.conf.template
    todoman.conf.template
    README.md
)
sha256sums=(
'5de4db322d759ae86ce1d5196e2908459b98943218a22acfbf2eaa3b9399b7b8'
    '7998ea2d0ada2f18c69e335b6402afaca70de47b942525354e9fece322a3d5f9'
    'e1cb74c98af6ddcd47f161281e98f6e28c83149dfc1f80c8c8e46f86c1cc61ff'
    'd0006208a3829ea08c73da18c5f16bf5b548af42937faedbabe8aa1b14d30d15'
    '1f0123434905906c6423ed2bd99c796dac6eed39741483d7c054a25f77533fed'
)

package() {
    install -Dm755 "$srcdir/apple-bridge-setup" \
        "$pkgdir/usr/bin/apple-bridge-setup"
    install -Dm755 "$srcdir/apple-bridge-planify-seed" \
        "$pkgdir/usr/bin/apple-bridge-planify-seed"
    install -Dm644 "$srcdir/vdirsyncer.conf.template" \
        "$pkgdir/usr/share/omarchy-apple-bridge/vdirsyncer.conf.template"
    install -Dm644 "$srcdir/todoman.conf.template" \
        "$pkgdir/usr/share/omarchy-apple-bridge/todoman.conf.template"
    install -Dm644 "$srcdir/README.md" \
        "$pkgdir/usr/share/doc/omarchy-apple-bridge/README.md"
}
