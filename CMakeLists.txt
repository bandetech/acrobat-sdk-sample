cmake_minimum_required(VERSION 3.16)

# macOS: build a Universal binary so the plug-in loads in both native and Rosetta Acrobat.
# Must be set before project().
set(CMAKE_OSX_ARCHITECTURES "arm64;x86_64" CACHE STRING "macOS target architectures")
set(CMAKE_OSX_DEPLOYMENT_TARGET "12.0" CACHE STRING "Minimum macOS version")

project(MyCustomPlugin VERSION 1.0.0 LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Fetch Acrobat SDK library
include(FetchContent)
FetchContent_Declare(
    AcrobatSDK
    GIT_REPOSITORY https://github.com/bandetech/acrobat-sdk-cmake
    GIT_TAG main
)
FetchContent_MakeAvailable(AcrobatSDK)

if(WIN32)
    add_library(MyCustomPlugin SHARED
        src/BasicPlugin.cpp
        src/BasicPluginInit.cpp
        ${acrobatsdk_SOURCE_DIR}/Headers/API/PIMain.c
    )

    # change the output file suffix for the plugin to .api
    set_target_properties(MyCustomPlugin PROPERTIES
        PREFIX ""
        SUFFIX ".api"
    )
elseif(APPLE)
    set(ACROBAT_PLUGIN_BUNDLE_IDENTIFIER "com.example.MyCustomPlugin" CACHE STRING
        "Reverse-DNS bundle identifier for the Acrobat plug-in"
    )
    # The Acrobat SDK headers are written for Objective-C++ on macOS
    # (e.g. WLHFT.h uses NSURL*), so compile all sources as Objective-C++.
    enable_language(OBJCXX)
    set_source_files_properties(
        src/BasicPlugin.cpp
        src/BasicPluginInit.cpp
        ${acrobatsdk_SOURCE_DIR}/Headers/API/PIMain.c
        PROPERTIES LANGUAGE OBJCXX
    )
    add_library(MyCustomPlugin MODULE
        src/BasicPlugin.cpp
        src/BasicPluginInit.cpp
        ${acrobatsdk_SOURCE_DIR}/Headers/API/PIMain.c
    )
    # Sources do not include PIHeaders.h on Mac; it is supplied as a prefix header
    target_precompile_headers(MyCustomPlugin PRIVATE <Carbon/Carbon.h> <PIHeaders.h>)
    set_target_properties(MyCustomPlugin PROPERTIES
        BUNDLE TRUE
        BUNDLE_EXTENSION "acroplugin"
        OUTPUT_NAME "MyCustomPlugin"
        MACOSX_BUNDLE_INFO_PLIST "${CMAKE_CURRENT_SOURCE_DIR}/Info.plist.in"
        MACOSX_BUNDLE_GUI_IDENTIFIER "${ACROBAT_PLUGIN_BUNDLE_IDENTIFIER}"
        MACOSX_BUNDLE_BUNDLE_NAME "${PROJECT_NAME}"
        MACOSX_BUNDLE_INFO_STRING "${PROJECT_NAME} ${PROJECT_VERSION}"
        MACOSX_BUNDLE_LONG_VERSION_STRING "${PROJECT_VERSION}"
        MACOSX_BUNDLE_SHORT_VERSION_STRING "${PROJECT_VERSION}"
        MACOSX_BUNDLE_BUNDLE_VERSION "${PROJECT_VERSION}"
    )
    # Mac向け定義 (MAC_PLATFORM / ACRO_SDK_LEVEL は Acrobat_SDK 側で定義済み)
    target_compile_definitions(MyCustomPlugin PRIVATE
        PLATFORM=\"MacPlatform.h\"
        PRODUCT=\"Plugin.h\"
        Platform_Carbon=1
    )
else()
    message(FATAL_ERROR "Unsupported platform: only Windows and macOS are supported")
endif()

target_include_directories(MyCustomPlugin PRIVATE
    "${CMAKE_CURRENT_SOURCE_DIR}/src"
)
target_link_libraries(MyCustomPlugin PRIVATE Acrobat_SDK)
if(MSVC)
    # Sources are UTF-8 (without BOM)
    target_compile_options(MyCustomPlugin PRIVATE /utf-8)
endif()
if(APPLE)
    find_library(COREFOUNDATION_FRAMEWORK CoreFoundation REQUIRED)
    target_link_libraries(MyCustomPlugin PRIVATE "${COREFOUNDATION_FRAMEWORK}")
endif()

# For checking the include directories of Acrobat_SDK
get_target_property(
    ACRO_INCLUDES
    Acrobat_SDK
    INTERFACE_INCLUDE_DIRECTORIES
)
message(STATUS "AcrobatSDK Includes=${ACRO_INCLUDES}")