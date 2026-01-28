cmake_minimum_required(VERSION 3.10)
project(tre VERSION 0.9.0 LANGUAGES C)

set(TRE_VERSION ${PROJECT_VERSION})
string(REPLACE "." ";" TRE_VERSION_LIST ${TRE_VERSION})
list(GET TRE_VERSION_LIST 0 TRE_VERSION_1)
list(GET TRE_VERSION_LIST 1 TRE_VERSION_2)
list(GET TRE_VERSION_LIST 2 TRE_VERSION_3)

include(CheckIncludeFiles)
include(CheckFunctionExists)
include(CheckTypeSize)
include(CheckSymbolExists)

# Basic checks
check_include_files(sys/types.h HAVE_SYS_TYPES_H)

# Options for features
option(TRE_ENABLE_WCHAR "Enable wide character support" OFF)
option(TRE_ENABLE_MULTIBYTE "Enable multibyte character support" OFF)
option(TRE_ENABLE_APPROX "Enable approximate matching support" OFF)

if(TRE_ENABLE_WCHAR)
    check_include_files(wchar.h HAVE_WCHAR_H)
    if(NOT HAVE_WCHAR_H)
        message(FATAL_ERROR "wchar.h not found but TRE_ENABLE_WCHAR is ON")
    endif()
    set(TRE_WCHAR 1)

    if(TRE_ENABLE_MULTIBYTE)
        check_function_exists(mbrtowc HAVE_MBRTOWC)
        check_function_exists(mbtowc HAVE_MBTOWC)
        if(HAVE_MBRTOWC OR HAVE_MBTOWC)
            set(TRE_MULTIBYTE 1)
            set(CMAKE_EXTRA_INCLUDE_FILES wchar.h)
            check_type_size(mbstate_t HAVE_MBSTATE_T)
            set(CMAKE_EXTRA_INCLUDE_FILES)
        endif()
    endif()
endif()

# User requested to disable wctype related features
# We explicitly do NOT check for wctype.h or wctype functions to stay minimal
# and avoid issues in toolchains that lack them (like some MinGW versions).
set(HAVE_WCTYPE_H OFF)
set(HAVE_ISWCTYPE OFF)
set(HAVE_WCTYPE OFF)
set(HAVE_ISWBLANK OFF)
set(HAVE_MBRTOWC OFF)
set(HAVE_MBTOWC OFF)
set(HAVE_MBSTATE_T OFF)

if(TRE_ENABLE_APPROX)
    set(TRE_APPROX 1)
endif()

# Reuse upstream tre-config.h.in
file(READ "${CMAKE_CURRENT_SOURCE_DIR}/local_includes/tre-config.h.in" CONFIG_CONTENT)

# Convert #undef to #cmakedefine
string(REPLACE "#undef " "#cmakedefine " CONFIG_CONTENT "${CONFIG_CONTENT}")

# Handle macros with values
string(REPLACE "#cmakedefine TRE_VERSION_1" "#define TRE_VERSION_1 @TRE_VERSION_1@" CONFIG_CONTENT "${CONFIG_CONTENT}")
string(REPLACE "#cmakedefine TRE_VERSION_2" "#define TRE_VERSION_2 @TRE_VERSION_2@" CONFIG_CONTENT "${CONFIG_CONTENT}")
string(REPLACE "#cmakedefine TRE_VERSION_3" "#define TRE_VERSION_3 @TRE_VERSION_3@" CONFIG_CONTENT "${CONFIG_CONTENT}")
string(REPLACE "#cmakedefine TRE_VERSION\n" "#define TRE_VERSION \"@TRE_VERSION@\"\n" CONFIG_CONTENT "${CONFIG_CONTENT}")

# Add custom/missing defines
set(CONFIG_CONTENT "${CONFIG_CONTENT}\n/* Define to a field in the regex_t struct where TRE should store a pointer to the internal tre_tnfa_t structure */\n#define TRE_REGEX_T_FIELD value\n")

file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/tre-config.h.cmake" "${CONFIG_CONTENT}")
configure_file("${CMAKE_CURRENT_BINARY_DIR}/tre-config.h.cmake" "${CMAKE_CURRENT_BINARY_DIR}/local_includes/tre-config.h" @ONLY)

set(TRE_SOURCES
    lib/tre-ast.c
    lib/tre-compile.c
    lib/tre-match-backtrack.c
    lib/tre-match-parallel.c
    lib/tre-mem.c
    lib/tre-parse.c
    lib/tre-stack.c
    lib/regcomp.c
    lib/regexec.c
    lib/regerror.c
    lib/xmalloc.c
)

if(TRE_ENABLE_APPROX)
    list(APPEND TRE_SOURCES lib/tre-match-approx.c)
endif()

add_library(tre STATIC ${TRE_SOURCES})

target_include_directories(tre PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/local_includes>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/local_includes>
    $<INSTALL_INTERFACE:include>
)

target_include_directories(tre PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/lib
)

set_target_properties(tre PROPERTIES
    OUTPUT_NAME tre
    CLEAN_DIRECT_OUTPUT 1
)

# Installation rules
install(TARGETS tre ARCHIVE DESTINATION lib)
install(FILES
    local_includes/regex.h
    DESTINATION include
)
install(FILES
    local_includes/tre.h
    ${CMAKE_CURRENT_BINARY_DIR}/local_includes/tre-config.h
    DESTINATION include/tre
)
