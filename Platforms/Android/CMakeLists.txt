cmake_minimum_required (VERSION 3.6)

project(Diligent-AndroidPlatform_LOG CXX)

set(INTERFACE 
    interface/AndroidDebug.hpp
    interface/AndroidFileSystem.hpp
    interface/AndroidPlatformDefinitions.h
    interface/AndroidPlatformMisc.hpp
    interface/AndroidNativeWindow.h
)

set(SOURCE 
    src/AndroidDebug.cpp
    src/AndroidFileSystem.cpp
)

add_library(Diligent-AndroidPlatform_LOG ${SOURCE} ${INTERFACE} ${PLATFORM_INTERFACE_HEADERS})
set_common_target_properties__LOG(Diligent-AndroidPlatform_LOG)

target_include_directories(Diligent-AndroidPlatform_LOG
PUBLIC
    interface
)

target_link_libraries(Diligent-AndroidPlatform_LOG
PRIVATE
    log
    android
    Diligent-BuildSettings__LOG 
PUBLIC
    Diligent-PlatformInterface
    Diligent-BasicPlatform_LOG
)

source_group("src" FILES ${SOURCE})
source_group("include" FILES ${INCLUDE})
source_group("interface" FILES ${PLATFORM_INTERFACE_HEADERS})

set_target_properties(Diligent-AndroidPlatform_LOG PROPERTIES
    FOLDER DiligentLog/Platforms
)

if(DILIGENT_INSTALL_CORE)
    install_core_lib__LOG(Diligent-AndroidPlatform_LOG)
endif()