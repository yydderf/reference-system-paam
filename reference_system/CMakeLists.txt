cmake_minimum_required(VERSION 3.5)
project(reference_system
  VERSION 0.0.1
)
find_package(CUDA REQUIRED)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_C_FLAGS "-pthread")
set(CMAKE_C_FLAGS_DEBUG "-g -O0")
set(CMAKE_C_FLAGS_RELEASE "-O3")
set(CMAKE_CXX_FLAGS "${CMAKE_C_FLAGS}")
set(CMAKE_CXX_FLAGS_DEBUG ${CMAKE_C_FLAGS_DEBUG})
set(CMAKE_CXX_FLAGS_RELEASE ${CMAKE_C_FLAGS_RELEASE})

# if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
#   add_compile_options(-Wall -Wextra -Wpedantic)
# endif()

find_package(ament_cmake_auto REQUIRED)
ament_auto_find_build_dependencies()
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/../../PAAM/include)


# Must match with commit that libedgetpu.so was built from.
set(TENSORFLOW_COMMIT d855adfc5a0195788bf5f92c3c7352e638aa1109)
set(EDGE_ARCH "${CMAKE_SYSTEM_PROCESSOR}")
message("${CMAKE_SYSTEM_PROCESSOR}")

if(EDGE_ARCH STREQUAL "x86_64" OR EDGE_ARCH STREQUAL "amd64")
  set(EDGE_ARCH "k8")          # libedgetpu uses "k8" for x86_64
elseif(EDGE_ARCH STREQUAL "arm64")
  set(EDGE_ARCH "aarch64")
endif()

if(EDGE_ARCH STREQUAL "aarch64")
  set(TF_INSTALL_PREFIX "generic-aarch64_armv8-a")
  set(CROSS_PREFIX "aarch64-linux-gnu-")
  set(TF_COMMAND make -j12 TARGET=generic-aarch64 TARGET_ARCH=armv8-a -C ${CMAKE_BINARY_DIR}/tensorflow/src/tf -f tensorflow/lite/tools/make/Makefile CC=${CROSS_PREFIX}g++ CXX=${CROSS_PREFIX}g++ AR=${CROSS_PREFIX}ar CFLAGS=-fpermissive lib)
else()
  set(TF_INSTALL_PREFIX "linux_x86_64")
  set(CROSS_PREFIX "")
  set(TF_COMMAND make -j12 -C ${CMAKE_BINARY_DIR}/tensorflow/src/tf -f tensorflow/lite/tools/make/Makefile CXXFLAGS="-std=c++17" lib)
endif()

include(ExternalProject)

ExternalProject_Add(absl_src
    GIT_REPOSITORY https://github.com/abseil/abseil-cpp
    GIT_PROGRESS 0
    GIT_SHALLOW 1
    PREFIX "absl"
    PATCH_COMMAND /bin/bash -lc
      "sed -i '/ABSL_RANDOM_HWAES_X64_FLAGS/d' '${CMAKE_BINARY_DIR}/absl/src/absl_src/absl/copts/AbseilConfigureCopts.cmake' && \
       sed -i '/random/d' '${CMAKE_BINARY_DIR}/absl/src/absl_src/absl/CMakeLists.txt'"
    CMAKE_ARGS
      -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}/absl_install
      -DCMAKE_INSTALL_LIBDIR=lib
      -DCMAKE_CXX_STANDARD=17
      -DCMAKE_CXX_STANDARD_REQUIRED=ON
      -DCMAKE_CXX_EXTENSIONS=OFF
      -DCMAKE_CXX_FLAGS=-std=c++17
    BUILD_BYPRODUCTS libabsl_synchronization.a libabsl_symbolize.a
)
set(ABSL_LIBS absl_synchronization absl_stacktrace absl_symbolize absl_demangle_internal absl_debugging_internal absl_dynamic_annotations absl_time absl_time_zone absl_graphcycles_internal absl_failure_signal_handler absl_malloc_internal absl_base absl_spinlock_wait)

ExternalProject_Add(glog_src
    GIT_REPOSITORY https://github.com/google/glog
    GIT_TAG v0.6.0
    GIT_PROGRESS 0
    GIT_SHALLOW 1
    CMAKE_ARGS -DWITH_GFLAGS=OFF
    -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}/glog_install
    -DCMAKE_INSTALL_LIBDIR=lib
    PREFIX "glog_src"
    BUILD_BYPRODUCTS  libglog.a
)
set(LIBGLOG ${CMAKE_BINARY_DIR}/glog_install/lib/libglog.a)

ExternalProject_Add(tf
    GIT_REPOSITORY https://github.com/tensorflow/tensorflow
    GIT_TAG ${TENSORFLOW_COMMIT}
    PREFIX "tensorflow"
    CONFIGURE_COMMAND /bin/bash -lc
      "cd ${CMAKE_BINARY_DIR}/tensorflow/src/tf && ./tensorflow/lite/tools/make/download_dependencies.sh"
    # BUILD_IN_SOURCE 1
    BUILD_COMMAND ${TF_COMMAND}
    BUILD_BYPRODUCTS
      ${CMAKE_BINARY_DIR}/libtensorflow-lite.a
      ${CMAKE_BINARY_DIR}/tensorflow/src/tf/tensorflow/lite/tools/make/downloads/fft2d/fftsg.c
      ${CMAKE_BINARY_DIR}/tensorflow/src/tf/tensorflow/lite/tools/optimize/sparsity/format_converter.cc
      ${CMAKE_BINARY_DIR}/tensorflow/src/tf/tensorflow/lite/c/common.c
    INSTALL_COMMAND /bin/bash -lc
      "cp -f ${CMAKE_BINARY_DIR}/tensorflow/src/tf/tensorflow/lite/tools/make/gen/${TF_INSTALL_PREFIX}/lib/libtensorflow-lite.a ${CMAKE_BINARY_DIR}/libtensorflow-lite.a"
)

set(TF_ET_SRC_LIB "${CMAKE_SOURCE_DIR}/libedgetpu/direct/${EDGE_ARCH}/libedgetpu.so.1.0")
set(TF_LITE_LIB "${CMAKE_BINARY_DIR}/libtensorflow-lite.a")
include_directories(${CMAKE_SOURCE_DIR}/)
include_directories(${CMAKE_SOURCE_DIR}/src/)
include_directories(${CMAKE_SOURCE_DIR}/include/)
include_directories(${CMAKE_SOURCE_DIR}/libedgetpu/)
include_directories(${CMAKE_BINARY_DIR}/tensorflow/src/tf/)
include_directories(${CMAKE_BINARY_DIR}/tensorflow/src/tf/tensorflow/lite/c/)
include_directories(${CMAKE_BINARY_DIR}/tensorflow/src/tf/tensorflow/lite/tools/make/downloads/absl/)
include_directories(${CMAKE_BINARY_DIR}/tensorflow/src/tf/tensorflow/lite/tools/make/downloads/flatbuffers/include/)
include_directories(${CMAKE_BINARY_DIR}/glog_install/include)
link_directories(${CMAKE_BINARY_DIR})
link_directories(${CMAKE_BINARY_DIR}/absl_install/lib)
link_directories(${CMAKE_BINARY_DIR}/glog_install/lib)
include_directories(include)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)
include_directories(${CUDA_INCLUDE_DIRS})



# Add header-only library
add_library(${PROJECT_NAME} INTERFACE)

set(TF_FFTSG_C   "${CMAKE_BINARY_DIR}/tensorflow/src/tf/tensorflow/lite/tools/make/downloads/fft2d/fftsg.c")
set(TF_FMT_CC    "${CMAKE_BINARY_DIR}/tensorflow/src/tf/tensorflow/lite/tools/optimize/sparsity/format_converter.cc")

set_source_files_properties(${TF_FFTSG_C} ${TF_FMT_CC}
  PROPERTIES GENERATED TRUE
)

cuda_add_library(cuda_gemm SHARED
  include/reference_system/gpu_operations.cu
  ${TF_FFTSG_C}
  ${TF_FMT_CC}
)

add_dependencies(cuda_gemm tf)
# cuda_add_library(cuda_gemm SHARED include/reference_system/gpu_operations.cu
#   ${CMAKE_BINARY_DIR}/tensorflow/src/tf/tensorflow/lite/tools/make/downloads/fft2d/fftsg.c
#   ${CMAKE_BINARY_DIR}/tensorflow/src/tf/tensorflow/lite/tools/optimize/sparsity/format_converter.cc
# )

add_library(tflite STATIC IMPORTED GLOBAL)
set_target_properties(tflite PROPERTIES
  IMPORTED_LOCATION "${CMAKE_BINARY_DIR}/libtensorflow-lite.a"
)
add_dependencies(tflite tf)

target_link_libraries(${PROJECT_NAME} INTERFACE cuda_gemm   ${TF_LITE_LIB} ${TF_ET_SRC_LIB})
# target_link_libraries(${PROJECT_NAME} INTERFACE cuda_gemm tflite ${TF_ET_SRC_LIB})

target_include_directories(${PROJECT_NAME} INTERFACE
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>
)

if(${BUILD_TESTING})
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()
endif()

# Install
install(TARGETS ${PROJECT_NAME} cuda_gemm
        EXPORT "export_${PROJECT_NAME}"
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        INCLUDES DESTINATION include
)

ament_export_targets("export_${PROJECT_NAME}")
ament_auto_package()

