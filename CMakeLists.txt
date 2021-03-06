cmake_minimum_required(VERSION 2.6)

# =============================================================================
# [AsmJit - Configuration]
# =============================================================================

# Embedding mode, asmjit will not create any targets (default FALSE).
# set(ASMJIT_EMBED FALSE)

# Whether to build a static library (default FALSE).
set(ASMJIT_STATIC TRUE)

# Whether to build tests and samples (default FALSE).
# set(ASMJIT_BUILD_TEST FALSE)

# =============================================================================
# [AsmJit - Build / Embed]
# =============================================================================

# Do not create a project if this CMakeLists.txt is included from another
# project. This makes it easy to embed or create a static library.
if(NOT CMAKE_PROJECT_NAME OR "${CMAKE_PROJECT_NAME}" MATCHES "^asmjit$")
  project(asmjit C CXX)
  set(ASMJIT_SIGNATURE "Standalone")
else()
  set(ASMJIT_SIGNATURE "Included")
endif()

if(ASMJIT_EMBED)
  set(ASMJIT_SIGNATURE "${ASMJIT_SIGNATURE} | Mode=Embed")
  set(ASMJIT_STATIC TRUE) # Implies ASMJIT_STATIC.
elseif(ASMJIT_STATIC)
  set(ASMJIT_SIGNATURE "${ASMJIT_SIGNATURE} | Mode=Static")
else()
  set(ASMJIT_SIGNATURE "${ASMJIT_SIGNATURE} | Mode=Shared")
endif()

if(ASMJIT_BUILD_TEST)
  set(ASMJIT_SIGNATURE "${ASMJIT_SIGNATURE} | Test=On")
else()
  set(ASMJIT_SIGNATURE "${ASMJIT_SIGNATURE} | Test=Off")
endif()

if(NOT ASMJIT_DIR)
  set(ASMJIT_DIR ${CMAKE_CURRENT_LIST_DIR})
endif()

message("-- [asmjit] ${ASMJIT_SIGNATURE}")
message("-- [asmjit] ASMJIT_DIR=${ASMJIT_DIR}")

# =============================================================================
# [AsmJit - Flags / Deps]
# =============================================================================

set(ASMJIT_SOURCE_DIR "${ASMJIT_DIR}/src")      # Asmjit source directory.
set(ASMJIT_INCLUDE_DIR "${ASMJIT_SOURCE_DIR}")  # Asmjit include directory.

set(ASMJIT_DEPS)                                # Asmjit dependencies (list of libraries) for the linker.
set(ASMJIT_LIBS)                                # Asmjit dependencies with asmjit included, for consumers.

# Internal, never use.
set(ASMJIT_D "-D")                              # Used to define a C/C++ preprocessor parameter (-D or /D).
set(ASMJIT_PRIVATE_CFLAGS "-fPIC")              # Compiler flags independent of build type.
set(ASMJIT_PRIVATE_LFLAGS "")                   # Linker flags used by the library and tests.

set(ASMJIT_PRIVATE_CFLAGS_DBG)                  # Compiler flags used only by debug build.
set(ASMJIT_PRIVATE_CFLAGS_REL)                  # Compiler flags used only by release build.

if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
  set(ASMJIT_D "/D")
  set(ASMJIT_PRIVATE_LFLAGS "/OPT:REF /OPT:ICF")

  list(APPEND ASMJIT_PRIVATE_CFLAGS /GF)
  list(APPEND ASMJIT_PRIVATE_CFLAGS_DBG /GS /GR-)
  list(APPEND ASMJIT_PRIVATE_CFLAGS_REL /Oi /Oy /GS- /GR-)
  if(NOT MSVC60 AND NOT MSVC70 AND NOT MSVC71)
    list(APPEND ASMJIT_PRIVATE_CFLAGS /MP)      # Enable multi-process compilation.
  endif()
endif()

if("${CMAKE_CXX_COMPILER_ID}" MATCHES "^(GNU|Clang)$")
  list(APPEND ASMJIT_PRIVATE_CFLAGS -fno-exceptions)
  list(APPEND ASMJIT_PRIVATE_CFLAGS_REL -fmerge-all-constants)
endif()

if(ASMJIT_EMBED)
  list(APPEND ASMJIT_PRIVATE_CFLAGS "${ASMJIT_D}ASMJIT_EMBED")
elseif(ASMJIT_STATIC)
  list(APPEND ASMJIT_PRIVATE_CFLAGS "${ASMJIT_D}ASMJIT_STATIC")
endif()

if(WIN32)
  list(APPEND ASMJIT_PRIVATE_CFLAGS "${ASMJIT_D}_UNICODE")
else()
  list(APPEND ASMJIT_DEPS pthread)
endif()

if("${CMAKE_SYSTEM_NAME}" MATCHES "Linux")
  list(APPEND ASMJIT_DEPS rt)
endif()

set(ASMJIT_LIBS ${ASMJIT_DEPS})
if(NOT ASMJIT_EMBED)
  list(INSERT ASMJIT_LIBS 0 asmjit)
endif()

set(ASMJIT_PRIVATE_CFLAGS_DBG ${ASMJIT_PRIVATE_CFLAGS} ${ASMJIT_PRIVATE_CFLAGS_DBG})
set(ASMJIT_PRIVATE_CFLAGS_REL ${ASMJIT_PRIVATE_CFLAGS} ${ASMJIT_PRIVATE_CFLAGS_REL})

message("-- [asmjit] ASMJIT_DEPS=${ASMJIT_DEPS}")
message("-- [asmjit] ASMJIT_LIBS=${ASMJIT_LIBS}")

# =============================================================================
# [AsmJit - Macros]
# =============================================================================

macro(asmjit_add_source _out_dst _src_dir)
  set(_src_path "${ASMJIT_SOURCE_DIR}/${_src_dir}")
  set(_src_list)

  foreach(_arg ${ARGN})
    set(_src_file "${_src_path}/${_arg}")
    list(APPEND _src_list ${_src_file})
  endforeach()

  list(APPEND "${_out_dst}" ${_src_list})
  source_group(${_src_dir} FILES ${_src_list})
endmacro()

macro(asmjit_add_library _target _src _deps _cflags _cflags_dbg _cflags_rel)
  if(NOT ASMJIT_STATIC)
    add_library(${_target} SHARED ${_src})
  else()
    add_library(${_target} STATIC ${_src})
  endif()

  target_link_libraries(${_target} ${_deps})
  set_target_properties(${_target} PROPERTIES LINK_FLAGS "${ASMJIT_PRIVATE_LFLAGS}")

  if(CMAKE_BUILD_TYPE)
    if("${CMAKE_BUILD_TYPE}" STREQUAL "Debug")
      target_compile_options(${_target} PRIVATE ${_cflags} ${_cflags_dbg})
    else()
      target_compile_options(${_target} PRIVATE ${_cflags} ${_cflags_rel})
    endif()
  else()
    target_compile_options(${_target} PRIVATE ${_cflags}
      $<$<CONFIG:Debug>:${_cflags_dbg}>
      $<$<NOT:$<CONFIG:Debug>>:${_cflags_rel}>)
  endif()

  if(NOT ASMJIT_STATIC)
    install(TARGETS ${_target} DESTINATION "lib${LIB_SUFFIX}")
  endif()
endmacro()

# =============================================================================
# [AsmJit - Source]
# =============================================================================

set(ASMJIT_SRC "")

asmjit_add_source(ASMJIT_SRC asmjit
  apibegin.h
  apiend.h

  asmjit.h
  base.h
  build.h
  host.h
  x86.h
)

asmjit_add_source(ASMJIT_SRC asmjit/base
  assembler.cpp
  assembler.h
  compiler.cpp
  compiler.h
  compilercontext.cpp
  compilercontext_p.h
  compilerfunc.h
  constpool.cpp
  constpool.h
  containers.cpp
  containers.h
  cpuinfo.cpp
  cpuinfo.h
  globals.cpp
  globals.h
  hlstream.cpp
  hlstream.h
  logger.cpp
  logger.h
  operand.cpp
  operand.h
  runtime.cpp
  runtime.h
  utils.cpp
  utils.h
  vectypes.h
  vmem.cpp
  vmem.h
  zone.cpp
  zone.h
)

asmjit_add_source(ASMJIT_SRC asmjit/x86
  x86assembler.cpp
  x86assembler.h
  x86compiler.cpp
  x86compiler.h
  x86compilercontext.cpp
  x86compilercontext_p.h
  x86compilerfunc.cpp
  x86compilerfunc.h
  x86cpuinfo.cpp
  x86cpuinfo.h
  x86inst.cpp
  x86inst.h
  x86operand.cpp
  x86operand_regs.cpp
  x86operand.h
  x86scheduler.cpp
  x86scheduler_p.h
)

# =============================================================================
# [AsmJit - Targets]
# =============================================================================

if(NOT ASMJIT_EMBED)
  # Add `asmjit` library.
  asmjit_add_library(asmjit
    "${ASMJIT_SRC}"
    "${ASMJIT_DEPS}"
    ""
    "${ASMJIT_PRIVATE_CFLAGS_DBG}"
    "${ASMJIT_PRIVATE_CFLAGS_REL}"
  )

  foreach(_src_file ${ASMJIT_SRC})
    get_filename_component(_src_dir ${_src_file} PATH)
    get_filename_component(_src_name ${_src_file} NAME)
    string(REGEX REPLACE "^${ASMJIT_SOURCE_DIR}/" "" targetpath "${_src_dir}")
    if("${_src_name}" MATCHES ".h$")
      if(NOT "${_src_name}" MATCHES "_p.h$")
        install(FILES ${_src_file} DESTINATION "include/${targetpath}")
      endif()
    endif()
  endforeach()

  # Add `asmjit` tests and samples.
  if(ASMJIT_BUILD_TEST)
    set(ASMJIT_TEST_SRC "")
    set(ASMJIT_TEST_CFLAGS ${ASMJIT_D}ASMJIT_TEST ${ASMJIT_D}ASMJIT_EMBED)
    asmjit_add_source(ASMJIT_TEST_SRC test asmjit_test_unit.cpp broken.cpp broken.h)

    add_executable(asmjit_test_unit ${ASMJIT_SRC} ${ASMJIT_TEST_SRC})
    target_link_libraries(asmjit_test_unit ${ASMJIT_DEPS})
    set_target_properties(asmjit_test_unit PROPERTIES LINK_FLAGS "${ASMJIT_PRIVATE_LFLAGS}")

    if(CMAKE_BUILD_TYPE)
      if("${CMAKE_BUILD_TYPE}" STREQUAL "Debug")
        target_compile_options(asmjit_test_unit PRIVATE ${ASMJIT_TEST_CFLAGS} ${ASMJIT_PRIVATE_CFLAGS_DBG})
      else()
        target_compile_options(asmjit_test_unit PRIVATE ${ASMJIT_TEST_CFLAGS} ${ASMJIT_PRIVATE_CFLAGS_REL})
      endif()
    else()
      target_compile_options(asmjit_test_unit PRIVATE ${ASMJIT_TEST_CFLAGS}
        $<$<CONFIG:Debug>:${ASMJIT_PRIVATE_CFLAGS_DBG}>
        $<$<NOT:$<CONFIG:Debug>>:${ASMJIT_PRIVATE_CFLAGS_REL}>)
    endif()

    foreach(_target asmjit_bench_x86 asmjit_test_opcode asmjit_test_x86)
      add_executable(${_target} "src/test/${_target}.cpp")
      target_link_libraries(${_target} ${ASMJIT_LIBS})
    endforeach()
  endif()
endif()
