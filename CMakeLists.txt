# this file was adapted from ammo.js CMakeList.txt
# https://github.com/kripken/ammo.js/blob/main/CMakeLists.txt
# Following is the license for that repository:

# Copyright (c) 2011 ammo.js contributors

# This software is provided 'as-is', without any express or implied
# warranty.  In no event will the authors be held liable for any damages
# arising from the use of this software.

# Permission is granted to anyone to use this software for any purpose,
# including commercial applications, and to alter it and redistribute it
# freely, subject to the following restrictions:

# 1. The origin of this software must not be misrepresented; you must not
#    claim that you wrote the original software. If you use this software
#    in a product, an acknowledgment in the product documentation would be
#    appreciated but is not required.
# 2. Altered source versions must be plainly marked as such, and must not be
#    misrepresented as being the original software.
# 3. This notice may not be removed or altered from any source distribution.

# ---

# See AUTHORS for list of contributors.

# See https://github.com/kripken/ammo.js for their AUTHORS file
# This file was then modified from their CMakeList.txt

cmake_minimum_required(VERSION 3.13 FATAL_ERROR)
set(CMAKE_POLICY_DEFAULT_CMP0077 NEW)

FIND_PACKAGE(Python3)
set(PYTHON ${Python3_EXECUTABLE} CACHE STRING "Python path")
set(EMSCRIPTEN_ROOT $ENV{EMSDK}/upstream/emscripten CACHE STRING "Emscripten path")
set(CMAKE_TOOLCHAIN_FILE ${EMSCRIPTEN_ROOT}/cmake/Modules/Platform/Emscripten.cmake)
set(WEBIDL_BINDER_SCRIPT ${EMSCRIPTEN_ROOT}/tools/webidl_binder.py)
set(PBD_FRONT_MATTER_FILE ${CMAKE_CURRENT_SOURCE_DIR}/front-matter.js)
set(PBD_HEADER_FILE ${CMAKE_CURRENT_SOURCE_DIR}/pbd.h)
set(PBD_IDL_FILE ${CMAKE_CURRENT_SOURCE_DIR}/pbd.idl)
set(PBD_ONLOAD_FILE ${CMAKE_CURRENT_SOURCE_DIR}/onload.js)
set(POSITION_BASED_DYNAMICS_SRC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/PositionBasedDynamics/)


set(POSITION_BASED_DYNAMICS_TARGETS Common Simulation Utils)
foreach(_TARGET ${POSITION_BASED_DYNAMICS_TARGETS})
  list(APPEND POSITION_BASED_DYNAMICS_LIBS $<TARGET_FILE:${_TARGET}>)
endforeach()

# Disable PositionBasedDynamics graphical benchmarks
set(USE_GRAPHICAL_BENCHMARK OFF)

# Disable PositionBasedDynamics GLUT
set(USE_GLUT OFF)

# Build Release by default
set(CMAKE_BUILD_TYPE Release CACHE STRING "Build Type")

# Default is 64*1024*1024 = 64MB
set(TOTAL_MEMORY 67108864 CACHE STRING "Total Memory")

# Enable for resizable heap, with some amount of slowness
set(ALLOW_MEMORY_GROWTH 0 CACHE STRING "Allow Memory Growth")

set(EMCC_ARGS
  --extern-pre-js ${PBD_FRONT_MATTER_FILE}
  --llvm-lto 1
  --post-js glue.js
  --post-js ${PBD_ONLOAD_FILE}
  -O3
  -s ALLOW_MEMORY_GROWTH=${ALLOW_MEMORY_GROWTH}
  -s ALLOW_TABLE_GROWTH=1
  -s EXPORTED_FUNCTIONS=["_malloc","_free"]
  -s EXPORTED_RUNTIME_METHODS=["UTF8ToString"]
  -s EXTRA_EXPORTED_RUNTIME_METHODS=["addFunction"]
  -s EXPORT_NAME="PBD"
  -s MODULARIZE=1
  -s NO_EXIT_RUNTIME=1
  -s NO_FILESYSTEM=1
  -s TOTAL_MEMORY=${TOTAL_MEMORY})

if(${CLOSURE})
  # Ignore closure errors about the PositionBasedDynamics Node class
  # (Node is a DOM thing too)
  LIST(APPEND EMCC_ARGS
    --closure 1
    -s IGNORE_CLOSURE_COMPILER_ERRORS=1)
else()
  LIST(APPEND EMCC_ARGS
    -s NO_DYNAMIC_EXECUTION=1)
endif()

set(EMCC_JS_ARGS ${EMCC_ARGS}
  -s AGGRESSIVE_VARIABLE_ELIMINATION=1
  -s ELIMINATE_DUPLICATE_FUNCTIONS=1
  -s LEGACY_VM_SUPPORT=1
  -s SINGLE_FILE=1
  -s WASM=0)

set(EMCC_WASM_ARGS ${EMCC_ARGS}
  -s BINARYEN_IGNORE_IMPLICIT_TRAPS=1
  -s WASM=1)

set(EMCC_GLUE_ARGS
  -c
  -I${POSITION_BASED_DYNAMICS_SRC_DIR}
  -include${PBD_HEADER_FILE})


#######################################
project("pbd")
# add_subdirectory(PositionBasedDynamics EXCLUDE_FROM_ALL)
add_subdirectory(PositionBasedDynamics)


#######################################
add_custom_command(
  OUTPUT glue.cpp glue.js
  BYPRODUCTS parser.out WebIDLGrammar.pkl
  COMMAND ${PYTHON} ${WEBIDL_BINDER_SCRIPT} ${PBD_IDL_FILE} glue
  DEPENDS ${PBD_IDL_FILE}
  COMMENT "Generating pbd bindings"
  VERBATIM)
add_custom_command(
  OUTPUT glue.o
  COMMAND emcc glue.cpp ${EMCC_GLUE_ARGS} -o glue.o
  DEPENDS glue.cpp ${PBD_HEADER_FILE}
  COMMENT "Building pbd bindings"
  VERBATIM)
add_custom_target(pbd-bindings ALL DEPENDS glue.js glue.o)


#######################################
add_custom_command(
  OUTPUT pbd.js
  COMMAND emcc glue.o ${POSITION_BASED_DYNAMICS_LIBS} ${EMCC_JS_ARGS} -o pbd.js
  DEPENDS pbd-bindings ${PBD_FRONT_MATTER_FILE} ${PBD_ONLOAD_FILE} ${POSITION_BASED_DYNAMICS_TARGETS}
  COMMENT "Building pbd javascript"
  VERBATIM)
add_custom_target(pbd-javascript ALL DEPENDS pbd.js)


#######################################
add_custom_command(
  OUTPUT pbd.wasm.js pbd.wasm.wasm
  COMMAND emcc glue.o ${POSITION_BASED_DYNAMICS_LIBS} ${EMCC_WASM_ARGS} -o pbd.wasm.js
  DEPENDS pbd-bindings ${PBD_FRONT_MATTER_FILE} ${PBD_ONLOAD_FILE} ${POSITION_BASED_DYNAMICS_TARGETS}
  COMMENT "Building pbd webassembly"
  VERBATIM)
add_custom_target(pbd-wasm ALL DEPENDS pbd.wasm.js pbd.wasm.wasm)
