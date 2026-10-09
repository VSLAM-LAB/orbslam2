cmake_minimum_required(VERSION 3.16)

project(DBoW2 CXX)

# Standard settings
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
endif()

# Output directories
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/bin)

# Dependencies
find_package(OpenCV REQUIRED)

# Interface library for common warnings
add_library(dbow2_warnings INTERFACE)
target_compile_options(dbow2_warnings INTERFACE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall -Wextra -Wpedantic>
    $<$<CXX_COMPILER_ID:MSVC>:/W4>
)

# Optimization/Architecture flags (-O3 comes with the Release build type). -march=native ties the binaries to
# the build machine's CPU, so it is opt-in (local builds: build.sh); MSVC needs /bigobj for the Eigen-heavy code
option(ORBSLAM2_NATIVE "Optimize for the build machine's CPU (-march=native)" OFF)
set(DBOW2_OPTIMIZATION_FLAGS
    $<$<CXX_COMPILER_ID:MSVC>:/bigobj>
    $<$<AND:$<BOOL:${ORBSLAM2_NATIVE}>,$<NOT:$<CXX_COMPILER_ID:MSVC>>>:-march=native>
)

# Configuration Header
configure_file(DBoW2/DBoW2.cmake.in "${PROJECT_BINARY_DIR}/DBoW2Config.cmake" @ONLY)

# Create shared library
set(SRCS_DBOW2
  DBoW2/BowVector.cpp
  DBoW2/FORB.cpp      
  DBoW2/FeatureVector.cpp
  DBoW2/ScoringObject.cpp)

set(SRCS_DUTILS
  DUtils/Random.cpp
  DUtils/Timestamp.cpp)

# Create static library (linked into libORB_SLAM2)
add_library(${PROJECT_NAME} STATIC ${SRCS_DBOW2} ${SRCS_DUTILS})
set_target_properties(${PROJECT_NAME} PROPERTIES POSITION_INDEPENDENT_CODE ON)

# Apply warnings and optimization flags
target_link_libraries(${PROJECT_NAME} PRIVATE dbow2_warnings)
target_compile_options(${PROJECT_NAME} PRIVATE ${DBOW2_OPTIMIZATION_FLAGS})

# Set include directories for the target
target_include_directories(${PROJECT_NAME} PUBLIC
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}>
    $<INSTALL_INTERFACE:include>
)

# Link dependencies
target_link_libraries(${PROJECT_NAME} PUBLIC ${OpenCV_LIBS})

# Install targets and headers
install(TARGETS ${PROJECT_NAME}
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib
        RUNTIME DESTINATION bin
        INCLUDES DESTINATION include)

# Install configuration header
install(FILES "${PROJECT_BINARY_DIR}/DBoW2Config.cmake" DESTINATION lib/cmake/DBoW2/)

# Install all other public headers
install(DIRECTORY DBoW2/ DESTINATION include/DBoW2 FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp")
install(DIRECTORY DUtils/ DESTINATION include/DUtils FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp")

