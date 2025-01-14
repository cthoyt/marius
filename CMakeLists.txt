cmake_minimum_required(VERSION 3.12.2)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
cmake_policy(SET CMP0048 NEW)
project(marius VERSION 0.1 LANGUAGES CXX)

include(FindPackageHandleStandardArgs)

add_compile_definitions(_GLIBCXX_USE_CXX11_ABI=0)

if ("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
    set(CLANG TRUE)
elseif ("${CMAKE_CXX_COMPILER_ID}" MATCHES "GNU")
    set(GCC TRUE)
elseif ("${CMAKE_CXX_COMPILER_ID}" MATCHES "Intel")
    set(ICC TRUE)
else ()
    message(FATAL_ERROR "Unknown compiler")
endif ()

if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(CMAKE_MACOSX_RPATH 1)
endif ()

if(NOT IS_ABSOLUTE ${MARIUS_TORCH_DIR})
    set(MARIUS_TORCH_DIR "${CMAKE_CURRENT_BINARY_DIR}/${MARIUS_TORCH_DIR}")
endif()

set(Torch_DIR ${MARIUS_TORCH_DIR}/share/cmake/Torch)
set(Boost_USE_MULTITHREADED TRUE)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${TORCH_CXX_FLAGS}")

if(${USE_CUDA})
    add_definitions(-DMARIUS_CUDA=${USE_CUDA})
endif()

# Find torch location
execute_process(
        COMMAND python3 -c "import torch; import os; print(os.path.dirname(torch.__file__), end='')"
        OUTPUT_VARIABLE TorchPath
)
list(APPEND CMAKE_PREFIX_PATH ${TorchPath})

execute_process(
        COMMAND python3 -c "import torch; print(torch.__version__, end='')"
        OUTPUT_VARIABLE TorchVersion
)
message(STATUS "Torch Version: ${TorchVersion}")

# Add the cmake folder so the FindSphinx module is found
set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake" ${CMAKE_MODULE_PATH})
set(project_INCLUDE_DIR ${CMAKE_CURRENT_LIST_DIR}/include)
set(project_SOURCE_DIR ${CMAKE_CURRENT_LIST_DIR}/src)
set(project_TEST_DIR ${CMAKE_CURRENT_LIST_DIR}/test)
set(project_DOCS_DIR ${CMAKE_CURRENT_LIST_DIR}/docs)
set(project_BINDINGS_DIR ${CMAKE_CURRENT_LIST_DIR}/python_bindings)
set(project_THIRD_PARTY_DIR ${CMAKE_CURRENT_LIST_DIR}/third_party)

add_definitions(-DMARIUS_BASE_DIRECTORY="${CMAKE_CURRENT_LIST_DIR}")
add_definitions(-DMARIUS_TEST_DIRECTORY="${project_TEST_DIR}")

if (EXISTS ${project_INCLUDE_DIR})
    file(GLOB_RECURSE project_HEADERS ${project_HEADERS} ${project_INCLUDE_DIR}/*.h)
endif ()
if (EXISTS ${project_SOURCE_DIR})
    file(GLOB_RECURSE project_SOURCES ${project_SOURCES} ${project_SOURCE_DIR}/*.cpp)
endif ()
if (EXISTS ${project_BINDINGS_DIR})
    file(GLOB_RECURSE project_BINDINGS ${project_BINDINGS} ${project_BINDINGS_DIR}/*.cpp)
endif ()

add_subdirectory(${project_THIRD_PARTY_DIR})
set_property(TARGET Boost_program_options PROPERTY POSITION_INDEPENDENT_CODE ON)
set_property(TARGET spdlog PROPERTY POSITION_INDEPENDENT_CODE ON)
include_directories(${project_INCLUDE_DIR})
include_directories(${TORCH_INCLUDE_DIRS})
add_library(${PROJECT_NAME} SHARED ${project_SOURCES} ${project_HEADERS})

find_package(Torch REQUIRED)
target_link_libraries(${PROJECT_NAME} ${TORCH_LIBRARIES})
target_link_libraries(${PROJECT_NAME} Boost::program_options)
target_link_libraries(${PROJECT_NAME} spdlog)

set_target_properties(${PROJECT_NAME} PROPERTIES PUBLIC_HEADER "${project_HEADERS}")

if (EXISTS ${project_INCLUDE_DIR})
    target_include_directories(${PROJECT_NAME} PUBLIC ${project_INCLUDE_DIR})
endif ()
if (EXISTS ${project_SOURCE_DIR})
    target_include_directories(${PROJECT_NAME} PRIVATE ${project_SOURCE_DIR})
endif ()

target_compile_options(${PROJECT_NAME}
        PRIVATE -fomit-frame-pointer
        PRIVATE -O$<$<CONFIG:Debug>:0 -g3>$<$<CONFIG:Release>:3>
        )

IF(CMAKE_BUILD_TYPE MATCHES Debug AND MARIUS_USE_ASAN)
   message("Using ASAN")
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address  -fsanitize=leak")
   set(CMAKE_MODULE_LINKER_FLAGS "{$CMAKE_MODULE_LINKER_FLAGS}  -fsanitize=address -fsanitize=leak")
ENDIF(CMAKE_BUILD_TYPE MATCHES Debug AND MARIUS_USE_ASAN)


IF(BUILD_DOCS)
    add_subdirectory(${project_DOCS_DIR})
ENDIF()

if (EXISTS ${project_TEST_DIR})
    enable_testing()
    add_subdirectory(${project_TEST_DIR})
endif ()

add_executable(marius_train src/marius.cpp)
add_executable(marius_eval src/marius.cpp)
target_link_libraries(marius_train ${PROJECT_NAME})
target_link_libraries(marius_eval ${PROJECT_NAME})

pybind11_add_module(_pymarius ${project_SOURCES} ${project_BINDINGS})

find_library(TORCH_PYTHON_LIBRARY torch_python PATHS "${TORCH_INSTALL_PREFIX}/lib")
message(STATUS "TORCH_PYTHON_LIBRARY: ${TORCH_PYTHON_LIBRARY}")
target_link_libraries(_pymarius PRIVATE ${PROJECT_NAME} ${TORCH_PYTHON_LIBRARY})
target_include_directories(_pymarius PRIVATE ${project_INCLUDE_DIR})
