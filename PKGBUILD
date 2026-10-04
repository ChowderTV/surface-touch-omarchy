# Surface IPTS touchscreen + pen on a stock (non linux-surface) Arch kernel:
#  - ipts driver as DKMS, extracted from linux-surface's kernel patch
#  - iptsd userspace daemon
#  - a boot-time service that binds mei_me to the touch chip and loads ipts
pkgname=surface-touch-omarchy
pkgver=1.1
pkgrel=1
pkgdesc='Surface IPTS touchscreen and pen (ipts DKMS + iptsd) without the linux-surface kernel'
arch=('x86_64')
url='https://github.com/ChowderTV/surface-touch-omarchy'
license=('GPL-2.0-or-later')
depends=('dkms' 'gcc-libs' 'glibc' 'systemd-libs')
makedepends=('meson' 'git')
provides=('iptsd')
conflicts=('iptsd' 'iptsd-git')
backup=('etc/iptsd.conf')
install=surface-touch.install

_iptsd_ver=3.1.0
# linux-surface commit that last touched patches/6.19/0005-ipts.patch
_ls_commit=73e26d02077e2b8abaebe7ca334ada7b40152044

source=("iptsd-$_iptsd_ver.tar.gz::https://github.com/linux-surface/iptsd/archive/refs/tags/v$_iptsd_ver.tar.gz"
        "0005-ipts-$_ls_commit.patch::https://raw.githubusercontent.com/linux-surface/linux-surface/$_ls_commit/patches/6.19/0005-ipts.patch"
        'dkms.conf'
        'surface-touch-setup'
        'surface-touch.service'
        '50-pen-sensitivity.conf.example'
        'surface-pen-tune')
sha256sums=('af0aab95387a5107dfa8ac1ef90a29289f6b9d06e1a59121c9e5e732b27a266c'
            'ccc2f38598d9a49e19aa88d01c797654468b86f07beda4d355067a9d9083880e'
            'b861d48416e2003fbeac2c42eb03e73fe9fbdc6b9e84619d8ebc8d6a6050a2a4'
            '2a74a0ce62f2c5f46417cb152acab898d104f35022fdee56a631844ef34d1168'
            '91d9a6979293f28609d33eefbb4e489df674ff8834bc190c139f804c71627839'
            '2e3c29683ac33b2f761d00d84a6fdf925449ddda48b9115fb1d8f4e6f4cd73b7'
            'ef4cfa374371a5c388a795e130188d742cef9405c6928f135032923b1cbe8b46')

prepare() {
	# Pull only the standalone driver out of the kernel patch
	rm -rf ipts-src && mkdir ipts-src
	git apply --directory=ipts-src --include='ipts-src/drivers/hid/ipts/*' \
		"0005-ipts-$_ls_commit.patch"
}

build() {
	# Dependencies are built from iptsd's pinned meson wraps (the versions it
	# was tested with) rather than whatever the system happens to ship.
	rm -rf build
	meson setup "iptsd-$_iptsd_ver" build \
		--prefix=/usr --libexecdir=lib --sbindir=bin \
		--buildtype=release -Db_lto=true -Db_pie=true \
		-Dwerror=false \
		-Dservice_manager=systemd \
		-Ddebug_tools=calibrate,dump \
		--wrap-mode=default \
		--force-fallback-for=cli11,eigen,fmt,inih,microsoft-gsl,spdlog
	meson compile -C build
}

package() {
	meson install -C build --skip-subprojects --destdir "$pkgdir"

	local src="$pkgdir/usr/src/ipts-6.19"
	install -d "$src"
	install -m644 ipts-src/drivers/hid/ipts/*.[ch] ipts-src/drivers/hid/ipts/Kconfig dkms.conf "$src/"
	sed 's/^obj-$(CONFIG_HID_IPTS)/obj-m/' ipts-src/drivers/hid/ipts/Makefile > "$src/Makefile"

	install -Dm755 surface-touch-setup "$pkgdir/usr/lib/surface-touch/setup"
	install -Dm644 surface-touch.service "$pkgdir/usr/lib/systemd/system/surface-touch.service"
	install -Dm755 surface-pen-tune "$pkgdir/usr/bin/surface-pen-tune"
	install -Dm644 50-pen-sensitivity.conf.example -t "$pkgdir/usr/share/doc/$pkgname/"
}
