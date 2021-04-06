#
# greeter
#

cmake_minimum_required(VERSION 3.12.4)

#
# Building in-tree is not allowed (we take care of your craziness).
#

if(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
    message(FATAL_ERROR "Prevented in-tree built. Please create a build directory outside of the source code and call cmake from there. Thank you.")
endif()

#
# Project configuration
#

project(
    greeter
    DESCRIPTION "My custom C++ template with CMake, Conan, Doxygen & Github Action "
    HOMEPAGE_URL "https://github.com/florianvazelle/cpp-template"
    LANGUAGES CXX
)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Debug)
endif()

option(GREETER_USE_LIBCPP "Use libc++ by adding -stdlib=libc++ flag if availbale." ON)
option(GREETER_USE_SANITIZER "Enable sanitizers by adding -fsanitize=address -fno-omit-frame-pointer -fsanitize=undefined flags" OFF)

#
# Compiler stuff
#

if(NOT WIN32 AND GREETER_USE_LIBCPP)
    include(CheckCXXSourceCompiles)
    include(CMakePushCheckState)

    cmake_push_check_state()

    set(CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS} -stdlib=libc++")

    check_cxx_source_compiles("
        #include<type_traits>
        int main() { return std::is_same_v<int, char>; }
    " GREETER_HAS_LIBCPP)

    if(NOT GREETER_HAS_LIBCPP)
        message(VERBOSE "The option GREETER_USE_LIBCPP is set (by default) but libc++ is not available. The flag will not be added to the target.")
    endif()

    cmake_pop_check_state()
endif()

#
# Add greeter library target
#

# ---- Tool ----

file(GLOB_RECURSE PROJECT_MODULES "${CMAKE_SOURCE_DIR}/cmake/*.cmake")
foreach(ARG ${PROJECT_MODULES})
  include(${ARG})
endforeach()

# ---- Conan package ----

if(CONAN_EXPORTED)
    include(${CMAKE_CURRENT_BINARY_DIR}/conanbuildinfo.cmake)
    conan_basic_setup()
else()
    include("${CMAKE_SOURCE_DIR}/cmake/manual/conan.cmake")
    conan_cmake_run(CONANFILE conanfile.txt BASIC_SETUP BUILD missing)
endif()

# ---- Create library ----

file(GLOB_RECURSE PROJECT_SOURCES "${CMAKE_SOURCE_DIR}/src/*.cpp")
file(GLOB_RECURSE PROJECT_HEADERS "${CMAKE_SOURCE_DIR}/include/*.hpp")

add_library(${PROJECT_NAME} OBJECT ${PROJECT_SOURCES} ${PROJECT_HEADERS})

# Lib needs its header files, and users of the library must also see these (PUBLIC).
target_include_directories(${PROJECT_NAME} PUBLIC ${PROJECT_SOURCE_DIR}/include)

target_link_libraries(${PROJECT_NAME} PRIVATE ${CONAN_LIBS})

if(GREETER_USE_SANITIZER)
    target_compile_options(${PROJECT_NAME} PRIVATE $<$<CONFIG:Debug>:-fsanitize=address -fno-omit-frame-pointer -fsanitize=undefined>)
    target_link_libraries(${PROJECT_NAME} PRIVATE $<$<CONFIG:Debug>:-fsanitize=address -fno-omit-frame-pointer -fsanitize=undefined>)
endif()

if(GREETER_HAS_LIBCPP)
    target_compile_options(${PROJECT_NAME} BEFORE PRIVATE -stdlib=libc++)
endif()

target_compile_features(${PROJECT_NAME} PRIVATE cxx_std_11)

#
# Entry point
#

option(GREETER_BUILD_APP "Enable building with entry point." ON)

if(GREETER_BUILD_APP)
    option(GREETER_INSTALL_APP "Enable installation of the entry point." OFF)

    # Add an executable for the file app/main.cc
    add_executable(${PROJECT_NAME}App app/main.cpp)

    # Link the executable to library
    target_link_libraries(${PROJECT_NAME}App PRIVATE ${PROJECT_NAME})

    # Install the executable
    if (GREETER_INSTALL_APP)
        install(TARGETS ${PROJECT_NAME}App DESTINATION bin)
    endif()
endif()

#
# Tests
#

option(GREETER_BUILD_TESTING "Enable building tests." OFF)

if(GREETER_BUILD_TESTING)
    # https://github.com/onqtam/doctest/blob/master/doc/markdown/build-systems.md
    include(ExternalProject)
    find_package(Git REQUIRED)

    ExternalProject_Add(
        doctest
        PREFIX ${CMAKE_BINARY_DIR}/doctest
        GIT_REPOSITORY https://github.com/onqtam/doctest.git
        TIMEOUT 10
        UPDATE_COMMAND ${GIT_EXECUTABLE} pull
        CONFIGURE_COMMAND ""
        BUILD_COMMAND ""
        INSTALL_COMMAND ""
        LOG_DOWNLOAD ON
    )

    # Expose required variable (DOCTEST_INCLUDE_DIR) to parent scope
    ExternalProject_Get_Property(doctest source_dir)
    set(DOCTEST_INCLUDE_DIR ${source_dir}/doctest CACHE INTERNAL "Path to include folder for doctest")

    include(CTest)
    enable_testing()
    add_subdirectory(test)
endif()

#
# Documentation
#

option(GREETER_BUILD_DOCS "Enable building with documentation." OFF)

if(GREETER_BUILD_DOCS)
    find_package(Doxygen 1.8)

    if(DOXYGEN_FOUND)
        add_subdirectory(docs)
    endif()
endif()