cmake_minimum_required(VERSION 3.20)
project(dartvm C CXX)

# ─── Input variables (set by build-dartvm.sh) ───
# BLUTTER_SRC_DIR     - path to blutter/src/ (contains all .cpp/.h)
# DARTVM_PACKAGES     - path to blutter's packages/ dir (dartvm_fetch_build.py output)
# DARTVM_LIB_NAME     - e.g. "dartvm3.12.2_android_arm64"
# DARTVM_STATIC_LIB   - path to libdart*.a (direct or from packages/)
# DART_INCLUDE_DIR    - optional override for Dart headers
# DART_VERSION        - e.g. "3.12.2"
# CAPSTONE_LIB        - path to libcapstone.so (dynamic linking)
# CAPSTONE_INCLUDE_DIR - path to capstone headers
# SQLITE_DIR          - path to SQLite amalgamation
# USE_SHARED_CAPSTONE - set to ON when linking against shared capstone

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# ─── Validate inputs ───
if(NOT DARTVM_STATIC_LIB)
    message(FATAL_ERROR "DARTVM_STATIC_LIB required")
endif()
if(NOT CAPSTONE_LIB AND NOT CAPSTONE_STATIC_LIB)
    message(FATAL_ERROR "CAPSTONE_LIB or CAPSTONE_STATIC_LIB required")
endif()
if(NOT BLUTTER_SRC_DIR)
    message(FATAL_ERROR "BLUTTER_SRC_DIR required")
endif()

# Determine Dart include dir
if(NOT DART_INCLUDE_DIR)
    if(DARTVM_PACKAGES AND DARTVM_LIB_NAME)
        if(EXISTS "${DARTVM_PACKAGES}/include/${DARTVM_LIB_NAME}")
            set(DART_INCLUDE_DIR "${DARTVM_PACKAGES}/include/${DARTVM_LIB_NAME}")
        else()
            file(GLOB _dartvm_inc "${DARTVM_PACKAGES}/include/dartvm*")
            list(GET _dartvm_inc 0 DART_INCLUDE_DIR)
        endif()
    endif()
endif()
if(NOT DART_INCLUDE_DIR)
    message(FATAL_ERROR "DART_INCLUDE_DIR not determined (set DARTVM_PACKAGES or DART_INCLUDE_DIR)")
endif()

message(STATUS "Dart VM lib:     ${DARTVM_STATIC_LIB}")
message(STATUS "Dart include:    ${DART_INCLUDE_DIR}")
message(STATUS "Blutter sources: ${BLUTTER_SRC_DIR}")

# ─── Capstone ───
if(NOT CAPSTONE_INCLUDE_DIR)
    set(CAPSTONE_INCLUDE_DIR "${BLUTTER_SRC_DIR}/../third_party/capstone/include")
endif()

# ─── Blutter sources ───
set(BLUTTER_SRCS
    ${BLUTTER_SRC_DIR}/CodeAnalyzer.cpp
    ${BLUTTER_SRC_DIR}/CodeAnalyzer_arm64.cpp
    ${BLUTTER_SRC_DIR}/DartApp.cpp
    ${BLUTTER_SRC_DIR}/DartClass.cpp
    ${BLUTTER_SRC_DIR}/DartDumper.cpp
    ${BLUTTER_SRC_DIR}/DartField.cpp
    ${BLUTTER_SRC_DIR}/DartFunction.cpp
    ${BLUTTER_SRC_DIR}/DartLibrary.cpp
    ${BLUTTER_SRC_DIR}/DartLoader.cpp
    ${BLUTTER_SRC_DIR}/DartStub.cpp
    ${BLUTTER_SRC_DIR}/DartThreadInfo.cpp
    ${BLUTTER_SRC_DIR}/DartTypes.cpp
    ${BLUTTER_SRC_DIR}/Disassembler.cpp
    ${BLUTTER_SRC_DIR}/Disassembler_arm64.cpp
    ${BLUTTER_SRC_DIR}/ElfHelper.cpp
    ${BLUTTER_SRC_DIR}/FridaWriter.cpp
    ${BLUTTER_SRC_DIR}/Util.cpp
    ${BLUTTER_SRC_DIR}/VarValue.cpp
    ${BLUTTER_SRC_DIR}/il.cpp
)

# ─── SQLite amalgamation ───
if(NOT SQLITE_DIR)
    set(SQLITE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/third_party/sqlite")
endif()
set(SQLITE_SRC "${SQLITE_DIR}/sqlite3.c")
if(NOT EXISTS ${SQLITE_SRC})
    message(FATAL_ERROR "SQLite amalgamation not found at ${SQLITE_SRC}")
endif()

# ─── Our wrapper ───
set(DARTVM_SO_SRCS
    ${CMAKE_CURRENT_SOURCE_DIR}/src/blutter_entry.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/src/regexp_stubs.cpp
    ${SQLITE_SRC}
)

# PCH must be disabled for regexp_stubs.cpp: the stubs forward-declare
# `namespace dart::BootstrapNatives`, which would ODR-conflict with the
# Dart SDK's `class dart::BootstrapNatives` if pulled in via pch.h.
set_source_files_properties(
    ${CMAKE_CURRENT_SOURCE_DIR}/src/regexp_stubs.cpp
    PROPERTIES SKIP_PRECOMPILE_HEADERS ON
)

# ─── Shared library ───
add_library(dartvm SHARED
    ${BLUTTER_SRCS}
    ${DARTVM_SO_SRCS}
)

# ─── Include paths ───
target_include_directories(dartvm PRIVATE
    ${BLUTTER_SRC_DIR}
    ${DART_INCLUDE_DIR}
    ${CAPSTONE_INCLUDE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    ${SQLITE_DIR}
)

# ─── Force-include atomic_ref compat (NDK r27 libc++ may lack std::atomic_ref) ───
if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    target_compile_options(dartvm PRIVATE
        "$<$<COMPILE_LANGUAGE:CXX>:-include;${CMAKE_CURRENT_SOURCE_DIR}/src/atomic_ref_compat.h>"
    )
endif()

# ─── Compile definitions ───
# 注意：DARTVM_STATIC_LIB 是以裸 .a 链接的（不是导出的 CMake target），
# 因此静态库构建时定义的 PUBLIC 宏（见 blutter scripts/CMakeLists.txt 模板）
# 不会传播到本目标，必须在此显式补齐。关键宏及其作用：
#   - DART_PRECOMPILED_RUNTIME：object.h 的 `Closure::entry_point()` 仅在其下声明；
#     且 Code/Closure 的 entry_point 读取走 AOT 直接字段路径（缺失走 JIT 分支 → 解析错乱）。
#   - PRODUCT：raw_object.h 用 `#if !defined(PRODUCT) && !defined(DART_PRECOMPILED_RUNTIME)`
#     守卫多个原始对象字段（UntaggedClass.direct_subclasses、UntaggedFunction.var_descriptors、
#     constant_coverage 等）。目标 APK 快照是 product 构建，缺失 PRODUCT 会导致结构体
#     布局多出字段 → 偏移错位 → vector 越界 / SIGSEGV@0x27。这是运行时正确性的关键。
#   - DART_TARGET_OS_ANDROID / TARGET_ARCH_ARM64 / EXCLUDE_CFE_AND_KERNEL_PLATFORM：
#     对齐目标平台/最小化运行时。
#   - U_USING_ICU_NAMESPACE=0：与宿主 blutter 完全一致的 ICU 命名空间定义。
target_compile_definitions(dartvm PRIVATE
    DART_VERSION="${DART_VERSION}"
    DART_COMPRESSED_POINTERS
    DART_PRECOMPILED_RUNTIME
    DART_TARGET_OS_ANDROID
    TARGET_ARCH_ARM64
    EXCLUDE_CFE_AND_KERNEL_PLATFORM
    PRODUCT
    U_USING_ICU_NAMESPACE=0
    SQLITE_THREADSAFE=0
    SQLITE_OMIT_LOAD_EXTENSION
    NDEBUG
    FRIDA_TEMPLATE_DIR="${BLUTTER_SRC_DIR}/../scripts"
)

# Version-specific defines
if(OLD_MAP_SET_NAME)
    target_compile_definitions(dartvm PRIVATE OLD_MAP_SET_NAME)
endif()
if(OLD_MAP_NO_IMMUTABLE)
    target_compile_definitions(dartvm PRIVATE OLD_MAP_NO_IMMUTABLE)
endif()
if(NO_LAST_INTERNAL_ONLY_CID)
    target_compile_definitions(dartvm PRIVATE NO_LAST_INTERNAL_ONLY_CID)
endif()
if(HAS_SHARED_CLASS_TABLE)
    target_compile_definitions(dartvm PRIVATE HAS_SHARED_CLASS_TABLE)
endif()
if(HAS_RECORD_TYPE)
    target_compile_definitions(dartvm PRIVATE HAS_RECORD_TYPE)
endif()
if(HAS_TYPE_REF)
    target_compile_definitions(dartvm PRIVATE HAS_TYPE_REF)
endif()
if(NO_INIT_LATE_STATIC_FIELD)
    target_compile_definitions(dartvm PRIVATE NO_INIT_LATE_STATIC_FIELD)
endif()
if(NO_CODE_ANALYSIS)
    target_compile_definitions(dartvm PRIVATE NO_CODE_ANALYSIS)
endif()
if(NO_METHOD_EXTRACTOR_STUB)
    target_compile_definitions(dartvm PRIVATE NO_METHOD_EXTRACTOR_STUB)
endif()
if(UNIFORM_INTEGER_ACCESS)
    target_compile_definitions(dartvm PRIVATE UNIFORM_INTEGER_ACCESS)
endif()
if(NO_OBJECT_STORE_STUB)
    target_compile_definitions(dartvm PRIVATE NO_OBJECT_STORE_STUB)
endif()
if(NO_CLOSURE_CONTEXT_FIELD)
    target_compile_definitions(dartvm PRIVATE NO_CLOSURE_CONTEXT_FIELD)
endif()
if(NO_EMBED_VM_SNAPSHOT)
    target_compile_definitions(dartvm PRIVATE NO_EMBED_VM_SNAPSHOT)
endif()

# ─── Link libraries ───
# Core: static DartVM (version-specific) + dynamic shared libs
target_link_libraries(dartvm PRIVATE
    "${DARTVM_STATIC_LIB}"
    atomic log z
)

# Capstone: dynamic or static based on USE_SHARED_CAPSTONE
if(USE_SHARED_CAPSTONE)
    target_link_libraries(dartvm PRIVATE "${CAPSTONE_LIB}")
elseif(CAPSTONE_LIB)
    target_link_libraries(dartvm PRIVATE "${CAPSTONE_LIB}")
else()
    target_link_libraries(dartvm PRIVATE "${CAPSTONE_STATIC_LIB}")
endif()

# ICU: dynamic linking on Android (shared across all Dart versions)
# ICU_DIR is passed by build-dartvm.sh via -DICU_DIR=...; only fallback
# to source-adjacent path if not provided (do NOT use set() which would
# shadow the cache variable).
if(ANDROID)
    if(NOT ICU_DIR)
        set(ICU_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../icu")
    endif()
    if(EXISTS "${ICU_DIR}/lib/libicuuc.so")
        add_library(icuuc SHARED IMPORTED)
        set_target_properties(icuuc PROPERTIES
            IMPORTED_LOCATION "${ICU_DIR}/lib/libicuuc.so"
            INTERFACE_INCLUDE_DIRECTORIES "${ICU_DIR}/include"
        )
        target_link_libraries(dartvm PRIVATE icuuc)
    endif()
    if(EXISTS "${ICU_DIR}/lib/libicudata.so")
        add_library(icudata SHARED IMPORTED)
        set_target_properties(icudata PROPERTIES
            IMPORTED_LOCATION "${ICU_DIR}/lib/libicudata.so"
        )
        target_link_libraries(dartvm PRIVATE icudata)
    endif()
endif()

# ─── Runtime path: ensure shared libs found at runtime ───
if(ANDROID)
    set_target_properties(dartvm PROPERTIES
        LINK_FLAGS "-Wl,-rpath,\$ORIGIN"
    )
endif()

# ─── Compile options ───
target_compile_options(dartvm PRIVATE
    -O3 -fno-rtti
    -fvisibility=hidden -fvisibility-inlines-hidden
    -fno-omit-frame-pointer
)

# ─── Precompiled header (CXX only; pch.h contains C++ headers) ───
if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    target_precompile_headers(dartvm PRIVATE
        "$<$<COMPILE_LANGUAGE:CXX>:${BLUTTER_SRC_DIR}/pch.h>"
    )
endif()

# ─── Properties ───
set_target_properties(dartvm PROPERTIES
    CXX_VISIBILITY_PRESET hidden
    VISIBILITY_INLINES_HIDDEN ON
)

# ─── Post-build strip ───
if(CMAKE_BUILD_TYPE STREQUAL "Release")
    add_custom_command(TARGET dartvm POST_BUILD
        COMMAND ${CMAKE_STRIP} --strip-unneeded "$<TARGET_FILE:dartvm>"
        COMMENT "Stripping debug symbols..."
    )
endif()
