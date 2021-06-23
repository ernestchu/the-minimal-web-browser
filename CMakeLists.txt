project(WebBrowser C CXX)

cmake_minimum_required(VERSION 3.3.2)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(SDK_ROOT "${CMAKE_BINARY_DIR}/SDK/")
if (UNIX)
    if (APPLE)
	set(PORT UltralightMac)
    else ()
	set(PORT UltralightLinux)
    endif ()
elseif (CMAKE_SYSTEM_NAME MATCHES "Windows")
    set(PORT UltralightWin)
else ()
    message(FATAL_ERROR "Unknown OS '${CMAKE_SYSTEM_NAME}'")
endif ()
include(${CMAKE_CURRENT_LIST_DIR}/Deps.cmake)

set(ULTRALIGHT_INCLUDE_DIR "${SDK_ROOT}/include")
set(ULTRALIGHT_BINARY_DIR "${SDK_ROOT}/bin")
set(ULTRALIGHT_INSPECTOR_DIR "${SDK_ROOT}/inspector")

if (CMAKE_SYSTEM_NAME MATCHES "Windows")
    set(ULTRALIGHT_LIBRARY_DIR "${SDK_ROOT}/lib")
else ()
    set(ULTRALIGHT_LIBRARY_DIR "${SDK_ROOT}/bin")
endif ()

if (PORT MATCHES "UltralightMac")
    SET(CMAKE_SKIP_BUILD_RPATH  FALSE)
    SET(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
    SET(CMAKE_INSTALL_RPATH "@executable_path/")
endif ()

if (PORT MATCHES "UltralightLinux")
    SET(CMAKE_SKIP_BUILD_RPATH  FALSE)
    SET(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
    SET(CMAKE_INSTALL_RPATH "$\{ORIGIN\}")
endif ()

add_custom_target(
    CopySDK ALL
    COMMAND ${CMAKE_COMMAND} -E make_directory "${SDK_ROOT}"
    COMMAND ${CMAKE_COMMAND} -E make_directory "${SDK_ROOT}/bin"
    COMMAND ${CMAKE_COMMAND} -E make_directory "${SDK_ROOT}/include"
    COMMAND ${CMAKE_COMMAND} -E make_directory "${SDK_ROOT}/inspector"
    COMMAND ${CMAKE_COMMAND} -E copy_directory "${ULTRALIGHTCORE_DIR}/bin/" "${SDK_ROOT}/bin"
    COMMAND ${CMAKE_COMMAND} -E copy_directory "${ULTRALIGHTCORE_DIR}/include/" "${SDK_ROOT}/include"
    COMMAND ${CMAKE_COMMAND} -E remove_directory "${SDK_ROOT}/include/Ultralight/private"
    COMMAND ${CMAKE_COMMAND} -E copy_directory "${WEBCORE_DIR}/bin/" "${SDK_ROOT}/bin"
    COMMAND ${CMAKE_COMMAND} -E copy_directory "${WEBCORE_DIR}/include/" "${SDK_ROOT}/include"
    COMMAND ${CMAKE_COMMAND} -E copy_directory "${WEBCORE_DIR}/inspector/" "${SDK_ROOT}/inspector"
    COMMAND ${CMAKE_COMMAND} -E copy_directory "${ULTRALIGHT_DIR}/bin/" "${SDK_ROOT}/bin"
    COMMAND ${CMAKE_COMMAND} -E copy_directory "${ULTRALIGHT_DIR}/include/" "${SDK_ROOT}/include"
    COMMAND ${CMAKE_COMMAND} -E copy_directory "${APPCORE_DIR}/bin/" "${SDK_ROOT}/bin"
    COMMAND ${CMAKE_COMMAND} -E copy_directory "${APPCORE_DIR}/include/" "${SDK_ROOT}/include"
    DEPENDS UltralightCoreBin WebCoreBin UltralightBin AppCoreBin
)

if (PORT MATCHES "UltralightWin")
    add_custom_command(TARGET CopySDK POST_BUILD
	COMMAND ${CMAKE_COMMAND} -E make_directory "${SDK_ROOT}/lib"
	COMMAND ${CMAKE_COMMAND} -E copy_directory "${ULTRALIGHTCORE_DIR}/lib/" "${SDK_ROOT}/lib"
	COMMAND ${CMAKE_COMMAND} -E copy_directory "${WEBCORE_DIR}/lib/" "${SDK_ROOT}/lib"
	COMMAND ${CMAKE_COMMAND} -E copy_directory "${ULTRALIGHT_DIR}/lib/" "${SDK_ROOT}/lib"
	COMMAND ${CMAKE_COMMAND} -E copy_directory "${APPCORE_DIR}/lib/" "${SDK_ROOT}/lib"
    )
endif ()

set(APP_NAME WebBrowser)
include_directories("${ULTRALIGHT_INCLUDE_DIR}")
link_directories("${ULTRALIGHT_LIBRARY_DIR}")
link_libraries(UltralightCore AppCore Ultralight WebCore)

set(SOURCES "src/Browser.h"
            "src/Browser.cpp"
            "src/Tab.h"
            "src/Tab.cpp"
            "src/UI.h"
            "src/UI.cpp"
            "src/main.cpp")

add_executable(${APP_NAME} WIN32 MACOSX_BUNDLE ${SOURCES})

if (APPLE)
  # Enable High-DPI on macOS through our custom Info.plist template
  set_target_properties(${APP_NAME} PROPERTIES MACOSX_BUNDLE_INFO_PLIST ${CMAKE_CURRENT_SOURCE_DIR}/Info.plist.in) 
endif()

if (MSVC)
  # Tell MSVC to use main instead of WinMain for Windows subsystem executables
  set_target_properties(${APP_NAME} PROPERTIES LINK_FLAGS "/ENTRY:mainCRTStartup")
endif()

# Copy all binaries to target directory
add_custom_command(TARGET ${APP_NAME} POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E copy_directory "${ULTRALIGHT_BINARY_DIR}" $<TARGET_FILE_DIR:${APP_NAME}>) 

if (APPLE)
  set(ASSETS_PATH "$<TARGET_FILE_DIR:${APP_NAME}>/../Resources/assets") 
else ()
  set(ASSETS_PATH "$<TARGET_FILE_DIR:${APP_NAME}>/assets") 
endif ()

# Copy assets to assets directory
add_custom_command(TARGET ${APP_NAME} POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E copy_directory "${CMAKE_CURRENT_SOURCE_DIR}/assets/" "${ASSETS_PATH}")
    
# Copy inspector to assets directory
add_custom_command(TARGET ${APP_NAME} POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E copy_directory "${ULTRALIGHT_INSPECTOR_DIR}" "${ASSETS_PATH}/inspector")

if (APPLE)
  # Copy resources to resources directory
  add_custom_command(TARGET ${APP_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory "${ULTRALIGHT_BINARY_DIR}/resources/" "$<TARGET_FILE_DIR:${APP_NAME}>/../Resources/resources")

  # Resources directory is also copied to target directory in the "Copy all binaries" step.
  # Remove this duplicate directory.
  add_custom_command(TARGET ${APP_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E remove_directory "$<TARGET_FILE_DIR:${APP_NAME}>/resources")
endif ()

add_dependencies(WebBrowser CopySDK)
