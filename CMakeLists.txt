cmake_minimum_required(VERSION 3.16...3.31)
project(kiss_matcher VERSION 0.3.0 LANGUAGES CXX)

# Only show the ASCII art if this is the top-level project
if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
    message("")
    message("██╗  ██╗██╗███████╗███████╗      ")
    message("██║ ██╔╝██║██╔════╝██╔════╝      ")
    message("█████╔╝ ██║███████╗███████╗█████╗")
    message("██╔═██╗ ██║╚════██║╚════██║╚════╝")
    message("██║  ██╗██║███████║███████║      ")
    message("╚═╝  ╚═╝╚═╝╚══════╝╚══════╝      ")
    message("")
    message("███╗   ███╗ █████╗ ████████╗ ██████╗██╗  ██╗███████╗██████╗ ")
    message("████╗ ████║██╔══██╗╚══██╔══╝██╔════╝██║  ██║██╔════╝██╔══██╗")
    message("██╔████╔██║███████║   ██║   ██║     ███████║█████╗  ██████╔╝")
    message("██║╚██╔╝██║██╔══██║   ██║   ██║     ██╔══██║██╔══╝  ██╔══██╗")
    message("██║ ╚═╝ ██║██║  ██║   ██║   ╚██████╗██║  ██║███████╗██║  ██║")
    message("╚═╝     ╚═╝╚═╝  ╚═╝   ╚═╝    ╚═════╝╚═╝  ╚═╝╚══════╝╚═╝  ╚═╝")
    message("")
endif()

# Options with CACHE to allow parent project to override
option(KISS_MATCHER_USE_CCACHE "Build using Ccache if found on the path" ON)
option(KISS_MATCHER_USE_SYSTEM_EIGEN3 "Use system pre-installed Eigen" ON)
option(KISS_MATCHER_USE_SYSTEM_TBB "Use system pre-installed oneAPI/tbb" OFF)
option(KISS_MATCHER_USE_SYSTEM_ROBIN "Use system pre-installed ROBIN from SPARK @ MIT" ON)
option(KISS_MATCHER_USE_ADDRESS_SANITIZER "Use address sanitizer" OFF)
option(KISS_MATCHER_BUILD_TESTS "Build tests" OFF)
option(KISS_MATCHER_INSTALL "Install KISS-Matcher" ${PROJECT_IS_TOP_LEVEL})

# Set a default build type if none is specified (only if top-level)
if(PROJECT_IS_TOP_LEVEL AND NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    message(STATUS "Setting build type to 'Release' as none was specified.")
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build." FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

if(KISS_MATCHER_USE_ADDRESS_SANITIZER)
    add_compile_options(-fsanitize=address -fsanitize-recover=address)
    add_link_options(-fsanitize=address -fsanitize-recover=address)
endif()

include(GNUInstallDirs)

# Include dependencies - ensure this file uses KISS_MATCHER_ prefixed options
include(3rdparty/find_dependencies.cmake)
include(cmake/CompilerOptions.cmake)

# NOTE(hlim): Without this line, pybinded KISS-Matcher does not work
find_library(LZ4_LIBRARY lz4 REQUIRED)

include(FindOpenMP)
if(OPENMP_FOUND)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
else()
    message(WARNING "OpenMP could not be found. Performance may be degraded.")
endif()

set(CMAKE_CXX_FLAGS_RELEASE "-O3 -Wall ${CMAKE_CXX_FLAGS}")

# Library target
set(TARGET_NAME kiss_matcher_core)
add_library(${TARGET_NAME} STATIC
    core/kiss_matcher/ROBINMatching.cpp
    core/kiss_matcher/FasterPFH.cpp
    core/kiss_matcher/KISSMatcher.cpp
    core/kiss_matcher/GncSolver.cpp
)

# Create alias for consistent usage (works with FetchContent and find_package)
add_library(kiss_matcher::kiss_matcher_core ALIAS ${TARGET_NAME})

target_link_libraries(${TARGET_NAME}
    PUBLIC 
        Eigen3::Eigen 
        robin::robin 
        TBB::tbb
    PRIVATE
        ${LZ4_LIBRARY}
)

if(OPENMP_FOUND)
    target_link_libraries(${TARGET_NAME} PUBLIC OpenMP::OpenMP_CXX)
endif()

# Set global target properties if function exists (from CompilerOptions.cmake)
if(COMMAND set_global_target_properties)
    set_global_target_properties(${TARGET_NAME})
endif()

target_include_directories(${TARGET_NAME}
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/core>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

# Only install if requested (typically false when used via FetchContent)
if(KISS_MATCHER_INSTALL)
    install(TARGETS ${TARGET_NAME}
        EXPORT ${PROJECT_NAME}-targets
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    )

    install(EXPORT ${PROJECT_NAME}-targets
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
        NAMESPACE kiss_matcher::
        FILE ${PROJECT_NAME}Targets.cmake
    )

    install(DIRECTORY core/kiss_matcher 
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    )

    # Install config file if it exists
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/cmake/${PROJECT_NAME}Config.cmake)
        install(FILES cmake/${PROJECT_NAME}Config.cmake
            DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
        )
    endif()
endif()