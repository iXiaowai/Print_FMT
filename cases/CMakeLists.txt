add_subdirectory(support/gtest)

set(TEST_MAIN_SRC
    support/test-main.cc
    support/gtest-extra.cc
    support/gtest-extra.h
    support/util.cc
)

add_library(test-main STATIC ${TEST_MAIN_SRC})

target_include_directories(test-main
    PUBLIC
        ${PROJECT_SOURCE_DIR}/include
        ${CMAKE_CURRENT_LIST_DIR}/support
            ${CMAKE_CURRENT_LIST_DIR}/embedded
)

target_link_libraries(test-main
    gtest
    fmt
)

function(add_fmt_test name)
    add_executable(${name} ${ARGN})

    target_include_directories(${name}
        PRIVATE
            ${PROJECT_SOURCE_DIR}/include
            ${CMAKE_CURRENT_LIST_DIR}/support
            ${CMAKE_CURRENT_LIST_DIR}/embedded
            ${CMAKE_CURRENT_LIST_DIR}
    )

    target_link_libraries(${name}
        test-main
        fmt
    )

    add_test(NAME ${name} COMMAND ${name})
endfunction()

add_fmt_test(args-test
    core/args-test.cc
)

add_fmt_test(assert-test
    core/assert-test.cc
)

add_fmt_test(base-test
    core/base-test.cc
)

add_fmt_test(compile-test
    core/compile-test.cc
)

add_fmt_test(enforce-checks-test
    core/enforce-checks-test.cc
)

add_fmt_test(enum-test
    core/enum-test.cc
)

add_fmt_test(format-test
    core/format-test.cc
    embedded/mock-allocator.h
)


add_fmt_test(printf-test
    core/printf-test.cc
)

add_fmt_test(chrono-test
    optional/chrono-test.cc
)

add_fmt_test(color-test
    optional/color-test.cc
)

add_fmt_test(ostream-test
    optional/ostream-test.cc
)

add_fmt_test(os-test
    optional/os-test.cc
)

add_fmt_test(ranges-test
    optional/ranges-test.cc
    optional/ranges-odr-test.cc
)


add_fmt_test(std-test
    optional/std-test.cc
)

add_fmt_test(xchar-test
    optional/xchar-test.cc
)

add_fmt_test(no-builtin-types-test
    embedded/no-builtin-types-test.cc
)

add_fmt_test(noexception-test
    embedded/noexception-test.cc
)

add_executable(perf-sanity
    embedded/perf-sanity.cc
)

target_include_directories(perf-sanity
    PRIVATE
        ${PROJECT_SOURCE_DIR}/include
)

target_link_libraries(perf-sanity
    fmt
)
