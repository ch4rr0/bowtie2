cmake_minimum_required(VERSION 2.8) 

cmake_policy(SET CMP0048 NEW)
cmake_policy(SET CMP0005 NEW)

project(bowtie2 LANGUAGES CXX VERSION "2.3.3.1")

enable_testing()

include(CTest)
include(ProcessorCount)
include(CheckSymbolExists)
include("cmake/TBBGet.cmake")

ProcessorCount(NUM_CORES)

option(BOWTIE_MM "enable bowtie2 memory mapping" ON)
option(BOWITE_SHARED_MM "enable shared memory mapping" ON)

set(EXTRA_FLAGS "${EXTRA_FLAGS} -DNO_SPINLOCK -DWITH_QUEUELOCK")
set(CMAKE_CXX_STANDARD 98)
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -O0 -msse2 -g3 -m64 -Wall")
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O3 -msse2 -m64 -funroll-loops -g3 -Wall")
set(INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/bin" CACHE PATH "Installation directory for executables")

set(BOWTIE2_BIN_LIST
    bowtie2-build-s
    bowtie2-build-l
    bowtie2-align-s
    bowtie2-align-l
    bowtie2-inspect-s
    bowtie2-inspect-l
)

set(SHARED_CPPS
    ccnt_lut.cpp
    ref_read.cpp
    alphabet.cpp
    shmem.cpp
    edit.cpp
    bt2_idx.cpp
    bt2_io.cpp
    bt2_util.cpp
    reference.cpp
    ds.cpp
    multikey_qsort.cpp
    limit.cpp
    random_source.cpp
)

set(SEARCH_CPPS
    qual.cpp pat.cpp sam.cpp
    read_qseq.cpp aligner_seed_policy.cpp 
    aligner_seed.cpp 
    aligner_seed2.cpp 
    aligner_sw.cpp 
    aligner_sw_driver.cpp aligner_cache.cpp 
    aligner_result.cpp ref_coord.cpp mask.cpp 
    pe.cpp aln_sink.cpp dp_framer.cpp 
    scoring.cpp presets.cpp unique.cpp 
    simple_func.cpp 
    random_util.cpp 
    aligner_bt.cpp sse_util.cpp 
    aligner_swsse.cpp outq.cpp 
    aligner_swsse_loc_i16.cpp 
    aligner_swsse_ee_i16.cpp 
    aligner_swsse_loc_u8.cpp 
    aligner_swsse_ee_u8.cpp 
    aligner_driver.cpp
    bowtie_main.cpp
    bt2_search.cpp
)

set(BUILD_CPPS
    bt2_build.cpp
    diff_sample.cpp
    bowtie_build_main.cpp)

set(INSPECT_CPPS
   bt2_inspect.cpp 
)

string(TIMESTAMP BUILD_DATE)
execute_process(COMMAND ${CMAKE_CXX_COMPILER} -v OUTPUT_VARIABLE COMPILER_VERSION)

add_definitions(
    -DWITH_TBB
    -DWITH_QUEUELOCK
    -DNO_SPINLOCK
    -DBOWTIE2
    -DBUILD_HOST="${CMAKE_HOST_SYSTEM}"
    -DBUILD_TIME="${BUILD_DATE}"
    -DCOMPILER_VERSION="${CMAKE_CXX_COMPILER} ${CMAKE_CXX_COMPILER_VERSION}"
    -DCOMPILER_OPTIONS="${CMAKE_CXX_FLAGS}"
    -DBOWTIE2_VERSION="${PROJECT_VERSION}"
)

if (CMAKE_BUILD_TYPE STREQUAL "Release")
    add_definitions(-DNDEBUG)
endif()

if (MINGW)
    set(WINDOWS 1)
    option(BOWTIE_MM "Memory mapped files not supported on Windows" OFF)
    option(BOWTIE_SHARED_MEM "Shared memory not supported on Windows" OFF)
endif(MINGW)

find_package(Threads)
find_path(TBB_INCLUDE_PATH tbb)
find_library(ZLIB_LIBRARY_PATH z)
find_library(TBB_LIBRARY_PATH tbb)
find_library(TBB_MALLOC_LIBRARY_PATH tbbmalloc)

if (NOT TBB_LIBRARY_PATH)
    tbb_get(SAVE_TO "${PROJECT_SOURCE_DIR}/.bowtie2-deps" TBB_ROOT tbb_root) 
    find_library(TBB_LIBRARY_PATH tbb PATHS "${tbb_root}/lib")
    find_library(TBB_MALLOC_LIBRARY_PATH tbbmalloc_proxy PATHS "${tbb_root}/lib")
    find_path(TBB_INCLUDE_PATH tbb PATHS "${tbb_root}/include")
endif()

include_directories(${PROJECT_SOURCE_DIR} ${TBB_INCLUDE_PATH})
link_libraries(${ZLIB_LIBRARY_PATH} ${CMAKE_THREAD_LIBS_INIT} ${TBB_LIBRARY_PATH} ${TBB_MALLOC_LIBRARY_PATH})

add_executable(bowtie2-align-s ${SEARCH_CPPS} ${SHARED_CPPS})
add_executable(bowtie2-align-l ${SEARCH_CPPS} ${SHARED_CPPS})
add_executable(bowtie2-build-s ${BUILD_CPPS} ${SHARED_CPPS})
add_executable(bowtie2-build-l ${BUILD_CPPS} ${SHARED_CPPS})
add_executable(bowtie2-inspect-s ${INSPECT_CPPS} ${SHARED_CPPS})
add_executable(bowtie2-inspect-l ${INSPECT_CPPS} ${SHARED_CPPS})

set_target_properties(bowtie2-align-l bowtie2-build-l bowtie2-inspect-l PROPERTIES COMPILE_FLAGS "-DBOWTIE2_64BIT_INDEX")
set_target_properties(bowtie2-inspect-s bowtie2-inspect-l PROPERTIES COMPILE_FLAGS "-DBOWTIE_INSPECT_MAIN")
set_target_properties(${BOWTIE2_BIN_LIST} PROPERTIES DEBUG_POSTFIX "-debug")

function(_get_perl_deps)
    file(DOWNLOAD "http://cpanmin.us" "${PROJECT_SOURCE_DIR}/.bowtie2-deps/cpanmin.pl")
    execute_process(COMMAND sh "-c"  "cat ${PROJECT_SOURCE_DIR}/.bowtie2-deps/cpanmin.pl | perl - -l ${PROJECT_SOURCE_DIR}/.bowtie2-deps App::cpanminus local::lib")
    execute_process(COMMAND perl -I ${PROJECT_SOURCE_DIR}/.bowtie2-deps/lib/perl5 -Mlocal::lib=${PROJECT_SOURCE_DIR}/.bowtie2-deps)
    execute_process(COMMAND cpanm --force Math::Random Clone Test::Deep Sys::Info)
endfunction(_get_perl_deps)

add_test(NAME simple-align COMMAND ${PROJECT_SOURCE_DIR}/bowtie2 -x example/index/lambda_virus example/reads/longreads.fq)
# _get_perl_deps()
# add_test(NAME simple-tests COMMAND
#     sh -c "eval `perl -I ${PROJECT_SOURCE_DIR}/.bowtie2-deps/lib/perl5 -Mlocal::lib=${PROJECT_SOURCE_DIR}/.bowtie2-deps`;
#     sh ${PROJECT_SOURCE_DIR}/scripts/test/simple_tests.sh")
# add_test(NAME random-tests COMMAND
#     "perl -I ${PROJECT_SOURCE_DIR}/.bowtie2-deps/lib/perl5 -Mlocal::lib=${PROJECT_SOURCE_DIR}/.bowtie2-deps; \
#     ${PROJECT_SOURCE_DIR}/scripts/sim/run.sh ${NUM_CORES}")
