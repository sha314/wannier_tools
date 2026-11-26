cmake_minimum_required(VERSION 3.16)
project(WannierTools LANGUAGES Fortran)

# ---------------------------------------------------------------------------
# Options
# ---------------------------------------------------------------------------
option(WT_USE_MPI        "Build with MPI support (-DMPI)"                  OFF)
option(WT_USE_ARPACK     "Link against ARPACK"                             OFF)
option(WT_COPY_TO_BIN    "Copy wt.x to ../bin after build (as in Makefile)" ON)
set(OPENBLAS_DIR "/opt/openblas" CACHE PATH
    "OpenBLAS prefix (lib64/ or lib/ is searched); falls back to system LAPACK/BLAS")

# Default to an optimized build (Makefile used -O3)
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

# ---------------------------------------------------------------------------
# Compiler flags
# ---------------------------------------------------------------------------
if(CMAKE_Fortran_COMPILER_ID STREQUAL "GNU")
  # gfortran -cpp ... -O3 -ffree-line-length-512
  add_compile_options($<$<COMPILE_LANGUAGE:Fortran>:-cpp>
                      $<$<COMPILE_LANGUAGE:Fortran>:-ffree-line-length-512>)
  set(CMAKE_Fortran_FLAGS_RELEASE "-O3")
elseif(CMAKE_Fortran_COMPILER_ID MATCHES "Intel")
  add_compile_options($<$<COMPILE_LANGUAGE:Fortran>:-fpp>)
  set(CMAKE_Fortran_FLAGS_RELEASE "-O3")
endif()

# Keep .mod files out of the source tree
set(CMAKE_Fortran_MODULE_DIRECTORY ${CMAKE_BINARY_DIR}/modules)

# ---------------------------------------------------------------------------
# Sources (same list as OBJ in the Makefile)
# ---------------------------------------------------------------------------
set(WT_SOURCES
  src/module.f90
  src/sparse.f90
  src/wt_aux.f90
  src/math_lib.f90
  src/mgmres.f90
  src/symmetry.f90
  src/readHmnR.f90
  src/inverse.f90
  src/proteus.f90
  src/eigen.f90
  src/ham_qlayer2qlayer.f90
  src/psi.f90
  src/unfolding.f90
  src/rand.f90
  src/ham_slab.f90
  src/ham_bulk.f90
  src/ek_slab.f90
  src/ek_bulk_polar.f90
  src/ek_bulk.f90
  src/readinput.f90
  src/fermisurface.f90
  src/surfgreen.f90
  src/surfstat.f90
  src/mat_mul.f90
  src/ham_ribbon.f90
  src/ek_ribbon.f90
  src/fermiarc.f90
  src/berrycurvature.f90
  src/wanniercenter.f90
  src/dos.f90
  src/orbital_momenta.f90
  src/landau_level_sparse.f90
  src/landau_level.f90
  src/lanczos_sparse.f90
  src/berry.f90
  src/wanniercenter_adaptive.f90
  src/effective_mass.f90
  src/findnodes.f90
  src/sigma_OHE.f90
  src/sigma.f90
  src/Boltz_transport_anomalous.f90
  src/2D_TSC.f90
  src/optic.f90
  src/orbital_hall.f90
  src/main.f90
)

add_executable(wt ${WT_SOURCES})
set_target_properties(wt PROPERTIES OUTPUT_NAME "wt.x")

# ---------------------------------------------------------------------------
# BLAS / LAPACK: prefer OpenBLAS in OPENBLAS_DIR, otherwise system -llapack -lblas
# ---------------------------------------------------------------------------
find_library(WT_OPENBLAS_LIB
  NAMES openblas
  PATHS ${OPENBLAS_DIR}/lib64 ${OPENBLAS_DIR}/lib
  NO_DEFAULT_PATH)

if(WT_OPENBLAS_LIB)
  message(STATUS "Using OpenBLAS: ${WT_OPENBLAS_LIB}")
  target_link_libraries(wt PRIVATE ${WT_OPENBLAS_LIB})
else()
  message(STATUS "OpenBLAS not found in ${OPENBLAS_DIR}; using system BLAS/LAPACK")
  find_package(LAPACK REQUIRED)
  find_package(BLAS REQUIRED)
  target_link_libraries(wt PRIVATE ${LAPACK_LIBRARIES} ${BLAS_LIBRARIES})
endif()

# ---------------------------------------------------------------------------
# Optional: ARPACK
# ---------------------------------------------------------------------------
if(WT_USE_ARPACK)
  find_library(ARPACK_LIBRARY NAMES arpack REQUIRED)
  target_link_libraries(wt PRIVATE ${ARPACK_LIBRARY})
endif()

# ---------------------------------------------------------------------------
# Optional: MPI (use with e.g. -DCMAKE_Fortran_COMPILER=mpif90 or let CMake find MPI)
# ---------------------------------------------------------------------------
if(WT_USE_MPI)
  find_package(MPI REQUIRED COMPONENTS Fortran)
  target_compile_definitions(wt PRIVATE MPI)
  target_link_libraries(wt PRIVATE MPI::MPI_Fortran)
endif()

# ---------------------------------------------------------------------------
# Mimic "cp -f wt.x ../bin"
# ---------------------------------------------------------------------------
if(WT_COPY_TO_BIN)
  set(WT_BIN_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../bin")
  add_custom_command(TARGET wt POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E make_directory "${WT_BIN_DIR}"
    COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_FILE:wt> "${WT_BIN_DIR}"
    COMMENT "Copying wt.x to ${WT_BIN_DIR}")
endif()

# ---------------------------------------------------------------------------
# make install  (PREFIX -> CMAKE_INSTALL_PREFIX, DESTDIR works as usual)
# ---------------------------------------------------------------------------
include(GNUInstallDirs)
install(TARGETS wt RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})