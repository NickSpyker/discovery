cmake_minimum_required(VERSION 3.28)

project(discovery
    VERSION 0.1.0
    DESCRIPTION "TODO"
    HOMEPAGE_URL "TODO"
    LANGUAGES C CXX)

if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif ()

message(VERBOSE "*")
message(VERBOSE "* ${PROJECT_NAME} v${PROJECT_VERSION} ${CMAKE_BUILD_TYPE}")
message(VERBOSE "* Copyright (c) 2024 Nicolas Spijkerman <nicolas.spijkerman@gmail.com>")
message(VERBOSE "*")

if (CMAKE_BUILD_TYPE STREQUAL Debug)
    add_compile_definitions(DEBUG)
elseif (CMAKE_BUILD_TYPE STREQUAL Release)
    add_compile_definitions(RELEASE)
endif ()

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(FetchContent)

# ----------------------------------------------------------------------------------------------------
# FMT : https://github.com/fmtlib/fmt
FetchContent_Declare(fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG master)
FetchContent_MakeAvailable(fmt)

# ----------------------------------------------------------------------------------------------------
# EnTT : https://github.com/skypjack/entt
FetchContent_Declare(EnTT
    GIT_REPOSITORY https://github.com/skypjack/entt.git
    GIT_TAG master)
FetchContent_MakeAvailable(EnTT)

# ----------------------------------------------------------------------------------------------------
# GLM : https://github.com/g-truc/glm
FetchContent_Declare(glm
    GIT_REPOSITORY https://github.com/g-truc/glm.git
    GIT_TAG master)
FetchContent_MakeAvailable(glm)

# ----------------------------------------------------------------------------------------------------
# GLFW : https://github.com/glfw/glfw
FetchContent_Declare(glfw
    GIT_REPOSITORY https://github.com/glfw/glfw.git
    GIT_TAG master)
FetchContent_MakeAvailable(glfw)

# ----------------------------------------------------------------------------------------------------
# GLAD : https://github.com/Dav1dde/glad
set(GLAD_GL_VERSION 4.5 CACHE STRING "Target GL version")
option(GLAD_GL_CORE "Target GL profile : ON = core profile, OFF = compatibility profile" ON)

if (GLAD_GL_CORE)
    set(GLAD_GL_PROFILE core)
else ()
    set(GLAD_GL_PROFILE compatibility)
endif ()

if ("${glad_INSTALLED_VERSION}" STREQUAL "${GLAD_GL_VERSION}-${GLAD_GL_PROFILE}")
    message(STATUS "Avoiding repeated download of GLAD GL ${GLAD_GL_VERSION}/${GLAD_GL_PROFILE}")
    set(glad_SOURCE_DIR ${glad_LAST_SOURCE_DIR})
else ()
    find_program(CURL NAMES curl curl.exe)
    if (NOT CURL)
        message(FATAL_ERROR "Could not find curl, please install curl and try again")
    endif ()

    execute_process(
        COMMAND ${CURL} -s -D - -X POST -d generator=c&api=egl%3Dnone&api=gl%3D${GLAD_GL_VERSION}&profile=gl%3D${GLAD_GL_PROFILE}&api=gles1%3Dnone&profile=gles1%3Dcommon&api=gles2%3Dnone&api=glsc2%3Dnone&api=glx%3Dnone&api=vulkan%3Dnone&api=wgl%3Dnone&options=LOADER https://gen.glad.sh/generate
        OUTPUT_VARIABLE out
        RESULT_VARIABLE res
        ERROR_VARIABLE err)

    if (NOT res EQUAL "0")
        if (err)
            message(FATAL_ERROR "Error : " ${err})
        endif ()
        message(FATAL_ERROR "${CURL} : " ${res})
    endif ()

    string(REGEX MATCH "[Ll][Oo][Cc][Aa][Tt][Ii][Oo][Nn]: ([A-Za-z0-9_\\:/\\.]+)" location ${out})
    set(location "${CMAKE_MATCH_1}")
    if (NOT location OR location STREQUAL "/")
        message(FATAL_ERROR "Could not extract location from http response : " ${out})
    endif ()

    string(REGEX REPLACE "/$" "" location ${location})
    string(APPEND location "/glad.zip")
    if (NOT ${location} MATCHES "^http")
        string(PREPEND location "https://gen.glad.sh")
    endif ()

    message(STATUS "Downloading glad loader sources for gl${GLAD_GL_VERSION}/${GLAD_GL_PROFILE} from ${location}")
    FetchContent_Declare(glad URL ${location})
    FetchContent_MakeAvailable(glad)
    set(glad_INSTALLED_VERSION ${GLAD_GL_VERSION}-${GLAD_GL_PROFILE} CACHE INTERNAL "")
    set(glad_LAST_SOURCE_DIR ${glad_SOURCE_DIR} CACHE INTERNAL "")
endif ()

add_library(glad
    ${glad_SOURCE_DIR}/src/gl.c
    ${glad_SOURCE_DIR}/include/glad/gl.h
    ${glad_SOURCE_DIR}/include/KHR/khrplatform.h)

target_include_directories(glad PUBLIC ${glad_SOURCE_DIR}/include)

# ----------------------------------------------------------------------------------------------------
# ImGui : https://github.com/ocornut/imgui
FetchContent_Declare(imgui
    GIT_REPOSITORY https://github.com/ocornut/imgui.git
    GIT_TAG master)
FetchContent_MakeAvailable(imgui)

file(GLOB imgui_SRC ${imgui_SOURCE_DIR}/*.cpp ${imgui_SOURCE_DIR}/*.h)
add_library(imgui
    ${imgui_SRC}
    ${imgui_SOURCE_DIR}/backends/imgui_impl_opengl3.cpp
    ${imgui_SOURCE_DIR}/backends/imgui_impl_opengl3.h
    ${imgui_SOURCE_DIR}/backends/imgui_impl_glfw.cpp
    ${imgui_SOURCE_DIR}/backends/imgui_impl_glfw.h)

target_link_libraries(imgui PUBLIC glad glfw)
target_include_directories(imgui PUBLIC ${imgui_SOURCE_DIR} ${imgui_SOURCE_DIR}/backends)

# ----------------------------------------------------------------------------------------------------
# Project Configuration
add_library(${PROJECT_NAME} STATIC
    src/discovery.h)

target_link_libraries(${PROJECT_NAME} fmt EnTT glm glfw glad imgui)
