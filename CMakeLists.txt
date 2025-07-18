cmake_minimum_required(VERSION 3.25)

include(FetchContent)

project(imgui-glfw-glad-glm VERSION 4.1.0)

set(CMAKE_CXX_STANDARD 23)

# Explicitly set ImGui to use the docking branch
set(IGG_IMGUI_TAG "docking" CACHE STRING "ImGui branch/tag to use" FORCE)

set(IGG_COMPONENTS "IMGUI;GLFW;GLAD;GLM" CACHE STRING "The components to import and build")

foreach (comp IN ITEMS ${IGG_COMPONENTS})
    set(IMPORT_${comp} ON)
endforeach ()

if (IMPORT_IMGUI)
    set(IMPORT_GLAD ON)
    set(IMPORT_GLFW ON)
endif ()

if (IMPORT_GLAD)
    option(IGG_GLAD_DOWNLOAD "If set to ON the glad gl loader will be generated and downloaded, if OFF the included version (gl4.5/core) will be used" ON)
    if (IGG_GLAD_DOWNLOAD)
        set(IGG_GLAD_GL_VERSION 4.5 CACHE STRING "The target gl version")
        option(IGG_GLAD_GL_CORE "The target gl profile. ON = core profile, OFF = compatibility profile" ON)
        if (IGG_GLAD_GL_CORE)
            set(GLAD_GL_PROFILE core)
        else ()
            set(GLAD_GL_PROFILE compatibility)
        endif ()
    else ()
        unset(IGG_GLAD_GL_VERSION CACHE)
        unset(IGG_GLAD_GL_CORE CACHE)
    endif ()
else ()
    unset(IGG_GLAD_DOWNLOAD CACHE)
    unset(IGG_GLAD_GL_VERSION CACHE)
    unset(IGG_GLAD_GL_CORE CACHE)
endif ()
if (IMPORT_IMGUI)
    set(IGG_IMGUI_TAG docking CACHE STRING "Defines the imgui version (e.g. docking, v1.89.4, v1.89.3, ...")
    set(IMPORT_GLFW ON)
else ()
    unset(IGG_IMGUI_TAG CACHE)
endif ()
if (IMPORT_GLFW)
    set(IGG_GLFW_TAG master CACHE STRING "Defines the glfw version (e.g. master, 3.3.8, 3.3.7, ...) ")
else ()
    unset(IGG_GLFW_TAG CACHE)
endif ()
if (IMPORT_GLM)
    set(IGG_GLM_TAG master CACHE STRING "Defines the glm version (e.g. master, 0.9.9.8, 0.9.9.7, ...")
else ()
    unset(IGG_GLM_TAG CACHE)
endif ()

##############################
#            glad            #
##############################
if (IMPORT_GLAD)
    if (NOT IGG_GLAD_DOWNLOAD)
        message(STATUS "Using included version of the glad loader sources (gl 4.5/core) ")
        set(glad_SOURCE_DIR glad)
    else ()
        if ("${glad_INSTALLED_VERSION}" STREQUAL "${IGG_GLAD_GL_VERSION}-${GLAD_GL_PROFILE}")
            message(STATUS "Avoiding repeated download of glad gl ${IGG_GLAD_GL_VERSION}/${GLAD_GL_PROFILE}")
            set(glad_SOURCE_DIR ${glad_LAST_SOURCE_DIR})
        else ()
            find_program(IGG_CURL NAMES curl curl.exe)
            if (NOT IGG_CURL)
                message(STATUS "Could not find curl, using included version of the glad loader sources (gl 4.5/core)")
                set(glad_SOURCE_DIR glad)
            else ()
                execute_process(
                        COMMAND ${IGG_CURL} -s -D - -X POST -d generator=c&api=egl%3Dnone&api=gl%3D${IGG_GLAD_GL_VERSION}&profile=gl%3D${GLAD_GL_PROFILE}&api=gles1%3Dnone&profile=gles1%3Dcommon&api=gles2%3Dnone&api=glsc2%3Dnone&api=glx%3Dnone&api=vulkan%3Dnone&api=wgl%3Dnone&options=LOADER https://gen.glad.sh/generate
                        OUTPUT_VARIABLE out
                        RESULT_VARIABLE res
                        ERROR_VARIABLE err
                )
                if (NOT res EQUAL "0")
                    message(WARNING "${IGG_CURL} returned: " ${res})
                    if (err)
                        message(WARNING "Error message: " ${err})
                    endif ()
                    message(STATUS "Using included version of the glad loader sources (gl 4.5/core)")
                    set(glad_SOURCE_DIR glad)
                else ()
                    string(REGEX MATCH "[Ll][Oo][Cc][Aa][Tt][Ii][Oo][Nn]: ([A-Za-z0-9_\\:/\\.]+)" location ${out})
                    set(location "${CMAKE_MATCH_1}")
                    if (NOT location OR location STREQUAL "/")
                        message(WARNING "Could not extract location from http response, using included version of the glad loader sources (gl 4.5/core)")
                        message(STATUS "Response: " ${out})
                        set(glad_SOURCE_DIR glad)
                    else ()
                        string(REGEX REPLACE "/$" "" location ${location})
                        string(APPEND location "/glad.zip")
                        if (NOT ${location} MATCHES "^http")
                            string(PREPEND location "https://gen.glad.sh")
                        endif ()
                        message(STATUS "Downloading glad loader sources for gl${IGG_GLAD_GL_VERSION}/${GLAD_GL_PROFILE} from ${location}")
                        FetchContent_Declare(
                                glad
                                URL ${location}
                        )
                        FetchContent_MakeAvailable(glad)
                        set(glad_INSTALLED_VERSION ${IGG_GLAD_GL_VERSION}-${GLAD_GL_PROFILE} CACHE INTERNAL "")
                        set(glad_LAST_SOURCE_DIR ${glad_SOURCE_DIR} CACHE INTERNAL "")
                    endif ()
                endif ()
            endif ()
        endif ()
    endif ()

    add_library(
            glad
            ${glad_SOURCE_DIR}/src/gl.c
            ${glad_SOURCE_DIR}/include/glad/gl.h
            ${glad_SOURCE_DIR}/include/KHR/khrplatform.h
    )
    target_include_directories(glad PUBLIC ${glad_SOURCE_DIR}/include)
endif ()

if (IMPORT_GLFW)
    if (NOT EMSCRIPTEN)
        FetchContent_Declare(
                glfw
                GIT_REPOSITORY https://github.com/glfw/glfw.git
                GIT_TAG ${IGG_GLFW_TAG}
        )
        list(APPEND components glfw)
    endif ()
endif ()

if (IMPORT_GLM)
    FetchContent_Declare(
            glm
            GIT_REPOSITORY https://github.com/g-truc/glm.git
            GIT_TAG ${IGG_GLM_TAG}
    )
    list(APPEND components glm)
endif ()

if (IMPORT_IMGUI)
    FetchContent_Declare(
            imgui
            GIT_REPOSITORY https://github.com/ocornut/imgui.git
            GIT_TAG ${IGG_IMGUI_TAG}
    )
    list(APPEND components imgui)
endif ()

if (components)
    FetchContent_MakeAvailable(${components})
endif ()

##############################
#            imgui           #
##############################
if (IMPORT_IMGUI)
    file(GLOB imgui_SRC ${imgui_SOURCE_DIR}/*.cpp ${imgui_SOURCE_DIR}/*.h)
    add_library(
            imgui
            ${imgui_SRC}
            ${imgui_SOURCE_DIR}/backends/imgui_impl_opengl3.cpp
            ${imgui_SOURCE_DIR}/backends/imgui_impl_opengl3.h
            ${imgui_SOURCE_DIR}/backends/imgui_impl_glfw.cpp
            ${imgui_SOURCE_DIR}/backends/imgui_impl_glfw.h
    )

    target_link_libraries(imgui PUBLIC glad)
    if (NOT EMSCRIPTEN)
        target_link_libraries(imgui PUBLIC glfw)
    endif ()
    target_include_directories(
            imgui
            PUBLIC
            ${imgui_SOURCE_DIR}
            ${imgui_SOURCE_DIR}/backends
    )
endif ()
