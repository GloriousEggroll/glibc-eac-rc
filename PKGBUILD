# Maintainer: Giancarlo Razzolini <grazzolini@archlinux.org>
# Maintainer: Frederik Schwan <freswa at archlinux dot org>
# Contributor: Bartłomiej Piotrowski <bpiotrowski@archlinux.org>
# Contributor: Allan McRae <allan@archlinux.org>

# toolchain build order: linux-api-headers->glibc->binutils->gcc->glibc->binutils->gcc
# NOTE: valgrind requires rebuilt with each major glibc version

pkgbase=glibc
pkgname=(glibc lib32-glibc)
pkgver=2.37
_commit=7c32cb7dd88cf100b0b412163896e30aa2ee671a
pkgrel=3
arch=(x86_64)
url='https://www.gnu.org/software/libc'
license=(GPL LGPL)
makedepends=(git gd lib32-gcc-libs python)
options=(staticlibs !lto)
source=(git+https://sourceware.org/git/glibc.git#commit=${_commit}
        locale.gen.txt
        locale-gen
        lib32-glibc.conf
        sdt.h sdt-config.h
        reenable_DT_HASH.patch
		0001-Revert-elf-Clean-up-GLIBC_PRIVATE-exports-of-interna.patch
		0001-Revert-Install-shared-objects-under-their-ABI-names.patch
		0002-Revert-elf-Generalize-name-based-DSO-recognition-in-.patch
        0003-Revert-Makerules-Remove-lib-version-subdir-version.patch
        0004-Revert-nptl_db-Install-libthread_db-under-a-regular-.patch
)
validpgpkeys=(7273542B39962DF7B299931416792B4EA25340F8 # Carlos O'Donell
              BC7C7372637EC10C57D7AA6579C43DFBF1CF2187) # Siddhesh Poyarekar
b2sums=('SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP')

prepare() {
  mkdir -p glibc-build lib32-glibc-build

  [[ -d glibc-$pkgver ]] && ln -s glibc-$pkgver glibc
  cd glibc

  # Fix Rogue Company EAC
  patch -Np1 -i "${srcdir}"/0001-Revert-elf-Clean-up-GLIBC_PRIVATE-exports-of-interna.patch
  patch -Np1 -i "${srcdir}"/0001-Revert-Install-shared-objects-under-their-ABI-names.patch
  patch --verbose -Np1 -i "${srcdir}"/0002-Revert-elf-Generalize-name-based-DSO-recognition-in-.patch
  patch -Np1 -i "${srcdir}"/0003-Revert-Makerules-Remove-lib-version-subdir-version.patch
  patch -Np1 -i "${srcdir}"/0004-Revert-nptl_db-Install-libthread_db-under-a-regular-.patch

  # Re-enable `--hash-style=both` for building shared objects due to issues with EPIC's EAC
  # which relies on DT_HASH to be present in these libs.
  # reconsider 2023-01
  patch -Np1 -i "${srcdir}"/reenable_DT_HASH.patch
}

build() {
  local _configure_flags=(
      --prefix=/usr
      --with-headers=/usr/include
      --with-bugurl=https://bugs.archlinux.org/
      --enable-bind-now
      --enable-cet
      --enable-kernel=4.4
      --enable-multi-arch
      --enable-stack-protector=strong
      --enable-systemtap
      --disable-crypt
      --disable-profile
      --disable-werror
  )

  cd "${srcdir}"/glibc-build

  echo "slibdir=/usr/lib" >> configparms
  echo "rtlddir=/usr/lib" >> configparms
  echo "sbindir=/usr/bin" >> configparms
  echo "rootsbindir=/usr/bin" >> configparms

  # Credits @allanmcrae
  # https://github.com/allanmcrae/toolchain/blob/f18604d70c5933c31b51a320978711e4e6791cf1/glibc/PKGBUILD
  # remove fortify for building libraries
  CFLAGS=${CFLAGS/-Wp,-D_FORTIFY_SOURCE=2/}

  "${srcdir}"/glibc/configure \
      --libdir=/usr/lib \
      --libexecdir=/usr/lib \
      "${_configure_flags[@]}"

  # build libraries with fortify disabled
  echo "build-programs=no" >> configparms
  make -O

  # re-enable fortify for programs
  sed -i "/build-programs=/s#no#yes#" configparms
  echo "CFLAGS += -Wp,-D_FORTIFY_SOURCE=2" >> configparms
  make -O

  # build info pages manually for reproducibility
  make info

  cd "${srcdir}"/lib32-glibc-build
  export CC="gcc -m32 -mstackrealign"
  export CXX="g++ -m32 -mstackrealign"

  echo "slibdir=/usr/lib32" >> configparms
  echo "rtlddir=/usr/lib32" >> configparms
  echo "sbindir=/usr/bin" >> configparms
  echo "rootsbindir=/usr/bin" >> configparms

  "${srcdir}"/glibc/configure \
      --host=i686-pc-linux-gnu \
      --libdir=/usr/lib32 \
      --libexecdir=/usr/lib32 \
      "${_configure_flags[@]}"

  # build libraries with fortify disabled
  echo "build-programs=no" >> configparms
  make -O

  # re-enable fortify for programs
  sed -i "/build-programs=/s#no#yes#" configparms
  echo "CFLAGS += -Wp,-D_FORTIFY_SOURCE=2" >> configparms
  make -O

  # pregenerate C.UTF-8 locale until it is built into glibc
  # (https://sourceware.org/glibc/wiki/Proposals/C.UTF-8, FS#74864)-
  elf/ld.so --library-path "$PWD" locale/localedef -c -f ../glibc/localedata/charmaps/UTF-8 -i ../glibc/localedata/locales/C ../C.UTF-8/
}

# Credits for skip_test() and check() @allanmcrae
# https://github.com/allanmcrae/toolchain/blob/f18604d70c5933c31b51a320978711e4e6791cf1/glibc/PKGBUILD
skip_test() {
  test=${1}
  file=${2}
  sed -i "s/\b${test}\b//" "${srcdir}"/glibc/${file}
}

check() {
  cd glibc-build

  # adjust/remove buildflags that cause false-positive testsuite failures
  sed -i '/FORTIFY/d' configparms                                     # failure to build testsuite
  sed -i 's/-Werror=format-security/-Wformat-security/' config.make   # failure to build testsuite
  sed -i '/CFLAGS/s/-fno-plt//' config.make                           # 16 failures
  sed -i '/CFLAGS/s/-fexceptions//' config.make                       # 1 failure
  LDFLAGS=${LDFLAGS/,-z,now/}                                         # 10 failures

  # The following tests fail due to restrictions in the Arch build system
  # The correct fix is to add the following to the systemd-nspawn call:
  # --system-call-filter="@clock @memlock @pkey"
  skip_test test-errno-linux        sysdeps/unix/sysv/linux/Makefile
  skip_test tst-mlock2              sysdeps/unix/sysv/linux/Makefile
  skip_test tst-ntp_gettime         sysdeps/unix/sysv/linux/Makefile
  skip_test tst-ntp_gettimex        sysdeps/unix/sysv/linux/Makefile
  skip_test tst-pkey                sysdeps/unix/sysv/linux/Makefile
  skip_test tst-process_mrelease    sysdeps/unix/sysv/linux/Makefile
  skip_test tst-ttyname             sysdeps/unix/sysv/linux/Makefile
  skip_test tst-adjtime             time/Makefile
  skip_test tst-clock2              time/Makefile

  make -O check
}

package_glibc() {
  pkgdesc='GNU C Library'
  depends=('linux-api-headers>=4.10' tzdata filesystem)
  optdepends=('gd: for memusagestat'
              'perl: for mtrace')
  install=glibc.install
  backup=(etc/gai.conf
          etc/locale.gen
          etc/nscd.conf)

  make -C glibc-build install_root="${pkgdir}" install
  rm -f "${pkgdir}"/etc/ld.so.cache

  # Shipped in tzdata
  rm -f "${pkgdir}"/usr/bin/{tzselect,zdump,zic}

  cd glibc

  install -dm755 "${pkgdir}"/usr/lib/{locale,systemd/system,tmpfiles.d}
  install -m644 nscd/nscd.conf "${pkgdir}"/etc/nscd.conf
  install -m644 nscd/nscd.service "${pkgdir}"/usr/lib/systemd/system
  install -m644 nscd/nscd.tmpfiles "${pkgdir}"/usr/lib/tmpfiles.d/nscd.conf
  install -dm755 "${pkgdir}"/var/db/nscd

  install -m644 posix/gai.conf "${pkgdir}"/etc/gai.conf

  install -m755 "${srcdir}"/locale-gen "${pkgdir}"/usr/bin

  # Create /etc/locale.gen
  install -m644 "${srcdir}"/locale.gen.txt "${pkgdir}"/etc/locale.gen
  sed -e '1,3d' -e 's|/| |g' -e 's|\\| |g' -e 's|^|#|g' \
    "${srcdir}"/glibc/localedata/SUPPORTED >> "${pkgdir}"/etc/locale.gen

  # Add SUPPORTED file to pkg
  sed -e '1,3d' -e 's|/| |g' -e 's| \\||g' \
    "${srcdir}"/glibc/localedata/SUPPORTED > "${pkgdir}"/usr/share/i18n/SUPPORTED

  # install C.UTF-8 so that it is always available
  install -dm755 "${pkgdir}"/usr/lib/locale
  cp -r "${srcdir}"/C.UTF-8 -t "${pkgdir}"/usr/lib/locale
  sed -i '/#C\.UTF-8 /d' "${pkgdir}"/etc/locale.gen

  # Provide tracing probes to libstdc++ for exceptions, possibly for other
  # libraries too. Useful for gdb's catch command.
  install -Dm644 "${srcdir}"/sdt.h "${pkgdir}"/usr/include/sys/sdt.h
  install -Dm644 "${srcdir}"/sdt-config.h "${pkgdir}"/usr/include/sys/sdt-config.h
}

package_lib32-glibc() {
  pkgdesc='GNU C Library (32-bit)'
  depends=("glibc=$pkgver")
  options+=('!emptydirs')

  cd lib32-glibc-build

  make install_root="${pkgdir}" install
  rm -rf "${pkgdir}"/{etc,sbin,usr/{bin,sbin,share},var}

  # We need to keep 32 bit specific header files
  find "${pkgdir}"/usr/include -type f -not -name '*-32.h' -delete

  # Dynamic linker
  install -d "${pkgdir}"/usr/lib
  ln -s ../lib32/ld-linux.so.2 "${pkgdir}"/usr/lib/

  # Add lib32 paths to the default library search path
  install -Dm644 "${srcdir}"/lib32-glibc.conf "${pkgdir}"/etc/ld.so.conf.d/lib32-glibc.conf

  # Symlink /usr/lib32/locale to /usr/lib/locale
  ln -s ../lib/locale "${pkgdir}"/usr/lib32/locale
}
