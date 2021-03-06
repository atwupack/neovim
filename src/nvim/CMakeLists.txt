include(CheckLibraryExists)

set(GENERATED_DIR ${PROJECT_BINARY_DIR}/src/nvim/auto)
set(DISPATCH_GENERATOR ${PROJECT_SOURCE_DIR}/scripts/msgpack-gen.lua)
file(GLOB API_HEADERS api/*.h)
file(GLOB MSGPACK_RPC_HEADERS msgpack_rpc/*.h)
set(MSGPACK_DISPATCH ${GENERATED_DIR}/msgpack_dispatch.c)
set(HEADER_GENERATOR ${PROJECT_SOURCE_DIR}/scripts/gendeclarations.lua)
set(GENERATED_INCLUDES_DIR ${PROJECT_BINARY_DIR}/include)
set(GENERATED_EX_CMDS_ENUM ${GENERATED_INCLUDES_DIR}/ex_cmds_enum.generated.h)
set(GENERATED_EX_CMDS_DEFS ${GENERATED_DIR}/ex_cmds_defs.generated.h)
set(EX_CMDS_GENERATOR ${PROJECT_SOURCE_DIR}/scripts/genex_cmds.lua)
set(EX_CMDS_DEFS_FILE ${PROJECT_SOURCE_DIR}/src/nvim/ex_cmds.lua)

include_directories(${GENERATED_DIR})
include_directories(${GENERATED_INCLUDES_DIR})

file(MAKE_DIRECTORY ${GENERATED_DIR})
file(MAKE_DIRECTORY ${GENERATED_DIR}/os)
file(MAKE_DIRECTORY ${GENERATED_DIR}/api)
file(MAKE_DIRECTORY ${GENERATED_DIR}/api/private)
file(MAKE_DIRECTORY ${GENERATED_DIR}/msgpack_rpc)
file(MAKE_DIRECTORY ${GENERATED_INCLUDES_DIR})
file(MAKE_DIRECTORY ${GENERATED_INCLUDES_DIR}/os)
file(MAKE_DIRECTORY ${GENERATED_INCLUDES_DIR}/api)
file(MAKE_DIRECTORY ${GENERATED_INCLUDES_DIR}/api/private)
file(MAKE_DIRECTORY ${GENERATED_INCLUDES_DIR}/msgpack_rpc)

file(GLOB NEOVIM_SOURCES *.c os/*.c api/*.c api/private/*.c msgpack_rpc/*.c)
file(GLOB_RECURSE NEOVIM_HEADERS *.h)

foreach(sfile ${NEOVIM_SOURCES})
  get_filename_component(f ${sfile} NAME)
  if(${f} MATCHES "^(regexp_nfa.c)$")
    list(APPEND to_remove ${sfile})
  endif()
endforeach()

list(REMOVE_ITEM NEOVIM_SOURCES ${to_remove})

set(CONV_SOURCES
  arabic.c
  cursor.c
  garray.c
  hashtab.c
  log.c
  map.c
  memfile.c
  memory.c
  misc2.c
  profile.c
  tempfile.c
  )

foreach(sfile ${CONV_SOURCES})
  if(NOT EXISTS "${PROJECT_SOURCE_DIR}/src/nvim/${sfile}")
    message(FATAL_ERROR "${sfile} doesn't exist(it was added to CONV_SOURCES)")
  endif()
endforeach()

file(GLOB_RECURSE EXTRA_CONV_SOURCES os/*.c api/*.c msgpack_rpc/*.c)
foreach(sfile ${EXTRA_CONV_SOURCES})
  file(RELATIVE_PATH f "${PROJECT_SOURCE_DIR}/src/nvim" "${sfile}")
  list(APPEND CONV_SOURCES ${f})
endforeach()

set_source_files_properties(
  ${CONV_SOURCES} PROPERTIES COMPILE_FLAGS "${COMPILE_FLAGS} -Wconversion")

if(CMAKE_C_COMPILER_ID MATCHES "Clang")
  if(DEFINED ENV{SANITIZE})
    message(STATUS "Enabling the sanitizers")
    add_definitions(-DEXITFREE) # is this necessary for LeakSanitizer?
    add_definitions(-fno-sanitize-recover -fno-omit-frame-pointer
      -fno-optimize-sibling-calls -fsanitize=address -fsanitize=undefined)
    set(CMAKE_EXE_LINKER_FLAGS
      "-fsanitize=address -fsanitize=undefined ${CMAKE_EXE_LINKER_FLAGS}")
    set(CMAKE_SHARED_LINKER_FLAGS
      "-fsanitize=address -fsanitize=undefined ${CMAKE_SHARED_LINKER_FLAGS}")
  endif()
endif()

get_directory_property(gen_cdefs COMPILE_DEFINITIONS)
foreach(gen_cdef ${gen_cdefs})
  set(gen_cflags "${gen_cflags} -D${gen_cdef}")
endforeach()

get_directory_property(gen_includes INCLUDE_DIRECTORIES)
foreach(gen_include ${gen_includes})
  set(gen_cflags "${gen_cflags} -I${gen_include}")
endforeach()
string(TOUPPER "${CMAKE_BUILD_TYPE}" _build_type)
set(gen_cflags "${gen_cflags} ${CMAKE_C_FLAGS_${_build_type}} ${CMAKE_C_FLAGS}")

foreach(sfile ${NEOVIM_SOURCES}
              "${PROJECT_SOURCE_DIR}/src/nvim/regexp_nfa.c")
  get_filename_component(full_d ${sfile} PATH)
  file(RELATIVE_PATH d "${PROJECT_SOURCE_DIR}/src/nvim" "${full_d}")
  get_filename_component(f ${sfile} NAME)
  get_filename_component(r ${sfile} NAME_WE)
  if(NOT ${d} EQUAL ".")
    set(f "${d}/${f}")
    set(r "${d}/${r}")
  endif()
  set(gf1 "${GENERATED_DIR}/${r}.c.generated.h")
  set(gf2 "${GENERATED_INCLUDES_DIR}/${r}.h.generated.h")
  add_custom_command(
    OUTPUT "${gf1}" "${gf2}"
    COMMAND "${LUA_PRG}" "${HEADER_GENERATOR}"
                             "${sfile}" "${gf1}" "${gf2}"
                             "${CMAKE_C_COMPILER} ${gen_cflags} -E"
    DEPENDS "${HEADER_GENERATOR}" "${sfile}"
    )
  list(APPEND NEOVIM_GENERATED_SOURCES "${gf1}")
  list(APPEND NEOVIM_GENERATED_SOURCES "${gf2}")
  if(${d} MATCHES "^api$" AND NOT ${f} MATCHES "^api/helpers.c$")
    list(APPEND API_HEADERS ${gf2})
  endif()
endforeach()

add_custom_command(OUTPUT ${MSGPACK_DISPATCH}
  COMMAND ${LUA_PRG} ${DISPATCH_GENERATOR} ${API_HEADERS} ${MSGPACK_DISPATCH}
  DEPENDS
    ${API_HEADERS}
    ${MSGPACK_RPC_HEADERS}
    ${DISPATCH_GENERATOR}
)

list(APPEND NEOVIM_GENERATED_SOURCES
  "${PROJECT_BINARY_DIR}/config/auto/pathdef.c"
  "${MSGPACK_DISPATCH}"
  "${GENERATED_EX_CMDS_ENUM}"
  "${GENERATED_EX_CMDS_DEFS}"
)

add_custom_command(OUTPUT ${GENERATED_EX_CMDS_ENUM} ${GENERATED_EX_CMDS_DEFS}
  COMMAND ${LUA_PRG} ${EX_CMDS_GENERATOR}
      ${PROJECT_SOURCE_DIR}/src/nvim ${GENERATED_INCLUDES_DIR} ${GENERATED_DIR}
  DEPENDS ${EX_CMDS_GENERATOR} ${EX_CMDS_DEFS_FILE}
)

# Our dependencies come first.

if (LibIntl_FOUND)
  list(APPEND NVIM_LINK_LIBRARIES ${LibIntl_LIBRARY})
endif()

check_library_exists(curses tgetent "" HAVE_LIBCURSES)
if (HAVE_LIBCURSES)
  list(APPEND NVIM_LINK_LIBRARIES curses)
else()
  check_library_exists(tinfo tgetent "" HAVE_LIBTINFO)
  if (HAVE_LIBTINFO)
    list(APPEND NVIM_LINK_LIBRARIES tinfo)
  else()
    find_package(Curses REQUIRED)
    list(APPEND NVIM_LINK_LIBRARIES ${CURSES_LIBRARY})
  endif()
endif()

if(HAVE_ICONV_LIB)
  list(APPEND NVIM_LINK_LIBRARIES iconv)
endif()

# Put these last on the link line, since multiple things may depend on them.
list(APPEND NVIM_LINK_LIBRARIES
    ${LIBUV_LIBRARIES}
    ${MSGPACK_LIBRARIES}
    ${LUAJIT_LIBRARIES}
    m
    ${CMAKE_THREAD_LIBS_INIT})

if(NOT DEFINED ENV{SKIP_EXEC})
  add_executable(nvim ${NEOVIM_GENERATED_SOURCES} ${NEOVIM_SOURCES}
    ${NEOVIM_HEADERS})
  target_link_libraries(nvim ${NVIM_LINK_LIBRARIES})
  install_helper(TARGETS nvim)
endif()

if(NOT DEFINED ENV{SKIP_UNITTEST})
  add_library(nvim-test MODULE EXCLUDE_FROM_ALL ${NEOVIM_GENERATED_SOURCES}
    ${NEOVIM_SOURCES} ${NEOVIM_HEADERS})
  target_link_libraries(nvim-test ${NVIM_LINK_LIBRARIES})
endif()

add_subdirectory(po)
