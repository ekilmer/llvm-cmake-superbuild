# Based off https://github.com/friendlyanon/cmake-init-gif-engine/blob/b1bf3b75c1e724ceadc4eaf11bd61a29ddadd5a5/fuzz/CMakeLists.txt
cmake_minimum_required(VERSION 3.20.0)

project(LLVMSuperBuild NONE)

# Allow the Ninja generators to output messages as they happen by assigning
# these jobs to the 'console' job pool
set(console_access "")
if(CMAKE_GENERATOR MATCHES "^Ninja")
  set(
      console_access
      USES_TERMINAL_CONFIGURE YES
      USES_TERMINAL_BUILD YES
      USES_TERMINAL_INSTALL YES
  )
endif()

set(toolchain "")
if(DEFINED CMAKE_TOOLCHAIN_FILE)
  set(toolchain "-DCMAKE_TOOLCHAIN_FILE:STRING=${CMAKE_TOOLCHAIN_FILE}")
endif()

set(common_flags
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
    -DLLVM_CCACHE_BUILD:BOOL=ON
    -DLLVM_INCLUDE_TESTS:BOOL=OFF
    # Needed for installing tblgen
    -DLLVM_BUILD_UTILS:BOOL=ON
)


include(ExternalProject)

# **** LLVM ****
ExternalProject_Add(
    llvm
    SOURCE_DIR "${PROJECT_SOURCE_DIR}/llvm-project/llvm"
    CMAKE_CACHE_ARGS
    ${toolchain}
    ${common_flags}
    # If we want to test usage of build directory
    STEP_TARGETS build
    BUILD_ALWAYS YES
    # EXCLUDE_FROM_ALL YES
    ${console_access}
    CONFIGURE_HANDLED_BY_BUILD YES
)
ExternalProject_Get_Property(llvm install_dir)
set(llvm_install_dir "${install_dir}")
ExternalProject_Get_Property(llvm binary_dir)
set(llvm_build_dir "${binary_dir}")

# **** Clang ****
ExternalProject_Add(
    clang
    SOURCE_DIR "${PROJECT_SOURCE_DIR}/llvm-project/clang"
    CMAKE_CACHE_ARGS
    -DLLVM_DIR:PATH=${llvm_install_dir}/lib/cmake/llvm
    ${toolchain}
    ${common_flags}
    BUILD_ALWAYS YES
    # EXCLUDE_FROM_ALL YES
    DEPENDS llvm
    ${console_access}
    CONFIGURE_HANDLED_BY_BUILD YES
)
ExternalProject_Get_Property(clang install_dir)
set(clang_install_dir "${install_dir}")
ExternalProject_Get_Property(clang binary_dir)
set(clang_build_dir "${binary_dir}")

# **** libc ****
set(libc_plat_opts "")
if(CMAKE_SYSTEM MATCHES Linux)
    set(libc_plat_opts
        -DLLVM_LIBC_FULL_BUILD:BOOL=ON
    )
endif()
ExternalProject_Add(
    libc
    SOURCE_DIR "${PROJECT_SOURCE_DIR}/llvm-project/runtimes"
    CMAKE_CACHE_ARGS
    ${libc_plat_opts}
    -DLLVM_ENABLE_RUNTIMES:STRING=libc
    -DLLVM_DIR:PATH=${llvm_install_dir}/lib/cmake/llvm
    -DClang_DIR:PATH=${clang_install_dir}/lib/cmake/clang
    ${toolchain}
    ${common_flags}
    INSTALL_COMMAND ${CMAKE_COMMAND} --build <BINARY_DIR> --target install-libc
    BUILD_ALWAYS YES
    # EXCLUDE_FROM_ALL YES
    DEPENDS llvm clang
    ${console_access}
    CONFIGURE_HANDLED_BY_BUILD YES
)
ExternalProject_Get_Property(libc install_dir)
set(libc_install_dir "${install_dir}")
ExternalProject_Get_Property(libc binary_dir)
set(libc_build_dir "${binary_dir}")

# **** libc++ ****
ExternalProject_Add(
    libcxx
    SOURCE_DIR "${PROJECT_SOURCE_DIR}/llvm-project/runtimes"
    CMAKE_CACHE_ARGS
    -DLLVM_ENABLE_RUNTIMES:STRING=libcxx\;libcxxabi\;libunwind
    -DLLVM_DIR:PATH=${llvm_install_dir}/lib/cmake/llvm
    -DClang_DIR:PATH=${clang_install_dir}/lib/cmake/clang
    ${toolchain}
    ${common_flags}
    BUILD_ALWAYS YES
    # EXCLUDE_FROM_ALL YES
    DEPENDS llvm clang
    ${console_access}
    CONFIGURE_HANDLED_BY_BUILD YES
)
ExternalProject_Get_Property(libcxx install_dir)
set(libcxx_install_dir "${install_dir}")
ExternalProject_Get_Property(libcxx binary_dir)
set(libcxx_build_dir "${binary_dir}")

# **** compiler-rt ****
ExternalProject_Add(
    compiler-rt
    SOURCE_DIR "${PROJECT_SOURCE_DIR}/llvm-project/compiler-rt"
    CMAKE_CACHE_ARGS
    -DLLVM_DIR:PATH=${llvm_install_dir}/lib/cmake/llvm
    ${toolchain}
    ${common_flags}
    BUILD_ALWAYS YES
    # EXCLUDE_FROM_ALL YES
    DEPENDS llvm
    ${console_access}
    CONFIGURE_HANDLED_BY_BUILD YES
)
ExternalProject_Get_Property(compiler-rt install_dir)
set(compiler-rt_install_dir "${install_dir}")
ExternalProject_Get_Property(compiler-rt binary_dir)
set(compiler-rt_build_dir "${binary_dir}")

# **** libclc ****
# TODO: Requires llvm-spirv, which can be built by following directions in
#   clang's UsersManual.rst under "SPIR-V Support"
# ExternalProject_Add(
#     libclc
#     SOURCE_DIR "${PROJECT_SOURCE_DIR}/llvm-project/libclc"
#     CMAKE_CACHE_ARGS
#     -DLLVM_DIR:PATH=${llvm_install_dir}/lib/cmake/llvm
#     -DLLVM_CLANG:PATH=${clang_install_dir}/bin/clang
#     -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
#     -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
#     ${toolchain}
#     BUILD_ALWAYS YES
#     EXCLUDE_FROM_ALL YES
#     DEPENDS llvm clang
#     ${console_access}
#     CONFIGURE_HANDLED_BY_BUILD YES
# )
# ExternalProject_Get_Property(libclc install_dir)
# set(libclc_install_dir "${install_dir}")
# ExternalProject_Get_Property(libclc binary_dir)
# set(libclc_build_dir "${binary_dir}")

# **** LLD ****
ExternalProject_Add(
    lld
    SOURCE_DIR "${PROJECT_SOURCE_DIR}/llvm-project/lld"
    CMAKE_CACHE_ARGS
    -DLLVM_DIR:PATH=${llvm_install_dir}/lib/cmake/llvm
    ${toolchain}
    ${common_flags}
    BUILD_ALWAYS YES
    # EXCLUDE_FROM_ALL YES
    DEPENDS llvm
    ${console_access}
    CONFIGURE_HANDLED_BY_BUILD YES
)
ExternalProject_Get_Property(lld install_dir)
set(lld_install_dir "${install_dir}")
ExternalProject_Get_Property(lld binary_dir)
set(lld_build_dir "${binary_dir}")

# **** LLDB ****
ExternalProject_Add(
    lldb
    SOURCE_DIR "${PROJECT_SOURCE_DIR}/llvm-project/lldb"
    CMAKE_CACHE_ARGS
    -DLLVM_DIR:PATH=${llvm_install_dir}/lib/cmake/llvm
    -DClang_DIR:PATH=${clang_install_dir}/lib/cmake/clang
    -DLLDB_INCLUDE_TESTS:BOOL=OFF
    ${toolchain}
    ${common_flags}
    BUILD_ALWAYS YES
    # EXCLUDE_FROM_ALL YES
    DEPENDS llvm clang
    ${console_access}
    CONFIGURE_HANDLED_BY_BUILD YES
)
ExternalProject_Get_Property(lldb install_dir)
set(lldb_install_dir "${install_dir}")
ExternalProject_Get_Property(lldb binary_dir)
set(lldb_build_dir "${binary_dir}")

# **** MLIR ****
ExternalProject_Add(
    mlir
    SOURCE_DIR "${PROJECT_SOURCE_DIR}/llvm-project/mlir"
    CMAKE_CACHE_ARGS
    -DLLVM_DIR:PATH=${llvm_install_dir}/lib/cmake/llvm
    ${toolchain}
    ${common_flags}
    BUILD_ALWAYS YES
    # EXCLUDE_FROM_ALL YES
    DEPENDS llvm
    ${console_access}
    CONFIGURE_HANDLED_BY_BUILD YES
)
ExternalProject_Get_Property(mlir install_dir)
set(mlir_install_dir "${install_dir}")
ExternalProject_Get_Property(mlir binary_dir)
set(mlir_build_dir "${binary_dir}")

# **** FLANG ****
ExternalProject_Add(
    flang
    SOURCE_DIR "${PROJECT_SOURCE_DIR}/llvm-project/flang"
    CMAKE_CACHE_ARGS
    -DLLVM_DIR:PATH=${llvm_install_dir}/lib/cmake/llvm
    -DClang_DIR:PATH=${clang_install_dir}/lib/cmake/clang
    -DMLIR_DIR:PATH=${mlir_install_dir}/lib/cmake/mlir
    -DFLANG_INCLUDE_TESTS:BOOL=OFF
    ${toolchain}
    ${common_flags}
    BUILD_ALWAYS YES
    # EXCLUDE_FROM_ALL YES
    DEPENDS llvm clang mlir
    ${console_access}
    CONFIGURE_HANDLED_BY_BUILD YES
)
ExternalProject_Get_Property(flang install_dir)
set(flang_install_dir "${install_dir}")
ExternalProject_Get_Property(flang binary_dir)
set(flang_build_dir "${binary_dir}")

# **** OpenMP ****
ExternalProject_Add(
    openmp
    SOURCE_DIR "${PROJECT_SOURCE_DIR}/llvm-project/openmp"
    CMAKE_CACHE_ARGS
    -DLLVM_DIR:PATH=${llvm_install_dir}/lib/cmake/llvm
    ${toolchain}
    ${common_flags}
    BUILD_ALWAYS YES
    # EXCLUDE_FROM_ALL YES
    DEPENDS llvm
    ${console_access}
    CONFIGURE_HANDLED_BY_BUILD YES
)
ExternalProject_Get_Property(openmp install_dir)
set(openmp_install_dir "${install_dir}")
ExternalProject_Get_Property(openmp binary_dir)
set(openmp_build_dir "${binary_dir}")

# **** Polly ****
ExternalProject_Add(
    polly
    SOURCE_DIR "${PROJECT_SOURCE_DIR}/llvm-project/polly"
    CMAKE_CACHE_ARGS
    -DLLVM_DIR:PATH=${llvm_install_dir}/lib/cmake/llvm
    ${toolchain}
    ${common_flags}
    BUILD_ALWAYS YES
    # EXCLUDE_FROM_ALL YES
    DEPENDS llvm
    ${console_access}
    CONFIGURE_HANDLED_BY_BUILD YES
)
ExternalProject_Get_Property(polly install_dir)
set(polly_install_dir "${install_dir}")
ExternalProject_Get_Property(polly binary_dir)
set(polly_build_dir "${binary_dir}")
