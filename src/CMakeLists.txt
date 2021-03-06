set(CMAKE_SKIP_BUILD_RPATH            FALSE)
set(CMAKE_BUILD_WITH_INSTALL_RPATH    FALSE)
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

# Treat MSYS as Cygwin
if (MSYS)
    set(CYGWIN 1)
    set(CMAKE_SYSTEM_NAME "CYGWIN")
endif ()

# Set RPATH when installing to a custom (non-system) directory
list(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/lib" isSystemDir)
if ("${isSystemDir}" STREQUAL "-1")
    set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_RPATH}:${CMAKE_INSTALL_PREFIX}/lib")
endif ()

include(${CMAKE_SOURCE_DIR}/cmake/EnsureLibraries.cmake)

if (NOT DEFINED LIBCLANG_LIBRARIES)
    set(LIBCLANG_LIBRARIES $ENV{LIBCLANG_LIBRARIES})
endif ()

if (NOT DEFINED LIBCLANG_CXXFLAGS)
    set(LIBCLANG_CXXFLAGS $ENV{LIBCLANG_CXXFLAGS})
endif ()

if (NOT DEFINED LIBCLANG_LIBDIR)
    set(LIBCLANG_LIBDIR $ENV{LIBCLANG_LIBDIR})
endif()

if (NOT DEFINED LIBCLANG_LIBRARIES OR NOT DEFINED LIBCLANG_CXXFLAGS OR NOT DEFINED LIBCLANG_LIBDIR)
    find_package(LibClang REQUIRED)
endif ()
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${LIBCLANG_CXXFLAGS}")

find_package(Curses REQUIRED)

if (EXISTS ${CLANG_COMPILATION_INCLUDE})
    add_definitions(-DHAVE_CXCOMPILATIONDATABASE)
endif ()

find_package(Lua)
If (LUA_FOUND)
    set(RCT_RTTI_ENABLED 1)
endif ()

set(RCT_NO_INSTALL 1)
set(RCT_NO_LIBRARY 1)
include(rct/rct.cmake)

include(${CMAKE_CURRENT_LIST_DIR}/rct/compiler.cmake)
# Set further compile options after including the rct compiler option.
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wstrict-aliasing=2 -Wcast-qual -fPIC")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wstrict-aliasing=2 -Wcast-qual -fPIC")
if (NOT CYGWIN)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fstack-protector-all -Wstack-protector")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fstack-protector-all -Wstack-protector")
endif ()

if (ASAN)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=address")
endif ()

if (CMAKE_BUILD_TYPE MATCHES "Debug")
    add_definitions("-DRTAGS_DEBUG")
    set(RCT_EVENTLOOP_CALLBACK_TIME_THRESHOLD 2000)
endif ()

if("${LIBCLANG_LIBDIR}" STREQUAL "")
    message(FATAL_ERROR "Define LIBCLANG_LIBDIR is empty, please provide the path to the clang library. See README for more help.")
endif()
add_definitions("-DCLANG_LIBDIR=${LIBCLANG_LIBDIR}")
add_definitions("-DOS_${CMAKE_SYSTEM_NAME}")
add_definitions(${RCT_DEFINITIONS})

if (CMAKE_SYSTEM_NAME MATCHES "FreeBSD")
    add_definitions(-D__LONG_LONG_SUPPORTED)
endif ()

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)

set(RTAGS_SOURCES
    ClangIndexer.cpp
    ClangThread.cpp
    ClassHierarchyJob.cpp
    CompilerManager.cpp
    CompletionThread.cpp
    DependenciesJob.cpp
    FileManager.cpp
    FindFileJob.cpp
    FindSymbolsJob.cpp
    FollowLocationJob.cpp
    IncludeFileJob.cpp
    IndexMessage.cpp
    IndexerJob.cpp
    JobScheduler.cpp
    ListSymbolsJob.cpp
    Location.cpp
    Preprocessor.cpp
    Project.cpp
    QueryJob.cpp
    QueryMessage.cpp
    RClient.cpp
    RTags.cpp
    ReferencesJob.cpp
    ScanThread.cpp
    Server.cpp
    Source.cpp
    StatusJob.cpp
    Symbol.cpp
    Symbol.cpp
    SymbolInfoJob.cpp
    ${RCT_SOURCES})

if (LUA_FOUND)
    list(APPEND RTAGS_SOURCES AST.cpp)
endif ()

add_library(rtags STATIC ${RTAGS_SOURCES})
if (RTAGS_COTIRE)
    include(cotire)
    set_target_properties(rtags PROPERTIES COTIRE_ADD_UNITY_BUILD FALSE)
    cotire(rtags)
endif ()

include_directories(${CMAKE_CURRENT_LIST_DIR} ${RCT_INCLUDE_DIRS} ${CMAKE_CURRENT_LIST_DIR}/selene/include)

if (CMAKE_SYSTEM_NAME MATCHES "Darwin")
    set(START_GROUP "")
    set(END_GROUP "")
else ()
    set(START_GROUP "-Wl,--start-group")
    set(END_GROUP "-Wl,--end-group")
endif ()

set(RTAGS_LIBRARIES rtags ${START_GROUP} ${LIBCLANG_LIBRARIES} ${END_GROUP} -lstdc++ ${CURSES_LIBRARIES} ${RCT_LIBRARIES})

if (LUA_FOUND)
    set(RTAGS_LIBRARIES ${RTAGS_LIBRARIES} ${LUA_LIBRARIES})
    include_directories(${LUA_INCLUDE_DIR})
    add_definitions(-DRTAGS_HAS_LUA)
    if (APPLE)
        set_source_files_properties(AST.cpp PROPERTIES COMPILE_FLAGS "-Wno-unused-parameter -Wno-unused-variable -ftemplate-depth=2000")
    elseif (UNIX)
        set_source_files_properties(AST.cpp PROPERTIES COMPILE_FLAGS "-Wno-unused-parameter -Wno-unused-variable -Wno-unused-but-set-parameter -ftemplate-depth=2000")
    endif ()
endif ()


add_executable(rc rc.cpp)
target_link_libraries(rc ${RTAGS_LIBRARIES})

add_executable(rdm rdm.cpp)

target_link_libraries(rdm ${RTAGS_LIBRARIES})

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)

add_executable(rp rp.cpp)
target_link_libraries(rp ${RTAGS_LIBRARIES})

if (CYGWIN)
    EnsureLibraries(rdm rct)
endif ()

if (EMACS)
    set(EMACS_EXECUTABLE ${EMACS})
else ()
    find_program(EMACS_EXECUTABLE emacs)
endif ()

if (EMACS_EXECUTABLE)
    execute_process(COMMAND ${EMACS_EXECUTABLE} --version
        RESULT_VARIABLE EMACS_ERROR
        OUTPUT_VARIABLE EMACS_VERSION_INFORMATION
        ERROR_QUIET)
    if (NOT EMACS_ERROR)
        string(REGEX MATCH "[0-9]+(\\.[0-9]+)*" EMACS_VERSION ${EMACS_VERSION_INFORMATION})
        if ("${EMACS_VERSION}" LESS "23")
            message(STATUS "Installed emacs version is to old, elisp files are not going to be installed, minimum required version is >=23!")
            set(RTAGS_NO_ELISP_FILES TRUE)
        endif ()
    else ()
        message(STATUS "Could not get emacs version (\"emacs --version\"), elisp files are not going to be installed")
        set(RTAGS_NO_ELISP_FILES TRUE)
    endif ()
else ()
    set(RTAGS_NO_ELISP_FILES TRUE)
    message(STATUS "Emacs was not found, elisp files are not going to be installed!")
endif ()

set(RTAGS_ELISP_FILES
    rtags.el
    rtags-ac.el
    company-rtags.el
    flycheck-rtags.el)

if (NOT RTAGS_NO_ELISP_FILES)
    if (NOT RTAGS_ELISP_INSTALL_LOCATION)
        set(RTAGS_ELISP_INSTALL_LOCATION ${CMAKE_INSTALL_PREFIX}/share/emacs/site-lisp/rtags/)
    endif ()
    set(RTAGS_ELISP_SOURCES "")
    foreach(el ${RTAGS_ELISP_FILES})
        if (NOT ${CMAKE_CURRENT_BINARY_DIR} STREQUAL ${CMAKE_CURRENT_SOURCE_DIR})
            add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${el}
        COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_SOURCE_DIR}/${el} ${CMAKE_CURRENT_BINARY_DIR}/${el}
        DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/${el}
        COMMENT "Copying ${CMAKE_CURRENT_SOURCE_DIR}/${el} -> ${CMAKE_CURRENT_BINARY_DIR}/${el}")
    endif ()
        list(APPEND RTAGS_ELISP_SOURCES ${CMAKE_CURRENT_BINARY_DIR}/${el})
        if (NOT RTAGS_NO_ELISP_BYTECOMPILE)
            list(APPEND RTAGS_ELISP_SOURCES ${CMAKE_CURRENT_BINARY_DIR}/${el}c)
            add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${el}c
                COMMAND ${EMACS_EXECUTABLE} -batch -l ${CMAKE_CURRENT_SOURCE_DIR}/compile-shim.elisp -l ${CMAKE_CURRENT_SOURCE_DIR}/rtags.el -f batch-byte-compile
                ${CMAKE_CURRENT_BINARY_DIR}/${el}
                DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${el}
                WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
                COMMENT "Creating byte-compiled Emacs lisp ${CMAKE_CURRENT_BINARY_DIR}/${el}c")
            add_custom_target(emacs_byte_compile_${el} ALL DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${el}c)
        endif ()
    endforeach()
endif ()

install(CODE "message(\"Installing rtags...\")")
include(BashCompletion)
if (BASH_COMPLETION_FOUND)
    file (MAKE_DIRECTORY "${PROJECT_BINARY_DIR}/completions")
    execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink rtags rc
        COMMAND ${CMAKE_COMMAND} -E create_symlink rtags rdm
        WORKING_DIRECTORY "${PROJECT_BINARY_DIR}/completions")
    install(FILES rtags-bash-completion.bash DESTINATION ${BASH_COMPLETION_COMPLETIONSDIR}
        PERMISSIONS OWNER_WRITE OWNER_READ GROUP_READ WORLD_READ
        RENAME rtags)
    install(FILES "${PROJECT_BINARY_DIR}/completions/rc" "${PROJECT_BINARY_DIR}/completions/rdm"
        DESTINATION ${BASH_COMPLETION_COMPLETIONSDIR})
endif ()
install(TARGETS rdm rc rp RUNTIME DESTINATION bin COMPONENT rtags)
install(FILES ../bin/gcc-rtags-wrapper.sh DESTINATION bin
    PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
install(FILES ../man/man7/rc.7 ../man/man7/rdm.7 DESTINATION share/man/man7/)
if (NOT RTAGS_NO_ELISP_FILES)
    install(FILES ${RTAGS_ELISP_SOURCES} DESTINATION ${RTAGS_ELISP_INSTALL_LOCATION})
endif ()
