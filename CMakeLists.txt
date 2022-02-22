# For more information about using CMake with Android Studio, read the
# documentation: https://d.android.com/studio/projects/add-native-code.html

# Sets the minimum version of CMake required to build the native library.

cmake_minimum_required(VERSION 3.10.2)

# Declares and names the project.

project(Diligent-Log)

# Define GNU standard installation directories such as CMAKE_INSTALL_INCLUDEDIR, CMAKE_INSTALL_LIBDIR, etc.
include(GNUInstallDirs)

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Debug CACHE STRING "" FORCE)
    message(STATUS "CMAKE_BUILD_TYPE is not specified, default to Debug. Note that this is only relevant for single-configuration generators (such as Makefile Generators and Ninja).")
endif()

set(DEBUG_CONFIGURATIONS DEBUG CACHE INTERNAL "Debug configurations")
set(RELEASE_CONFIGURATIONS RELEASE RELWITHDEBINFO MINSIZEREL CACHE INTERNAL "Release configurations")

if(BUILD_CONFIGURATION_FILE)
    message("Using build configuration file " ${CUSTOM_BUILD_SCRIPT})
    include(${CMAKE_SOURCE_DIR}/${BUILD_CONFIGURATION_FILE})

    if(COMMAND custom_configure_build)
        custom_configure_build()
    else()
        message("custom_configure_build() function not found in " ${CUSTOM_BUILD_SCRIPT})
    endif()
endif()

# Generate XCode schemes
set(CMAKE_XCODE_GENERATE_SCHEME TRUE)
# Make malloc write 0xAA to newly allocated memory and 0x55 to deallocated memory
set(CMAKE_XCODE_SCHEME_MALLOC_SCRIBBLE YES)
# Place guard pages on each side of large (4096 bytes or more) buffers
set(CMAKE_XCODE_SCHEME_MALLOC_GUARD_EDGES YES)

if("${CMAKE_SIZEOF_VOID_P}" EQUAL "8")
    set(ARCH 64 CACHE INTERNAL "64-bit architecture")
else()
    set(ARCH 32 CACHE INTERNAL "32-bit architecture")
endif()

# we need to rename due to conflics, normally this would link with PublicBuildSettings
# however PublicBuildSettings is related to Diligent Backends
add_library(Diligent-BuildSettings__LOG INTERFACE)
target_link_libraries(Diligent-BuildSettings__LOG INTERFACE)

if (NOT PLATFORM_HAS_BEEN_DEFINED)
    if(WIN32)
        if(${CMAKE_SYSTEM_NAME} STREQUAL "WindowsStore")
            target_compile_definitions(Diligent-BuildSettings__LOG INTERFACE -DPLATFORM_UNIVERSAL_WINDOWS)
            set(PLATFORM_UNIVERSAL_WINDOWS TRUE CACHE INTERNAL "Target platform: Windows Store")
            message("Target platform: Universal Windows. SDK Version: " ${CMAKE_SYSTEM_VERSION})
        else()
            target_compile_definitions(Diligent-BuildSettings__LOG INTERFACE -DPLATFORM_WIN32)
            set(PLATFORM_WIN32 TRUE CACHE INTERNAL "Target platform: Win32") #WIN32 is a variable, so we cannot use string "WIN32"
            message("Target platform: Win32. SDK Version: " ${CMAKE_SYSTEM_VERSION})
        endif()
    else()
        if(${CMAKE_SYSTEM_NAME} STREQUAL "Android")
            target_compile_definitions(Diligent-BuildSettings__LOG INTERFACE -DPLATFORM_ANDROID)
            set(PLATFORM_ANDROID TRUE CACHE INTERNAL "Target platform: Android")
            message("Target platform: Android")
        elseif(${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
            target_compile_definitions(Diligent-BuildSettings__LOG INTERFACE -DPLATFORM_LINUX)
            set(PLATFORM_LINUX TRUE CACHE INTERNAL "Target platform: Linux")
            message("Target Platform: Linux")
        elseif(${CMAKE_SYSTEM_NAME} STREQUAL "Darwin")
            if(IOS)
                target_compile_definitions(Diligent-BuildSettings__LOG INTERFACE -DPLATFORM_IOS)
                set(PLATFORM_IOS TRUE CACHE INTERNAL "Target platform: iOS")
                message("Target Platform: iOS")
            else()
                target_compile_definitions(Diligent-BuildSettings__LOG INTERFACE -DPLATFORM_MACOS)
                set(PLATFORM_MACOS TRUE CACHE INTERNAL "Target platform: MacOS")
                message("Target Platform: MacOS")
            endif()
        elseif(${CMAKE_SYSTEM_NAME} STREQUAL "iOS")
            target_compile_definitions(Diligent-BuildSettings__LOG INTERFACE -DPLATFORM_IOS)
            set(PLATFORM_IOS TRUE CACHE INTERNAL "Target platform: iOS")
            message("Target Platform: iOS")
        elseif(${CMAKE_SYSTEM_NAME} STREQUAL "tvOS")
            target_compile_definitions(Diligent-BuildSettings__LOG INTERFACE -DPLATFORM_TVOS)
            set(PLATFORM_TVOS TRUE CACHE INTERNAL "Target platform: tvOS")
            message("Target Platform: tvOS")
        else()
            message(FATAL_ERROR "Unsupported platform")
        endif()
    endif(WIN32)

    if(PLATFORM_MACOS OR PLATFORM_IOS OR PLATFORM_TVOS)
        target_compile_definitions(Diligent-BuildSettings__LOG INTERFACE -DPLATFORM_APPLE -DGL_SILENCE_DEPRECATION)
        set(PLATFORM_APPLE TRUE CACHE INTERNAL "Apple platform (macOS, iOS, or tvOS)")
    endif()

    set(PLATFORM_HAS_BEEN_DEFINED TRUE "Platform has been defined")
endif()

foreach(DBG_CONFIG ${DEBUG_CONFIGURATIONS})
    target_compile_definitions(Diligent-BuildSettings__LOG INTERFACE "$<$<CONFIG:${DBG_CONFIG}>:_DEBUG;DEBUG>")
endforeach()

foreach(REL_CONFIG ${RELEASE_CONFIGURATIONS})
    target_compile_definitions(Diligent-BuildSettings__LOG INTERFACE "$<$<CONFIG:${REL_CONFIG}>:NDEBUG>")
endforeach()

if(MSVC)
    # For msvc, enable level 4 warnings and treat warnings as errors, except for
    # - w4100 - unreferenced formal parameter
    # - w4505 - unreferenced local function has been removed
    # - w4201 - nonstandard extension used: nameless struct/union
    target_compile_options(Diligent-BuildSettings__LOG INTERFACE /W4 /WX /wd4100 /wd4505 /wd4201 /MP)
    # In all release modes also:
    # - disable w4189 - local variable is initialized but not referenced
    # - Disable RTTI (/GR-)
    # - Enable whole program optimization (/GL)
    # - Enable string pooling (/GF)
    set(MSVC_ALL_RELEASE_COMPILE_OPTIONS /wd4189 /GR- /GL /GF)
    #target_compile_options(Diligent-BuildSettings__LOG INTERFACE "$<$<CONFIG:RELEASE>:/wd4189 /Ot")
    # In RELEASE mode:
    # - Set favor fast code option (/Ot)
    # - Enable intrinsic functions (/Oi)
    # - Maximize Speed (/O2)
    # - Inline any suitable function (/Ob2)
    set(MSVC_RELEASE_COMPILE_OPTIONS ${MSVC_ALL_RELEASE_COMPILE_OPTIONS} /Ot /Oi /Ob2 /O2)
    set(MSVC_RELWITHDEBINFO_COMPILE_OPTIONS ${MSVC_RELEASE_COMPILE_OPTIONS})
    # In MINSIZEREL mode set favor small code option (/Os)
    set(MSVC_MINSIZEREL_COMPILE_OPTIONS ${MSVC_ALL_RELEASE_COMPILE_OPTIONS} /Os)
    target_compile_options(Diligent-BuildSettings__LOG INTERFACE "$<$<CONFIG:RELEASE>:${MSVC_RELEASE_COMPILE_OPTIONS}>")
    target_compile_options(Diligent-BuildSettings__LOG INTERFACE "$<$<CONFIG:MINSIZEREL>:${MSVC_MINSIZEREL_COMPILE_OPTIONS}>")
    target_compile_options(Diligent-BuildSettings__LOG INTERFACE "$<$<CONFIG:RELWITHDEBINFO>:${MSVC_RELWITHDEBINFO_COMPILE_OPTIONS}>")
    # !!!NOTE!!! For some reason above is the only form of generator expression that works
    # For instance, this way
    # target_compile_options(Diligent-BuildSettings__LOG INTERFACE "$<$<CONFIG:RELEASE>:/Ot>")
    # does not work as expected
else()
    # Todo: use __attribute__((always_inline)), but it needs to be defined in a header file
    target_compile_definitions(Diligent-BuildSettings__LOG INTERFACE __forceinline=inline)
endif(MSVC)


if (CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR
        CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    if(PLATFORM_APPLE)
        set(WHOLE_ARCHIVE_FLAG "-Wl,-all_load" CACHE INTERNAL "all_load flag")
        set(NO_WHOLE_ARCHIVE_FLAG "-Wl,-noall_load" CACHE INTERNAL "noall_load flag")
    else()
        set(WHOLE_ARCHIVE_FLAG "-Wl,--whole-archive" CACHE INTERNAL "whole-archive flag")
        set(NO_WHOLE_ARCHIVE_FLAG "-Wl,--no-whole-archive" CACHE INTERNAL "no-whole-archive flag")
    endif()
else()
    set(WHOLE_ARCHIVE_FLAG "")
    set(NO_WHOLE_ARCHIVE_FLAG "")
endif()

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    target_compile_options(Diligent-BuildSettings__LOG INTERFACE
            # All warnings are errors
            -Werror
            # Some extra warnings
            -Wall -Wextra -Wuninitialized -Wconditional-uninitialized -Wextra-tokens -Wpointer-arith -Wloop-analysis -Wunused
            # Disable few warnings
            -Wno-overloaded-virtual -Wno-incompatible-pointer-types-discards-qualifiers -Wno-unknown-pragmas
            -Wno-zero-as-null-pointer-constant -Wno-unused-parameter
            )
    set(CLANG_RELEASE_OPTIONS -Wno-unused-variable)
    target_compile_options(Diligent-BuildSettings__LOG INTERFACE $<$<NOT:$<CONFIG:Debug>>:${CLANG_RELEASE_OPTIONS}>)

    if ((PLATFORM_IOS  AND CMAKE_OSX_SYSROOT STREQUAL "iphonesimulator")  OR PLATFORM_IOS_SIMULATOR OR
    (PLATFORM_TVOS AND CMAKE_OSX_SYSROOT STREQUAL "appletvsimulator") OR PLATFORM_TVOS_SIMULATOR)
        # There is a known long-standing issue in simulator SDK:
        # the compiler generates a lot of bogus warnings in Metal
        # headers about unavailable API
        target_compile_options(Diligent-BuildSettings__LOG
                INTERFACE
                -Wno-error=unguarded-availability-new
                )
    endif()
endif()

# Returns path to the target relative to CMake root
function(get_target_relative_dir__LOG _TARGET _DIR)
    get_target_property(TARGET_SOURCE_DIR ${_TARGET} SOURCE_DIR)
    file(RELATIVE_PATH TARGET_RELATIVE_PATH "${CMAKE_SOURCE_DIR}" "${TARGET_SOURCE_DIR}")
    set(${_DIR} ${TARGET_RELATIVE_PATH} PARENT_SCOPE)
endfunction()

# Performs installation steps for the core library
function(install_core_lib__LOG _TARGET)
    get_target_relative_dir__LOG(${_TARGET} TARGET_RELATIVE_PATH)

    get_target_property(TARGET_TYPE ${_TARGET} TYPE)
    if(TARGET_TYPE STREQUAL STATIC_LIBRARY)
        list(APPEND DILIGENT_CORE_INSTALL_LIBS_LIST ${_TARGET})
        set(DILIGENT_CORE_INSTALL_LIBS_LIST ${DILIGENT_CORE_INSTALL_LIBS_LIST} CACHE INTERNAL "Core libraries installation list")
    elseif(TARGET_TYPE STREQUAL SHARED_LIBRARY)
        install(TARGETS				 ${_TARGET}
                ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}/${DILIGENT_CORE_DIR}/$<CONFIG>"
                LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}/${DILIGENT_CORE_DIR}/$<CONFIG>"
                RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}/${DILIGENT_CORE_DIR}/$<CONFIG>"
                )
        if (DILIGENT_INSTALL_PDB)
            install(FILES $<TARGET_PDB_FILE:${_TARGET}> DESTINATION "${CMAKE_INSTALL_BINDIR}/${DILIGENT_CORE_DIR}/$<CONFIG>" OPTIONAL)
        endif()
    endif()

    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/interface")
        install(DIRECTORY    interface
                DESTINATION  "${CMAKE_INSTALL_INCLUDEDIR}/${TARGET_RELATIVE_PATH}/"
                )
    endif()
endfunction()

function(set_common_target_properties__LOG TARGET)

    if(COMMAND custom_pre_configure_target)
        custom_pre_configure_target(${TARGET})
        if(TARGET_CONFIGURATION_COMPLETE)
            return()
        endif()
    endif()

    get_target_property(TARGET_TYPE ${TARGET} TYPE)

    if(MSVC)
        # For msvc, enable link-time code generation for release builds (I was not able to
        # find any way to set these settings through interface library BuildSettings)
        if(TARGET_TYPE STREQUAL STATIC_LIBRARY)

            foreach(REL_CONFIG ${RELEASE_CONFIGURATIONS})
                set_target_properties(${TARGET} PROPERTIES
                        STATIC_LIBRARY_FLAGS_${REL_CONFIG} /LTCG
                        )
            endforeach()

        else()

            foreach(REL_CONFIG ${RELEASE_CONFIGURATIONS})
                set_target_properties(${TARGET} PROPERTIES
                        LINK_FLAGS_${REL_CONFIG} "/LTCG /OPT:REF /INCREMENTAL:NO"
                        )
            endforeach()

            if(PLATFORM_UNIVERSAL_WINDOWS)
                # On UWP, disable incremental link to avoid linker warnings
                foreach(DBG_CONFIG ${DEBUG_CONFIGURATIONS})
                    set_target_properties(${TARGET} PROPERTIES
                            LINK_FLAGS_${DBG_CONFIG} "/INCREMENTAL:NO"
                            )
                endforeach()
            endif()
        endif()
    else()
        set_target_properties(${TARGET} PROPERTIES
                CXX_VISIBILITY_PRESET hidden # -fvisibility=hidden
                C_VISIBILITY_PRESET hidden # -fvisibility=hidden
                VISIBILITY_INLINES_HIDDEN TRUE

                # Without -fPIC option GCC fails to link static libraries into dynamic library:
                #  -fPIC
                #      If supported for the target machine, emit position-independent code, suitable for
                #      dynamic linking and avoiding any limit on the size of the global offset table.
                POSITION_INDEPENDENT_CODE ON

                # It is crucial to set CXX_STANDARD flag to only affect c++ files and avoid failures compiling c-files:
                # error: invalid argument '-std=c++14' not allowed with 'C/ObjC'
                CXX_STANDARD 14
                CXX_STANDARD_REQUIRED ON

                C_STANDARD 11
                )

        if(NOT MINGW_BUILD)
            # Do not disable extensions when building with MinGW!
            set_target_properties(${TARGET} PROPERTIES
                    CXX_EXTENSIONS OFF
                    )
        endif()
    endif()

    if(COMMAND custom_post_configure_target)
        custom_post_configure_target(${TARGET})
    endif()

endfunction()

if(PLATFORM_MACOS)
    find_library(APP_KIT AppKit)
    if (NOT APP_KIT)
        message(FATAL_ERROR "AppKit not found")
    endif()
elseif(PLATFORM_IOS)
    find_library(CORE_FOUNDATION CoreFoundation)
    if(NOT CORE_FOUNDATION)
        message(FATAL_ERROR "Cannot find CoreFoundation framework")
    endif()

    find_library(FOUNDATION Foundation)
    if(NOT FOUNDATION)
        message(FATAL_ERROR "Cannot find Foundation framework")
    endif()
elseif(PLATFORM_TVOS)
    find_library(CORE_FOUNDATION CoreFoundation)
    if(NOT CORE_FOUNDATION)
        message(FATAL_ERROR "Cannot find CoreFoundation framework")
    endif()

    find_library(FOUNDATION Foundation)
    if(NOT FOUNDATION)
        message(FATAL_ERROR "Cannot find Foundation framework")
    endif()
endif()

add_subdirectory(Primitives)
add_subdirectory(Platforms)

add_library(
        Diligent-Log SHARED
        common.cpp
)

target_link_libraries(
        Diligent-Log
        Diligent-Primitives
        Diligent-TargetPlatform
)
