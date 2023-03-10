# ------------------------------------------------------------------------------------------------------------
# Copyright (c) 2006-2023, Knut Reinert & Freie Universität Berlin
# Copyright (c) 2016-2023, Knut Reinert & MPI für molekulare Genetik
# This file may be used, modified and/or redistributed under the terms of the 3-clause BSD-License
# shipped with this file and also available at: https://github.com/seqan/library-template/blob/main/LICENSE.md
# ------------------------------------------------------------------------------------------------------------

cmake_minimum_required (VERSION 3.10...3.22)
project (library_template_test_coverage CXX)

include (../library_template-test.cmake)

if (CMAKE_BUILD_TYPE AND NOT ("${CMAKE_BUILD_TYPE}" STREQUAL "Debug"))
    message (WARNING "Coverage test must be build in debug mode [build type = ${CMAKE_BUILD_TYPE}]")
endif ()

find_program (GCOVR_COMMAND NAMES gcovr)

if (NOT GCOVR_COMMAND)
    message (FATAL_ERROR "gcovr not found! Aborting...")
endif ()

# Holds all target's defined by library_template_test
set_property (GLOBAL PROPERTY GLOBAL_TEST_COVERAGE_ALL_TESTS "")

set (LIBRARY_TEMPLATE_COVERAGE_PARALLEL_LEVEL
     "1"
     CACHE STRING "Number of threads to use for coverage report generation.")

set (LIBRARY_TEMPLATE_GCOVR_ARGUMENTS "")

macro (library_template_append_gcovr_args)
    set (key ${ARGV0})
    if (${ARGC} EQUAL 1)
        list (APPEND LIBRARY_TEMPLATE_GCOVR_ARGUMENTS ${key})
    else ()
        string (REPLACE "'^" "\\''^" value "${ARGV1}")
        string (REPLACE "$'" "$$'\\'" value "${value}")
        string (REPLACE ";" "\\\\\\;" value "${value}")
        list (APPEND LIBRARY_TEMPLATE_GCOVR_ARGUMENTS ${key} ${value})
    endif ()
endmacro ()

# Arguments for gcovr.
# Certain chracters will be escaped by library_template_append_gcovr_args.
# Special CMake characters such as "\" must be escaped manually with "\\".
# We can prevent one level of CMake's string evaluation by using bracket arguments, which are similar to raw strings:
# (https://cmake.org/cmake/help/latest/manual/cmake-language.7.html#bracket-argument)
# gcovr uses python regex expressions.
# See https://gcovr.com/en/5.0/guide.html#the-gcovr-command for an overview of parameters.
# See https://gcovr.com/en/5.0/faq.html#why-does-c-code-have-so-many-uncovered-branches for an explanation on branches.

# The directory of the CMakeLists.txt used for invoking cmake.
library_template_append_gcovr_args ("--root" "${CMAKE_CURRENT_LIST_DIR}")
# The build directory.
library_template_append_gcovr_args ("${PROJECT_BINARY_DIR}")
# Include all files whose path match '${LIBRARY_TEMPLATE_CLONE_DIR}/include/library_template/.*'.
library_template_append_gcovr_args ("--filter" "${LIBRARY_TEMPLATE_CLONE_DIR}/include/library_template")
# Include all files whose path match '${LIBRARY_TEMPLATE_CLONE_DIR}/test/include/library_template/test/.*'.
library_template_append_gcovr_args ("--filter" "${LIBRARY_TEMPLATE_CLONE_DIR}/test/include/library_template/test")
# Remove line coverage for all lines that match '^\s*$', i.e. empty lines.
library_template_append_gcovr_args ("--exclude-lines-by-pattern" [['^\\s*$']])
# Remove line coverage for all lines that match "^\s*[{}]{0,2}\s*;*\s*(//.*)?$", i.e. lines with a single '{' or '}'
# or a combination of those (max 2) which might be followed by a semicolon or a comment '//'.
library_template_append_gcovr_args ("--exclude-lines-by-pattern" [['^\\s*[{}]{0,2}\\s*;*\\s*(//.*)?$']])
# Will exclude branches that are unreachable.
library_template_append_gcovr_args ("--exclude-unreachable-branches")
# Will exclude branches that are only generated for exception handling.
library_template_append_gcovr_args ("--exclude-throw-branches")
# Run up to this many gcov instances in parallel.
library_template_append_gcovr_args ("-j" "${LIBRARY_TEMPLATE_COVERAGE_PARALLEL_LEVEL}")

add_custom_command (OUTPUT ${PROJECT_BINARY_DIR}/coverage.xml
                    # Run tests.
                    COMMAND ${CMAKE_CTEST_COMMAND} -j ${LIBRARY_TEMPLATE_COVERAGE_PARALLEL_LEVEL} --output-on-failure
                    # Run gcovr and create XML report.
                    COMMAND ${GCOVR_COMMAND} ${LIBRARY_TEMPLATE_GCOVR_ARGUMENTS} --xml --output
                            ${PROJECT_BINARY_DIR}/coverage.xml
                    WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
                    COMMENT "Processing code coverage counters and generating XML report."
                    COMMAND_EXPAND_LISTS)

add_custom_target (coverage ALL DEPENDS ${PROJECT_BINARY_DIR}/coverage.xml)

add_custom_command (OUTPUT ${PROJECT_BINARY_DIR}/html/index.html
                    # Run tests.
                    COMMAND ${CMAKE_CTEST_COMMAND} -j ${LIBRARY_TEMPLATE_COVERAGE_PARALLEL_LEVEL} --output-on-failure
                    # Create output directory.
                    COMMAND ${CMAKE_COMMAND} -E make_directory ${PROJECT_BINARY_DIR}/html/
                    # Run gcovr and create HTML report.
                    COMMAND ${GCOVR_COMMAND} ${LIBRARY_TEMPLATE_GCOVR_ARGUMENTS} --html-details --output
                            ${PROJECT_BINARY_DIR}/html/index.html
                    WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
                    COMMENT "Processing code coverage counters and generating HTML report."
                    COMMAND_EXPAND_LISTS)

add_custom_target (coverage_html
                   DEPENDS ${PROJECT_BINARY_DIR}/html/index.html
                   COMMENT "Generate coverage report.")

add_custom_command (TARGET coverage_html
                    POST_BUILD
                    COMMAND ;
                    COMMENT "Open ${PROJECT_BINARY_DIR}/html/index.html in your browser to view the coverage report.")

macro (library_template_test unit_test_cpp)
    file (RELATIVE_PATH unit_test "${CMAKE_SOURCE_DIR}/../unit" "${CMAKE_CURRENT_LIST_DIR}/${unit_test_cpp}")
    library_template_test_component (target "${unit_test}" TARGET_NAME)
    library_template_test_component (test_name "${unit_test}" TEST_NAME)

    add_executable (${target} ${unit_test_cpp})
    target_link_libraries (${target} library_template::test::coverage)
    add_test (NAME "${test_name}" COMMAND ${target})

    # any change of a target will invalidate the coverage result;
    # NOTE that this is a GLOBAL variable, because a normal
    # `set(GLOBAL_TEST_COVERAGE_ALL_TESTS)` would not propagate the result when
    # CMakeLists.txt goes out of scope due to a `add_subdirectory`
    set_property (GLOBAL APPEND PROPERTY GLOBAL_TEST_COVERAGE_ALL_TESTS ${target})

    unset (unit_test)
    unset (target)
    unset (test_name)
endmacro ()

library_template_require_ccache ()
library_template_require_test ()

# add all unit tests
add_subdirectories_of ("${CMAKE_CURRENT_SOURCE_DIR}/../unit")

# add collected test cases as dependency
get_property (TEST_COVERAGE_ALL_TESTS GLOBAL PROPERTY GLOBAL_TEST_COVERAGE_ALL_TESTS)
add_custom_command (OUTPUT ${PROJECT_BINARY_DIR}/coverage.xml
                    DEPENDS ${TEST_COVERAGE_ALL_TESTS}
                    APPEND)
