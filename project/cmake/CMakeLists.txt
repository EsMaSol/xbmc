project(kodi)

cmake_minimum_required(VERSION 2.8.8)

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/modules)
if(DEPENDS_DIR)
  list(APPEND CMAKE_PREFIX_PATH ${DEPENDS_DIR})
endif()

enable_language(CXX)
enable_language(C)

# general
option(VERBOSE            "Enable verbose output?" OFF)
option(ENABLE_DVDCSS      "Enable libdvdcss support?" ON)
option(ENABLE_UPNP        "Enable UPnP support?" ON)
option(ENABLE_NONFREE     "Enable non-free components?" ON)
option(ENABLE_MICROHTTPD  "Enable MicroHttpd webserver?" ON)
option(ENABLE_MYSQLCLIENT "Enable MySql support?" ON)
option(ENABLE_AVAHI       "Enable Avahi support?" ON)
option(ENABLE_RTMP        "Enable RTMP support?" ON)
option(ENABLE_BLURAY      "Enable BluRay support?" ON)
option(ENABLE_AFPCLIENT   "Enable AFP support?" ON)
option(ENABLE_PLIST       "Enable AirPlay support?" ON)
option(ENABLE_AIRTUNES    "Enable AirTunes support?" ON)
option(ENABLE_CEC         "Enable CEC support?" ON)
if(UNIX)
  option(ENABLE_INTERNAL_FFMPEG "Enable internal ffmpeg?" ON)
  option(FFMPEG_PATH        "Path to external ffmpeg?" "")
  option(ENABLE_INTERNAL_CROSSGUID "Enable internal crossguid?" ON)
  option(ENABLE_ALSA        "Enable ALSA support?" ON)
  option(ENABLE_PULSEAUDIO  "Enable PulseAudio support?" ON)
  option(ENABLE_DBUS        "Enable dbus support?" ON)
  option(ENABLE_LIBUSB      "Enable libusb support?" ON)
  option(ENABLE_UDEV        "Enable UDev support?" ON)
  option(ENABLE_OPENGL      "Enable OpenGL?" ON)
  option(ENABLE_SDL         "Enable SDL?" OFF)
  option(ENABLE_X11         "Enable X11 support?" ON)
  option(ENABLE_WAYLAND     "Enable Wayland support?" OFF)
  option(ENABLE_EGL         "Enable EGL?" OFF)
  option(ENABLE_VAAPI       "Enable VAAPI support?" ON)
  option(ENABLE_VDPAU       "Enable VDPAU support?" ON)
  option(ENABLE_OPTICAL     "Enable optical support?" ON)
endif()
# System options
option(WITH_ARCH              "build with given arch" OFF)
option(WITH_CPU               "build with given cpu" OFF)

set(CORE_SOURCE_DIR ${PROJECT_SOURCE_DIR}/../..)
set(CORE_BUILD_DIR build)

include(scripts/common/addoptions.cmake)
include(scripts/common/archsetup.cmake)
include(scripts/common/macros.cmake)
include(scripts/common/managestring.cmake)
include(scripts/common/projectmacros.cmake)
include(scripts/common/pathsetup.cmake)
include(ExternalProject)

core_find_git_rev()
core_find_versions()

set(INCLUDES ${CORE_SOURCE_DIR}
             ${CORE_SOURCE_DIR}/addons/library.xbmc.addon
             ${CORE_SOURCE_DIR}/lib
             ${CORE_SOURCE_DIR}/lib/gtest/include
             ${CORE_SOURCE_DIR}/xbmc
             ${CORE_SOURCE_DIR}/xbmc/linux
             ${CORE_SOURCE_DIR}/xbmc/cores/VideoPlayer
             ${CORE_SOURCE_DIR}/tools/depends/native/libsquish-native/src
             ${CMAKE_BINARY_DIR}/${CORE_BUILD_DIR})

find_package(PkgConfig)
if(CMAKE_CROSSCOMPILING)
  if(PKG_CONFIG_EXECUTABLE)
    message(STATUS "CMAKE_CROSSCOMPILING: ${CMAKE_CROSSCOMPILING}")
    set(PKG_CONFIG_FOUND TRUE)
  endif()
endif()
find_package(Threads REQUIRED)
list(APPEND DEPLIBS ${CMAKE_THREAD_LIBS_INIT})

# Required dependencies
set(required_deps Sqlite3 FreeType PCRE Cpluff OpenSSL
                  TinyXML Python Yajl LibSmbClient ZLIB Xslt
                  JPEG Lzo2 Fribidi TagLib Boost FFMPEG CrossGUID)

# Optional dependencies
set(optional_deps MicroHttpd MySqlClient SSH
                  Alsa UDev Dbus Avahi Wayland
                  PulseAudio VDPAU VAAPI)

# Required, dyloaded deps
set(required_dyload Curl ASS Mpeg2 LibDvd)

# Optional, dyloaded deps
set(dyload_optional RTMP CEC Bluray
                    AfpClient Plist NFS)

# Required by shared objects we link
set(required_dep_libs Jasper PNG TIFF EXPAT)

foreach(dep ${required_dep_libs})
  find_package(${dep} REQUIRED)
endforeach()

foreach(dep ${required_deps})
  core_require_dep(${dep})
endforeach()

foreach(dep ${optional_deps})
  core_optional_dep(${dep})
endforeach()

if(NOT UDEV_FOUND)
  core_optional_dep(LibUSB)
endif()

foreach(dep ${required_dyload})
  core_require_dyload_dep(${dep})
endforeach()

foreach(dep ${dyload_optional})
  core_optional_dyload_dep(${dep})
endforeach()

if(ENABLE_UPNP)
  list(APPEND DEP_DEFINES "-DUSE_UPNP=1")
endif()

if(ENABLE_NONFREE)
  list(APPEND DEP_DEFINES "-DHAVE_XBMC_NONFREE=1")
endif()

if(ENABLE_OPTICAL)
  list(APPEND DEP_DEFINES -DHAS_DVD_DRIVE)
  core_require_dep(Cdio)
endif()

core_optional_dep(Shairplay ENABLE_AIRTUNES)
if(NOT SHAIRPLAY_FOUND)
  core_optional_dep(Shairport ENABLE_AIRTUNES)
endif()

if(NOT WIN32)
  core_optional_dep(OpenGl)
  if(OPENGL_FOUND)
    core_require_dep(GLEW)
    core_optional_dep(X ENABLE_X11)
    core_optional_dep(XRandR ENABLE_X11)
    core_optional_dep(Sdl)
  endif()
endif()

if(ENABLE_SKIN_TOUCHED)
  list(APPEND DEP_DEFINES "-DHAS_SKIN_TOUCHED=1")
endif()

add_custom_command(OUTPUT ${CORE_BUILD_DIR}/xbmc/CompileInfo.cpp
                   COMMAND ${CMAKE_COMMAND} -DCORE_SOURCE_DIR=${CORE_SOURCE_DIR}
                                            -DCORE_SYSTEM_NAME=${CORE_SYSTEM_NAME}
                                            -DCORE_BUILD_DIR=${CORE_BUILD_DIR}
                                            -DCMAKE_BINARY_DIR=${CMAKE_BINARY_DIR}
                                            -DARCH_DEFINES="${ARCH_DEFINES}"
                                            -DAPP_SCMID=${APP_SCMID}
                                            -Dprefix=${CMAKE_BINARY_DIR}/${CORE_BUILD_DIR}
                                            -P ${PROJECT_SOURCE_DIR}/scripts/common/generateversionedfiles.cmake
                  DEPENDS ${CORE_SOURCE_DIR}/version.txt
                          ${CORE_SOURCE_DIR}/addons/xbmc.addon/addon.xml.in
                          ${CORE_SOURCE_DIR}/addons/kodi.guilib/addon.xml.in
                          ${CORE_SOURCE_DIR}/xbmc/CompileInfo.cpp.in)
list(APPEND install_data addons/xbmc.addon/addon.xml)
add_library(compileinfo OBJECT ${CORE_BUILD_DIR}/xbmc/CompileInfo.cpp)

include_directories(${INCLUDES} ${SYSTEM_INCLUDES})
add_definitions(${ARCH_DEFINES} ${SYSTEM_DEFINES} ${DEP_DEFINES} ${PATH_DEFINES})

set(core_DEPENDS "" CACHE STRING "" FORCE)
set(test_archives "" CACHE STRING "" FORCE)
set(test_sources "" CACHE STRING "" FORCE)
mark_as_advanced(core_DEPENDS)
mark_as_advanced(test_archives)
mark_as_advanced(test_sources)

file(STRINGS ${PROJECT_SOURCE_DIR}/installdata/addon-bindings.txt bindings)
foreach(binding ${bindings})
  list(APPEND addon_bindings ${CORE_SOURCE_DIR}/${binding})
endforeach()

add_subdirectory(${CORE_SOURCE_DIR}/lib/gtest ${CORE_BUILD_DIR}/gtest EXCLUDE_FROM_ALL)

# Subdirs
core_add_subdirs_from_filelist(${PROJECT_SOURCE_DIR}/treedata/common/*.txt
                               ${PROJECT_SOURCE_DIR}/treedata/${CORE_SYSTEM_NAME}/*.txt)
core_add_optional_subdirs_from_filelist(${PROJECT_SOURCE_DIR}/treedata/optional/common/*.txt
                                        ${PROJECT_SOURCE_DIR}/treedata/optional/${CORE_SYSTEM_NAME}/*.txt)

# copy files to build tree
copy_files_from_filelist_to_buildtree(${PROJECT_SOURCE_DIR}/installdata/common/*.txt
                                      ${PROJECT_SOURCE_DIR}/installdata/${CORE_SYSTEM_NAME}/*.txt)

list(APPEND SKINS "${CORE_SOURCE_DIR}/addons/skin.confluence\;${CORE_SOURCE_DIR}")

# These are skins that are copied into place from the source tree
foreach(skin ${SKINS})
  list(GET skin 0 dir)
  list(GET skin 1 relative)
  copy_skin_to_buildtree(${dir} ${relative})
endforeach()

add_custom_target(pack-skins ALL
                  DEPENDS TexturePacker export-files ${XBT_FILES})

set(CXIMAGE_DEPLIBS ${ZLIB_LIBRARIES} ${JPEG_LIBRARIES}
                    ${TIFF_LIBRARIES} ${PNG_LIBRARIES}
                    ${JASPER_LIBRARIES} )

core_link_library(cximage system/ImageLib "nowrap" "${CXIMAGE_DEPLIBS}")
core_link_library(exif system/libexif)

core_link_library(XBMC_addon addons/library.xbmc.addon/libXBMC_addon)
core_link_library(XBMC_codec addons/library.xbmc.codec/libXBMC_codec)
core_link_library(KODI_guilib addons/library.kodi.guilib/libKODI_guilib)
core_link_library(KODI_adsp addons/library.kodi.adsp/libKODI_adsp)
core_link_library(XBMC_pvr   addons/library.xbmc.pvr/libXBMC_pvr)

file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/system/players/VideoPlayer)
add_custom_target(wrap-libraries ALL DEPENDS ${WRAP_FILES})

configure_file(${CORE_SOURCE_DIR}/xbmc/DllPaths_generated.h.in
               ${CORE_BUILD_DIR}/DllPaths_generated.h @ONLY)

# Build helpers
add_executable(JsonSchemaBuilder ${CORE_SOURCE_DIR}/tools/depends/native/JsonSchemaBuilder/src/JsonSchemaBuilder.cpp)
add_subdirectory(${CORE_SOURCE_DIR}/tools/depends/native/TexturePacker build/texturepacker)

# Codegen
add_subdirectory(${CORE_SOURCE_DIR}/xbmc/interfaces/swig build/swig)

# main binary
add_executable(${APP_NAME_LC}.bin ${CORE_SOURCE_DIR}/xbmc/main/main.cpp $<TARGET_OBJECTS:compileinfo>)
target_link_libraries(${APP_NAME_LC}.bin ${SYSTEM_LDFLAGS} -Wl,--whole-archive ${core_DEPENDS} -Wl,--no-whole-archive
                      ${DEPLIBS} ${CMAKE_DL_LIBS})
add_dependencies(${APP_NAME_LC}.bin export-files)

# testing
if(NOT CMAKE_CROSSCOMPILING) # Cannot execute tests when cross compiling
  copy_files_from_filelist_to_buildtree(${PROJECT_SOURCE_DIR}/installdata/test-reference-data.txt 1)
  add_executable(${APP_NAME_LC}-test EXCLUDE_FROM_ALL ${CORE_SOURCE_DIR}/xbmc/test/xbmc-test.cpp $<TARGET_OBJECTS:compileinfo>)
  target_link_libraries(${APP_NAME_LC}-test ${SYSTEM_LDFLAGS} -Wl,--whole-archive ${core_DEPENDS} gtest ${test_archives} -Wl,--no-whole-archive
                        ${DEPLIBS} ${CMAKE_DL_LIBS})
  enable_testing()
  gtest_add_tests(${APP_NAME_LC}-test "" ${test_sources})
  add_dependencies(${APP_NAME_LC}-test export-files)
  add_custom_target(check ${CMAKE_CTEST_COMMAND} WORKING_DIRECTORY ${PROJECT_BINARY_DIR})
  add_dependencies(check ${APP_NAME_LC}-test)

  # For testing commit series
  add_custom_target(check-commits ${CMAKE_COMMAND} -P ${PROJECT_SOURCE_DIR}/scripts/common/checkcommits.cmake
                                                   -DCMAKE_BINARY_DIR=${CMAKE_BINARY_DIR})
endif()

# link wrapper
if(FFMPEG_LINK_EXECUTABLE)
  set(CMAKE_CXX_LINK_EXECUTABLE "${FFMPEG_LINK_EXECUTABLE}")
endif()

# randr
add_executable(kodi-xrandr ${CORE_SOURCE_DIR}/xbmc-xrandr.c)
target_link_libraries(kodi-xrandr ${SYSTEM_LDFLAGS} ${X_LIBRARIES} m ${XRANDR_LIBRARIES})

include(scripts/${CORE_SYSTEM_NAME}/install.cmake)

# Status
message("#---- CONFIGURATION ----#")
foreach(msg ${final_message})
  message(STATUS ${msg})
endforeach()
message("#-----------------------#")
