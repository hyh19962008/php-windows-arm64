# php-windows-arm64
A guide to build php 8 on Windows arm64

Tested on:
Windows 11(24H2 26100) + Visual Studio 2022 ARM64 + Windows SDK 20348

## 1. Dependencies
Prebuilt libraries are here: [releases/tag/Dependencies](https://github.com/hyh19962008/php-windows-arm64/releases/tag/Dependencies)

### 1.0 vcpkg
Many dependecies can be installed via vcpkg.
- vcpkg new --name php --version 8.4
- replace vcpkg.json with the one in this repository
- vcpkg install

### 1.1 QDBM 1.8.78
> [winlibs/qdbm](https://github.com/winlibs/qdbm)
- nmake /f VCMakefile all 
- misc\win32check.bat
- **All tests Passed**

### 1.2 libsodium 1.0.20
> [libsodium.org](https://download.libsodium.org/libsodium/releases/libsodium-1.0.20-stable-msvc.zip)
- The official binary
- vcpkg also provides it. (Failed to download on my machine)

### 1.3 wineditline 2.206
> [winlibs/wineditline](https://github.com/winlibs/wineditline)
- Build with cmake(check the file INSTALL)

### 1.4 oniguruma 6.9.10
> [kkos/oniguruma](https://github.com/kkos/oniguruma)
- mkdir build && cd build
- cmake -DCMAKE_SYSTEM_VERSION=10.0.20348.0 -DCMAKE_INSTALL_PREFIX=D:\onig -G "Visual Studio 17" -A arm64 -DBUILD_SHARED_LIBS=OFF ..
- msbuild /m /p:Configuration=Release oniguruma.sln
- **All tests Passed**

### 1.5 net-snmp 5.9.4
> [net-snmp Files](https://sourceforge.net/projects/net-snmp/files/net-snmp/5.9.4/)
- Apply this commit to 5.9.4 src: [win32: add build support for Windows on ARM](https://github.com/net-snmp/net-snmp/commit/23cfd15bf05aad242af958831b2b472e24e0ce04)
- Install StrawberryPerl (x64 version is OK)
- Require openssl
- cd win32 && perl Configure --config=release --linktype=dynamic --prefix=d:/netsnmp --with-ssl --with-ipv6
- nmake
- nmake install
- nmake install_devel

### 1.6 libiconv 1.17.1
> [winlibs/libiconv](https://github.com/winlibs/libiconv)
- Open .sln file and Build with Visual Studio
- copy source\include\iconv.h %deps%\include
- **All tests Passed**

### 1.7 gettext/libintl 0.18.3-9
> [winlibs/gettext](https://github.com/winlibs/gettext)
- add "#include <stdint.h>" in source\gettext-runtime\intl\printf-parse.c:49
- remove "#define uintmax_t unsigned __int64" in source\gettext-runtime\config.h
- add "#define HAVE_STDINT_H 1" in source\gettext-runtime\config.h
- Open .sln file and Build with Visual Studio
- copy source\gettext-runtime\intl\libgnuintl.h %deps%\include\libintl.h

### 1.8 libxml2 2.11.9-2
> [winlibs/libxml2](https://github.com/winlibs/libxml2)
- Require libiconv
- cd win32
- cscript configure.js lib="<path to iconv lib dir>" include="<path to iconv header dir>" vcmanifest=yes
- nmake /f Makefile.msvc
- **All tests Passed**

### 1.9 libxslt 1.1.39
> [winlibs/libxslt](https://github.com/winlibs/libxslt)
- Require iconv, libxml2(dynamic lib)
- cd win32
- cscript configure.js lib="path to iconv lib dir;path to libxml2 lib dir;" include="path to iconv header dir;path to libxml2 header dir" modules=yes crypto=no vcmanifest=yes
- nmake /f Makefile.msvc
- **All tests Passed**

### 1.10 glib 2.84.2
> [download.gnome.org](https://download.gnome.org/sources/glib/2.84/glib-2.84.2.tar.xz)
- vcpkg also provides it. (Failed to install on my machine)
- Require zlib, libintl
- Build with meson + ninja
- copy %BUILDIR%/glib/glibconfig.h %deps%/include/glib-2.0
- Running tests requires elevation
- Produce libffi
- **Some tests Failed**

### 1.11 libenchant2 2.2.8
> [winlibs/enchant](https://github.com/winlibs/enchant)
- Require glib
- Open .sln file and Build with Visual Studio
- Project - Properties - linker - advance - DYNAMICBASE:YES
- convert the encoding of src\lib.c to UTF8-BOM if error is reported
- copy src/enchant.h %deps%/include

### 1.12 krb5 1.22-beta1
> [krb5/krb5](https://github.com/krb5/krb5)
- Make sure you have these in PATH: sed, awk, cat, cp, mv, rm, hhc.exe (HTML Help Workshop)
- set PATH=%PATH%;"%WindowsSdkVerBinPath%"\x86
- set KRB_INSTALL_DIR=d:\krb5
- set NO_LEASH=1
- cd src
- nmake -f Makefile.in prep-windows
- nmake NDEBUG=1
#### for 1.21
- Apply changes in this PR [Get arm64-windows builds working](https://github.com/krb5/krb5/pull/1302/files)
- The last step is: nmake NDEBUG=1 CPU=ARM64

### 1.13 cyrus-sasl 2.1.27-3
> [winlibs/cyrus-sasl](https://github.com/winlibs/cyrus-sasl)
- Require krb5, openssl, lmdb
- initialize tmp in common\plugin_common.c:641
- Open .sln file and Build with Visual Studio

### 1.14 openldap 2.4.47-1
> [winlibs/openldap](https://github.com/winlibs/openldap)
- Require openssl, libsasl
- Open .sln file and Build with Visual Studio
- copy include/*.h %deps%/include/openldap
- copy include/ac/*.h %deps%/include/openldap/ac

### 1.15 Firebird 6.0.0
> [FirebirdSQL/snapshots](https://github.com/FirebirdSQL/snapshots/releases/tag/snapshot-master)
- This is nightly Pre-Alpha Build

## 2. php-src

### 2.1 libraries
Due to the way php links libraries, some libraries can only use static or static+dynamic mixed version. Otherwise, you will get trouble in the linking pharse.
- libxml2: use libxml2_a_dll.lib
- libiconv: use libiconv_a.lib
- libxslt: use libxslt_a.lib that dynamic link to libxml
### 2.2 copy headers to %deps%\include
Some header files need to be copy into %deps%\include from vcpkg_installed\arm64-windows\include. Otherwise, php configure script won't find them.
- libwebp, libavif, libxpm
- libssh2, nghttp2
- openssl/thread.h
- argon2.h
### 2.3 rename libs
php build scripts check libraries for specific names(e.g. libxml2_a_dll.lib), you will have to make symlink or copy the library in vcpkg_installed\arm64-windows\lib and then rename them. Check the warning messages generate by the configure script.

### 2.4 netsnmp  
Add these to %deps%\include\net-snmp\net-snmp-config.h
```c  
#define HAVE_SSIZE_T 1
#define HAVE_READDIR 1
#define HAVE_STRTOK_R 1
#undef HAVE_SYSLOG_H
```
use get_logh_head() instead of extern logh_head in php-8.4.6\ext\snmp\snmp.c.
```c
// extern netsnmp_log_handler *logh_head;
#define shutdown_snmp_logging() \
	{ \
		netsnmp_log_handler *logh_head = get_logh_head();	\
```

### 2.5 gd
Add X11.lib into php-8.4.6\ext\gd\config.w32 so that it will link it. You need to re-run buildconf.bat after changing this file.
```javascript
if (CHECK_LIB("libXpm_a.lib", "gd", PHP_GD) && CHECK_LIB("X11.lib", "gd", PHP_GD) &&
```

Disable these statements in php-8.4.6\ext\gd\libgd\gd_interpolation.c, they only work in X86 platform.
```c
# pragma optimize("t", on)
# include <emmintrin.h>
```

### 2.6 Zend (php >= 8.4.7)
Limit this section to X64 platform in php-8.4.7\Zend\zend_vm_execute.h.
```c
#if defined(_WIN64) && defined(_M_X64)
/* See save_xmm_x86_64_ms_masm.asm */
void execute_ex_real(zend_execute_data *ex)
```

## 3. Building
Follow this tutorial [Build your own PHP on Windows](https://wiki.php.net/internals/windows/stepbystepbuild_sdk_2) to setup your build environment.
### 3.1 php-sdk
The phpsdk_deps won't work, because the php team haven't built depedencies for windows arm64. The Second thing is you need this file `php-sdk\phpsdk-vs17-arm64.bat` in this repository.
### 3.2 php-src
php >= 8.4.7 should apply the changes in 2.6.
### 3.3 start building
`--enable-snapshot-build` will try to enable everything, make `--with-extra-includes` and `--with-extra-libs` point to the vcpkg dirs.
```
configure --enable-snapshot-build --with-prefix=D:\php --disable-zts --without-analyzer --with-extra-includes=D:\PHP8.4\php-sdk-2.3.0\phpmaster\vs17\arm64\php-8.4.6\vcpkg_installed\arm64-windows\include --with-extra-libs=D:\PHP8.4\php-sdk-2.3.0\phpmaster\vs17\arm64\php-8.4.6\vcpkg_installed\arm64-windows\lib
nmake
nmake test
nmake install
```

## 4. Known issues
- --with-gmp can't be enabled because mpir can't be built. It uses lots of assembly code.
- --with-db Berkeley DB can't be built.
- --with-mhash, --enable-phar-native-ssl can't be enabled.
- opcache-jit is not supported for ARM64 yet.