if (MULTITHREADS)
    find_package(Threads REQUIRED)
endif()

set (ARGON2_SRC
    "argon2.c"
    "core.c"
    "blake2/blake2b.c"
    "thread.c"
    "encoding.c"
    )

message(STATUS "Checking support for hardware optimization:")
try_compile(WITH_OPTIMIZATIONS
    ${CMAKE_CURRENT_BINARY_DIR}/optimization
    ${CMAKE_CURRENT_LIST_DIR}/optimization
    check_cpu_features
    OUTPUT_VARIABLE OUTPUT_TEST_SUPPORT_OPTIMIZATION)
message(STATUS "${OUTPUT_TEST_SUPPORT_OPTIMIZATION}")

message(STATUS "Build with hardware optimization? ${WITH_OPTIMIZATIONS}")
if (WITH_OPTIMIZATIONS)
    list(APPEND ARGON2_SRC "opt.c")
else()
    list(APPEND ARGON2_SRC "ref.c")
endif()

##########
# SHARED #
##########
add_library (argon2 SHARED ${ARGON2_SRC})

if (UNIX)
    target_compile_definitions(argon2 PRIVATE "A2_VISCTL")
endif ()

target_include_directories(argon2
                            PUBLIC
                                $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
                            PRIVATE
                                $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
                                $<INSTALL_INTERFACE:include>)

if (MULTITHREADS)
    target_link_libraries(argon2 PRIVATE Threads::Threads)
else()
    target_compile_definitions(argon2 PRIVATE "ARGON2_NO_THREADS=1")
endif()

if (WIN32 OR MINGW)
    set_target_properties(argon2 PROPERTIES
        C_STANDARD 90
        OUTPUT_NAME "argon2"
        PDB_OUTPUT_NAME "argon2"
        SUFFIX ".dll")
else()
    set_target_properties(argon2 PROPERTIES
        C_STANDARD 90
        OUTPUT_NAME "argon2"
        PDB_OUTPUT_NAME "argon2"
        SUFFIX ".so.${ABI_VERSION}")
endif()

install(TARGETS argon2
        EXPORT argon2-targets
        RUNTIME  DESTINATION ${CMAKE_INSTALL_BINDIR}
        LIBRARY  DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE  DESTINATION ${CMAKE_INSTALL_LIBDIR}
        )

install(EXPORT argon2-targets
        NAMESPACE argon2::
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/argon2
        )

if (UNIX)
    macro(install_symlink filepath sympath)
        install(CODE "execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink ${filepath} ${sympath} \
                                      WORKING_DIRECTORY ${CMAKE_INSTALL_LIBDIR})")
        install(CODE "message(\"-- Created symlink: ${sympath} -> ${filepath}\")")
    endmacro(install_symlink)

    install_symlink(${CMAKE_SHARED_LIBRARY_PREFIX}argon2${CMAKE_SHARED_LIBRARY_SUFFIX}.${ABI_VERSION}
                    ${CMAKE_SHARED_LIBRARY_PREFIX}argon2${CMAKE_SHARED_LIBRARY_SUFFIX})
endif()

##########
# STATIC #
##########
add_library (argon2-static STATIC ${ARGON2_SRC})

if (UNIX)
    target_compile_definitions(argon2-static PRIVATE "A2_VISCTL")
endif ()

if (MULTITHREADS)
    target_link_libraries(argon2-static PRIVATE Threads::Threads)
else()
    target_compile_definitions(argon2-static PRIVATE "ARGON2_NO_THREADS=1")
endif()

target_include_directories(argon2-static
                                PUBLIC
                                    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
                                PRIVATE
                                    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
                                    $<INSTALL_INTERFACE:include>)

if (UNIX)
    set(ARGON2_STATIC_LIB_NAME "argon2")
else()
    set(ARGON2_STATIC_LIB_NAME "argon2-static")
endif()
set_target_properties(argon2-static PROPERTIES
    C_STANDARD 90
    OUTPUT_NAME ${ARGON2_STATIC_LIB_NAME}
    PDB_OUTPUT_NAME "argon2")

install(TARGETS argon2-static
        ARCHIVE  DESTINATION ${CMAKE_INSTALL_LIBDIR}
        )


install(FILES "${CMAKE_SOURCE_DIR}/include/argon2.h"
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/argon2)

if (UNIX)
    set (CPACK_PACKAGE_NAME argon2)
    set (PKGCONFIG_INSTALL_DIR lib/cmake/argon2)
    if (MULTITHREADS)
        set(CPACK_PACKAGE_LINKFLAGS "-pthread")
    endif()

    configure_file ("${PROJECT_SOURCE_DIR}/cmake/argon2.pc.in" "${PROJECT_BINARY_DIR}/lib${CPACK_PACKAGE_NAME}.pc" @ONLY)
    install (FILES "${PROJECT_BINARY_DIR}/lib${CPACK_PACKAGE_NAME}.pc"
             DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/argon2")
endif()

######################################################
# STATIC for genkat (provide sew points for logging) #
######################################################
add_library (argon2-static-genkat STATIC ${ARGON2_SRC})

if (MULTITHREADS)
    target_link_libraries(argon2-static-genkat PRIVATE Threads::Threads)
else()
    target_compile_definitions(argon2-static-genkat PRIVATE "ARGON2_NO_THREADS=1")
endif()

target_include_directories(argon2-static-genkat PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>)

target_compile_definitions(argon2-static-genkat PRIVATE "GENKAT")
if (UNIX)
    target_compile_definitions(argon2-static-genkat PRIVATE "A2_VISCTL")
endif ()

set_target_properties(argon2-static-genkat PROPERTIES
    C_STANDARD 90
    OUTPUT_NAME "argon2-genkat")

