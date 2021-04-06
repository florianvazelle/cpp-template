#
# Tests configuration
#

set(TEST_MAIN ${PROJECT_NAME}Tests)
set(TEST_RUNNER_PARAMS "")

#
# Entry Point
#

file(GLOB_RECURSE PROJECT_TEST_SOURCES "${CMAKE_SOURCE_DIR}/test/src/*.cpp")

add_executable(${TEST_MAIN} ${PROJECT_TEST_SOURCES})

# Link the executable to library
target_link_libraries(${TEST_MAIN} PRIVATE ${PROJECT_NAME})

target_include_directories(${TEST_MAIN} PRIVATE ${DOCTEST_INCLUDE_DIR})

# Add in ctest
add_test(
    # Use some per-module/project prefix so that it is easier to run only tests for this module
    NAME ${PROJECT_NAME}.${TEST_MAIN}
    COMMAND ${TEST_MAIN} ${TEST_RUNNER_PARAMS})