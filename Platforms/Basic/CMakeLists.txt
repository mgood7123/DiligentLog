cmake_minimum_required (VERSION 3.6)

project(Diligent-BasicPlatform_LOG CXX)

set(SOURCE 
    src/BasicFileSystem.cpp
    src/BasicPlatformDebug.cpp
)

set(INTERFACE 
    interface/BasicAtomics.hpp
    interface/BasicFileSystem.hpp
    interface/BasicPlatformDebug.hpp
    interface/BasicPlatformMisc.hpp
    interface/DebugUtilities.hpp
)

if(PLATFORM_LINUX OR PLATFORM_WIN32 OR PLATFORM_APPLE)
    list(APPEND SOURCE src/StandardFile.cpp)
    list(APPEND INTERFACE interface/StandardFile.hpp)
endif()

add_library(Diligent-BasicPlatform_LOG STATIC ${SOURCE} ${INTERFACE})
set_common_target_properties__LOG(Diligent-BasicPlatform_LOG)

target_include_directories(Diligent-BasicPlatform_LOG
PUBLIC
    interface
)

target_link_libraries(Diligent-BasicPlatform_LOG
PRIVATE
    Diligent-BuildSettings 
PUBLIC
    Diligent-Primitives
)

source_group("src" FILES ${SOURCE})
source_group("interface" FILES ${INTERFACE})

set_target_properties(Diligent-BasicPlatform_LOG PROPERTIES
    FOLDER DiligentLog/Platforms
)


if(DILIGENT_INSTALL_CORE)
    install_core_lib__LOG(Diligent-BasicPlatform_LOG)
    get_target_relative_dir__LOG(Diligent-BasicPlatform_LOG BASIC_PLATFORM_DIR)
    install(DIRECTORY    "../interface"
            DESTINATION  "${CMAKE_INSTALL_INCLUDEDIR}/${BASIC_PLATFORM_DIR}/../"
    )
endif()
