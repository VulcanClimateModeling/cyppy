# Set default project to unknown
if(NOT PROJECT)
  message(STATUS "Setting CCPP project to 'unknown' as none was specified.")
  set(PROJECT "Unknown")
endif (NOT PROJECT)

#------------------------------------------------------------------------------
cmake_minimum_required(VERSION 2.8.11)

# Use rpaths on MacOSX
set(CMAKE_MACOSX_RPATH 1)

if(POLICY CMP0048)
    cmake_policy(SET CMP0048 NEW)
    project(ccppphys VERSION 3.0.0)
else(POLICY CMP0048)
    project(ccppphys)
    set(PROJECT_VERSION 3.0.0)
    set(PROJECT_VERSION_MAJOR 3)
    set(PROJECT_VERSION_MINOR 0)
    set(PROJECT_VERSION_PATCH 0)
endif(POLICY CMP0048)

if(POLICY CMP0042)
    cmake_policy(SET CMP0042 NEW)
endif(POLICY CMP0042)

#------------------------------------------------------------------------------
set(PACKAGE "ccpp-physics")
set(AUTHORS  "Grant J. Firl" "Dom Heinzeller")

#------------------------------------------------------------------------------
# Enable Fortran
enable_language(Fortran)

#------------------------------------------------------------------------------
# Set OpenMP flags for C/C++/Fortran
if (OPENMP)
  include(detect_openmp)
  detect_openmp()
  set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
  set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
  set (CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} ${OpenMP_Fortran_FLAGS}")
  message(STATUS "Enable OpenMP support for C/C++/Fortran compiler")
else (OPENMP)
  message (STATUS "Disable OpenMP support for C/C++/Fortran compiler")
endif()

#------------------------------------------------------------------------------
# The Fortran compiler/linker flag inserted by cmake to create shared libraries
# with the Intel compiler is deprecated (-i_dynamic), correct here.
# CMAKE_Fortran_COMPILER_ID = {"Intel", "PGI", "GNU", "Clang", "MSVC", ...}
if ("${CMAKE_Fortran_COMPILER_ID}" STREQUAL "Intel")
    string(REPLACE "-i_dynamic" "-shared-intel"
           CMAKE_SHARED_LIBRARY_CREATE_Fortran_FLAGS
           "${CMAKE_SHARED_LIBRARY_CREATE_Fortran_FLAGS}")
    string(REPLACE "-i_dynamic" "-shared-intel"
           CMAKE_SHARED_LIBRARY_LINK_Fortran_FLAGS
           "${CMAKE_SHARED_LIBRARY_LINK_Fortran_FLAGS}")
endif()

#------------------------------------------------------------------------------
# Set a default build type if none was specified
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    message(STATUS "Setting build type to 'Release' as none was specified.")
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)

    # Set the possible values of build type for cmake-gui
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Bitforbit" "Release" "Coverage")
endif()

#------------------------------------------------------------------------------
# We want a shared library
option(BUILD_SHARED_LIBS "Build a shared library" ON)

#------------------------------------------------------------------------------
# Add the CCPP include/module directory
set(CCPP_INCLUDE_DIRS "" CACHE FILEPATH "Path to ccpp includes")
set_property(DIRECTORY PROPERTY INCLUDE_DIRECTORIES ${CCPP_INCLUDE_DIRS})

#------------------------------------------------------------------------------
# Add the CCPP library
set(CCPP_LIB_DIRS "" CACHE FILEPATH "Path to ccpp library")
link_directories(${CCPP_LIB_DIRS})
list(APPEND LIBS "ccpp")

#------------------------------------------------------------------------------
if (${CMAKE_Fortran_COMPILER_ID} STREQUAL "GNU")
  set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -ffree-line-length-none")
  SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_bfmicrophysics.f
                              ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/sflx.f
                              ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/sfc_diff.f
                              ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/sfc_diag.f
                              PROPERTIES COMPILE_FLAGS -fdefault-real-8)
  SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_nst_model.f90
                              ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/calpreciptype.f90
                              PROPERTIES COMPILE_FLAGS "-fdefault-real-8 -ffree-form")
  SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/mersenne_twister.f
                              PROPERTIES COMPILE_FLAGS "-fdefault-real-8 -fno-range-check")
  SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_nst_water_prop.f90
                              PROPERTIES COMPILE_FLAGS "-ffree-line-length-none -fdefault-real-8 -ffree-form")
  SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/aer_cloud.F
                              ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/wv_saturation.F
                              ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/cldwat2m_micro.F
                              ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/surface_perturbation.F90
                              PROPERTIES COMPILE_FLAGS "-fdefault-real-8 -fdefault-double-8")
  SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_mp_thompson_make_number_concentrations.F90
                              PROPERTIES COMPILE_FLAGS "-fdefault-real-8 -fdefault-double-8")
  SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_SF_JSFC.F90
                              ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_BL_MYJPBL.F90
                              PROPERTIES COMPILE_FLAGS "-fdefault-real-8 -fdefault-double-8")
  if (PROJECT STREQUAL "CCPP-FV3")
    # Set 32-bit floating point precision flags for certain files
    # that are executed in the dynamics (fast physics part)
    if (DYN32)
      # Reduce floating point precision from 64-bit to 32-bit, if necessary
      set(CMAKE_Fortran_FLAGS_PREC32 ${CMAKE_Fortran_FLAGS_DEFAULT_PREC})
      string(REPLACE "-fdefault-real-8" ""
             CMAKE_Fortran_FLAGS_PREC32 "${CMAKE_Fortran_FLAGS_PREC32}")
      string(REPLACE "-fdefault-double-8" ""
             CMAKE_Fortran_FLAGS_PREC32 "${CMAKE_Fortran_FLAGS_PREC32}")
      SET_PROPERTY(SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/gfdl_fv_sat_adj.F90
                   APPEND_STRING PROPERTY COMPILE_FLAGS " ${CMAKE_Fortran_FLAGS_PREC32} ")
      # Add all of the above files to the list of schemes with special floating point precision flags
      list(APPEND SCHEMES_SFX_PREC ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/gfdl_fv_sat_adj.F90)
    endif (DYN32)

    # Remove files with special floating point precision flags from list
    # of files with standard floating point precision flags flags
    if (SCHEMES_SFX_PREC)
      list(REMOVE_ITEM SCHEMES2 ${SCHEMES_SFX_PREC})
    endif (SCHEMES_SFX_PREC)
    # Assign standard floating point precision flags to all remaining schemes and caps
    SET_PROPERTY(SOURCE ${SCHEMES2} ${CAPS}
                 APPEND_STRING PROPERTY COMPILE_FLAGS " ${CMAKE_Fortran_FLAGS_DEFAULT_PREC} ")
  endif (PROJECT STREQUAL "CCPP-FV3")

elseif (${CMAKE_Fortran_COMPILER_ID} STREQUAL "Intel")
  # Adjust settings for bit-for-bit reproducibility of NEMSfv3gfs
  if (PROJECT STREQUAL "CCPP-FV3")
    SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_bfmicrophysics.f
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/sflx.f
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/sfc_diff.f
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/sfc_diag.f
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_nst_model.f90
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/calpreciptype.f90
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/mersenne_twister.f
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_nst_water_prop.f90
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/aer_cloud.F
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/wv_saturation.F
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/cldwat2m_micro.F
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/surface_perturbation.F90
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/radiation_aerosols.f
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/cu_gf_deep.F90
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/cu_gf_sh.F90
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_bl_mynn.F90
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_MYNNPBL_wrapper.F90
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_sf_mynn.F90
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_MYNNSFC_wrapper.F90
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_MYNNrad_pre.F90
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_MYNNrad_post.F90
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_mp_thompson_make_number_concentrations.F90
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_SF_JSFC.F90
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_BL_MYJPBL.F90
                                PROPERTIES COMPILE_FLAGS "-r8 -ftz")

    # Replace -xHost or -xCORE-AVX2 with -xCORE-AVX-I for certain files
    set(CMAKE_Fortran_FLAGS_LOPT1 ${CMAKE_Fortran_FLAGS_OPT})
    string(REPLACE "-xHOST" "-xCORE-AVX-I"
           CMAKE_Fortran_FLAGS_LOPT1
           "${CMAKE_Fortran_FLAGS_LOPT1}")
    string(REPLACE "-xCORE-AVX2" "-xCORE-AVX-I"
           CMAKE_Fortran_FLAGS_LOPT1
           "${CMAKE_Fortran_FLAGS_LOPT1}")
    string(REPLACE "-axSSE4.2,AVX,CORE-AVX2,CORE-AVX512" "-axSSE4.2,AVX,CORE-AVX-I"
           CMAKE_Fortran_FLAGS_LOPT1
           "${CMAKE_Fortran_FLAGS_LOPT1}")
    SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/radiation_aerosols.f
                                PROPERTIES COMPILE_FLAGS "${CMAKE_Fortran_FLAGS_LOPT1}")
    # Add all of the above files to the list of schemes with special compiler flags
    list(APPEND SCHEMES_SFX_OPT ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/radiation_aerosols.f)

    # Remove files with special compiler flags from list of files with standard compiler flags
    list(REMOVE_ITEM SCHEMES ${SCHEMES_SFX_OPT})
    # Assign standard compiler flags to all remaining schemes and caps
    SET_SOURCE_FILES_PROPERTIES(${SCHEMES} ${CAPS}
                                PROPERTIES COMPILE_FLAGS "${CMAKE_Fortran_FLAGS_OPT}")

    # Set 32-bit floating point precision flags for certain files
    # that are executed in the dynamics (fast physics part)
    if (DYN32)
      # Reduce floating point precision from 64-bit to 32-bit, if necessary
      set(CMAKE_Fortran_FLAGS_PREC32 ${CMAKE_Fortran_FLAGS_DEFAULT_PREC})
      string(REPLACE "-real-size 64" "-real-size 32"
             CMAKE_Fortran_FLAGS_PREC32 "${CMAKE_Fortran_FLAGS_PREC32}")
      SET_PROPERTY(SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/gfdl_fv_sat_adj.F90
                   APPEND_STRING PROPERTY COMPILE_FLAGS " ${CMAKE_Fortran_FLAGS_PREC32} ")
      # Add all of the above files to the list of schemes with special floating point precision flags
      list(APPEND SCHEMES_SFX_PREC ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/gfdl_fv_sat_adj.F90)
    endif (DYN32)

    # Remove files with special floating point precision flags from list
    # of files with standard floating point precision flags flags
    if (SCHEMES_SFX_PREC)
      list(REMOVE_ITEM SCHEMES2 ${SCHEMES_SFX_PREC})
    endif (SCHEMES_SFX_PREC)
    # Assign standard floating point precision flags to all remaining schemes and caps
    SET_PROPERTY(SOURCE ${SCHEMES2} ${CAPS}
                 APPEND_STRING PROPERTY COMPILE_FLAGS " ${CMAKE_Fortran_FLAGS_DEFAULT_PREC} ")

  else (PROJECT STREQUAL "CCPP-FV3")
    SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_bfmicrophysics.f
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/sflx.f
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/sfc_diff.f
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/sfc_diag.f
                                PROPERTIES COMPILE_FLAGS -r8)
    SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_nst_model.f90
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/calpreciptype.f90
                                PROPERTIES COMPILE_FLAGS "-r8 -free")
    SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/mersenne_twister.f
                                PROPERTIES COMPILE_FLAGS "-r8 -ftz")
    SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_nst_water_prop.f90
                                PROPERTIES COMPILE_FLAGS "-extend-source 132 -r8 -free")
    SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/aer_cloud.F
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/wv_saturation.F
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/cldwat2m_micro.F
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/surface_perturbation.F90
                                PROPERTIES COMPILE_FLAGS "-r8")
    SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_mp_thompson_make_number_concentrations.F90
                                PROPERTIES COMPILE_FLAGS "-r8")
    SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_SF_JSFC.F90
                                ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_BL_MYJPBL.F90
                                PROPERTIES COMPILE_FLAGS "-r8")
  endif (PROJECT STREQUAL "CCPP-FV3")
elseif (${CMAKE_Fortran_COMPILER_ID} STREQUAL "PGI")
  SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_bfmicrophysics.f
                              ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/sflx.f
                              ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/sfc_diff.f
                              ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/sfc_diag.f
                              PROPERTIES COMPILE_FLAGS -r8)
  SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_nst_model.f90
                              ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/calpreciptype.f90
                              PROPERTIES COMPILE_FLAGS "-r8 -Mfree")
  SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/mersenne_twister.f
                              PROPERTIES COMPILE_FLAGS "-r8 -Mnofptrap")
  SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_nst_water_prop.f90
                              PROPERTIES COMPILE_FLAGS "-r8 -Mfree")
  SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/aer_cloud.F
                              ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/wv_saturation.F
                              ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/cldwat2m_micro.F
                              ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/surface_perturbation.F90
                              PROPERTIES COMPILE_FLAGS "-r8")
  SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_mp_thompson_make_number_concentrations.F90
                              PROPERTIES COMPILE_FLAGS "-r8")
  SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_SF_JSFC.F90
                              ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/module_BL_MYJPBL.F90
                              PROPERTIES COMPILE_FLAGS "-r8")
  if (PROJECT STREQUAL "CCPP-FV3")
    # Set 32-bit floating point precision flags for certain files
    # that are executed in the dynamics (fast physics part)
    if (DYN32)
      # Reduce floating point precision from 64-bit to 32-bit, if necessary
      set(CMAKE_Fortran_FLAGS_PREC32 ${CMAKE_Fortran_FLAGS_DEFAULT_PREC})
      string(REPLACE "-r8" "-r4"
             CMAKE_Fortran_FLAGS_PREC32 "${CMAKE_Fortran_FLAGS_PREC32}")
      SET_PROPERTY(SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/gfdl_fv_sat_adj.F90
                   APPEND_STRING PROPERTY COMPILE_FLAGS " ${CMAKE_Fortran_FLAGS_PREC32} ")
      # Add all of the above files to the list of schemes with special floating point precision flags
      list(APPEND SCHEMES_SFX_PREC ${CMAKE_CURRENT_SOURCE_DIR}/ccpp-physics/physics/gfdl_fv_sat_adj.F90)
    endif (DYN32)

    # Remove files with special floating point precision flags from list
    # of files with standard floating point precision flags flags
    if (SCHEMES_SFX_PREC)
      list(REMOVE_ITEM SCHEMES2 ${SCHEMES_SFX_PREC})
    endif (SCHEMES_SFX_PREC)
    # Assign standard floating point precision flags to all remaining schemes and caps
    SET_PROPERTY(SOURCE ${SCHEMES2} ${CAPS}
                 APPEND_STRING PROPERTY COMPILE_FLAGS " ${CMAKE_Fortran_FLAGS_DEFAULT_PREC} ")

  endif (PROJECT STREQUAL "CCPP-FV3")
else (${CMAKE_Fortran_COMPILER_ID} STREQUAL "GNU")
  message ("CMAKE_Fortran_COMPILER full path: " ${CMAKE_Fortran_COMPILER})
  message ("Fortran compiler: " ${CMAKE_Fortran_COMPILER_ID})
  message (FATAL_ERROR "This program has only been compiled with gfortran, pgf90 and ifort. If another compiler is needed, the appropriate flags must be added in ${GFS_PHYS_SRC}/CMakeLists.txt")
endif (${CMAKE_Fortran_COMPILER_ID} STREQUAL "GNU")

# The auto-generated caps can contain calls to physics schemes in
# which some of the arguments (pointers, arrays) are not associated/allocated.
# This is on purpose to avoid allocating fields that are not used inside the
# scheme if, for example, certain conditions are not met. To avoid
# Fortran runtime errors, it is necessary to remove checks for pointers
# that are not associated and for array bounds from the caps ONLY. For the
# physics schemes, these checks can and should remain enabled. Overwriting
# the pointer check flags explicitly works for Intel and GNU, but not for PGI.
if (${CMAKE_Fortran_COMPILER_ID} STREQUAL "GNU")
  set_property(SOURCE ${CAPS} APPEND_STRING PROPERTY COMPILE_FLAGS " -fcheck=no-pointer,no-bounds ")
elseif (${CMAKE_Fortran_COMPILER_ID} STREQUAL "Intel")
  set_property(SOURCE ${CAPS} APPEND_STRING PROPERTY COMPILE_FLAGS " -check nopointers,nobounds ")
elseif (${CMAKE_Fortran_COMPILER_ID} STREQUAL "PGI")
  if (CMAKE_Fortran_FLAGS MATCHES ".*chkptr.*")
    message (FATAL_ERROR "PGI compiler option chkptr cannot be used for CCPP physics")
  endif (CMAKE_Fortran_FLAGS MATCHES ".*chkptr.*")
  set_property(SOURCE ${CAPS} APPEND_STRING PROPERTY COMPILE_FLAGS " -Mnobounds ")
endif (${CMAKE_Fortran_COMPILER_ID} STREQUAL "GNU")

if (PROJECT STREQUAL "CCPP-SCM")
  INCLUDE_DIRECTORIES(${CMAKE_BINARY_DIR}/ccpp/framework/src)
endif (PROJECT STREQUAL "CCPP-SCM")

#------------------------------------------------------------------------------

add_library(ccppphys SHARED ${SCHEMES} ${SCHEMES_SFX_OPT} ${CAPS})

target_link_libraries(ccppphys LINK_PUBLIC ${LIBS} ${BACIO_LIB4} ${SP_LIBd} ${W3NCO_LIBd})

set_target_properties(ccppphys PROPERTIES VERSION ${PROJECT_VERSION}
                                     SOVERSION ${PROJECT_VERSION_MAJOR})

if (PROJECT STREQUAL "CCPP-FV3")
  # Define where to install the library
  install(TARGETS ccppphys
          EXPORT ccppphys-targets
          ARCHIVE DESTINATION lib
          LIBRARY DESTINATION lib
          RUNTIME DESTINATION lib
  )
  # Export our configuration
  install(EXPORT ccppphys-targets
          FILE ccppphys-config.cmake
          DESTINATION lib/cmake
  )
  if(STATIC)
    # Define where to install the C headers and Fortran modules
    #install(FILES ${HEADERS_C} DESTINATION include)
    install(FILES ${MODULES_F90} DESTINATION include)
  endif(STATIC)
endif (PROJECT STREQUAL "CCPP-FV3")
