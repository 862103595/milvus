#-------------------------------------------------------------------------------
# Copyright (C) 2019-2020 Zilliz. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance
# with the License. You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software distributed under the License
# is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express
# or implied. See the License for the specific language governing permissions and limitations under the License.
#-------------------------------------------------------------------------------

# RDKit integration for Milvus
# Supports building from local modified source or from Git repository

# Priority 1: Check if RDKIT_SOURCE_DIR environment variable is set (for local modified source)
if(DEFINED ENV{RDKIT_SOURCE_DIR})
    set(RDKIT_SOURCE_DIR "$ENV{RDKIT_SOURCE_DIR}" CACHE PATH "Path to modified RDKit source")
    message(STATUS "Using RDKit from environment variable: ${RDKIT_SOURCE_DIR}")
endif()

# Priority 2: Check if RDKIT_SOURCE_DIR CMake variable is set
if(NOT DEFINED RDKIT_SOURCE_DIR)
    # Default: try to use rdkit-src subdirectory (if user places source there)
    set(RDKIT_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/rdkit-src" CACHE PATH "Path to modified RDKit source")
endif()

# Check if local source exists
if(EXISTS "${RDKIT_SOURCE_DIR}/CMakeLists.txt")
    message(STATUS "Building RDKit from local source: ${RDKIT_SOURCE_DIR}")
    
    # Set RDKit build options
    set(RDKIT_BUILD_DIR "${CMAKE_CURRENT_BINARY_DIR}/rdkit-build")
    
    # RDKit CMake configuration options
    # Disable Python wrappers and other unnecessary components for Milvus
    set(RDKIT_CMAKE_ARGS
        -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
        -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
        -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
        -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
        -DRDK_BUILD_PYTHON_WRAPPERS=OFF
        -DRDK_BUILD_INCHI_SUPPORT=ON
        -DRDK_BUILD_AVALON_SUPPORT=ON
        -DRDK_BUILD_CPP_TESTS=OFF
        -DRDK_BUILD_SWIG_WRAPPERS=OFF
        -DRDK_BUILD_DESCRIPTORS3D=OFF
        -DRDK_BUILD_MOLINTERCHANGE_SUPPORT=OFF
        -DRDK_BUILD_FREETYPE_SUPPORT=OFF
        -DRDK_BUILD_COORDGEN_SUPPORT=OFF
        -DBOOST_ROOT=${CONAN_BOOST_ROOT}
        -DBoost_NO_BOOST_CMAKE=ON
    )
    
    # Enable thread-safe SSS if available (for concurrent access)
    # Uncomment if your modified RDKit supports this option
    # set(RDKIT_CMAKE_ARGS ${RDKIT_CMAKE_ARGS} -DRDK_BUILD_THREADSAFE_SSS=ON)
    
    # Use ExternalProject to build RDKit from local source
    include(ExternalProject)
    
    ExternalProject_Add(
        rdkit_ep
        SOURCE_DIR ${RDKIT_SOURCE_DIR}
        BINARY_DIR ${RDKIT_BUILD_DIR}
        CMAKE_ARGS ${RDKIT_CMAKE_ARGS}
        ${EP_LOG_OPTIONS}
        BUILD_BYPRODUCTS 
            "${CMAKE_INSTALL_PREFIX}/lib/libRDKitGraphMol${CMAKE_SHARED_LIBRARY_SUFFIX}"
            "${CMAKE_INSTALL_PREFIX}/lib/libRDKitGraphMol${CMAKE_STATIC_LIBRARY_SUFFIX}"
            "${CMAKE_INSTALL_PREFIX}/lib/libRDKitDescriptors${CMAKE_SHARED_LIBRARY_SUFFIX}"
            "${CMAKE_INSTALL_PREFIX}/lib/libRDKitFingerprints${CMAKE_SHARED_LIBRARY_SUFFIX}"
    )
    
    # Create imported library targets for RDKit components
    # Main GraphMol library
    add_library(RDKit::RDKitGraphMol SHARED IMPORTED)
    set_target_properties(RDKit::RDKitGraphMol
        PROPERTIES
        IMPORTED_LOCATION "${CMAKE_INSTALL_PREFIX}/lib/libRDKitGraphMol${CMAKE_SHARED_LIBRARY_SUFFIX}"
        INTERFACE_INCLUDE_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/include/rdkit"
    )
    add_dependencies(RDKit::RDKitGraphMol rdkit_ep)
    
    # Descriptors library (for fingerprint generation)
    add_library(RDKit::RDKitDescriptors SHARED IMPORTED)
    set_target_properties(RDKit::RDKitDescriptors
        PROPERTIES
        IMPORTED_LOCATION "${CMAKE_INSTALL_PREFIX}/lib/libRDKitDescriptors${CMAKE_SHARED_LIBRARY_SUFFIX}"
        INTERFACE_INCLUDE_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/include/rdkit"
        INTERFACE_LINK_LIBRARIES RDKit::RDKitGraphMol
    )
    add_dependencies(RDKit::RDKitDescriptors rdkit_ep)
    
    # Fingerprints library
    add_library(RDKit::RDKitFingerprints SHARED IMPORTED)
    set_target_properties(RDKit::RDKitFingerprints
        PROPERTIES
        IMPORTED_LOCATION "${CMAKE_INSTALL_PREFIX}/lib/libRDKitFingerprints${CMAKE_SHARED_LIBRARY_SUFFIX}"
        INTERFACE_INCLUDE_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/include/rdkit"
        INTERFACE_LINK_LIBRARIES RDKit::RDKitGraphMol
    )
    add_dependencies(RDKit::RDKitFingerprints rdkit_ep)
    
    # Convenience target that links all RDKit libraries
    add_library(RDKit::RDKit INTERFACE IMPORTED)
    set_target_properties(RDKit::RDKit
        PROPERTIES
        INTERFACE_LINK_LIBRARIES 
            "RDKit::RDKitGraphMol;RDKit::RDKitDescriptors;RDKit::RDKitFingerprints"
        INTERFACE_INCLUDE_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/include/rdkit"
    )
    
    # Set global variables for other modules
    set(RDKIT_INCLUDE_DIR "${CMAKE_INSTALL_PREFIX}/include" CACHE INTERNAL "RDKit include directory")
    set(RDKIT_LIB_DIR "${CMAKE_INSTALL_PREFIX}/lib" CACHE INTERNAL "RDKit library directory")
    
    message(STATUS "RDKit will be installed to: ${CMAKE_INSTALL_PREFIX}")
    
else()
    # Fallback: Use FetchContent to get RDKit from Git (if local source not found)
    message(STATUS "Local RDKit source not found at ${RDKIT_SOURCE_DIR}")
    message(STATUS "Attempting to fetch RDKit from Git repository...")
    
    include(FetchContent)
    
    # RDKit Git repository and version
    set(RDKIT_VERSION "Release_2024_03_1" CACHE STRING "RDKit version/tag to use")
    set(RDKIT_GIT_REPOSITORY "https://github.com/rdkit/rdkit.git" CACHE STRING "RDKit Git repository")
    
    FetchContent_Declare(
        rdkit
        GIT_REPOSITORY ${RDKIT_GIT_REPOSITORY}
        GIT_TAG ${RDKIT_VERSION}
        SOURCE_DIR ${CMAKE_CURRENT_BINARY_DIR}/rdkit-src
        BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/rdkit-build
    )
    
    FetchContent_GetProperties(rdkit)
    if(NOT rdkit_POPULATED)
        FetchContent_Populate(rdkit)
        
        # Configure RDKit build options
        set(RDKIT_CMAKE_ARGS
            -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
            -DRDK_BUILD_PYTHON_WRAPPERS=OFF
            -DRDK_BUILD_INCHI_SUPPORT=ON
            -DRDK_BUILD_AVALON_SUPPORT=ON
            -DRDK_BUILD_CPP_TESTS=OFF
            -DRDK_BUILD_SWIG_WRAPPERS=OFF
            -DBOOST_ROOT=${CONAN_BOOST_ROOT}
            -DBoost_NO_BOOST_CMAKE=ON
        )
        
        # Build RDKit using ExternalProject
        include(ExternalProject)
        
        ExternalProject_Add(
            rdkit_ep
            SOURCE_DIR ${rdkit_SOURCE_DIR}
            BINARY_DIR ${rdkit_BINARY_DIR}
            CMAKE_ARGS ${RDKIT_CMAKE_ARGS}
            ${EP_LOG_OPTIONS}
        )
        
        # Create imported targets (same as above)
        add_library(RDKit::RDKitGraphMol SHARED IMPORTED)
        set_target_properties(RDKit::RDKitGraphMol
            PROPERTIES
            IMPORTED_LOCATION "${CMAKE_INSTALL_PREFIX}/lib/libRDKitGraphMol${CMAKE_SHARED_LIBRARY_SUFFIX}"
            INTERFACE_INCLUDE_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/include/rdkit"
        )
        add_dependencies(RDKit::RDKitGraphMol rdkit_ep)
        
        set(RDKIT_INCLUDE_DIR "${CMAKE_INSTALL_PREFIX}/include" CACHE INTERNAL "RDKit include directory")
        set(RDKIT_LIB_DIR "${CMAKE_INSTALL_PREFIX}/lib" CACHE INTERNAL "RDKit library directory")
    endif()
endif()

message(STATUS "RDKit integration configured")
message(STATUS "  Include directory: ${RDKIT_INCLUDE_DIR}")
message(STATUS "  Library directory: ${RDKIT_LIB_DIR}")

