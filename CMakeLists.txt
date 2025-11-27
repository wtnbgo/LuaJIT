cmake_minimum_required(VERSION 3.10)

project(LuaJIT VERSION 2.1.0)

# --- Basic Settings ---
set(LUAJIT_VERSION_MAJOR 2)
set(LUAJIT_VERSION_MINOR 1)
set(LUAJIT_VERSION_BUILD 0) # This should be set dynamically based on git commit, but is a fixed value here
set(LUAJIT_ABIVER 5.1)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)

# --- Build Options ---
option(BUILD_SHARED_LIBS "Build a shared library" ON)
option(BUILD_STATIC_LIBS "Build a static library" ON)
option(BUILD_EXECUTABLE "BUild the luajit executable" ON)

option(LUAJIT_ENABLE_LUA52COMPAT "Enable Lua 5.2 compatibility" OFF)
option(LUAJIT_DISABLE_FFI "Disable the FFI extension" OFF)
option(LUAJIT_DISABLE_JIT "Disable the JIT compiler (interpreter only)" OFF)
option(LUAJIT_USE_SYSMALLOC "Use system malloc instead of LuaJIT's custom allocator" OFF)

# Host tool directory paths
set(MINILUA_HOST_PATH "${CMAKE_BINARY_DIR}/minilua_host")
set(BUILDVM_HOST_PATH "${CMAKE_BINARY_DIR}/buildvm_host")

# --- Compiler Settings ---
include(CheckCCompilerFlag)
if(MSVC)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
else()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall")
    # Try to omit frame pointer as in Makefile
    check_c_compiler_flag("-fomit-frame-pointer" C_HAS_FOMIT_FRAME_POINTER)
    if(C_HAS_FOMIT_FRAME_POINTER)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fomit-frame-pointer")
    endif()
endif()

# --- Source Files ---
set(LUAJIT_CORE_SOURCES
    src/lj_alloc.c
    src/lj_api.c
    src/lj_asm.c
    src/lj_assert.c
    src/lj_bc.c
    src/lj_bcread.c
    src/lj_bcwrite.c
    src/lj_buf.c
    src/lj_carith.c
    src/lj_ccall.c
    src/lj_ccallback.c
    src/lj_cconv.c
    src/lj_cdata.c
    src/lj_char.c
    src/lj_clib.c
    src/lj_cparse.c
    src/lj_crecord.c
    src/lj_ctype.c
    src/lj_debug.c
    src/lj_dispatch.c
    src/lj_err.c
    src/lj_ffrecord.c
    src/lj_func.c
    src/lj_gc.c
    src/lj_gdbjit.c
    src/lj_ir.c
    src/lj_lex.c
    src/lj_lib.c
    src/lj_load.c
    src/lj_mcode.c
    src/lj_meta.c
    src/lj_obj.c
    src/lj_opt_dce.c
    src/lj_opt_fold.c
    src/lj_opt_loop.c
    src/lj_opt_mem.c
    src/lj_opt_narrow.c
    src/lj_opt_sink.c
    src/lj_opt_split.c
    src/lj_parse.c
    src/lj_profile.c
    src/lj_prng.c
    src/lj_record.c
    src/lj_serialize.c
    src/lj_snap.c
    src/lj_state.c
    src/lj_str.c
    src/lj_strfmt.c
    src/lj_strfmt_num.c
    src/lj_strscan.c
    src/lj_tab.c
    src/lj_trace.c
    src/lj_udata.c
    src/lj_vmevent.c
    src/lj_vmmath.c
    src/lib_aux.c
    src/lib_base.c
    src/lib_bit.c
    src/lib_buffer.c
    src/lib_debug.c
    src/lib_ffi.c
    src/lib_init.c
    src/lib_io.c
    src/lib_jit.c
    src/lib_math.c
    src/lib_os.c
    src/lib_package.c
    src/lib_string.c
    src/lib_table.c
)

set(LUAJIT_LIB_SOURCES
    ${LUAJIT_CORE_SOURCES}
)

# --- Code Generation ---
set(GENERATED_DIR ${CMAKE_CURRENT_BINARY_DIR}/generated)
file(MAKE_DIRECTORY ${GENERATED_DIR})

# --- Generate luajit.h ---
find_package(Git)
if(GIT_FOUND AND EXISTS "${CMAKE_SOURCE_DIR}/.git")
    # Get git timestamp and write it to luajit_relver.txt
    execute_process(
        COMMAND ${GIT_EXECUTABLE} show -s --format=%ct
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE GIT_TIMESTAMP
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    file(WRITE ${GENERATED_DIR}/luajit_relver.txt "${GIT_TIMESTAMP}")
else()
    # Fallback if git is not found - use current timestamp
    string(TIMESTAMP CURRENT_TIMESTAMP "%s" UTC)
    file(WRITE ${GENERATED_DIR}/luajit_relver.txt "${CURRENT_TIMESTAMP}")
endif()

# The genversion script expects to be run from a directory containing
# luajit_relver.txt and luajit_rolling.h.
file(COPY src/luajit_rolling.h DESTINATION ${GENERATED_DIR})

# Generate luajit.h using host/genversion.lua
add_custom_command(
    OUTPUT ${GENERATED_DIR}/luajit.h
    COMMAND ${MINILUA_HOST_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/src/host/genversion.lua
    DEPENDS ${MINILUA_HOST_PATH}
    COMMENT "Generating luajit.h"
    WORKING_DIRECTORY ${GENERATED_DIR}
)

# Create a custom target for luajit.h generation
add_custom_target(luajit_h_target DEPENDS ${GENERATED_DIR}/luajit.h)

# Check if TARGET_TESTARCH is empty or undefined
# TARGET_DIRが指定されている場合は、そこからTARGET_TESTARCHを読み取る
if(DEFINED ENV{TARGET_DIR} AND EXISTS "$ENV{TARGET_DIR}/target_testarch.txt")
    file(READ "$ENV{TARGET_DIR}/target_testarch.txt" TARGET_TESTARCH)
    string(strip "${TARGET_TESTARCH}" TARGET_TESTARCH)
    message(STATUS "TARGET_TESTARCH loaded from $ENV{TARGET_DIR}/target_testarch.txt: ${TARGET_TESTARCH}")
elseif(NOT TARGET_TESTARCH OR TARGET_TESTARCH STREQUAL "")
    if(WIN32 AND MSVC)
        # Windows MSVC: Define TARGET_TESTARCH based on msvcbuild.bat logic
        if(CMAKE_SIZEOF_VOID_P EQUAL 8)
            # 64-bit host compiler
            if(CMAKE_VS_PLATFORM_NAME STREQUAL "ARM64" OR CMAKE_GENERATOR_PLATFORM STREQUAL "ARM64")
                # ARM64 target
                set(TARGET_TESTARCH "LJ_TARGET_ARM64 1 LJ_LE 1 LJ_ARCH_BITS 64 LJ_ARCH_HASFPU 1 LJ_ABI_SOFTFP 0 LJ_HASJIT 1 LJ_HASFFI 1 WIN")
            else()
                # x64 target (default for 64-bit)
                set(TARGET_TESTARCH "LJ_TARGET_X64 1 LJ_LE 1 LJ_ARCH_BITS 64 LJ_ARCH_HASFPU 1 LJ_ABI_SOFTFP 0 LJ_HASJIT 1 LJ_HASFFI 1 LJ_FR2 1 WIN")
            endif()
        else()
            # 32-bit host compiler (x86)
            set(TARGET_TESTARCH "LJ_TARGET_X86 1 LJ_LE 1 LJ_ARCH_BITS 32 LJ_ARCH_HASFPU 1 LJ_ABI_SOFTFP 0 LJ_HASJIT 1 LJ_HASFFI 1")
        endif()
    else()
        # Non-Windows or non-MSVC: Use compiler command (like Makefile does)
        execute_process(
            COMMAND ${CMAKE_C_COMPILER} -E ${CMAKE_CURRENT_SOURCE_DIR}/src/lj_arch.h -dM
            OUTPUT_VARIABLE TARGET_TESTARCH
            ERROR_QUIET
        )
    endif()
endif()

# CMAKE_BINARY_DIRにTARGET_TESTARCHの内容を出力
file(WRITE "${CMAKE_BINARY_DIR}/target_testarch.txt" "${TARGET_TESTARCH}")
message(STATUS "TARGET_TESTARCH written to ${CMAKE_BINARY_DIR}/target_testarch.txt")

if(TARGET_TESTARCH MATCHES "LJ_TARGET_X64")
    set(LUAJIT_TARGET "LUAJIT_ARCH_X64")
    set(TARGET_LJARCH "x64")
    # Check LJ_FR2 setting like Makefile does
    if(NOT TARGET_TESTARCH MATCHES "LJ_FR2 1")
        set(DASM_ARCH "x86")
    else()
        set(DASM_ARCH "x64")
    endif()
elseif(TARGET_TESTARCH MATCHES "LJ_TARGET_X86")
    set(LUAJIT_TARGET "LUAJIT_ARCH_X86")
    set(TARGET_LJARCH "x86")
    set(DASM_ARCH "x86")
elseif(TARGET_TESTARCH MATCHES "LJ_TARGET_ARM64")
    set(LUAJIT_TARGET "LUAJIT_ARCH_ARM64")
    set(TARGET_LJARCH "arm64")
    set(DASM_ARCH "arm64")
elseif(TARGET_TESTARCH MATCHES "LJ_TARGET_ARM")
    set(LUAJIT_TARGET "LUAJIT_ARCH_ARM")
    set(TARGET_LJARCH "arm")
    set(DASM_ARCH "arm")
elseif(TARGET_TESTARCH MATCHES "LJ_TARGET_PPC")
    set(LUAJIT_TARGET "LUAJIT_ARCH_PPC")
    set(TARGET_LJARCH "ppc")
    set(DASM_ARCH "ppc")
elseif(TARGET_TESTARCH MATCHES "LJ_TARGET_MIPS64")
    set(LUAJIT_TARGET "LUAJIT_ARCH_MIPS64")
    set(TARGET_LJARCH "mips64")
    set(DASM_ARCH "mips64")
elseif(TARGET_TESTARCH MATCHES "LJ_TARGET_MIPS")
    set(LUAJIT_TARGET "LUAJIT_ARCH_MIPS")
    set(TARGET_LJARCH "mips")
    set(DASM_ARCH "mips")
else()
    # Fallback to CMAKE detection
    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
        if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|AMD64")
            set(LUAJIT_TARGET "LUAJIT_ARCH_X64")
            set(TARGET_LJARCH "x64")
            set(DASM_ARCH "x86") # Default to x86 dasc for x64
        elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64|ARM64")
            set(LUAJIT_TARGET "LUAJIT_ARCH_ARM64")
            set(TARGET_LJARCH "arm64")
            set(DASM_ARCH "arm64")
        else()
            set(LUAJIT_TARGET "LUAJIT_ARCH_X64")
            set(TARGET_LJARCH "x64")
            set(DASM_ARCH "x86")
        endif()
    else()
        if(CMAKE_SYSTEM_PROCESSOR MATCHES "i.86|x86")
            set(LUAJIT_TARGET "LUAJIT_ARCH_X86")
            set(TARGET_LJARCH "x86")
            set(DASM_ARCH "x86")
        elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "arm")
            set(LUAJIT_TARGET "LUAJIT_ARCH_ARM")
            set(TARGET_LJARCH "arm")
            set(DASM_ARCH "arm")
        else()
            set(LUAJIT_TARGET "LUAJIT_ARCH_X86")
            set(TARGET_LJARCH "x86")
            set(DASM_ARCH "x86")
        endif()
    endif()
endif()

set(DASM_DASC_FILE "src/vm_${DASM_ARCH}.dasc")

# Set DASM flags based on target architecture (like Makefile)
set(DASM_FLAGS "-LN")
if(TARGET_TESTARCH MATCHES "WIN")
    list(APPEND DASM_FLAGS "-D" "WIN")
endif()
if(TARGET_TESTARCH MATCHES "LJ_HASJIT 1")
    list(APPEND DASM_FLAGS "-D" "JIT")
endif()
if(TARGET_TESTARCH MATCHES "LJ_HASFFI 1")
    list(APPEND DASM_FLAGS "-D" "FFI")
endif()
if(TARGET_TESTARCH MATCHES "LJ_LE 1")
    list(APPEND DASM_FLAGS "-D" "ENDIAN_LE")
else()
    list(APPEND DASM_FLAGS "-D" "ENDIAN_BE")
endif()
if(TARGET_TESTARCH MATCHES "LJ_ARCH_BITS 64")
    list(APPEND DASM_FLAGS "-D" "P64")
endif()
if(TARGET_TESTARCH MATCHES "LJ_ARCH_HASFPU 1")
    list(APPEND DASM_FLAGS "-D" "FPU")
endif()
if(NOT TARGET_TESTARCH MATCHES "LJ_ABI_SOFTFP 1")
    list(APPEND DASM_FLAGS "-D" "HFABI")
endif()
if(TARGET_TESTARCH MATCHES "LJ_NO_UNWIND 1")
    list(APPEND DASM_FLAGS "-D" "NO_UNWIND")
endif()
if(TARGET_TESTARCH MATCHES "LJ_ABI_PAUTH 1")
    list(APPEND DASM_FLAGS "-D" "PAUTH")
endif()

# Generate buildvm_arch.h using DynASM
add_custom_command(
    OUTPUT ${GENERATED_DIR}/buildvm_arch.h
    COMMAND ${MINILUA_HOST_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/dynasm/dynasm.lua 
            -D LUAJIT_TARGET=${LUAJIT_TARGET}
            ${DASM_FLAGS} -o ${GENERATED_DIR}/buildvm_arch.h ${CMAKE_CURRENT_SOURCE_DIR}/${DASM_DASC_FILE}
    DEPENDS ${MINILUA_HOST_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/${DASM_DASC_FILE}
    COMMENT "Generating buildvm_arch.h using DynASM (${TARGET_LJARCH})"
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)

# Create a custom target for buildvm_arch.h generation
add_custom_target(buildvm_arch_target DEPENDS ${GENERATED_DIR}/buildvm_arch.h)

# Create a custom target for all buildvm-generated files
add_custom_target(buildvm_generated_files DEPENDS
    ${GENERATED_DIR}/lj_bcdef.h
    ${GENERATED_DIR}/lj_ffdef.h
    ${GENERATED_DIR}/lj_libdef.h
    ${GENERATED_DIR}/lj_recdef.h
    ${GENERATED_DIR}/lj_folddef.h
    ${CMAKE_CURRENT_SOURCE_DIR}/src/jit/vmdef.lua
    ${VM_OUTPUT_FILE}
)

# Generate various files using buildvm
set(LIB_SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_base.c ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_math.c 
    ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_bit.c ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_string.c 
    ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_table.c ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_io.c 
    ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_os.c ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_package.c 
    ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_debug.c ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_jit.c 
    ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_ffi.c ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_buffer.c
)

add_custom_command(
    OUTPUT ${GENERATED_DIR}/lj_bcdef.h
    COMMAND ${BUILDVM_HOST_PATH} -m bcdef -o ${GENERATED_DIR}/lj_bcdef.h ${LIB_SOURCES}
    DEPENDS ${BUILDVM_HOST_PATH}
    COMMENT "Generating lj_bcdef.h"
)

add_custom_command(
    OUTPUT ${GENERATED_DIR}/lj_ffdef.h
    COMMAND ${BUILDVM_HOST_PATH} -m ffdef -o ${GENERATED_DIR}/lj_ffdef.h ${LIB_SOURCES}
    DEPENDS ${BUILDVM_HOST_PATH}
    COMMENT "Generating lj_ffdef.h"
)

add_custom_command(
    OUTPUT ${GENERATED_DIR}/lj_libdef.h
    COMMAND ${BUILDVM_HOST_PATH} -m libdef -o ${GENERATED_DIR}/lj_libdef.h ${LIB_SOURCES}
    DEPENDS ${BUILDVM_HOST_PATH}
    COMMENT "Generating lj_libdef.h"
)

add_custom_command(
    OUTPUT ${GENERATED_DIR}/lj_recdef.h
    COMMAND ${BUILDVM_HOST_PATH} -m recdef -o ${GENERATED_DIR}/lj_recdef.h ${LIB_SOURCES}
    DEPENDS ${BUILDVM_HOST_PATH}
    COMMENT "Generating lj_recdef.h"
)

add_custom_command(
    OUTPUT ${GENERATED_DIR}/lj_folddef.h
    COMMAND ${BUILDVM_HOST_PATH} -m folddef -o ${GENERATED_DIR}/lj_folddef.h ${CMAKE_CURRENT_SOURCE_DIR}/src/lj_opt_fold.c
    DEPENDS ${BUILDVM_HOST_PATH}
    COMMENT "Generating lj_folddef.h"
)

add_custom_command(
    OUTPUT ${CMAKE_CURRENT_SOURCE_DIR}/src/jit/vmdef.lua
    COMMAND ${BUILDVM_HOST_PATH} -m vmdef -o ${CMAKE_CURRENT_SOURCE_DIR}/src/jit/vmdef.lua ${LIB_SOURCES}
    DEPENDS ${BUILDVM_HOST_PATH}
    COMMENT "Generating jit/vmdef.lua"
)

# Determine buildvm mode and output file based on platform (like Makefile)
if(WIN32)
    set(BUILDVM_MODE "peobj")
    set(VM_OUTPUT_FILE "${GENERATED_DIR}/lj_vm.obj")
elseif(APPLE)
    set(BUILDVM_MODE "machasm")
    set(VM_OUTPUT_FILE "${GENERATED_DIR}/lj_vm.s")
else()
    set(BUILDVM_MODE "elfasm")
    set(VM_OUTPUT_FILE "${GENERATED_DIR}/lj_vm.s")
endif()

# Add architecture-specific compiler flags (like Makefile CCOPT_$(TARGET_LJARCH))
if(TARGET_LJARCH STREQUAL "x86")
    if(NOT MSVC)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -march=i686 -msse -msse2 -mfpmath=sse")
    endif()
endif()

add_custom_command(
    OUTPUT ${VM_OUTPUT_FILE}
    COMMAND ${BUILDVM_HOST_PATH} -m ${BUILDVM_MODE} -o ${VM_OUTPUT_FILE}
    DEPENDS ${BUILDVM_HOST_PATH}
    COMMENT "Generating VM source for ${TARGET_LJARCH} (${BUILDVM_MODE})"
)

# Set VM file for inclusion in sources based on platform
if(WIN32)
    # Windows: VM file is object file, handle separately (empty for sources)
    set(VM_SOURCE_FILE "")
    # Windows: VM file needs to be added directly to targets
    set(VM_LINK_FILE ${VM_OUTPUT_FILE})
else()
    # Unix: VM file is assembly source, include in sources
    set(VM_SOURCE_FILE ${VM_OUTPUT_FILE})
    # Unix: VM file is already included in luajit-core
    set(VM_LINK_FILE "")
endif()

set(GENERATED_SOURCES
    ${GENERATED_DIR}/lj_bcdef.h
    ${GENERATED_DIR}/lj_ffdef.h
    ${GENERATED_DIR}/lj_libdef.h
    ${GENERATED_DIR}/lj_recdef.h
    ${GENERATED_DIR}/lj_folddef.h
    ${CMAKE_CURRENT_SOURCE_DIR}/src/jit/vmdef.lua
    ${VM_SOURCE_FILE}
    ${GENERATED_DIR}/luajit.h
    ${GENERATED_DIR}/buildvm_arch.h
)

# --- Build Host Tools ---
# minilua (Lua interpreter) - ホスト用ツール準備

# Find C compiler
find_program(CC_EXECUTABLE 
    NAMES gcc
    DOC "GCC compiler"
)
if (NOT CC_EXECUTABLE)
    find_program(CC_EXECUTABLE 
        NAMES clang
        DOC "Clang compiler"
    )
endif()
if (NOT CC_EXECUTABLE)
    message(FATAL_ERROR "Could not find a C compiler (gcc or clang). Please install a C compiler or ensure it's in your PATH.")
endif()

add_custom_command(
    OUTPUT ${MINILUA_HOST_PATH}
    COMMAND ${CC_EXECUTABLE} -o ${MINILUA_HOST_PATH} src/host/minilua.c
    DEPENDS src/host/minilua.c
    COMMENT "Generating minilua_host using cc command"
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)

# buildvm (code generator) - ホスト用ツール準備
set(BUILDVM_SOURCES
    src/host/buildvm.c
    src/host/buildvm_asm.c
    src/host/buildvm_peobj.c
    src/host/buildvm_lib.c
    src/host/buildvm_fold.c
)

add_custom_command(
    OUTPUT ${BUILDVM_HOST_PATH}
    COMMAND ${CC_EXECUTABLE} -o ${BUILDVM_HOST_PATH} 
            -I src -I ${GENERATED_DIR} -I dynasm 
            -D LUAJIT_TARGET=${LUAJIT_TARGET}
            ${BUILDVM_SOURCES} 
    DEPENDS ${BUILDVM_SOURCES} luajit_h_target buildvm_arch_target
    COMMENT "Generating buildvm_host using cc command"
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)

# --- Build LuaJIT library ---
enable_language(ASM)

add_library(luajit-core OBJECT ${LUAJIT_LIB_SOURCES} ${GENERATED_SOURCES})
target_include_directories(luajit-core PUBLIC src ${GENERATED_DIR})

# Add dependencies for generated files
add_dependencies(luajit-core luajit_h_target buildvm_generated_files)

target_compile_definitions(luajit-core PRIVATE
    LUAJIT_TARGET=${LUAJIT_TARGET}
)

# Add definitions based on options
if(LUAJIT_ENABLE_LUA52COMPAT)
    target_compile_definitions(luajit-core PRIVATE LUAJIT_ENABLE_LUA52COMPAT)
endif()
if(LUAJIT_DISABLE_FFI)
    target_compile_definitions(luajit-core PRIVATE LUAJIT_DISABLE_FFI)
endif()
if(LUAJIT_DISABLE_JIT)
    target_compile_definitions(luajit-core PRIVATE LUAJIT_DISABLE_JIT)
endif()
if (LUAJIT_USE_SYSMALLOC)
    target_compile_definitions(luajit-core PRIVATE LUAJIT_USE_SYSMALLOC)
endif()

# --- Final Targets ---
# luajit executable
if(BUILD_EXECUTABLE)
    add_executable(luajit src/luajit.c $<TARGET_OBJECTS:luajit-core> ${VM_LINK_FILE})
    target_include_directories(luajit PRIVATE src ${GENERATED_DIR})
    # Ensure luajit.h is generated before building luajit
    add_dependencies(luajit luajit_h_target)
    if(NOT WIN32)
        target_link_libraries(luajit PRIVATE m dl)
    endif()
    set_property(TARGET luajit PROPERTY OUTPUT_NAME "luajit-${LUAJIT_VERSION_MAJOR}.${LUAJIT_VERSION_MINOR}")
endif()

# Libraries
if(BUILD_SHARED_LIBS)
    add_library(luajit-shared SHARED $<TARGET_OBJECTS:luajit-core> ${VM_LINK_FILE})
    target_include_directories(luajit-shared PRIVATE src ${GENERATED_DIR})
    set_target_properties(luajit-shared PROPERTIES
        OUTPUT_NAME "luajit-${LUAJIT_ABIVER}"
        SOVERSION ${LUAJIT_VERSION_MAJOR}
        VERSION ${PROJECT_VERSION}
    )
    if(NOT WIN32)
        target_link_libraries(luajit-shared PRIVATE m dl)
    endif()
endif()

if(BUILD_STATIC_LIBS)
    add_library(luajit-static STATIC $<TARGET_OBJECTS:luajit-core> ${VM_LINK_FILE})
    target_include_directories(luajit-static PRIVATE src ${GENERATED_DIR})
    if(BUILD_SHARED_LIBS)
        # If both shared and static libraries are built, differentiate names
        set_target_properties(luajit-static PROPERTIES
            OUTPUT_NAME "luajit-static-${LUAJIT_ABIVER}"
        )
    else()
        # If only static library is built, use standard name
        set_target_properties(luajit-static PROPERTIES
            OUTPUT_NAME "luajit-${LUAJIT_ABIVER}"
        )
    endif()
endif()


# --- Installation ---
include(GNUInstallDirs)

if(BUILD_EXECUTABLE)
    install(TARGETS luajit
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    )
endif()

if(BUILD_SHARED_LIBS)
    install(TARGETS luajit-shared
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    )
endif()

if(BUILD_STATIC_LIBS)
    install(TARGETS luajit-static
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    )
endif()

# Header files
install(FILES
    src/lua.h
    src/lualib.h
    src/lauxlib.h
    src/luaconf.h
    ${GENERATED_DIR}/luajit.h
    src/lua.hpp
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/luajit-${LUAJIT_VERSION_MAJOR}.${LUAJIT_VERSION_MINOR}
)

# jit library
install(DIRECTORY src/jit/
    DESTINATION ${CMAKE_INSTALL_DATADIR}/luajit-${LUAJIT_VERSION_MAJOR}.${LUAJIT_VERSION_MINOR}
    FILES_MATCHING PATTERN "*.lua"
)

# pkg-config file
# ... (set etc/luajit.pc as a template)

# man page
# install(FILES etc/luajit.1 DESTINATION ${CMAKE_INSTALL_MANDIR}/man1)
