cmake_minimum_required(VERSION 3.13)
project(src/pyjion)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(CODE_COVERAGE "Enable coverage reporting" OFF)

add_definitions(-DDEFAULT_RECURSION_LIMIT=1000)
add_definitions(-DDEFAULT_CODEOBJECT_SIZE_LIMIT=10000)
option(COMPILER_DEBUG "Emit debug messages in the compiler" OFF)
option(BUILD_TESTS "Build the unit tests" OFF)
option(GENERATE_PROFILE "Enable Profile Generation" OFF)
option(DUMP_JIT_TRACES "Dump .net JIT traces on compilation" OFF)
option(REPORT_CLR_FAULTS "Report .NET CLR faults" OFF)
option(DUMP_SEQUENCE_POINTS "Dump IL and native sequence points after compiling" OFF)
option(DUMP_INSTRUCTION_GRAPHS "Dump graphviz instruction graphs during compilation" OFF)
option(DEBUG_VERBOSE "Verbose messages during execution" OFF)
option(ASAN "Compile with address sanitizer" OFF)
option(NET_SRC_LIB "Use .NET source library" OFF)
option(FUZZER "Build a test fuzzer" OFF)
include(eng/CMakeOptimizations.txt)
include(eng/CMakeProfiling.txt)

if (UNIX AND NOT APPLE)
    message(STATUS "Enabling PIC")
    set(CMAKE_POSITION_INDEPENDENT_CODE ON)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fpic")
endif()

if (SKBUILD)
    find_package(PythonExtensions REQUIRED)
    message(STATUS "Using Python libraries for skbuild " ${PYTHON_LIBRARIES})
    message(STATUS "Using Python headers for skbuild " ${PYTHON_INCLUDE_DIRS})
endif (SKBUILD)

if (NOT PYTHON_LIBRARIES)
    find_package (Python3 3.10 EXACT COMPONENTS Interpreter Development)
else()
    set(Python3_LIBRARIES ${PYTHON_LIBRARIES})
    set(Python3_INCLUDE_DIRS ${PYTHON_INCLUDE_DIRS})
endif()

message(STATUS "Using Python libraries " ${Python3_LIBRARIES})
message(STATUS "Using Python headers " ${Python3_INCLUDE_DIRS})

message("CMAKE_HOST_SYSTEM_PROCESSOR: ${CMAKE_HOST_SYSTEM_PROCESSOR}")
message("CMAKE_SYSTEM_PROCESSOR: ${CMAKE_SYSTEM_PROCESSOR}")

include_directories(${Python3_INCLUDE_DIRS})

set (CLR_DIR CoreCLR/src/coreclr)

add_definitions(-DUSE_STL)

IF(CMAKE_BUILD_TYPE MATCHES Debug OR COMPILER_DEBUG)
    message(STATUS "Enabling verbose messages")
    add_definitions(-DDEBUG)
    set(BUILD_TESTS ON)
    set(REPORT_CLR_FAULTS ON)
ENDIF(CMAKE_BUILD_TYPE MATCHES Debug OR COMPILER_DEBUG)

IF(CMAKE_BUILD_TYPE MATCHES Release)
    add_definitions(-DNDEBUG)
ENDIF(CMAKE_BUILD_TYPE MATCHES Release)

if(DEBUG_VERBOSE)
    message(STATUS "Enabling verbose debug messages")
    add_definitions(-DDEBUG_VERBOSE)
endif()
if(DUMP_JIT_TRACES)
    message(STATUS "Enabling very-very-verbose messages")
    add_definitions(-DDUMP_JIT_TRACES)
endif()
if(DUMP_INSTRUCTION_GRAPHS)
    message(STATUS "Dumping instruction graphs during compile")
    add_definitions(-DDUMP_INSTRUCTION_GRAPHS)
endif()
if(DUMP_SEQUENCE_POINTS)
    message(STATUS "Enabling sequence point IL dumps")
    add_definitions(-DDUMP_SEQUENCE_POINTS)
endif()
if(REPORT_CLR_FAULTS)
    message(STATUS "Enabling CLR fault reporting")
    add_definitions(-DREPORT_CLR_FAULTS)
endif()
if(ASAN)
    message(STATUS "Enabling address sanitizer")
    if (WIN32)
        add_compile_options(/fsanitize=address)
        add_link_options(/fsanitize=address)
    else()
        add_compile_options(-fsanitize=address,fuzzer-no-link)
    endif()
endif()
if(FUZZER)
    message(STATUS "Enabling address sanitizer and libfuzzer")
    set(BUILD_FUZZER ON)
endif()


if(NOT WIN32)
    set(CMAKE_CXX_STANDARD 17)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
    include_directories(CoreCLR/src/coreclr/pal/inc/rt CoreCLR/src/coreclr/pal/inc CoreCLR/src/coreclr/pal/prebuilt/inc)
    add_compile_options(-DPAL_STDCPP_COMPAT)
    add_compile_options(-fexceptions)
    add_compile_options(-fvisibility=hidden)

    add_definitions(-DTARGET_UNIX)
    message(STATUS "Enabling UNIX Patches")
    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        add_compile_options(-Wno-null-arithmetic -Wno-switch -Wno-pragma-pack)
    else(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        add_compile_options(-Wno-conversion-null -Wno-pointer-arith -Wno-switch -Wno-missing-profile)
    endif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
else()
    add_definitions(-DWINDOWS=1)
    add_definitions(-D_HAS_STD_BYTE=0)
    if (MSVC)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /std:c++latest")
        set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /STACK:80000000")
    endif(MSVC)
endif(NOT WIN32)

add_definitions(-DPROFILING_SUPPORTED -DUNICODE)

if (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "x86_64")
    set(IS_X64 1)
elseif (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "amd64")
    set(IS_X64 1)
elseif (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "AMD64")
    if (CMAKE_CL_64)
        set(IS_X64 1)
    endif()
elseif (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "arm64")
    set(IS_ARM64 1)
elseif (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "aarch64")
    set(IS_ARM64 1)
else()
    message(STATUS "Don't recognise CPU arch " ${CMAKE_SYSTEM_PROCESSOR})
endif()
message(STATUS "Detected CPU arch " ${CMAKE_SYSTEM_PROCESSOR})
if (IS_X64)
    add_definitions(-DFEATURE_SIMD)
    add_definitions(-DFEATURE_HW_INTRINSICS)
    message(STATUS "Enabling AMD64")
    if (NOT WIN32)
        add_definitions(-DUNIX_AMD64_ABI)
        add_definitions(-DUNIX_AMD64_ABI_ITF)
        message(STATUS "Enabling AMD64 ABI")
    endif()
elseif (IS_ARM64)
    add_definitions(-DFEATURE_HW_INTRINSICS)
    message(STATUS "Enabling ARM64")
else()
    message(STATUS "Enabling x86")
endif()

include_directories(CoreCLR/src/coreclr/inc CoreCLR/src/coreclr/jit)

if (UNIX)
    if(DEFINED ENV{DOTNET_ROOT})
        file(GLOB DOTNETPATH $ENV{DOTNET_ROOT}/shared/Microsoft.NETCore.App*/7.0.0/)
    else()
        file(GLOB DOTNETPATH /usr/local/share/dotnet/shared/Microsoft.NETCore.App*/7.0.0/)
        if (DOTNETPATH STREQUAL "")
            file(GLOB DOTNETPATH /usr/share/dotnet/shared/Microsoft.NETCore.App*/7.0.0/)
        endif()
    endif()
    if (DOTNETPATH STREQUAL "")
        message(FATAL_ERROR "Cant locate .NET 7")
    endif()
    message(STATUS "Found .NET 7 in " ${DOTNETPATH})
endif()

if (WIN32)
    set(CLR_OS_BUILD Windows.x64.Debug)
    if(DEFINED ENV{DOTNET_ROOT})
        file(GLOB DOTNETPATH $ENV{DOTNET_ROOT}/shared/Microsoft.NETCore.App*/7.0.0/)
        message ("Scanning " $ENV{DOTNET_ROOT}/shared/Microsoft.NETCore.App*/7.0.0/)
    else()
        file(GLOB DOTNETPATH $ENV{PROGRAMFILES}/dotnet/shared/Microsoft.NETCore.App*/7.0.0/)
    endif()
    if (DOTNETPATH STREQUAL "")
        message(FATAL_ERROR "Cant locate .NET 7")
    endif()
    message(STATUS "Found .NET 7 in " ${DOTNETPATH})
endif()

add_definitions(-DGIL)  # use Python GIL on sub function calls

if (UNIX AND NOT APPLE)
    if (IS_X64)
        set(CLR_OS_BUILD Linux.x64.Debug)
    elseif(IS_ARM64)
        set(CLR_OS_BUILD Linux.arm64.Debug)
    else()
        set(CLR_OS_BUILD Linux.x86.Debug)
    endif()
    set(CLR_JIT_LIB "libclrjit.so")
    set(CLR_CMAKE_HOST_UNIX 1)
    add_compile_options(-fPIC)
    add_definitions(-DINDIRECT_HELPERS)
endif()

if (APPLE)
    if (IS_X64)
        set(CLR_OS_BUILD OSX.x64.Debug)
    elseif(IS_ARM64)
        set(CLR_OS_BUILD OSX.arm64.Debug)
    else()
        set(CLR_OS_BUILD OSX.x86.Debug)
    endif()
    include_directories(src/pyjion/compat)
    set(CLR_CMAKE_HOST_UNIX 1)
    set(CLR_JIT_LIB "libclrjit.dylib")
    add_definitions(-D_XOPEN_SOURCE)
    add_definitions(-DTARGET_OSX)
    add_definitions(-DHOST_OSX)
    add_compile_options(-fdeclspec)
endif()

if(NET_SRC_LIB)  # use the libraries compiled from source
    set(DOTNETPATH ${CMAKE_SOURCE_DIR}/CoreCLR/artifacts/bin/coreclr/${CLR_OS_BUILD})
    message(STATUS "Using .NET builds " ${DOTNETPATH})
endif()

set(SOURCES src/pyjion/absint.cpp src/pyjion/absvalue.cpp src/pyjion/intrins.cpp src/pyjion/jitinit.cpp src/pyjion/pycomp.cpp src/pyjion/pyjit.cpp src/pyjion/exceptionhandling.cpp src/pyjion/stack.cpp src/pyjion/codemodel.cpp src/pyjion/binarycomp.cpp src/pyjion/instructions.cpp src/pyjion/unboxing.cpp src/pyjion/frame.h src/pyjion/pgc.cpp src/pyjion/base.cpp src/pyjion/objects/unboxedrangeobject.cpp src/pyjion/attrtable.cpp)

if (WIN32)
    enable_language(ASM_MASM)
    add_definitions(-DINDIRECT_HELPERS)
    if (IS_X64)
        set(HOST_ASM_MASM_X86_64 true)
    endif()
    set(SOURCES ${SOURCES} src/pyjion/helpers.asm)
endif()

add_library(pyjionlib OBJECT ${SOURCES})
add_library(_pyjion MODULE $<TARGET_OBJECTS:pyjionlib>)

if (WIN32)
    add_custom_command(
        TARGET _pyjion POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy
        ${DOTNETPATH}/clrjit.dll
        ${CMAKE_CURRENT_BINARY_DIR}/
    )
endif(WIN32)

set_target_properties(
        _pyjion
        PROPERTIES
        PREFIX ""
        OUTPUT_NAME "_pyjion"
        LINKER_LANGUAGE C
)

if (NOT SKBUILD)
    target_link_libraries(_pyjion ${Python3_LIBRARIES})
endif()

if (BUILD_TESTS)
    # Testing
    add_subdirectory(Tests/Catch)
    set(TEST_SOURCES Tests/testing_util.cpp Tests/test_basics.cpp Tests/test_compiler.cpp Tests/Tests.cpp Tests/test_wrappers.cpp Tests/test_exceptions.cpp Tests/test_scopes.cpp Tests/test_tracing.cpp Tests/test_inference.cpp Tests/test_math.cpp Tests/test_pgc.cpp Tests/test_unpack.cpp Tests/test_class.cpp Tests/test_coro.cpp Tests/test_graph.cpp Tests/test_big_build.cpp Tests/test_ilgen.cpp Tests/test_with.cpp Tests/test_containers.cpp Tests/test_bigint.cpp Tests/test_globals.cpp)

    add_executable(unit_tests ${TEST_SOURCES} $<TARGET_OBJECTS:pyjionlib>)
    if (NOT WIN32)
        set_property(TARGET unit_tests PROPERTY CXX_STANDARD 17)
        set_property(TARGET unit_tests PROPERTY CXX_EXTENSIONS OFF)
    endif(NOT WIN32)
    target_include_directories(unit_tests PRIVATE src/pyjion)
    target_link_libraries(unit_tests Catch2::Catch2)
    target_link_libraries(unit_tests ${Python3_LIBRARIES})

    if (NOT WIN32)
        target_link_libraries(unit_tests ${DOTNETPATH}/${CLR_JIT_LIB})
    endif()
    include(CTest)
endif(BUILD_TESTS)

if (BUILD_FUZZER)
    # Testing
    set(FUZZ_SOURCES Tests/fuzzing/fuzz_target.cpp)
    add_executable(fuzz_target ${FUZZ_SOURCES} $<TARGET_OBJECTS:pyjionlib>)
    if (NOT WIN32)
        target_compile_options(fuzz_target PRIVATE -fsanitize=address,fuzzer)
        target_link_options(fuzz_target PRIVATE -fsanitize=address,fuzzer)
        set_property(TARGET fuzz_target PROPERTY CXX_STANDARD 17)
        set_property(TARGET fuzz_target PROPERTY CXX_EXTENSIONS OFF)
    endif(NOT WIN32)
    target_include_directories(fuzz_target PRIVATE src/pyjion)
    target_link_libraries(fuzz_target ${Python3_LIBRARIES})
    if (NOT WIN32)
        target_link_libraries(fuzz_target ${DOTNETPATH}/${CLR_JIT_LIB})
    endif()
endif(BUILD_FUZZER)

# Code Coverage Configuration
add_library(coverage_config INTERFACE)

if(CODE_COVERAGE AND CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    # Add required flags (GCC & LLVM/Clang)
    target_compile_options(coverage_config INTERFACE
            -O0        # no optimization
            -g         # generate debug info
            --coverage # sets all required flags
            )
    target_link_options(coverage_config INTERFACE --coverage)
endif(CODE_COVERAGE AND CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")

if (NOT WIN32)
    target_link_libraries(_pyjion ${DOTNETPATH}/${CLR_JIT_LIB})
endif()

if (SKBUILD)
    python_extension_module(_pyjion)
    install(TARGETS _pyjion LIBRARY DESTINATION src/pyjion)
else()
    set(CMAKE_INSTALL_PREFIX ${CMAKE_SOURCE_DIR})
    install(TARGETS _pyjion
            CONFIGURATIONS Debug
            LIBRARY DESTINATION src/pyjion
            RUNTIME DESTINATION Debug/bin)
    install(TARGETS _pyjion
            CONFIGURATIONS Release
            LIBRARY DESTINATION src/pyjion
            RUNTIME DESTINATION Release/bin)
endif()
