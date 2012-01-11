# Original kernel maintainers:
#	Tobias Powalowski <tpowa@archlinux.org>
#	Thomas Baechler <thomas@archlinux.org>
# Contributors:
#	henning mueller <henning@orgizm.net>

pkgname=linux-pax
_kernelname=${pkgname#linux}
_basekernel=3.2
_paxver=test3
pkgver=${_basekernel}
pkgrel=4
arch=(i686 x86_64)
url="http://www.kernel.org/"
license=(GPL2)
options=(!strip)

pkgdesc="The Linux Kernel and modules with PaX patches"
groups=('base')
depends=('paxctl' 'coreutils' 'linux-firmware' 'module-init-tools>=3.16' 'mkinitcpio>=0.7')
optdepends=('crda: to set the correct wireless channels of your country')
provides=('kernel26-pax')
conflicts=('kernel26-pax')
replaces=('kernel26-pax')
backup=("etc/mkinitcpio.d/${pkgname}.preset")
install=$pkgname.install

_menuconfig=0
[ ! -z $MENUCONFIG ] && _menuconfig=1

source=(
	ftp://ftp.kernel.org/pub/linux/kernel/v3.0/linux-$pkgver.tar.bz2
	http://grsecurity.net/test/pax-linux-$pkgver-$_paxver.patch
	change-default-console-loglevel.patch
	i915-fix-ghost-tv-output.patch
	config
	config.x86_64
	$pkgname.install
	$pkgname.preset
	$pkgname-fix-permissions
)
md5sums=(
	7ceb61f87c097fc17509844b71268935
	b9abf5eabb8f502d3b82910798e80bfa
	9d3c56a4b999c8bfbd4018089a62f662
	342071f852564e1ad03b79271a90b1a5
	4079a2ae3e2ee308e6db188f7bc04959
	f1397e95d7f6e9f4180b8b16bbc49d52
	c792439f6f6c0dcedd1c71af63673d30
	5d29c2995ffa1ac918dd6b269ec09ecc
	d50670fac5d343d50e19ceb6e86dcd35
)

build() {
  cd $srcdir/linux-$pkgver

  # Some chips detect a ghost TV output
  # mailing list discussion: http://lists.freedesktop.org/archives/intel-gfx/2011-April/010371.html
  # Arch Linux bug report: FS#19234
  #
  # It is unclear why this patch wasn't merged upstream, it was accepted,
  # then dropped because the reasoning was unclear. However, it is clearly
  # needed.
  patch -Np1 -i "${srcdir}/i915-fix-ghost-tv-output.patch"

  # set DEFAULT_CONSOLE_LOGLEVEL to 4 (same value as the 'quiet' kernel param)
  # remove this when a Kconfig knob is made available by upstream
  # (relevant patch sent upstream: https://lkml.org/lkml/2011/7/26/227)
  patch -Np1 -i "${srcdir}/change-default-console-loglevel.patch"

  # Add PaX patches
  patch -Np1 -i $srcdir/pax-linux-$pkgver-$_paxver.patch

  if [ "${CARCH}" = "x86_64" ]; then
    cat "${srcdir}/config.x86_64" > ./.config
  else
    cat "${srcdir}/config" > ./.config
  fi

  if [ "${_kernelname}" != "" ]; then
    sed -i "s|CONFIG_LOCALVERSION=.*|CONFIG_LOCALVERSION=\"${_kernelname}\"|g" ./.config
  fi

  # remove the sublevel from Makefile
  # this ensures our kernel version is always 3.X-ARCH
  # this way, minor kernel updates will not break external modules
  # we need to change this soon, see FS#16702
  sed -ri 's|^(SUBLEVEL =).*|\1|' Makefile

  # get kernel version
  [ "$_menuconfig" = "0" ] && {
    make prepare
  }

  # load configuration
  # Configure the kernel. Replace the line below with one of your choice.
  [ "$_menuconfig" = "1" ] && {
    make menuconfig # CLI menu for configuration
    #make nconfig # new CLI menu for configuration
    #make xconfig # X-based configuration
    #make oldconfig # using old config from previous kernel version
    # ... or manually edit .config
  }

  ####################
  # stop here
  # this is useful to configure the kernel
  [ "$_menuconfig" = "1" ] && {
    msg "Stopping build"
    return 1
  }
  ####################

  yes "" | make config

  # build!
  make ${MAKEFLAGS} bzImage modules
}

package() {
  cd $srcdir/linux-$pkgver

  KARCH=x86

  # get kernel version
  _kernver="$(make kernelrelease)"

  mkdir -p "${pkgdir}"/{lib/modules,lib/firmware,boot}
  make INSTALL_MOD_PATH="${pkgdir}" modules_install
  cp arch/$KARCH/boot/bzImage "${pkgdir}/boot/vmlinuz-${pkgname}"

  # add vmlinux
  install -D -m644 vmlinux "${pkgdir}/usr/src/linux-${_kernver}/vmlinux"

  # install fallback mkinitcpio.conf file and preset file for kernel
  install -D -m644 "${srcdir}/${pkgname}.preset" "${pkgdir}/etc/mkinitcpio.d/${pkgname}.preset"

  # set correct depmod command for install
  sed \
    -e  "s/KERNEL_NAME=.*/KERNEL_NAME=${_kernelname}/g" \
    -e  "s/KERNEL_VERSION=.*/KERNEL_VERSION=${_kernver}/g" \
    -i "${startdir}/${pkgname}.install"
  sed \
    -e "s|ALL_kver=.*|ALL_kver=\"/boot/vmlinuz-${pkgname}\"|g" \
    -e "s|default_image=.*|default_image=\"/boot/initramfs-${pkgname}.img\"|g" \
    -e "s|fallback_image=.*|fallback_image=\"/boot/initramfs-${pkgname}-fallback.img\"|g" \
    -i "${pkgdir}/etc/mkinitcpio.d/${pkgname}.preset"

  # remove build and source links
  rm -f "${pkgdir}"/lib/modules/${_kernver}/{source,build}
  # remove the firmware
  rm -rf "${pkgdir}/lib/firmware"
  # gzip -9 all modules to safe 100MB of space
  find "${pkgdir}" -name '*.ko' -exec gzip -9 {} \;
  # make room for external modules
  ln -s "../extramodules-${_basekernel}${_kernelname:--ARCH}" "${pkgdir}/lib/modules/${_kernver}/extramodules"
  # add real version for building modules and running depmod from post_install/upgrade
  mkdir -p "${pkgdir}/lib/modules/extramodules-${_basekernel}${_kernelname:--ARCH}"
  echo "${_kernver}" > "${pkgdir}/lib/modules/extramodules-${_basekernel}${_kernelname:--ARCH}/version"

  install -D -m755 $srcdir/$pkgname-fix-permissions \
	  $pkgdir/usr/bin/$pkgname-fix-permissions
}
