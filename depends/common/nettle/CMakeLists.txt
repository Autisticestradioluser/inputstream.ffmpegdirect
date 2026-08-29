project(nettle)

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

list(APPEND nettle_conf CPPFLAGS=-I${CMAKE_INSTALL_PREFIX}/include)
list(APPEND nettle_conf LDFLAGS=-L${CMAKE_INSTALL_PREFIX}/lib)
list(APPEND nettle_conf LIBS=-lgmp)

include(ExternalProject)
externalproject_add(nettle
                    SOURCE_DIR ${CMAKE_SOURCE_DIR}
                    CONFIGURE_COMMAND <SOURCE_DIR>/configure
                      ${AND_HOST_FLAG}
                      CC=${AND_CC}
                      --prefix=${CMAKE_INSTALL_PREFIX}
                      --libdir=${CMAKE_INSTALL_PREFIX}/lib
                      --disable-shared
                      --disable-openssl
                      --disable-documentation
                      --enable-pic
                      CCPIC=
                      ${nettle_conf}
                    INSTALL_COMMAND ""
                    BUILD_IN_SOURCE 1)

install(CODE "execute_process(COMMAND make install WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})")
