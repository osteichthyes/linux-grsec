# Original kernel maintainers:
#   Tobias Powalowski <tpowa@archlinux.org>
#   Thomas Baechler <thomas@archlinux.org>
#
# Contributors:
#   henning mueller <henning@orgizm.net>
#   Thomas Dwyer http://tomd.tel
#
# Find this package in the AUR:
#   https://aur.archlinux.org/packages/linux-grsec
#
# Please report bugs and feature requests on GitHub:
#   https://github.com/nning/linux-grsec
#

pkgname=linux-grsec
true && pkgname=(linux-grsec linux-grsec-headers)
_kernelname=${pkgname#linux}
_basekernel=4.8
_kernelpatchver=7
_grsecver=3.1
_timestamp=201611102210
if [ "$_kernelpatchver" == 0 ]; then
	_kernelver=$_basekernel
	sourcea=(
		https://www.kernel.org/pub/linux/kernel/v4.x/linux-$_basekernel.tar.xz
		)
else
	_kernelver=$_basekernel.$_kernelpatchver
		sourcea="https://www.kernel.org/pub/linux/kernel/v4.x/linux-$_basekernel.tar.xz https://www.kernel.org/pub/linux/kernel/v4.x/patch-$_kernelver.xz"
		
fi
_grsecpatchver=$_grsecver-$_kernelver
pkgver=$_kernelver.$_timestamp
pkgrel=2
arch=(x86_64)
url='https://github.com/nning/linux-grsec'
license=(GPL2)
options=(!strip)
makedepends=(bc)
conflicts=(linux-grsec-lts)

[ "$1" = '-v' ] && echo $pkgver $_timestamp

# The MENUCONFIG environment variable controls the invokation of the kernel
# configuration (see line 91). 0 does not run menuconfig (default), 1 runs
# menuconfig and exits, 2 runs menuconfig and builds the kernel.
_menuconfig=0
[ ! -z $MENUCONFIG ] && _menuconfig=$MENUCONFIG

_grsec_patch="grsecurity-$_grsecpatchver-$_timestamp.patch"

source=(
  $sourcea
  https://grsecurity.net/test/$_grsec_patch{,.sig}
  config.x86_64
  $pkgname.install
  $pkgname.preset
  sysctl.conf
  module-blacklist.conf
)

validpgpkeys=('DE9452CE46F42094907F108B44D1C0F82525FE49')

build() {
  cd "$srcdir/linux-$_basekernel"

  # add upstream patch
  [ -f "$srcdir/patch-$_kernelver" ] &&
    patch -p1 -i "$srcdir/patch-$_kernelver"

  # set DEFAULT_CONSOLE_LOGLEVEL to 4 (same value as the 'quiet' kernel param)
  # remove this when a Kconfig knob is made available by upstream
  # (relevant patch sent upstream: https://lkml.org/lkml/2011/7/26/227)
  sed -i 's/DEFAULT_CONSOLE_LOGLEVEL 7/DEFAULT_CONSOLE_LOGLEVEL 4/' kernel/printk/printk.c

  # Add grsecurity patches
  patch -Np1 -i "$srcdir/$_grsec_patch"
  rm localversion-grsec

  cat "${srcdir}/config.${CARCH}" > ./.config

  if [ "${_kernelname}" != "" ]; then
    sed -i "s|CONFIG_LOCALVERSION=.*|CONFIG_LOCALVERSION=\"${_kernelname}\"|g" ./.config
  fi

  # set extraversion to pkgrel
  sed -ri "s|^(EXTRAVERSION =).*|\1 -${pkgrel}|" Makefile

  # don't run depmod on 'make install'. We'll do this ourselves in packaging
  sed -i '2iexit 0' scripts/depmod.sh

  # get kernel version
  [ "$_menuconfig" = "0" ] && {
    make prepare
  }

  # Configure the kernel. Replace the line below with one of your choice.
  [ "$_menuconfig" -gt "0" ] && {
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

package_linux-grsec() {
  pkgdesc="The Linux Kernel and modules with grsecurity/PaX patches"
  groups=('base')
  depends=('gradm' 'paxctld' 'coreutils' 'linux-firmware' 'kmod' 'mkinitcpio>=0.7')
  optdepends=('crda: to set the correct wireless channels of your country')
  provides=('kernel26-grsec')
  conflicts=('kernel26-grsec')
  replaces=('kernel26-grsec')
  backup=(
    etc/mkinitcpio.d/$pkgname.preset
    etc/modprobe.d/dma.conf
    etc/sysctl.d/05-grsecurity.conf
  )
  install=${pkgname}.install

  cd "$srcdir/linux-$_basekernel"

  KARCH=x86

  # get kernel version
  _kernver="$(make kernelrelease)"

  mkdir -p "${pkgdir}"/{lib/modules,lib/firmware,boot}
  make INSTALL_MOD_PATH="${pkgdir}" modules_install
  cp arch/$KARCH/boot/bzImage "${pkgdir}/boot/vmlinuz-${pkgname}"

  # add vmlinux and gcc plugins TURN OFF GCC PLUGINS ELSE BUILD FAILS
  install -Dm644 vmlinux "$pkgdir/usr/src/linux-$_kernver/vmlinux"
  mkdir -p "$pkgdir/usr/src/linux-$_kernver/tools/gcc"
#  install -m644 tools/gcc/*.so "$pkgdir/usr/src/linux-$_kernver/tools/gcc/"

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
  # gzip -9 all modules to save 100MB of space
  find "${pkgdir}" -name '*.ko' -exec gzip -9 {} \;
  # make room for external modules
  ln -s "../extramodules-${_basekernel}${_kernelname:--ARCH}" "${pkgdir}/lib/modules/${_kernver}/extramodules"
  # add real version for building modules and running depmod from post_install/upgrade
  mkdir -p "${pkgdir}/lib/modules/extramodules-${_basekernel}${_kernelname:--ARCH}"
  echo "${_kernver}" > "${pkgdir}/lib/modules/extramodules-${_basekernel}${_kernelname:--ARCH}/version"

  # Now we call depmod...
  depmod -b "$pkgdir" -F System.map "$_kernver"

  # move module tree /lib -> /usr/lib
  mv "$pkgdir/lib" "$pkgdir/usr"

  # Copy sysctl configuration
  install -Dm600 "$srcdir/sysctl.conf" "$pkgdir/etc/sysctl.d/05-grsecurity.conf"

  # Copy kernel module blacklist
  install -Dm600 "$srcdir/module-blacklist.conf" "$pkgdir/etc/modprobe.d/dma.conf"

  # Copy current grsec patch
  install -Dm644 "$srcdir/$_grsec_patch" \
    "$pkgdir/usr/share/doc/$pkgname/$_grsec_patch"
}

package_linux-grsec-headers() {
  true && pkgdesc="Header files and scripts for building modules for linux kernel with grsecurity/PaX patches"
  provides=('kernel26-grsec-headers')
  conflicts=('kernel26-grsec-headers')
  replaces=('kernel26-grsec-headers')
  
  cd "$srcdir/linux-$_basekernel"

  install -dm755 "${pkgdir}/usr/lib/modules/${_kernver}"

  install -D -m644 Makefile \
    "${pkgdir}/usr/lib/modules/${_kernver}/build/Makefile"
  install -D -m644 kernel/Makefile \
    "${pkgdir}/usr/lib/modules/${_kernver}/build/kernel/Makefile"
  install -D -m644 .config \
    "${pkgdir}/usr/lib/modules/${_kernver}/build/.config"

  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/include"

  for i in acpi asm-generic config crypto drm generated keys linux math-emu \
    media net pcmcia scsi sound trace uapi video xen; do
    cp -a include/${i} "${pkgdir}/usr/lib/modules/${_kernver}/build/include/"
  done

  # copy arch includes for external modules
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/arch/x86"
  cp -a arch/x86/include "${pkgdir}/usr/lib/modules/${_kernver}/build/arch/x86/"

  # copy files necessary for later builds, like nvidia and vmware
  cp Module.symvers "${pkgdir}/usr/lib/modules/${_kernver}/build"
  cp -a scripts "${pkgdir}/usr/lib/modules/${_kernver}/build"

  # fix permissions on scripts dir
  chmod og-w -R "${pkgdir}/usr/lib/modules/${_kernver}/build/scripts"
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/.tmp_versions"

  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/arch/${KARCH}/kernel"

  cp arch/${KARCH}/Makefile "${pkgdir}/usr/lib/modules/${_kernver}/build/arch/${KARCH}/"

  if [ "${CARCH}" = "i686" ]; then
    cp arch/${KARCH}/Makefile_32.cpu "${pkgdir}/usr/lib/modules/${_kernver}/build/arch/${KARCH}/"
  fi

  cp arch/${KARCH}/kernel/asm-offsets.s "${pkgdir}/usr/lib/modules/${_kernver}/build/arch/${KARCH}/kernel/"

  # add docbook makefile
  install -D -m644 Documentation/DocBook/Makefile \
    "${pkgdir}/usr/lib/modules/${_kernver}/build/Documentation/DocBook/Makefile"

  # add dm headers
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/drivers/md"
  cp drivers/md/*.h "${pkgdir}/usr/lib/modules/${_kernver}/build/drivers/md"

  # add inotify.h
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/include/linux"
  cp include/linux/inotify.h "${pkgdir}/usr/lib/modules/${_kernver}/build/include/linux/"

  # add wireless headers
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/net/mac80211/"
  cp net/mac80211/*.h "${pkgdir}/usr/lib/modules/${_kernver}/build/net/mac80211/"

  # add dvb headers for external modules
  # in reference to:
  # http://bugs.archlinux.org/task/9912
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/drivers/media/dvb-core"
  cp drivers/media/dvb-core/*.h "${pkgdir}/usr/lib/modules/${_kernver}/build/drivers/media/dvb-core/"
  # and...
  # http://bugs.archlinux.org/task/11194
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/include/config/dvb/"
  cp include/config/dvb/*.h "${pkgdir}/usr/lib/modules/${_kernver}/build/include/config/dvb/"

  # add dvb headers for http://mcentral.de/hg/~mrec/em28xx-new
  # in reference to:
  # http://bugs.archlinux.org/task/13146
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/drivers/media/dvb-frontends/"
  cp drivers/media/dvb-frontends/lgdt330x.h "${pkgdir}/usr/lib/modules/${_kernver}/build/drivers/media/dvb-frontends/"
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/drivers/media/i2c/"
  cp drivers/media/i2c/msp3400-driver.h "${pkgdir}/usr/lib/modules/${_kernver}/build/drivers/media/i2c/"

  # add dvb headers
  # in reference to:
  # http://bugs.archlinux.org/task/20402
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/drivers/media/usb/dvb-usb"
  cp drivers/media/usb/dvb-usb/*.h "${pkgdir}/usr/lib/modules/${_kernver}/build/drivers/media/usb/dvb-usb/"
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/drivers/media/dvb-frontends"
  cp drivers/media/dvb-frontends/*.h "${pkgdir}/usr/lib/modules/${_kernver}/build/drivers/media/dvb-frontends/"
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/drivers/media/tuners"
  cp drivers/media/tuners/*.h "${pkgdir}/usr/lib/modules/${_kernver}/build/drivers/media/tuners/"

  # add xfs and shmem for aufs building
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/fs/xfs/libxfs"
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/mm"
  cp fs/xfs/libxfs/xfs_sb.h "${pkgdir}/usr/lib/modules/${_kernver}/build/fs/xfs/libxfs/xfs_sb.h"

  # copy in Kconfig files
  for i in $(find . -name "Kconfig*"); do
    mkdir -p "${pkgdir}"/usr/lib/modules/${_kernver}/build/`echo ${i} | sed 's|/Kconfig.*||'`
    cp ${i} "${pkgdir}/usr/lib/modules/${_kernver}/build/${i}"
  done

  chown -R root.root "${pkgdir}/usr/lib/modules/${_kernver}/build"
  find "${pkgdir}/usr/lib/modules/${_kernver}/build" -type d -exec chmod 755 {} \;

  # strip scripts directory
  find "${pkgdir}/usr/lib/modules/${_kernver}/build/scripts" -type f -perm -u+w 2>/dev/null | while read binary ; do
    case "$(file -bi "${binary}")" in
      *application/x-sharedlib*) # Libraries (.so)
        /usr/bin/strip ${STRIP_SHARED} "${binary}";;
      *application/x-archive*) # Libraries (.a)
        /usr/bin/strip ${STRIP_STATIC} "${binary}";;
      *application/x-executable*) # Binaries
        /usr/bin/strip ${STRIP_BINARIES} "${binary}";;
    esac
  done

  # remove unneeded architectures
  rm -rf "${pkgdir}"/usr/lib/modules/${_kernver}/build/arch/{alpha,arc,arm,arm26,arm64,avr32,blackfin,c6x,cris,frv,h8300,hexagon,ia64,m32r,m68k,m68knommu,metag,mips,microblaze,mn10300,openrisc,parisc,powerpc,ppc,s390,score,sh,sh64,sparc,sparc64,tile,unicore32,um,v850,xtensa}
}

sha256sums=('3e9150065f193d3d94bcf46a1fe9f033c7ef7122ab71d75a7fb5a2f0c9a7e11a'
            '94213e7557d192d1054e352aec18e93275ed5a84abe190d43fd43847d1d86efe'
            'd00203ad69480242aeddf1eab70ac8396c14624a140d3f7972d212ff37dcead8'
            'SKIP'
            '4587eef81c2002849195c82057414d2fdd67961136b599aec7c1356ced47e566'
            '38ad5fbb2b85407ec602854fd77a88c1755b6231ec602243bd61b7c4bf655b13'
            'ca7e718375b3790888756cc0a64a7500cd57dddb9bf7e10a0df22c860d91f74d'
            '10479bae8a966f0aedbea5ddf24bb6e7da120c705099e9098990224e9f16eb03'
            '520fb5c0b117e2abf6378c7677ab905be89293350661f895dd7b7a06d3622cb3')

