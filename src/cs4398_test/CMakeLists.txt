# BSD 2-Clause License
# 
# Copyright (c) 2017, Alexander Lutsai <s.lyra@ya.ru>
# All rights reserved.
# 
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are met:
# 
# * Redistributions of source code must retain the above copyright notice, this
#   list of conditions and the following disclaimer.
# 
# * Redistributions in binary form must reproduce the above copyright notice,
#   this list of conditions and the following disclaimer in the documentation
#   and/or other materials provided with the distribution.
# 
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
# AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
# DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
# FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
# DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
# SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
# CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
# OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

# Requirements:
# cmake > 2.6
# arm-none-eabi-gdb
# python3

# HOWTO:
# 1) Create STM32cubeMX SW4STM32 project.
# 2) Copy this CMakeLists.txt & CubeMX2_cmake.py in folder with created project.
# 3) Execute: "mkdir build; cd build"
# 4) Execute: "cmake ../; make"
# 5) DONE


#USER CUSTOM SETTINGS
#You can change values here. It's not recommended to change whole script
set(OPT "-O0")
set(USER_LINKER_FLAGS "-u _printf_float")
set(USER_CFLAGS "")

#You cam add your own defenitions
add_definitions(-DIS_STM32=1)
#add_definitions(-DYour_defenition=here)

#generate compile_commands.json
set(CMAKE_EXPORT_COMPILE_COMMANDS "ON")
#USER END

#CMAKE
cmake_minimum_required (VERSION 2.6)
include(CMakeForceCompiler)

set(CMAKE_C_COMPILER_ID "Clang")
set(CMAKE_BUILD_TYPE "Debug")

enable_language(ASM)

set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)


# Configure the cross toolchain
if (CMAKE_C_COMPILER_ID STREQUAL "GNU")
  if (DEFINED ENV{CROSS_COMPILE})
    set(CMAKE_C_COMPILER $ENV{CROSS_COMPILE}gcc)
    set(CMAKE_CXX_COMPILER $ENV{CROSS_COMPILE}g++)
    set(CMAKE_ASM_COMPILER $ENV{CROSS_COMPILE}gcc)
    set(CMAKE_OBJCOPY $ENV{CROSS_COMPILE}objcopy)
    set(CMAKE_SIZE $ENV{CROSS_COMPILE}size)
    set(CMAKE_AR $ENV{CROSS_COMPILE}ar)
  else()
    # Test if the cross-compilation toolchain is setup
    find_program(CROSS_COMPILE_GCC arm-none-eabi-gcc)
    if (NOT CROSS_COMPILE_GCC)
      message(FATAL_ERROR "Either add your cross-compilation toolchain to your PATH or define the environment variable CROSS_COMPILE.")
    endif()

    set(CMAKE_C_COMPILER arm-none-eabi-gcc)
    set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
    set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)
    set(CMAKE_OBJCOPY arm-none-eabi-objcopy)
    set(CMAKE_SIZE arm-none-eabi-size)
    set(CMAKE_AR arm-none-eabi-ar)
  endif()

  if(WIN32)
    # Redefine the linker command line as some GCC versions do not implement '--out-implib'
    set(CMAKE_C_LINK_EXECUTABLE "<CMAKE_C_COMPILER> <FLAGS> <CMAKE_C_LINK_FLAGS> <LINK_FLAGS> <OBJECTS>  -o <TARGET> <LINK_LIBRARIES>")
  endif()
elseif (CMAKE_C_COMPILER_ID STREQUAL "Clang")
  if (DEFINED ENV{CROSS_COMPILE})
    set(CMAKE_ASM_COMPILER $ENV{CROSS_COMPILE}gcc)
    set(CMAKE_OBJCOPY $ENV{CROSS_COMPILE}objcopy)
    set(CMAKE_SIZE $ENV{CROSS_COMPILE}size)
    # CMake generally calls CMAKE_C_COMPILER to link the executable. Clang invokes itself the linker installed on the host machine
    set(CMAKE_C_LINK_EXECUTABLE "$ENV{CROSS_COMPILE}gcc <CMAKE_C_LINK_FLAGS> <LINK_FLAGS> <OBJECTS>  -o <TARGET> <LINK_LIBRARIES>")
    set(CMAKE_CXX_LINK_EXECUTABLE "$ENV{CROSS_COMPILE}g++ <CMAKE_CXX_LINK_FLAGS> <LINK_FLAGS> <OBJECTS>  -o <TARGET> <LINK_LIBRARIES>")
    set(CROSS_COMPILE_GCC $ENV{CROSS_COMPILE}gcc)
  else()
    # Test if the cross-compilation toolchain is setup
    find_program(CROSS_COMPILE_GCC arm-none-eabi-gcc)
    if (NOT CROSS_COMPILE_GCC)
      message(FATAL_ERROR "Either add your cross-compilation toolchain to your PATH or define the environment variable CROSS_COMPILE.")
    endif()

    set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)
    set(CMAKE_OBJCOPY arm-none-eabi-objcopy)
    set(CMAKE_SIZE arm-none-eabi-size)
    # CMake generally calls CMAKE_C_COMPILER to link the executable. Clang invokes itself the linker installed on the host machine
    set(CMAKE_C_LINK_EXECUTABLE "arm-none-eabi-gcc <CMAKE_C_LINK_FLAGS> <LINK_FLAGS> <OBJECTS> -o <TARGET> <LINK_LIBRARIES>")
    set(CMAKE_CXX_LINK_EXECUTABLE "arm-none-eabi-g++ <CMAKE_CXX_LINK_FLAGS> <LINK_FLAGS> <OBJECTS> -o <TARGET> <LINK_LIBRARIES>")
  endif()
else()
  message(FATAL_ERROR "${CMAKE_C_COMPILER_ID} Toolchain not supported")
endif()

if(WIN32)
  # Prevent Windows Standard Libraries to be added
  set(CMAKE_C_STANDARD_LIBRARIES "")
endif()

#
# Macro to get the list of include path of GCC
#
MACRO(GET_GCC_INCLUDE_PATH is_cxx gcc_path gcc_include_path)
  if (${is_cxx} STREQUAL "TRUE")
    if (WIN32)
      execute_process(COMMAND ${gcc_path} -v -x c++ -E NUL ERROR_VARIABLE _gcc_output OUTPUT_QUIET)
    else()
      execute_process(COMMAND ${gcc_path} -v -x c++ -E - INPUT_FILE /dev/null ERROR_VARIABLE _gcc_output OUTPUT_QUIET)
    endif()
  else()
    if (WIN32)
      execute_process(COMMAND ${gcc_path} -v -x c -E NUL ERROR_VARIABLE _gcc_output OUTPUT_QUIET)
    else()
      execute_process(COMMAND ${gcc_path} -v -x c -E - INPUT_FILE /dev/null ERROR_VARIABLE _gcc_output OUTPUT_QUIET)
    endif()
  endif()

  # Build an array of string from the GCC output
  string(REPLACE "\n" ";" _gcc_output "${_gcc_output}")

  set(_capture_include FALSE)
  set(_include_path "")

  # Go through the lines and capture between '"#include <...> search starts here:"' and 'End of search list.'
  foreach(_line ${_gcc_output})
    if(${_line} STREQUAL "End of search list.")
      set(_capture_include FALSE)
    endif()

    if(_capture_include)
      # Remove the leading and trailing empty characters
      string(REPLACE "\r" "" _line ${_line})
      string(SUBSTRING "${_line}" 1 -1 _line)

      set(_include_path "${_include_path} -I${_line}")
    endif()

    if(${_line} STREQUAL "#include <...> search starts here:")
      set(_capture_include TRUE)
    endif()
  endforeach()
  set(${gcc_include_path} ${_include_path})
ENDMACRO()

#
# Toolchain support
#
set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -DDEBUG")
set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -g -Wno-unused-but-set-variable -Wno-unused-variable")
if (CMAKE_C_COMPILER_ID STREQUAL "GNU")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=gnu99 -fno-common -fmessage-length=0 -Wall -fno-exceptions -ffunction-sections -fdata-sections -fomit-frame-pointer")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-common -fmessage-length=0 -Wall -fno-exceptions -ffunction-sections -fdata-sections -fomit-frame-pointer")
  set(CMAKE_ASM_FLAGS "${CMAKE_ASM_FLAGS} -x assembler-with-cpp")
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--gc-sections")
  set(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS "")
  set(CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS "")
elseif (CMAKE_C_COMPILER_ID STREQUAL "Clang")
  # Retrieve the GCC include paths for C and C++
  GET_GCC_INCLUDE_PATH(FALSE ${CROSS_COMPILE_GCC} CROSS_COMPILE_GCC_C_INCLUDE_PATH)
  GET_GCC_INCLUDE_PATH(TRUE ${CROSS_COMPILE_GCC} CROSS_COMPILE_GCC_CXX_INCLUDE_PATH)

  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -target arm-none-eabi -std=gnu99 -fno-common -fmessage-length=0 -Wall -fno-exceptions -ffunction-sections -fdata-sections -fomit-frame-pointer -fshort-enums ${CROSS_COMPILE_GCC_C_INCLUDE_PATH}")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -target arm-none-eabi -fno-common -fmessage-length=0 -Wall -fno-exceptions -ffunction-sections -fdata-sections -fomit-frame-pointer -fshort-enums ${CROSS_COMPILE_GCC_CXX_INCLUDE_PATH}")
  set(CMAKE_ASM_FLAGS "${CMAKE_ASM_FLAGS} -x assembler-with-cpp")
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--gc-sections")
  set(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS "")
  set(CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS "")
  set(EXTERN_C_FLAGS "-target arm-none-eabi")
  set(EXTERN_CXX_FLAGS "-target arm-none-eabi")

  # Prevent the warning related to non supported function attribute - see: https://sourceware.org/ml/newlib/2015/msg00714.html
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-unknown-attributes")
  set(EXTERN_C_FLAGS "${EXTERN_C_FLAGS} -Wno-unknown-attributes ${CROSS_COMPILE_GCC_C_INCLUDE_PATH}")
else()
  message(FATAL_ERROR "${CMAKE_C_COMPILER_ID} Toolchain not supported")
endif()

#set(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS "") #remove -rdynamic

#LOAD CUBE MX CONFIGURATION
#USING PYTHON SCRIPT CubeMX2_cmake.py
set(cubemx_dir ${CMAKE_CURRENT_LIST_DIR})
message("CubeMX dir: " ${cubemx_dir})
string(REPLACE " " "" cubemx_dir ${cubemx_dir})
execute_process(COMMAND bash "-c"
  "python ${cubemx_dir}/CubeMX2_cmake.py ${cubemx_dir}"
  OUTPUT_VARIABLE cubemx_conf
  RESULT_VARIABLE cubemx_res
  ERROR_VARIABLE  cubemx_err)
if(${cubemx_res} EQUAL "0")
  message("CubeMX ok")
else()
  message(FATAL_ERROR ${cubemx_err})
endif()

foreach(i ${cubemx_conf})
  message(${i})
endforeach()

list(GET cubemx_conf 0 STM32_PRJ_NAME     )
list(GET cubemx_conf 1 STM32_PRJ_CFLAGS   )
list(GET cubemx_conf 2 STM32_PRJ_DEFS     )
list(GET cubemx_conf 3 STM32_PRJ_MCU      )
list(GET cubemx_conf 4 STM32_PRJ_LD_SCRIPT)

message("CUBE project name: " ${STM32_PRJ_NAME})
message("CUBE MCU: "          ${STM32_PRJ_MCU})
message("CUBE CFLAGS: "       ${STM32_PRJ_CFLAGS})
message("CUBE DEFs: "         ${STM32_PRJ_DEFS})
message("CUBE LD script "     ${STM32_PRJ_LD_SCRIPT})

string(REPLACE " " "" STM32_PRJ_NAME ${STM32_PRJ_NAME})
string(REPLACE "\"" "" STM32_PRJ_DEFS ${STM32_PRJ_DEFS})
string(REPLACE " " ";" STM32_PRJ_DEFS ${STM32_PRJ_DEFS})
string(REPLACE " " "" STM32_PRJ_LD_SCRIPT ${STM32_PRJ_LD_SCRIPT})
string(REPLACE "\n" "" STM32_PRJ_LD_SCRIPT ${STM32_PRJ_LD_SCRIPT})
set(STM32_PRJ_LD_SCRIPT "${cubemx_dir}/${STM32_PRJ_LD_SCRIPT}")

#start project
project(${STM32_PRJ_NAME})

#find and add all headers & sources & asm to target
MACRO(HEADER_DIRECTORIES return_list)
    FILE(GLOB_RECURSE new_list *.h)
    SET(dir_list "")
    FOREACH(file_path ${new_list})
        GET_FILENAME_COMPONENT(dir_path ${file_path} PATH)
        SET(dir_list ${dir_list} ${dir_path})
    ENDFOREACH()
    LIST(REMOVE_DUPLICATES dir_list)
    SET(${return_list} ${dir_list})
ENDMACRO()


#HEADERS
header_directories(INC)
#set(INC "${INC};/usr/lib/gcc/arm-none-eabi/7.2.0/include;/usr/lib/gcc/arm-none-eabi/7.2.0/include-fixed;/usr/arm-none-eabi/include/")
include_directories(${INC})

#SOURCES
file(GLOB_RECURSE SRC RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} *.c)
string(REGEX REPLACE "[^;]*CMakeFiles/[^;]+;?" "" SRC "${SRC}")

#ASSEMBLER files
file(GLOB_RECURSE asm_SRC "*.s")
set_source_files_properties(${asm_SRC} "-x assembler-with-cpp")

#list all files
message("INCLUDES:")
set(incs "")
foreach(f ${INC})
  set(incs "${incs} -I${f}")
  message(${f})
endforeach()
message("SOURCES: ")
foreach(f ${SRC})
  message(${f})
endforeach()
message("DEFINITIONS: ")
foreach(def ${STM32_PRJ_DEFS})
  message(${def})
  add_definitions("${def}")
endforeach()

#setup flags

set(CMAKE_C_FLAGS "${EXTERN_C_FLAGS} ${CMAKE_C_FLAGS} ${STM32_PRJ_CFLAGS} ${OPT} ${USER_CFLAGS}")
set(CMAKE_CXX_FLAGS "${EXTERN_C_FLAGS} ${CMAKE_CXX_FLAGS} ${STM32_PRJ_CFLAGS} ${OPT} ${USER_CFLAGS}")
set(CMAKE_ASM_FLAGS "${CMAKE_ASM_FLAGS} ${STM32_PRJ_CFLAGS} ${OPT} ${USER_CFLAGS}")


set(LDSCRIPT "-T${STM32_PRJ_LD_SCRIPT}")
set(LINKER_FLAGS "-specs=nosys.specs ${LDSCRIPT} -lc -lm -lnosys -Wl,--gc-sections ${STM32_PRJ_CFLAGS} ${OPT} ${USER_LINKER_FLAGS} ")
SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${LINKER_FLAGS}")


SET(TARGET ${CMAKE_PROJECT_NAME})
message(${CMAKE_C_COMPILER})

#setup targets
get_directory_property(C_DEFS COMPILE_DEFINITIONS)
#.elf
add_executable(${TARGET}.elf ${SRC} ${asm_SRC})
#print size
add_custom_command(TARGET ${TARGET}.elf POST_BUILD COMMAND ${CMAKE_SIZE} ${TARGET}.elf)

#other
ADD_CUSTOM_TARGET(${TARGET}.hex DEPENDS ${TARGET}.elf COMMAND ${CMAKE_OBJCOPY} -Oihex ${TARGET} ${TARGET}.hex)
ADD_CUSTOM_TARGET(${TARGET}.bin DEPENDS ${TARGET}.elf COMMAND ${CMAKE_OBJCOPY} -Obinary ${TARGET} ${TARGET}.bin)
