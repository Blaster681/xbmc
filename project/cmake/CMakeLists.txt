cmake_minimum_required(VERSION 3.1)
project(kodi)

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/modules)
if(DEPENDS_DIR)
  list(APPEND CMAKE_PREFIX_PATH ${DEPENDS_DIR})
endif()

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

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
option(ENABLE_PLIST       "Enable AirPlay support?" ON)
option(ENABLE_NFS         "Enable NFS support?" ON)
option(ENABLE_AIRTUNES    "Enable AirTunes support?" ON)
option(ENABLE_CEC         "Enable CEC support?" ON)
if(UNIX)
  if(NOT APPLE)
    option(ENABLE_INTERNAL_FFMPEG "Enable internal ffmpeg?" ON)
  endif()
  option(FFMPEG_PATH        "Path to external ffmpeg?" "")
  option(ENABLE_INTERNAL_CROSSGUID "Enable internal crossguid?" ON)
  option(ENABLE_ALSA        "Enable ALSA support?" ON)
  option(ENABLE_PULSEAUDIO  "Enable PulseAudio support?" ON)
  option(ENABLE_DBUS        "Enable dbus support?" ON)
  option(ENABLE_LIBUSB      "Enable libusb support?" ON)
  option(ENABLE_UDEV        "Enable UDev support?" ON)
  option(ENABLE_OPENGL      "Enable OpenGL?" ON)
  option(ENABLE_OPENGLES    "Enable OpenGLES?" ON)
  option(ENABLE_OPENSSL     "Enable OpenSSL?" ON)
  option(ENABLE_SDL         "Enable SDL?" OFF)
  option(ENABLE_X11         "Enable X11 support?" ON)
  option(ENABLE_EGL         "Enable EGL?" OFF)
  option(ENABLE_VAAPI       "Enable VAAPI support?" ON)
  option(ENABLE_VDPAU       "Enable VDPAU support?" ON)
  option(ENABLE_OPTICAL     "Enable optical support?" ON)
endif()
# System options
if(NOT WIN32)
  option(WITH_ARCH              "build with given arch" OFF)
  option(WITH_CPU               "build with given cpu" OFF)
  option(ENABLE_CCACHE          "Enable Ccache support" ON)
endif()
if(CMAKE_SYSTEM_NAME STREQUAL Linux)
  option(ENABLE_LIRC            "Enable LIRC support?" ON)
endif()

get_filename_component(CORE_SOURCE_DIR ${PROJECT_SOURCE_DIR}/../.. ABSOLUTE)
set(CORE_BUILD_DIR build)

message(STATUS "Source directory: ${CORE_SOURCE_DIR}")
message(STATUS "Build directory: ${CMAKE_BINARY_DIR}")

include(scripts/common/GeneratorSetup.cmake)
include(scripts/common/AddOptions.cmake)
include(scripts/common/ArchSetup.cmake)
include(scripts/common/Macros.cmake)
include(scripts/common/ManageString.cmake)
include(scripts/common/ProjectMacros.cmake)
include(scripts/common/PathSetup.cmake)
include(ExternalProject)

core_find_git_rev()
core_find_versions()

set(INCLUDES ${CORE_SOURCE_DIR}
             ${CORE_SOURCE_DIR}/addons/library.xbmc.addon
             ${CORE_SOURCE_DIR}/lib
             ${CORE_SOURCE_DIR}/lib/gtest/include
             ${CORE_SOURCE_DIR}/xbmc
             ${CORE_SOURCE_DIR}/xbmc/${PLATFORM_DIR}
             ${CORE_SOURCE_DIR}/xbmc/cores/VideoPlayer
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
set(required_deps Sqlite3 FreeType PCRE Cpluff LibDvd
                  TinyXML Python Yajl Xslt
                  Lzo2 Fribidi TagLib FFMPEG CrossGUID)
if(NOT WIN32)
  list(APPEND required_deps LibSmbClient ZLIB)
else()
  list(APPEND required_deps D3DX11Effects)
endif()
if(CORE_SYSTEM_NAME STREQUAL android)
  list(APPEND required_deps Zip)
endif()

# Optional dependencies
set(optional_deps MicroHttpd MySqlClient SSH
                  Alsa UDev Dbus Avahi
                  PulseAudio VDPAU VAAPI)

# Required, dyloaded deps
set(required_dyload Curl ASS)

# Optional, dyloaded deps
set(dyload_optional RTMP CEC Bluray Plist NFS)

# Required by shared objects we link
set(required_dep_libs PNG EXPAT)

# Required tools
find_package(TexturePacker REQUIRED)
find_package(JsonSchemaBuilder REQUIRED)

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

if(ENABLE_OPENSSL)
  core_require_dep(OpenSSL)
  list(APPEND DEP_DEFINES "-DHAVE_OPENSSL=1")
endif()

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

if(ENABLE_AIRTUNES)
  find_package(Shairplay)
  if(SHAIRPLAY_FOUND)
    core_require_dyload_dep(Shairplay)
  endif()
endif()

if(NOT WIN32)
  core_optional_dep(OpenGl)
  if(OPENGL_FOUND)
    if(NOT APPLE)
      core_optional_dep(Sdl)
    else()
      core_require_dep(Sdl)
    endif()
    core_optional_dep(X ENABLE_X11)
    core_optional_dep(XRandR ENABLE_X11)
  else()
    core_optional_dep(OpenGLES ENABLE_OPENGLES)
    if(OPENGLES_FOUND)
      core_optional_dep(EGL ENABLE_EGL)
      core_optional_dep(OMX ENABLE_OMX)
      core_optional_dep(AML ENABLE_AML)
      core_optional_dep(X ENABLE_X11)
      core_optional_dep(Sdl)
    endif()
  endif()

  if(CORE_SYSTEM_NAME STREQUAL rbpi)
    core_require_dep(MMAL)
  endif()
endif()

if(ENABLE_CCACHE)
  core_optional_dep(CCache)
endif()

# Compile Info
add_custom_command(OUTPUT ${CORE_BUILD_DIR}/xbmc/CompileInfo.cpp
                          ${CMAKE_BINARY_DIR}/addons/xbmc.addon/addon.xml
                          ${CMAKE_BINARY_DIR}/addons/kodi.guilib/addon.xml
                   COMMAND ${CMAKE_COMMAND} -DCORE_SOURCE_DIR=${CORE_SOURCE_DIR}
                                            -DCORE_SYSTEM_NAME=${CORE_SYSTEM_NAME}
                                            -DCORE_BUILD_DIR=${CORE_BUILD_DIR}
                                            -DCMAKE_BINARY_DIR=${CMAKE_BINARY_DIR}
                                            -DARCH_DEFINES="${ARCH_DEFINES}"
                                            -DAPP_SCMID=${APP_SCMID}
                                            -Dprefix=${CMAKE_BINARY_DIR}/${CORE_BUILD_DIR}
                                            -P ${PROJECT_SOURCE_DIR}/scripts/common/GenerateVersionedFiles.cmake
                  DEPENDS ${CORE_SOURCE_DIR}/version.txt
                          ${CORE_SOURCE_DIR}/addons/xbmc.addon/addon.xml.in
                          ${CORE_SOURCE_DIR}/addons/kodi.guilib/addon.xml.in
                          ${CORE_SOURCE_DIR}/xbmc/CompileInfo.cpp.in)
list(APPEND install_data addons/xbmc.addon/addon.xml)
list(APPEND install_data addons/xbmc.json/addon.xml)
list(APPEND install_data addons/kodi.guilib/addon.xml)
add_library(compileinfo OBJECT ${CORE_BUILD_DIR}/xbmc/CompileInfo.cpp)
target_compile_options(compileinfo PRIVATE "${SYSTEM_DEFINES}")

# RC File
if(WIN32)
  configure_file(${CORE_SOURCE_DIR}/xbmc/win32/XBMC_PC.rc.in
                 ${CORE_BUILD_DIR}/xbmc/win32/XBMC_PC.rc @ONLY)
  add_library(resources OBJECT ${CORE_BUILD_DIR}/xbmc/win32/XBMC_PC.rc)
  target_include_directories(resources PRIVATE ${CORE_SOURCE_DIR}/tools/windows/packaging/media)
  set(RESOURCES $<TARGET_OBJECTS:resources>)
endif()

include_directories(${INCLUDES} ${SYSTEM_INCLUDES})
add_compile_options(${ARCH_DEFINES} "${SYSTEM_DEFINES}" ${DEP_DEFINES} ${PATH_DEFINES})

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

file(STRINGS ${PROJECT_SOURCE_DIR}/installdata/cmake-files.txt cmakefiles)
foreach(cmakefile ${cmakefiles})
  list(APPEND cmake_files ${CORE_SOURCE_DIR}/${cmakefile})
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

list(APPEND SKINS "${CORE_SOURCE_DIR}/addons/skin.estuary\;${CORE_SOURCE_DIR}")
list(APPEND SKINS "${CORE_SOURCE_DIR}/addons/skin.estouchy\;${CORE_SOURCE_DIR}")

# These are skins that are copied into place from the source tree
foreach(skin ${SKINS})
  list(GET skin 0 dir)
  list(GET skin 1 relative)
  copy_skin_to_buildtree(${dir} ${relative})
endforeach()

add_custom_target(pack-skins ALL
                  DEPENDS TexturePacker::TexturePacker export-files ${XBT_FILES})

core_link_library(exif system/libexif)
if(CORE_SYSTEM_NAME STREQUAL linux)
  core_link_library(sse4 system/libsse4)
endif()

core_link_library(XBMC_addon addons/library.xbmc.addon/libXBMC_addon nowrap)
core_link_library(XBMC_codec addons/library.xbmc.codec/libXBMC_codec nowrap)
core_link_library(XBMC_pvr addons/library.xbmc.pvr/libXBMC_pvr nowrap)
core_link_library(KODI_adsp addons/library.kodi.adsp/libKODI_adsp nowrap)
core_link_library(KODI_audioengine addons/library.kodi.audioengine/libKODI_audioengine nowrap)
core_link_library(KODI_guilib addons/library.kodi.guilib/libKODI_guilib nowrap)
core_link_library(KODI_inputstream addons/library.kodi.inputstream/libKODI_inputstream nowrap)
core_link_library(KODI_peripheral addons/library.kodi.peripheral/libKODI_peripheral nowrap)

file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/system/players/VideoPlayer)
add_custom_target(wrap-libraries ALL DEPENDS ${WRAP_FILES})

set(LIBCEC_SONAME "${CEC_SONAME}")
if(NOT CORE_SYSTEM_NAME STREQUAL windows AND NOT CORE_SYSTEM_NAME STREQUAL android)
  configure_file(${CORE_SOURCE_DIR}/xbmc/DllPaths_generated.h.in
                 ${CORE_BUILD_DIR}/DllPaths_generated.h @ONLY)
elseif(CORE_SYSTEM_NAME STREQUAL android)
  configure_file(${CORE_SOURCE_DIR}/xbmc/DllPaths_generated_android.h.in
                 ${CORE_BUILD_DIR}/DllPaths_generated_android.h @ONLY)
endif()

# Codegen
add_subdirectory(${CORE_SOURCE_DIR}/xbmc/interfaces/swig build/swig)

# Other files (IDE)
set(OTHER_FILES README.md)

# main binary
add_executable(${APP_NAME_LC} ${CORE_MAIN_SOURCE} $<TARGET_OBJECTS:compileinfo> "${RESOURCES}" ${OTHER_FILES})
whole_archive(_MAIN_LIBRARIES ${core_DEPENDS})
target_link_libraries(${APP_NAME_LC} ${SYSTEM_LDFLAGS} ${_MAIN_LIBRARIES} ${DEPLIBS} ${CMAKE_DL_LIBS})
unset(_MAIN_LIBRARIES)
add_dependencies(${APP_NAME_LC} export-files)
if(NOT WIN32)
  set_target_properties(${APP_NAME_LC} PROPERTIES SUFFIX ".bin")
else()
  set_target_properties(${APP_NAME_LC} PROPERTIES WIN32_EXECUTABLE ON)
  copy_main_dlls_to_buildtree()
endif()

# testing
copy_files_from_filelist_to_buildtree(${PROJECT_SOURCE_DIR}/installdata/test-reference-data.txt NO_INSTALL)
add_executable(${APP_NAME_LC}-test EXCLUDE_FROM_ALL ${CORE_SOURCE_DIR}/xbmc/test/xbmc-test.cpp $<TARGET_OBJECTS:compileinfo>)
whole_archive(_TEST_LIBRARIES ${core_DEPENDS} gtest ${test_archives})
target_link_libraries(${APP_NAME_LC}-test ${SYSTEM_LDFLAGS} ${_TEST_LIBRARIES} ${DEPLIBS} ${CMAKE_DL_LIBS})
unset(_TEST_LIBRARIES)
add_dependencies(${APP_NAME_LC}-test export-files)

# make test and make check (cannot be executed when cross compiling)
if(NOT CMAKE_CROSSCOMPILING)
  enable_testing()
  gtest_add_tests(${APP_NAME_LC}-test "" ${test_sources})
  add_custom_target(check ${CMAKE_CTEST_COMMAND} WORKING_DIRECTORY ${PROJECT_BINARY_DIR})
  add_dependencies(check ${APP_NAME_LC}-test)

  # For testing commit series
  add_custom_target(check-commits ${CMAKE_COMMAND} -P ${PROJECT_SOURCE_DIR}/scripts/common/CheckCommits.cmake
                                                   -DCMAKE_BINARY_DIR=${CMAKE_BINARY_DIR})
endif()

# link wrapper
if(FFMPEG_LINK_EXECUTABLE)
  set(CMAKE_CXX_LINK_EXECUTABLE "${FFMPEG_LINK_EXECUTABLE}")
endif()

# randr
if(ENABLE_X11 AND XRANDR_FOUND)
  add_executable(kodi-xrandr ${CORE_SOURCE_DIR}/xbmc-xrandr.c)
  target_link_libraries(kodi-xrandr ${SYSTEM_LDFLAGS} ${X_LIBRARIES} m ${XRANDR_LIBRARIES})
endif()

# XBMCHelper
if(CORE_SYSTEM_NAME STREQUAL darwin)
  add_subdirectory(${CORE_SOURCE_DIR}/tools/EventClients/Clients/OSXRemote build/XBMCHelper)
endif()

include(scripts/${CORE_SYSTEM_NAME}/Install.cmake)

# Prepare add-on build env
foreach(binding ${addon_bindings})
  get_filename_component(file ${binding} NAME)
  set(BINDING_TARGET ${CORE_BUILD_DIR}/include/${APP_NAME_LC}/${file})
  configure_file(${binding} ${BINDING_TARGET} COPYONLY)
endforeach()

set(APP_LIB_DIR ${CMAKE_BINARY_DIR}/${CORE_BUILD_DIR}/lib/${APP_NAME_LC})
set(APP_INCLUDE_DIR ${CMAKE_BINARY_DIR}/${CORE_BUILD_DIR}/include/${APP_NAME_LC})
set(CXX11_SWITCH "-std=c++11")
configure_file(${PROJECT_SOURCE_DIR}/KodiConfig.cmake.in
               ${CORE_BUILD_DIR}/lib/${APP_NAME_LC}/${APP_NAME_LC}Config.cmake @ONLY)
configure_file(${PROJECT_SOURCE_DIR}/scripts/common/AddonHelpers.cmake
               ${CORE_BUILD_DIR}/lib/${APP_NAME_LC}/AddonHelpers.cmake COPYONLY)
configure_file(${PROJECT_SOURCE_DIR}/scripts/common/AddOptions.cmake
               ${CORE_BUILD_DIR}/lib/${APP_NAME_LC}/AddOptions.cmake COPYONLY)

# Status
message(STATUS "#---- CONFIGURATION ----#")
foreach(msg ${final_message})
  message(STATUS ${msg})
endforeach()

if(VERBOSE)
  message(STATUS "\n#--------------- Internal Variables -------------#")
  message(STATUS "DEPLIBS: ${DEPLIBS}")
  message(STATUS "core_DEPENDS: ${core_DEPENDS}")
  message(STATUS "#---------------------------------------------#")
  message(STATUS "SYSTEM_LDFLAGS        : ${SYSTEM_LDFLAGS}")
  message(STATUS "CMAKE_DL_LIBS         : ${CMAKE_DL_LIBS}")
  message(STATUS "CMAKE_C_FLAGS         : ${CMAKE_C_FLAGS}")
  message(STATUS "CMAKE_CXX_FLAGS       : ${CMAKE_CXX_FLAGS}")
  message(STATUS "CMAKE_EXE_LINKER_FLAGS: ${CMAKE_EXE_LINKER_FLAGS}")
  message(STATUS "#---------------------------------------------#")
  message(STATUS "bindir     : ${bindir}")
  message(STATUS "includedir : ${includedir}")
  message(STATUS "libdir     : ${libdir}")
  message(STATUS "datarootdir: ${datarootdir}")
  message(STATUS "#---------------------------------------------#")
  message(STATUS "GIT_REV: ${APP_SCMID}")
endif()
message(STATUS "#---------------------------------------------#")

