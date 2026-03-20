cmake_minimum_required(VERSION 3.13)
project(speederv2 LANGUAGES CXX)

# Generate compile_commands.json
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Generate git_version.h (matching makefile's 'git_version' target)
find_package(Git QUIET)
if(GIT_FOUND)
    execute_process(
        COMMAND ${GIT_EXECUTABLE} rev-parse HEAD
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE GIT_COMMIT_HASH
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
        RESULT_VARIABLE GIT_RESULT
    )
endif()
if(NOT GIT_FOUND OR GIT_RESULT)
    set(GIT_COMMIT_HASH "unknown")
endif()
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/git_version.h
    "const char *gitversion = \"${GIT_COMMIT_HASH}\";\n")

# Source files (matching makefile's SOURCES0 + my_ev.cpp)
set(SOURCE_FILES
    main.cpp
    log.cpp
    common.cpp
    lib/fec.cpp
    lib/rs.cpp
    crc32/Crc32.cpp
    packet.cpp
    delay_manager.cpp
    fd_manager.cpp
    connection.cpp
    fec_manager.cpp
    misc.cpp
    tunnel_client.cpp
    tunnel_server.cpp
    my_ev.cpp
)

add_executable(speederv2 ${SOURCE_FILES})

# Include directories (matching makefile's -I. and -isystem libev)
target_include_directories(speederv2 SYSTEM PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/libev)
target_include_directories(speederv2 PRIVATE . ${CMAKE_CURRENT_BINARY_DIR})

# Compiler warning flags (matching makefile FLAGS)
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    target_compile_options(speederv2 PRIVATE
        -Wall -Wextra
        -Wno-unused-variable -Wno-unused-parameter -Wno-missing-field-initializers
    )
endif()

# Build configuration options
option(SPEEDERV2_STATIC "Build with static linking (matching makefile)" ON)

# Platform-specific configuration (matching makefile targets)
if(MINGW)
    # Matching makefile's 'mingw' target
    target_link_libraries(speederv2 PRIVATE ws2_32)
elseif(APPLE)
    # Matching makefile's 'mac' target: no -lrt, no -static
elseif(CYGWIN)
    # Matching makefile's 'cygwin' target
    target_link_libraries(speederv2 PRIVATE rt)
    target_compile_definitions(speederv2 PRIVATE _GNU_SOURCE)
else()
    # Linux / FreeBSD (matching makefile's 'all' / 'freebsd' targets)
    target_link_libraries(speederv2 PRIVATE rt)
endif()

if(SPEEDERV2_STATIC AND NOT APPLE)
    target_link_options(speederv2 PRIVATE -static)
endif()
