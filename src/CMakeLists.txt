if(HAVE_BFD_DISASM)
  set(BFD_DISASM_SRC bfd-disasm.cpp)
endif()

if(BUILD_FUZZ)
  message(STATUS "Build for fuzzing")
  set(MAIN_SRC fuzz_main.cpp)
  set(BPFTRACE bpftrace_fuzz)
else()
  set(MAIN_SRC main.cpp)
  if (NOT DEFINED BPFTRACE)
    set(BPFTRACE bpftrace)
  endif ()
endif()

add_library(runtime
  attached_probe.cpp
  bpffeature.cpp
  bpftrace.cpp
  btf.cpp
  child.cpp
  disasm.cpp
  dwarf_parser.cpp
  fake_map.cpp
  format_string.cpp
  imap.cpp
  log.cpp
  map.cpp
  mapkey.cpp
  output.cpp
  probe_matcher.cpp
  procmon.cpp
  printf.cpp
  relocator.cpp
  resolve_cgroupid.cpp
  required_resources.cpp
  struct.cpp
  tracefs.cpp
  debugfs.cpp
  types.cpp
  usdt.cpp
  utils.cpp
  pcap_writer.cpp
  ${BFD_DISASM_SRC}
)
# Ensure flex+bison outputs are built first
add_dependencies(runtime parser)

add_library(libbpftrace
  build_info.cpp
  clang_parser.cpp
  driver.cpp
  lockdown.cpp
  tracepoint_format_parser.cpp
)
# So it's not "liblibbpftrace"
set_target_properties(libbpftrace PROPERTIES PREFIX "")

add_executable(${BPFTRACE}
  ${MAIN_SRC}
)

install(TARGETS ${BPFTRACE} DESTINATION ${CMAKE_INSTALL_BINDIR})
target_link_libraries(${BPFTRACE} libbpftrace)

if (BUILD_FUZZ)
  target_compile_options(${BPFTRACE} PUBLIC "-DFUZZ")

  if(FUZZ_TARGET STREQUAL "semantic")
      message(STATUS "Fuzzing seantic analyzer")
      target_compile_options(${BPFTRACE} PUBLIC -DTEST_SEMANTIC)
  elseif(FUZZ_TARGET STREQUAL "codegen")
      message(STATUS "Fuzzing codegen")
  else()
      message(FATAL_ERROR "Unsupported FUZZ_TARGET")
  endif()

  if (USE_LIBFUZZER)
      if (NOT CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
          message(FATAL_ERROR "Use Clang for libfuzzer")
      endif()
      target_compile_options(${BPFTRACE} PUBLIC "-DLIBFUZZER")
      message(STATUS "Use libfuzzer")
  endif()
endif()

# compile definitions

set(KERNEL_HEADERS_DIR "" CACHE PATH "Hard-code kernel headers directory")
if (KERNEL_HEADERS_DIR)
  MESSAGE(STATUS "Using KERNEL_HEADERS_DIR=${KERNEL_HEADERS_DIR}")
  target_compile_definitions(runtime PUBLIC KERNEL_HEADERS_DIR="${KERNEL_HEADERS_DIR}")
endif()

execute_process(
  COMMAND git describe --abbrev=4 --dirty --tags
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
  OUTPUT_VARIABLE BPFTRACE_VERSION
  ERROR_VARIABLE GIT_DESCRIBE_ERROR
  OUTPUT_STRIP_TRAILING_WHITESPACE
  RESULT_VARIABLE retcode
)

# If the build is not done from a git repo, get the version information from
# the version variables in main CMakeLists.txt
if(NOT "${retcode}" STREQUAL "0")
  set(BPFTRACE_VERSION "v${bpftrace_VERSION_MAJOR}.${bpftrace_VERSION_MINOR}.${bpftrace_VERSION_PATCH}")
endif()

add_definitions("-DBPFTRACE_VERSION=\"${BPFTRACE_VERSION}\"")

target_include_directories(runtime PRIVATE ${CMAKE_BINARY_DIR})
target_include_directories(runtime PRIVATE ${CMAKE_SOURCE_DIR}/src)
target_include_directories(runtime PRIVATE ${CMAKE_SOURCE_DIR}/src/ast)
target_compile_definitions(runtime PRIVATE ${BPFTRACE_FLAGS})
target_compile_definitions(libbpftrace PRIVATE ${BPFTRACE_FLAGS})

# Linking
if(STATIC_LINKING)
  if(STATIC_LIBC)
    target_link_options(${BPFTRACE} BEFORE PRIVATE "-static")
  else()
    target_link_options(${BPFTRACE} BEFORE PRIVATE "-static-libgcc" "-static-libstdc++")
  endif(STATIC_LIBC)
endif(STATIC_LINKING)


target_link_libraries(runtime ${LIBBPF_LIBRARIES} ${ZLIB_LIBRARIES})
target_link_libraries(libbpftrace parser resources runtime aot ast arch cxxdemangler_llvm)

if(LIBPCAP_FOUND)
  target_link_libraries(libbpftrace ${LIBPCAP_LIBRARIES})
endif(LIBPCAP_FOUND)

if(HAVE_BFD_DISASM)
  if(STATIC_LINKING)
    add_library(LIBBFD STATIC IMPORTED)
    set_property(TARGET LIBBFD PROPERTY IMPORTED_LOCATION ${LIBBFD_LIBRARIES})
    target_link_libraries(runtime LIBBFD)
    add_library(LIBOPCODES STATIC IMPORTED)
    set_property(TARGET LIBOPCODES PROPERTY IMPORTED_LOCATION ${LIBOPCODES_LIBRARIES})
    target_link_libraries(runtime LIBOPCODES)
    add_library(LIBIBERTY STATIC IMPORTED)
    set_property(TARGET LIBIBERTY PROPERTY IMPORTED_LOCATION ${LIBIBERTY_LIBRARIES})
    target_link_libraries(runtime LIBIBERTY)
  else()
    target_link_libraries(runtime ${LIBBFD_LIBRARIES})
    target_link_libraries(runtime ${LIBOPCODES_LIBRARIES})
  endif(STATIC_LINKING)
endif(HAVE_BFD_DISASM)

# Link to bcc libraries (without LLVM) if possible
if(LIBBCC_BPF_CONTAINS_RUNTIME)
  target_link_libraries(runtime ${LIBBCC_BPF_LIBRARIES})
else()
  target_link_libraries(runtime ${LIBBCC_LIBRARIES})
endif()

if(STATIC_BPF_BCC)
  # These are not part of the static libbcc so have to be added separate
  target_link_libraries(runtime ${LIBBCC_BPF_LIBRARIES})
  target_link_libraries(runtime ${LIBBPF_LIBRARIES})
  target_link_libraries(runtime ${LIBBCC_LOADER_LIBRARY_STATIC})

  add_library(LIBELF STATIC IMPORTED)
  set_property(TARGET LIBELF PROPERTY IMPORTED_LOCATION ${LIBELF_LIBRARIES})
  target_link_libraries(runtime LIBELF)
else()
  target_link_libraries(runtime ${LIBELF_LIBRARIES})
endif(STATIC_BPF_BCC)

if (LIBDW_FOUND)
  if(STATIC_LINKING)
    find_package(LibBz2)
    find_package(LibLzma)
    find_package(LibEbl)

    add_library(LIBBZ2 STATIC IMPORTED)
    set_property(TARGET LIBBZ2 PROPERTY IMPORTED_LOCATION ${LIBBZ2_LIBRARIES})

    add_library(LIBLZMA STATIC IMPORTED)
    set_property(TARGET LIBLZMA PROPERTY IMPORTED_LOCATION ${LIBLZMA_LIBRARIES})

    add_library(LIBDW STATIC IMPORTED)
    set_property(TARGET LIBDW PROPERTY IMPORTED_LOCATION ${LIBDW_LIBRARIES})

    set(LIBDW_LIBS LIBBZ2 LIBELF LIBLZMA)

    if (${LIBEBL_FOUND})
      # libebl is not necessary on some systems (e.g. Alpine)
      add_library(LIBEBL STATIC IMPORTED)
      set_property(TARGET LIBEBL PROPERTY IMPORTED_LOCATION ${LIBEBL_LIBRARIES})
      set(LIBDW_LIBS ${LIBDW_LIBS} LIBEBL)
    endif()

    target_link_libraries(LIBDW INTERFACE ${LIBDW_LIBS})

    target_link_libraries(runtime LIBDW)
  else()
    target_link_libraries(runtime ${LIBDW_LIBRARIES})
  endif()
endif()

# Support for std::filesystem
# GCC version <9 and Clang (all versions) require -lstdc++fs
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang" OR ${CMAKE_CXX_COMPILER_VERSION} VERSION_LESS "9")
  target_link_libraries(runtime "stdc++fs")
  target_link_libraries(libbpftrace "stdc++fs")
endif()

if (BUILD_ASAN)
  target_compile_options(${BPFTRACE} PUBLIC "-fsanitize=address")
  target_link_options(${BPFTRACE} PUBLIC "-fsanitize=address")
endif()

if (USE_LIBFUZZER)
  if (BUILD_ASAN)
    target_compile_options(${BPFTRACE} PUBLIC "-fsanitize=fuzzer,address")
    target_link_options(${BPFTRACE} PUBLIC "-fsanitize=fuzzer")
  else()
    target_compile_options(${BPFTRACE} PUBLIC "-fsanitize=fuzzer")
    target_link_options(${BPFTRACE} PUBLIC "-fsanitize=fuzzer,address")
  endif()
endif()

if (STATIC_LINKING)
  if(ANDROID)
    target_link_libraries(libbpftrace "-Wl,-Bdynamic" "-ldl" "-lm" "-lz")
    target_link_libraries(runtime "-Wl,-Bdynamic" "-ldl" "-lm" "-lz")
  else()
    target_link_libraries(libbpftrace "-Wl,-Bdynamic" "-lrt" "-lpthread" "-ldl" "-lm")
    target_link_libraries(libbpftrace "-Wl,-Bstatic" "-lz")
    target_link_libraries(runtime "-Wl,-Bdynamic" "-lrt" "-lpthread" "-ldl" "-lm")
    target_link_libraries(runtime "-Wl,-Bstatic" "-lz")
  endif()
elseif(STATIC_BPF_BCC)
  # partial static build, libbpf needs zlib, bcc needs liblzma
  target_link_libraries(runtime "-lz" "-llzma")
endif()

unset(MAIN_SRC)
unset(BPFTRACE)

add_subdirectory(aot)
add_subdirectory(arch)
add_subdirectory(ast)
add_subdirectory(cxxdemangler)
