cmake_minimum_required(VERSION 3.15 FATAL_ERROR)

# with at least one base tag, this can have dynamic (git) versioning:
#  git describe last tag = 1.1.0
#  git describe last tag + rev count = 1.1.0.45
#
# for now, set -DSCM_VERSION_INFO in the build env to override
if(NOT SCM_VERSION_INFO)
    set(LIBRARY_VERSION 1.1.0)
    set(SCM_VERSION_INFO ${LIBRARY_VERSION})
endif()
set(LIBRARY_SOVERSION 1)

project(
    abc
    LANGUAGES C CXX
    VERSION ${SCM_VERSION_INFO}
)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_VERBOSE_MAKEFILE ON)

# Default c++ standard used unless otherwise specified in target_compile_features.
set(CMAKE_CXX_STANDARD 17 CACHE STRING "the C++ standard to use for this project")
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(CMAKE_CXX_COMPILER_ID STREQUAL Clang)
    set(CLANG_DEFAULT_CXX_STDLIB libc++)
    set(CLANG_DEFAULT_RTLIB compiler-rt)
endif()

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE
        "RelWithDebInfo"
        CACHE STRING "Default build type: RelWithDebInfo" FORCE
    )
endif()

include(CMakeParseArguments)
include(CheckCCompilerFlag)
include(CheckCXXCompilerFlag)
include(GNUInstallDirs)

# LTO requires cmake min version 3.9 and clang LTO requires lld
if(ABC_ENABLE_LTO AND CMAKE_BUILD_TYPE MATCHES "Release")
    if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
        find_package(LLVM CONFIG)
    endif()

    include(CheckIPOSupported)
    check_ipo_supported(RESULT ipo_supported OUTPUT error)

    if(ipo_supported)
        message(STATUS "IPO / LTO supported")
        set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
    else()
        message(WARNING "IPO / LTO not supported: <${error}>")
    endif()
endif()

function(addprefix var prefix)
    foreach( s ${ARGN} )
        list(APPEND tmp "-I${s}")
    endforeach()
    set(${var} ${tmp} PARENT_SCOPE)
endfunction()

# filter out flags that are not appropriate for the compiler being used
function(target_compile_options_filtered target visibility)
    foreach( flag ${ARGN} )
        if( flag MATCHES "^-D.*" )
            target_compile_options( ${target} ${visibility} ${flag} )
        else()
            check_c_compiler_flag( ${flag} C_COMPILER_SUPPORTS__${flag} )
            if( C_COMPILER_SUPPORTS__${flag} )
                target_compile_options( ${target} ${visibility} $<$<COMPILE_LANGUAGE:C>:${flag}> )
            endif()

            check_cxx_compiler_flag( ${flag} CXX_COMPILER_SUPPORTS__${flag} )
            if( CXX_COMPILER_SUPPORTS__${flag} )
                target_compile_options( ${target} ${visibility} $<$<COMPILE_LANGUAGE:CXX>:${flag}>  )
            endif()
        endif()
    endforeach()
endfunction()

project(abc)

if(ABC_USE_NAMESPACE)
    set(ABC_USE_NAMESPACE_FLAGS "ABC_USE_NAMESPACE=${ABC_USE_NAMESPACE}")
endif()

if( APPLE )
    set(make_env ${CMAKE_COMMAND} -E env SDKROOT=${CMAKE_OSX_SYSROOT})
endif()

# run make to extract compiler options, linker options and list of source files
execute_process(
  COMMAND
    ${make_env}
    make
        ${ABC_READLINE_FLAGS}
        ${ABC_USE_NAMESPACE_FLAGS}
        ARCHFLAGS_EXE=${CMAKE_CURRENT_BINARY_DIR}/abc_arch_flags_program.exe
        ABC_MAKE_NO_DEPS=1
        CC=${CMAKE_C_COMPILER}
        CXX=${CMAKE_CXX_COMPILER}
        LD=${CMAKE_CXX_COMPILER}
        cmake_info
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  OUTPUT_VARIABLE MAKE_OUTPUT
)

# extract options from make output
function(extract_var SEPARATOR DEST_VARIABLE MAKE_OUTPUT)
  string(REGEX MATCH "${SEPARATOR} .* ${SEPARATOR}" TMP "${MAKE_OUTPUT}")
  string(REGEX REPLACE "${SEPARATOR} (.*) ${SEPARATOR}" "\\1" TMP "${TMP}")

  separate_arguments(TMP)

  set(${DEST_VARIABLE} ${TMP} PARENT_SCOPE)
endfunction()

extract_var(SEPARATOR_SRC ABC_SRC ${MAKE_OUTPUT})
extract_var(SEPARATOR_LIBS ABC_LIBS ${MAKE_OUTPUT})
extract_var(SEPARATOR_CFLAGS ABC_CFLAGS ${MAKE_OUTPUT})
extract_var(SEPARATOR_CXXFLAGS ABC_CXXFLAGS ${MAKE_OUTPUT})

set(ABC_HEADERS
    src/base/main/abcapis_old.h
    src/base/main/abcapis.h
)

string(REPLACE "src/" "${CMAKE_CURRENT_SOURCE_DIR}/src/" ABC_HEADERS
               "${ABC_HEADERS}"
)

# Cache variable so this can be used in parent projects
set(abc_INCLUDE_DIR
    "${CMAKE_CURRENT_LIST_DIR}/src"
    CACHE INTERNAL "Directory where common headers are located"
)

if(ABC_USE_NAMESPACE)
    set_source_files_properties(${ABC_SRC} PROPERTIES LANGUAGE CXX)  # generates deprecation warnings
    set(IAM_CPP "-x c++")
    add_compile_options($<$<COMPILE_LANG_AND_ID:C,AppleClang,Clang>:"${IAM_CPP}">)  # silence deprecation warnings
endif()

function(abc_properties target visibility)
    target_include_directories(
        ${target} ${visibility} $<BUILD_INTERFACE:${abc_INCLUDE_DIR}>
                                $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
    )
    target_compile_options_filtered(${target} ${visibility} ${ABC_CFLAGS} ${ABC_CXXFLAGS} -Wno-unused-but-set-variable )
    target_link_libraries(${target} ${visibility} ${ABC_LIBS})

    if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU" OR CMAKE_CXX_COMPILER_ID STREQUAL "Clang" OR CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang")
        target_compile_options(${target}
            ${visibility}
                -Wall -Wno-unused-function -Wno-write-strings -Wno-sign-compare -Wno-deprecated
        )
        target_link_libraries(${target} ${visibility} m)
        if(WIN32 OR ABC_USE_NAMESPACE)
            target_compile_options(${target}
                ${visibility}
                    $<$<COMPILE_LANGUAGE:CXX>:-fvisibility=hidden>
            )
        endif()
    elseif(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
        target_compile_definitions(${target}
            ${visibility}
                "_CONSOLE"
                "_MBCS"
                "_SCL_SECURE_NO_WARNINGS"
                "_CRT_SECURE_NO_WARNINGS"
                "_XKEYCHECK_H"
        )
    endif()

    if(WIN32)
        if(MINGW OR MSYS)
            add_definitions(-DWIN32 -D__MINGW32__ -DHAVE_STRUCT_TIMESPEC)
            if(DEFINED ENV{CONDA_PREFIX})
                include_directories($ENV{CONDA_PREFIX}/Library/include)
            endif()
        endif()
        target_compile_definitions(${target}
            ${visibility}
                $<$<BOOL:${ABC_USE_NO_PTHREADS}>:HAVE_STRUCT_TIMESPEC>
                $<$<BOOL:${BUILD_SHARED_LIBS}>:ABC_DLL=ABC_DLLEXPORT>
                $<$<NOT:$<BOOL:${BUILD_SHARED_LIBS}>>:WIN32_NO_DLL>
                $<$<NOT:$<BOOL:${BUILD_SHARED_LIBS}>>:ABC_NO_DYNAMIC_LINKING>
        )
        target_link_libraries(${target} ${visibility} shlwapi)
    endif()

    if(ABC_USE_LIBSTDCXX OR NOT ABC_USE_NAMESPACE)
        if(NOT CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
            message(STATUS "Using explicit -lstdc++")
            target_link_libraries(${target} ${visibility} stdc++)
        endif()
    endif()

    if(NOT ABC_USE_NO_READLINE)
        message(STATUS "Using libreadline")
        target_compile_definitions(${target} ${visibility} ABC_USE_READLINE)
        find_path(ABC_READLINE_INCLUDES
            NAMES readline.h
            HINTS
                $ENV{READLINE_ROOT}
            PATH_SUFFIXES include include/readline
        )
        find_library(ABC_READLINE_LIBRARIES
            NAMES readline libreadline
            HINTS
                $ENV{READLINE_ROOT}
        )
        if(NOT ABC_READLINE_INCLUDES OR NOT ABC_READLINE_LIBRARIES)
            message(FATAL_ERROR "Could not find readline")
        endif()
        target_include_directories(${target} ${visibility} ${ABC_READLINE_INCLUDES})
        target_link_libraries(${target} ${visibility} ${ABC_READLINE_LIBRARIES})
    endif()
endfunction()

set(ABC_MAIN_SRC src/base/main/main.c)
list(REMOVE_ITEM ABC_SRC ${ABC_MAIN_SRC})

add_library(libabc EXCLUDE_FROM_ALL ${ABC_SRC})
abc_properties(libabc PUBLIC)
set_property(TARGET libabc PROPERTY OUTPUT_NAME abc)

add_executable(abc ${ABC_MAIN_SRC})
target_link_libraries(abc PRIVATE libabc)
abc_properties(abc PRIVATE)

add_library(libabc-pic EXCLUDE_FROM_ALL ${ABC_SRC})
abc_properties(libabc-pic PUBLIC)
set_property(TARGET libabc-pic PROPERTY POSITION_INDEPENDENT_CODE ON)
set_property(TARGET libabc-pic PROPERTY OUTPUT_NAME abc-pic)

if(BUILD_SHARED_LIBS)
    if(ABC_USE_SONAME)
        set_target_properties(
            libabc PROPERTIES VERSION ${LIBRARY_VERSION}
                              SOVERSION ${LIBRARY_SOVERSION}
                              WINDOWS_EXPORT_ALL_SYMBOLS TRUE
        )
    endif()
endif()

if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    if(ABC_ENABLE_LTO AND ipo_supported)
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fuse-ld=lld")
        set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fuse-ld=lld")
    endif()
endif()

install(FILES ${ABC_HEADERS}
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/abc
)

install(
    TARGETS libabc abc
    EXPORT abcConfig
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    INCLUDES
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

install(EXPORT abcConfig
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/abc NAMESPACE abc::)

if(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    if(BUILD_SHARED_LIBS)
        install(FILES $<TARGET_PDB_FILE:libabc> DESTINATION bin OPTIONAL)
    endif()
endif()

if(NOT DEFINED ABC_SKIP_TESTS)
    enable_testing()
    include(FetchContent)
    FetchContent_Declare(
    googletest
    DOWNLOAD_EXTRACT_TIMESTAMP TRUE
    # Specify the commit you depend on and update it regularly.
    URL "https://github.com/google/googletest/archive/refs/tags/v1.14.0.zip"
    )
    FetchContent_MakeAvailable(googletest)
    include(GoogleTest)
    add_subdirectory(test)
endif()
