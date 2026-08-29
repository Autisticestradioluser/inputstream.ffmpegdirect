cmake_minimum_required(VERSION 3.5)
project(gmp)

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

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR})

include(ExternalProject)

if(CORE_SYSTEM_NAME STREQUAL osx OR
   CORE_SYSTEM_NAME STREQUAL darwin_embedded)

  if(CORE_SYSTEM_NAME STREQUAL darwin_embedded)
    set(EXTRA_ARGS "--disable-assembly")
    set(EXTRA_ARGS "${EXTRA_ARGS} --host=aarch64-apple-darwin")
  else()
    set(EXTRA_ARGS "--with-pic")
    if(CPU STREQUAL "arm64")
      set(EXTRA_ARGS "${EXTRA_ARGS} --host=aarch64-apple-darwin")
    else()
      set(EXTRA_ARGS "${EXTRA_ARGS} --host=x86_64-apple-darwin")
    endif()
    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
      set(EXTRA_ARGS "${EXTRA_ARGS} ABI=64")
    else()
      set(EXTRA_ARGS "${EXTRA_ARGS} ABI=32")
    endif()
  endif()

  externalproject_add(gmp
                      SOURCE_DIR ${CMAKE_SOURCE_DIR}
                      PATCH_COMMAND autoreconf -vif
                      CONFIGURE_COMMAND
                          bash --login -c
                          "./configure \
                          --prefix=${CMAKE_BINARY_DIR}/gmpbuild \
                          CC='${CMAKE_C_COMPILER} ${CMAKE_C_FLAGS}' \
                          LDFLAGS='${CMAKE_EXE_LINKER_FLAGS}' \
                          CPP='${CMAKE_C_COMPILER} -E' \
                          CPPFLAGS='${CMAKE_C_FLAGS}' \
                          ${EXTRA_ARGS} \
                          --enable-static \
                          --disable-shared"
                      BUILD_COMMAND  make
                      INSTALL_COMMAND
                        make install &&
                        # Is installed to different place to become no conflict during configure with
                        # "./share/config.site" on addon depends binary folder
                        cmake -E copy_directory ${CMAKE_BINARY_DIR}/gmpbuild ${CMAKE_INSTALL_PREFIX}
                      BUILD_IN_SOURCE TRUE)
else()
  if(CORE_SYSTEM_NAME MATCHES "linux" AND CPU MATCHES "i.86")
      list(APPEND gmp_conf ABI=32)
  endif()

  externalproject_add(gmp
                      SOURCE_DIR ${CMAKE_SOURCE_DIR}
                      PATCH_COMMAND autoreconf -vif
                      CONFIGURE_COMMAND <SOURCE_DIR>/configure
                        ${AND_HOST_FLAG}
                        CC=${AND_CC}
                        --prefix=${CMAKE_INSTALL_PREFIX}
                        --disable-assembly
                        --enable-static
                        --disable-shared
                        --with-pic
                        ${gmp_conf})
endif()

install(CODE "Message(Done)")
