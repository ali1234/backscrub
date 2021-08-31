cmake_minimum_required(VERSION 3.16)
project(Backscrub CXX)

# allow override of Tensorflow location
if(NOT DEFINED TENSORFLOW)
  set(TENSORFLOW tensorflow)
  set(TF_CHECKOUT true)
endif()

# find Git and use it to get backscrub version
find_package(Git)
if(GIT_FOUND)
  execute_process(
    COMMAND ${GIT_EXECUTABLE} describe --all --long --always --dirty
    OUTPUT_STRIP_TRAILING_WHITESPACE
    OUTPUT_VARIABLE DEEPSEG_VERSION)
else()
  set(DEEPSEG_VERSION v0.2.0-no-git)
endif()
message("Version: ${DEEPSEG_VERSION}")

# always build PIC everywhere
set(CMAKE_POSITION_INDEPENDENT_CODE TRUE)

# use local platform OpenCV libraries
find_package(OpenCV REQUIRED COMPONENTS core imgproc imgcodecs video highgui)

# use .gitmodules defined Tensorflow version unless a path was provided
if (TF_CHECKOUT)
  if (GIT_FOUND)
    message(STATUS "Updating Tensorflow source")
    execute_process(COMMAND ${GIT_EXECUTABLE} submodule update --init --recursive)
  else()
    message(FATAL_ERROR "Git not found. Unable to checkout required Tensorflow version")
  endif()
else()
  message(STATUS "Using specified Tensorflow: ${TENSORFLOW}")
endif(TF_CHECKOUT)

# force compilation of XNNPACK delegate (without this the too clever-by-half use
# of weak/strong symbol linking fails in a static library)
add_compile_definitions(TFLITE_BUILD_WITH_XNNPACK_DELEGATE)

# Stop broken xnnpack/FP16 cmake goop from downloading PSimd to our source folder!
# TODO: check once tensorflow is updated post 2.5.0
set(CONFU_DEPENDENCIES_SOURCE_DIR ${CMAKE_BINARY_DIR})
set(CONFU_DEPENDENCIES_BINARY_DIR ${CMAKE_BINARY_DIR}/_deps)

# pull in Tensorflow Lite source build
add_subdirectory(${TENSORFLOW}/tensorflow/lite
  "${CMAKE_CURRENT_BINARY_DIR}/tensorflow-lite" EXCLUDE_FROM_ALL)

# build backscrub code
add_compile_definitions(DEEPSEG_VERSION=${DEEPSEG_VERSION})
include_directories(BEFORE .)

add_library(libbackscrub STATIC
  lib/libbackscrub.cc
  lib/transpose_conv_bias.cc)
# don't add lib prefix as the name already has it
set_target_properties(libbackscrub PROPERTIES PREFIX "")

set_target_properties(libbackscrub PROPERTIES PUBLIC_HEADER "lib/libbackscrub.h")
target_include_directories(libbackscrub PUBLIC lib)

target_link_libraries(libbackscrub
  tensorflow-lite ${CMAKE_DL_LIBS}
  opencv_core
  opencv_imgproc
)

# installation names for library, header and models
install(TARGETS libbackscrub)
install(DIRECTORY models
  DESTINATION ${CMAKE_INSTALL_PREFIX}/share/backscrub
  FILES_MATCHING PATTERN "*.tflite" PATTERN "*.md")

if(NOT WIN32) # shouldn't this be "if linux" since the app uses v4l?
    add_subdirectory(app)
endif()