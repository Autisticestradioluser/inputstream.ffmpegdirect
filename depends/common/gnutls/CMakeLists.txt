project(gnutls)

# Cross-compilation setup for bare-autoconf recipes: ANDROID_ABI is forwarded
# by HandleDepends; the NDK toolchain leaves env CC unusable here, so derive
# both --host and a target-wrapper CC from the ABI and the configured compiler.
if(CORE_SYSTEM_NAME STREQUAL android AND ANDROID_ABI)
  get_filename_component(_ndk_bin "${CMAKE_C_COMPILER}" DIRECTORY)
  if(ANDROID_ABI STREQUAL arm64-v8a)
    set(AND_HOST_FLAG "--host=aarch64-linux-android")
    set(AND_CC "${_ndk_bin}/aarch64-linux-android21-clang")
    set(AND_CXX "${_ndk_bin}/aarch64-linux-android21-clang++")
  elseif(ANDROID_ABI STREQUAL armeabi-v7a)
    set(AND_HOST_FLAG "--host=arm-linux-androideabi")
    set(AND_CC "${_ndk_bin}/armv7a-linux-androideabi21-clang")
    set(AND_CXX "${_ndk_bin}/armv7a-linux-androideabi21-clang++")
  endif()
endif()

cmake_minimum_required(VERSION 3.5)

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR})

# GnuTLS automake can request a rebuild even though we don't have
# all the tools installed so we disable it for the build
list(APPEND gnutls_conf --disable-maintainer-mode)

list(APPEND gnutls_conf PKG_CONFIG_PATH=${CMAKE_INSTALL_PREFIX}/lib/pkgconfig)
list(APPEND gnutls_conf CPPFLAGS=-I${CMAKE_INSTALL_PREFIX}/include)
list(APPEND gnutls_conf LDFLAGS=-L${CMAKE_INSTALL_PREFIX}/lib)
list(APPEND gnutls_conf LIBS=-lgmp)

if(CORE_SYSTEM_NAME STREQUAL darwin_embedded)
  list(APPEND gnutls_conf ac_cv_func_vfork_works=no)
  list(APPEND gnutls_conf ac_cv_func_fork=no)
endif()

include(ExternalProject)
externalproject_add(gnutls
                    SOURCE_DIR ${CMAKE_SOURCE_DIR}
                    CONFIGURE_COMMAND <SOURCE_DIR>/configure
                      ${AND_HOST_FLAG}
                      CC=${AND_CC}
                      --prefix=${CMAKE_INSTALL_PREFIX}
                      --disable-shared
                      --without-p11-kit
                      --without-idn
                      --without-zstd
                      --without-brotli
                      --disable-nls
                      --with-included-unistring
                      --with-included-libtasn1
                      --enable-local-libopts
                      --without-nettle-mini
                      --disable-doc
                      --disable-tests
                      --disable-guile
                      --disable-tools
                      --with-pic
                      ${gnutls_conf})

if(CORE_SYSTEM_NAME STREQUAL darwin_embedded)
  externalproject_add_step(gnutls
                           RunSedCommand
                           # Apple CCAS requires stripping ; from comment block # to build successfully
                           COMMAND echo "Stripping semi-colon from comment blocks for Apple CCAS to build successfully"
                           COMMAND bash -c "sed -ie 's|\\(^# .*\\)[;]\\(.*\\)|\\1\\2|g' ${CMAKE_CURRENT_SOURCE_DIR}/lib/accelerated/aarch64/macosx/*.s"
                           DEPENDERS configure)
endif()

install(CODE "Message(Done)")
