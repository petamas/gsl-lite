# Copyright 2015-2018 by Martin Moene
#
# gsl-lite is based on GSL: Guidelines Support Library,
# https://github.com/microsoft/gsl
#
# This code is licensed under the MIT License (MIT).

if( NOT DEFINED CMAKE_MINIMUM_REQUIRED_VERSION )
    cmake_minimum_required( VERSION 3.5 FATAL_ERROR )
endif()

project( gsl-lite-test LANGUAGES CXX )

set( unit_name "gsl" )
set( PACKAGE   ${unit_name}-lite )
set( PROGRAM   ${unit_name}-lite )

message( STATUS "Subproject '${PROJECT_NAME}', programs '${PROGRAM}-*'")

# Test sources:

set( SOURCES
    gsl-lite.t.cpp
    assert.t.cpp
    at.t.cpp
    byte.t.cpp
    issue.t.cpp
    not_null.t.cpp
    owner.t.cpp
    span.t.cpp
    string_span.t.cpp
    util.t.cpp
)

# Configure gsl-lite for testing:

set( GSL_CONFIG -Dgsl_CONFIG_CONTRACT_VIOLATION_THROWS -Dgsl_CONFIG_CONTRACT_LEVEL_AUDIT )

# Preset available C++ language compiler flags:

set( HAS_STD_FLAGS  FALSE )
set( HAS_CPP98_FLAG FALSE )
set( HAS_CPP11_FLAG FALSE )
set( HAS_CPP14_FLAG FALSE )
set( HAS_CPP17_FLAG FALSE )
set( HAS_CPP20_FLAG FALSE )
set( HAS_CPPLATEST_FLAG FALSE )

# Determine compiler-specifics for MSVC, GNUC, Clang:

if( MSVC )
    message( STATUS "Matched: MSVC")

    set( HAS_STD_FLAGS TRUE )

    set( OPTIONS     -W3 -EHsc )
    set( DEFINITIONS -D_SCL_SECURE_NO_WARNINGS ${GSL_CONFIG} )

    if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 19.00 )
        set( HAS_CPP14_FLAG TRUE )
        set( HAS_CPPLATEST_FLAG TRUE )
    endif()
    if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 19.11 )
        set( HAS_CPP17_FLAG TRUE )
    endif()

elseif( CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang|AppleClang" )
    message( STATUS "CompilerId: '${CMAKE_CXX_COMPILER_ID}'")

    set( HAS_STD_FLAGS  TRUE )
    set( HAS_CPP98_FLAG TRUE )

    set( OPTIONS
        -O2
        -Wall
        -Wno-missing-braces
        -Wconversion
        -Wsign-conversion
        -Wno-string-conversion
        -fno-elide-constructors
        -fstrict-aliasing -Wstrict-aliasing=2
    )
    set( DEFINITIONS ${GSL_CONFIG} )

    # GNU: available -std flags depends on version
    if( CMAKE_CXX_COMPILER_ID MATCHES "GNU" )
        message( STATUS "Matched: GNU")

        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.8.0 )
            set( HAS_CPP11_FLAG TRUE )
        endif()
        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.9.2 )
            set( HAS_CPP14_FLAG TRUE )
        endif()
        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 7.1.0 )
            set( HAS_CPP17_FLAG TRUE )
        endif()

    # AppleClang: available -std flags depends on version
    elseif( CMAKE_CXX_COMPILER_ID MATCHES "AppleClang" )
        message( STATUS "Matched: AppleClang")

        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 5.0.0 )
            set( HAS_CPP11_FLAG TRUE )
        endif()
        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 5.1.0 )
            set( HAS_CPP14_FLAG TRUE )
        endif()
        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 9.2.0 )
            set( HAS_CPP17_FLAG TRUE )
        endif()

    # Clang: available -std flags depends on version
    elseif( CMAKE_CXX_COMPILER_ID MATCHES "Clang" )
        message( STATUS "Matched: Clang")

        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 3.3.0 )
            set( HAS_CPP11_FLAG TRUE )
        endif()
        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 3.4.0 )
            set( HAS_CPP14_FLAG TRUE )
        endif()
        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 5.0.0 )
            set( HAS_CPP17_FLAG TRUE )
        endif()
    endif()

elseif( CMAKE_CXX_COMPILER_ID MATCHES "Intel" )
    # as is
    message( STATUS "Matched: Intel")
else()
    # as is
    message( STATUS "Matched: nothing")
endif()

# Enable MS C++ Core Guidelines checker if MSVC:

function( enable_msvs_guideline_checker target )
    if( MSVC )
        set_target_properties( ${target} PROPERTIES
        VS_GLOBAL_EnableCppCoreCheck true
        VS_GLOBAL_CodeAnalysisRuleSet CppCoreCheckRules.ruleset
        VS_GLOBAL_RunCodeAnalysis true )
    endif()
endfunction()

# Make target, compile for given standard if specified:

function( make_target target std extra )
    message( STATUS "Make target: '${std}'" )

    add_executable            ( ${target} ${SOURCES}  )
    target_link_libraries     ( ${target} PRIVATE ${PACKAGE} )
    target_compile_options    ( ${target} PRIVATE ${OPTIONS} )
    set( def ${DEFINITIONS} ${extra} )
    target_compile_definitions( ${target} PRIVATE ${def} )

    if( std )
        if( MSVC )
            target_compile_options( ${target} PRIVATE -std:c++${std} )
        else()
            # Necessary for clang 3.x:
            target_compile_options( ${target} PRIVATE -std=c++${std} )
            # Ok for clang 4 and later:
            # set( CMAKE_CXX_STANDARD ${std} )
            # set( CMAKE_CXX_STANDARD_REQUIRED ON )
            # set( CMAKE_CXX_EXTENSIONS OFF )
        endif()
    endif()
endfunction()

function( make_targets postfix extra )
    # Add generic executable, unless -std flags can be specified:

    if( NOT HAS_STD_FLAGS )
        make_target( ${PROGRAM}${postfix}.t "" ${extra} )
    else()
        # unconditionally add C++98 variant as MSVC has no option for it:
        if( HAS_CPP98_FLAG )
            make_target( ${PROGRAM}${postfix}-cpp98.t 98 "${extra}" )
        else()
            make_target( ${PROGRAM}${postfix}-cpp98.t "" "${extra}" )
        endif()

        if( HAS_CPP11_FLAG )
            make_target( ${PROGRAM}${postfix}-cpp11.t 11 "${extra}" )
        endif()

        if( HAS_CPP14_FLAG )
            make_target( ${PROGRAM}${postfix}-cpp14.t 14 "${extra}" )
        endif()

        if( HAS_CPP17_FLAG )
            make_target( ${PROGRAM}${postfix}-cpp17.t 17 "${extra}" )
            enable_msvs_guideline_checker( ${PROGRAM}${postfix}-cpp17.t )
        endif()

        if( HAS_CPPLATEST_FLAG )
            make_target( ${PROGRAM}${postfix}-cpplatest.t latest "${extra}" )
        endif()
    endif()
endfunction()

make_targets( "" "" )
make_targets( "-nnconstref" "-Dgsl_CONFIG_NOT_NULL_GET_BY_CONST_REF=1" )
make_targets( "-nnexplicit" "-Dgsl_CONFIG_NOT_NULL_EXPLICIT_CTOR=1" )

# Configure unit tests via CTest:

enable_testing()

function( add_tests postfix )
    if( HAS_STD_FLAGS )
        # Unconditionally add C++98 variant for MSVC:
        add_test(     NAME test${postfix}-cpp98     COMMAND ${PROGRAM}${postfix}-cpp98.t )

        if( HAS_CPP11_FLAG )
            add_test( NAME test${postfix}-cpp11     COMMAND ${PROGRAM}${postfix}-cpp11.t )
        endif()
        if( HAS_CPP14_FLAG )
            add_test( NAME test${postfix}-cpp14     COMMAND ${PROGRAM}${postfix}-cpp14.t )
        endif()
        if( HAS_CPP17_FLAG )
            add_test( NAME test${postfix}-cpp17     COMMAND ${PROGRAM}${postfix}-cpp17.t )
        endif()
        if( HAS_CPPLATEST_FLAG )
            add_test( NAME test${postfix}-cpplatest COMMAND ${PROGRAM}${postfix}-cpplatest.t )
        endif()
    else()
        add_test(     NAME test${postfix}           COMMAND ${PROGRAM}${postfix}. --pass )
        add_test(     NAME list_version${postfix}   COMMAND ${PROGRAM}${postfix}. --version )
        add_test(     NAME list_tags${postfix}      COMMAND ${PROGRAM}${postfix}. --list-tags )
        add_test(     NAME list_tests${postfix}     COMMAND ${PROGRAM}${postfix}. --list-tests )
    endif()
endfunction()

add_tests( "" )
add_tests( "-nnconstref" )
add_tests( "-nnexplicit" )

# end of file
