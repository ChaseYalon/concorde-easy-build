cmake_minimum_required(VERSION 3.14)
project(concorde LANGUAGES C)

# -----------------
# Options
# -----------------
option(USE_QSOPT "Use QSopt LP backend" ON)
set(QSOPT_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/src/LP" CACHE PATH "Path to QSopt installation")

# -----------------
# Compiler and linker options
# -----------------
set(WARNING_OPTIONS   -Wall -Wextra)
set(RELEASE_OPTIONS   -O3)
set(DEBUG_OPTIONS     -O0 -g)
set(GDB_DEBUG_OPTIONS -ggdb3)

add_compile_options(${WARNING_OPTIONS})
add_compile_options($<$<CONFIG:RELEASE>:${RELEASE_OPTIONS}>)
add_compile_options($<$<CONFIG:DEBUG>:${DEBUG_OPTIONS}>)
add_compile_options($<$<AND:$<CONFIG:DEBUG>,$<CXX_COMPILER_ID:GNU>>:${GDB_DEBUG_OPTIONS}>)

include_directories("${CMAKE_CURRENT_SOURCE_DIR}/src/INCLUDE")
include_directories(${QSOPT_ROOT})

# -----------------
# QSopt LP backend
# -----------------
set(QSOPT_DIR "${CMAKE_CURRENT_SOURCE_DIR}/src/LP")

if(EXISTS "${QSOPT_DIR}/qsopt.h" AND EXISTS "${QSOPT_DIR}/qsopt.a")
    message(STATUS "Using local QSopt at ${QSOPT_DIR}")
    
    add_library(concorde_lp STATIC
        src/LP/qsopt.c
    )
    
    target_compile_features(concorde_lp PUBLIC c_std_11)
    target_include_directories(concorde_lp PUBLIC ${QSOPT_DIR})
    target_link_libraries(concorde_lp PUBLIC ${QSOPT_DIR}/qsopt.a)
    set_target_properties(concorde_lp PROPERTIES POSITION_INDEPENDENT_CODE ON)
    
else()
    message(FATAL_ERROR "QSopt not found in ${QSOPT_DIR}. Make sure QSopt is placed there.")
endif()

# -----------------
# Utility functions
# -----------------
macro(add_concorde_module name sources)
    add_library(${name} STATIC ${sources})
    target_compile_features(${name} PUBLIC c_std_11)
    target_include_directories(${name} PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/src/INCLUDE")
    set_target_properties(${name} PROPERTIES POSITION_INDEPENDENT_CODE ON)
    # Suppress warnings for legacy code
    target_compile_options(${name} PRIVATE 
        -Wno-unused-function 
        -Wno-stringop-truncation 
        -Wno-unused-result
        -Wno-stringop-overflow
        -Wno-format-overflow
    )
endmacro()

# -----------------
# UTIL module
# -----------------
add_concorde_module(concorde_utils
    src/UTIL/allocrus.c
    src/UTIL/util.c
    src/UTIL/edgelen.c
    src/UTIL/zeit.c
    src/UTIL/urandom.c
    src/UTIL/signal.c
    src/UTIL/getdata.c
    src/UTIL/edgeutil.c
    src/UTIL/bgetopt.c
    src/UTIL/safe_io.c
    src/UTIL/sortrus.c
    src/UTIL/genhash.c
    src/UTIL/edgemap.c
    src/UTIL/priority.c
    src/UTIL/dheaps_i.c
    src/UTIL/fastread.c
)
target_link_libraries(concorde_utils PUBLIC dl)

# -----------------
# BIGGUY module
# -----------------
add_concorde_module(concorde_bigguy src/BIGGUY/bigguy.c)

# -----------------
# HELDKARP module  
# -----------------
add_concorde_module(concorde_heldkarp src/HELDKARP/heldkarp.c)

# -----------------
# CUT module
# -----------------
add_concorde_module(concorde_cut
    src/CUT/shrink.c
    src/CUT/connect.c
    src/CUT/segments.c
    src/CUT/mincut.c
    src/CUT/gomoryhu.c
    src/CUT/cut_st.c
)

# -----------------
# FMATCH module
# -----------------
add_concorde_module(concorde_fmatch src/FMATCH/fmatch.c)

# -----------------
# KDTREE module
# -----------------
add_concorde_module(concorde_kdtree
    src/KDTREE/kdbuild.c
    src/KDTREE/kdnear.c
    src/KDTREE/kdspan.c
    src/KDTREE/kdtwoopt.c
)

# -----------------
# PQ module
# -----------------
add_concorde_module(concorde_pq
    src/PQ/cuttree.c
    src/PQ/consec1.c
    src/PQ/necklace.c
    src/PQ/pq.c
)

# -----------------
# LINKERN module (contains flipper functions)
# -----------------
add_concorde_module(concorde_linkern
    src/LINKERN/linkern.c
    src/LINKERN/flip_two.c
    src/LINKERN/flipper.c
)

# -----------------
# EDGEGEN module
# -----------------
add_concorde_module(concorde_edgegen
    src/EDGEGEN/edgegen.c
    src/EDGEGEN/xnear.c
    src/EDGEGEN/delaunay.c
    src/EDGEGEN/mlinkern.c
    src/EDGEGEN/x_k_near.c
    src/EDGEGEN/junk_nn.c
    src/EDGEGEN/quad_nn.c
    src/EDGEGEN/greedy.c
    src/EDGEGEN/f2match.c
    src/EDGEGEN/spanning.c
)

# -----------------
# LOCALCUT module (contains chunk functions)
# -----------------
add_concorde_module(concorde_localcut
    src/LOCALCUT/first.c
    src/LOCALCUT/localcut.c
    src/LOCALCUT/separate.c
    src/LOCALCUT/tsporacl.c
    src/LOCALCUT/chunks.c
    src/LOCALCUT/lift.c
    src/LOCALCUT/intmat.c
)

# -----------------
# COMBS module
# -----------------
add_concorde_module(concorde_combs
    src/COMBS/dngreedy.c
    src/COMBS/block.c
)

# -----------------
# VERIFY module
# -----------------
add_concorde_module(concorde_verify src/VERIFY/verify.c)

# -----------------
# TINY module
# -----------------
add_concorde_module(concorde_tiny
    src/TINY/bnbmsp.c
    src/TINY/tinytsp.c
)

# -----------------
# Main TSP library
# -----------------
add_library(concorde STATIC
    src/TSP/concorde.c
    src/TSP/control.c
    src/TSP/prob_io.c
    src/TSP/branch.c
    src/TSP/cutpool.c
    src/TSP/bcontrol.c
    src/TSP/tsp_lp.c
    src/TSP/ex_price.c
    src/TSP/generate.c
    src/TSP/cutcall.c
    src/TSP/blossom.c
    src/TSP/blkcomb.c
    src/TSP/growcomb.c
    src/TSP/prclique.c
    src/TSP/xtour.c
    src/TSP/cliqwork.c
    src/TSP/skeleton.c
    src/TSP/cliqhash.c
    src/TSP/qsparse.c
    src/TSP/tighten.c
    src/TSP/teething.c
    src/TSP/ddecker.c
    src/TSP/combcliq.c
    src/TSP/tsp_call.c
)

target_compile_features(concorde PUBLIC c_std_11)
target_include_directories(concorde PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/src/INCLUDE")
set_target_properties(concorde PROPERTIES POSITION_INDEPENDENT_CODE ON)
target_compile_options(concorde PRIVATE 
    -Wno-unused-function 
    -Wno-stringop-truncation 
    -Wno-unused-result
    -Wno-stringop-overflow
    -Wno-format-overflow
)

# Link all dependencies in correct order
target_link_libraries(concorde PUBLIC
    concorde_utils
    concorde_lp
    concorde_bigguy
    concorde_heldkarp
    concorde_cut
    concorde_fmatch
    concorde_kdtree
    concorde_pq
    concorde_linkern
    concorde_edgegen
    concorde_localcut
    concorde_combs
    concorde_verify
    concorde_tiny
    m
    pthread
    dl
)

# -----------------
# Executable
# -----------------
add_executable(concorde-bin src/TSP/concorde.c)
target_compile_features(concorde-bin PUBLIC c_std_11)
target_link_libraries(concorde-bin PUBLIC concorde)

# Set executable output name
set_target_properties(concorde-bin PROPERTIES OUTPUT_NAME concorde)
