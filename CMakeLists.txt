cmake_minimum_required(VERSION 3.14)
project (bredis)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_definitions(-DBOOST_ERROR_CODE_HEADER_ONLY)
add_definitions(-DBOOST_COROUTINES_NO_DEPRECATION_WARNING)
add_definitions(-DBOOST_ASIO_NO_DEPRECATED)
#add_definitions(-DBREDIS_DEBUG)

find_package(
    Boost
    COMPONENTS
        chrono
        coroutine
        filesystem
        regex
        system
        thread
        program_options
    REQUIRED
)

if(CMAKE_COMPILER_IS_GNUCXX)
    set(CMAKE_CXX_FLAGS "-g -O0 -fprofile-arcs -ftest-coverage")
    set(CMAKE_C_FLAGS "-g -O0 -fprofile-arcs -ftest-coverage")

    execute_process(COMMAND
        ${CMAKE_CXX_COMPILER} -dumpversion
        OUTPUT_VARIABLE GCC_VERSION)
    string(STRIP ${GCC_VERSION} GCC_VERSION)
    MESSAGE(STATUS "gcc version: [" ${GCC_VERSION} "]")

    # this works on gentoo, but not on travis/ubuntu, let's hard-code the value
    #execute_process(COMMAND
    #    "which" "gcov-${GCC_VERSION}"
    #    OUTPUT_VARIABLE GCOV_TOOL)
    #MESSAGE(STATUS "gcov: [" ${GCOV_TOOL} "]")
    #string(STRIP ${GCOV_TOOL} GCOV_TOOL)
    #MESSAGE(STATUS "gcov(fixed): [" ${GCOV_TOOL} "]")
    set(GCOV_TOOL "gcov-5")

    add_custom_target("coverage"
        COMMAND "lcov" --directory . --zerocounters
        COMMAND ctest
        COMMAND "lcov" --directory . --capture --output-file coverage.info --gcov-tool ${GCOV_TOOL}
        COMMAND "lcov" --remove coverage.info "'/t/*'" "'/boost_1_63_0/*'" "'/usr/*'" --output-file coverage.info.cleaned
        COMMAND "rm" coverage.info

        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    )
    add_custom_target("coverage-report"
        COMMAND "genhtml" -o coverage coverage.info.cleaned
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
        DEPENDS "coverage"
    )
endif()

include_directories(${bredis_SOURCE_DIR}/include ${Boost_INCLUDE_DIRS})
MESSAGE(STATUS "boost inc: " ${Boost_INCLUDE_DIRS})
MESSAGE(STATUS "boost libs: " ${Boost_LIBRARIES})

# add_library(catch_lib t/tests-main.cpp)

if(WIN32)
    set(Boost_USE_MULTITHREADED ON)
    set(LINK_DEPENDENCIES ${Boost_LIBRARIES})
    add_definitions(-DBOOST_ALL_DYN_LINK -DBOOST_THREAD_USE_LIB -D_WIN32_WINNT=0x0501)
else()
    #add_definitions(-Wall -Wextra -pedantic -Werror)
    add_definitions(-Wall -Wextra -pedantic)
    #add_definitions(-fsanitize=undefined -fsanitize=address -fno-omit-frame-pointer)
    set(LINK_DEPENDENCIES pthread ${Boost_LIBRARIES}) #  asan ubsan
endif()

if ((Boost_MAJOR_VERSION GREATER_EQUAL 1) AND (Boost_MINOR_VERSION GREATER_EQUAL 70))
    set(BOOST_VERSION_OK true)
else()
    set(BOOST_VERSION_OK false)
endif()

MESSAGE(STATUS "BOOST_VERSION_OK ${BOOST_VERSION_OK}")

add_subdirectory(examples)
add_subdirectory(dependencies/catch2)

enable_testing()
add_subdirectory(t)