cmake_minimum_required(VERSION 3.14)
project(concorde LANGUAGES C)

# -----------------
# Options
# -----------------
option(USE_QSOPT "Use QSopt LP backend" ON)
set(QSOPT_ROOT "/usr/lib/QS" CACHE PATH "Path to QSopt installation")

# -----------------
# Compiler and linker options
# -----------------
set(WARNING_OPTIONS   -Wall -Wextra -Werror)
set(RELEASE_OPTIONS   -O3)
set(DEBUG_OPTIONS     -O0)
set(GDB_DEBUG_OPTIONS -ggdb3)
set(LINKER_OPTIONS    -flto -Wl,--no-as-needed)

add_compile_options(${WARNING_OPTIONS})
add_compile_options($<$<CONFIG:RELEASE>:${RELEASE_OPTIONS}>)
add_compile_options($<$<CONFIG:DEBUG>:${DEBUG_OPTIONS}>)
add_compile_options($<$<AND:$<CONFIG:DEBUG>,$<CXX_COMPILER_ID:GNU>>:${GDB_DEBUG_OPTIONS}>)
add_link_options(-fPIC -ldl)

include_directories("${CMAKE_CURRENT_SOURCE_DIR}/src/INCLUDE")
include_directories(${QSOPT_ROOT})   # QSopt headers

# -----------------
# UTIL
# -----------------
add_library(concorde_utils
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
target_compile_features(concorde_utils PUBLIC c_std_11)
target_link_libraries(concorde_utils PUBLIC ${LINKER_OPTIONS} PRIVATE dl)
target_compile_options(concorde_utils PRIVATE -Wno-unused-function -Wno-stringop-truncation -Wno-unused-result)
set_target_properties(concorde_utils PROPERTIES POSITION_INDEPENDENT_CODE ON)

# -----------------
# LP (QSopt) - local
# -----------------
set(QSOPT_DIR "${CMAKE_CURRENT_SOURCE_DIR}/src/LP")

if(EXISTS "${QSOPT_DIR}/qsopt.h" AND EXISTS "${QSOPT_DIR}/qsopt.a")
    message(STATUS "Using local QSopt at ${QSOPT_DIR}")

    add_library(concorde_lp
        src/LP/qsopt.c
    )

    target_compile_features(concorde_lp PUBLIC c_std_11)
    target_include_directories(concorde_lp PUBLIC ${QSOPT_DIR})
    target_link_libraries(concorde_lp PUBLIC ${LINKER_OPTIONS} ${QSOPT_DIR}/qsopt.a)
    set_target_properties(concorde_lp PROPERTIES POSITION_INDEPENDENT_CODE ON)

else()
    message(FATAL_ERROR "QSopt not found in ${QSOPT_DIR}. Make sure QSopt is placed there.")
endif()

# -----------------
# Remaining Concorde modules
# -----------------
macro(add_concorde_module name sources)
    add_library(${name} ${sources})
    target_compile_features(${name} PUBLIC c_std_11)
    target_link_libraries(${name} PUBLIC ${LINKER_OPTIONS})
    set_target_properties(${name} PROPERTIES POSITION_INDEPENDENT_CODE ON)
endmacro()

add_concorde_module(concorde_bigguy src/BIGGUY/bigguy.c)
add_concorde_module(concorde_heldkarp src/HELDKARP/heldkarp.c)
add_concorde_module(concorde_edgegen
    src/EDGEGEN/edgegen.c
    src/EDGEGEN/xnear.c
    src/EDGEGEN/delaunay.c
    src/EDGEGEN/mlinkern.c
)
add_concorde_module(concorde_linkern
    src/LINKERN/linkern.c
    src/LINKERN/flip_two.c
)
add_concorde_module(concorde_localcut
    src/LOCALCUT/first.c
    src/LOCALCUT/localcut.c
    src/LOCALCUT/separate.c
    src/LOCALCUT/tsporacl.c
    src/LOCALCUT/chunks.c
    src/LOCALCUT/lift.c
    src/LOCALCUT/intmat.c
)
add_concorde_module(concorde_cut
    src/CUT/shrink.c
    src/CUT/connect.c
    src/CUT/segments.c
    src/CUT/mincut.c
    src/CUT/gomoryhu.c
    src/CUT/cut_st.c
)
add_concorde_module(concorde_pq
    src/PQ/cuttree.c
    src/PQ/consec1.c
    src/PQ/necklace.c
    src/PQ/pq.c
)
target_compile_options(concorde_pq PRIVATE -Wno-unused-function)

add_concorde_module(concorde_fmatch src/FMATCH/fmatch.c)
add_concorde_module(concorde_kdtree
    src/KDTREE/kdbuild.c
    src/KDTREE/kdnear.c
    src/KDTREE/kdspan.c
    src/KDTREE/kdtwoopt.c
)
add_concorde_module(concorde_combs
    src/COMBS/dngreedy.c
    src/COMBS/block.c
)
add_concorde_module(concorde_verify src/VERIFY/verify.c)
add_concorde_module(concorde_tiny
    src/TINY/bnbmsp.c
    src/TINY/tinytsp.c
)

# -----------------
# Main Concorde library
# -----------------
add_library(concorde
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
target_link_libraries(concorde PUBLIC
    ${LINKER_OPTIONS}
    concorde_utils
    concorde_lp
    concorde_bigguy
    concorde_heldkarp
    concorde_edgegen
    concorde_linkern
    concorde_localcut
    concorde_pq
    concorde_cut
    concorde_fmatch
    concorde_kdtree
    concorde_combs
    concorde_verify
    concorde_tiny
    m
    pthread
)
target_compile_options(concorde PRIVATE -Wno-unused-function -Wno-stringop-truncation -Wno-stringop-overflow -Wno-format-overflow -Wno-unused-result)
set_target_properties(concorde PROPERTIES POSITION_INDEPENDENT_CODE ON)

# -----------------
# Executable
# -----------------
add_executable(concorde-bin src/TSP/concorde.c)
target_compile_features(concorde-bin PUBLIC c_std_11)
target_link_libraries(concorde-bin PUBLIC ${LINKER_OPTIONS} concorde)

# -----------------
# Post build: shared + static lib
# -----------------
add_custom_command(TARGET concorde POST_BUILD
    COMMAND gcc -fPIC -shared -o libconcorde_full.so -Wl,--whole-archive *.a ${QSOPT_ROOT}/libqsopt.a -lstdc++ -Wl,--no-whole-archive
    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
    COMMENT "Creating libconcorde_full.so"
)

add_custom_command(TARGET concorde POST_BUILD
    COMMAND ar -x *.a
    COMMAND ar -qc libconcorde_full.a *.o
    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
    COMMENT "Creating libconcorde_full.a"
)

add_custom_command(TARGET concorde POST_BUILD
    COMMAND cat config.h machdefs.h util.h bigguy.h combs.h cut.h delaunay.h edgegen.h fmatch.h heldkarp.h kdtree.h linkern.h lp.h tsp.h consec1.h localcut.h macrorus.h mlinkern.h necklace.h pqsets.h pq.h cuttree.h tinytsp.h tsp.h verify.h | grep -v '\#include \"' > ${CMAKE_CURRENT_BINARY_DIR}/concorde.h
    WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/src/INCLUDE"
    COMMENT "Creating concorde.h"
)

add_custom_command(TARGET concorde-bin POST_BUILD
    COMMAND rm -f *.c.o *.a
    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
    COMMENT "Cleaning up temporary object files"
)
