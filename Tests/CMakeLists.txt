set(EXECUTABLE_OUTPUT_PATH ${CMAKE_CURRENT_BINARY_DIR})
include_directories(${CMAKE_CURRENT_BINARY_DIR})

set(TEST_BASE_PATH "./tmp")
add_definitions("-DBASE_PATH=\"${TEST_BASE_PATH}\"")

add_executable(ExecuteTest ExecuteTest.cpp QueryRunner.cpp)
add_executable(RunQueryLoop RunQueryLoop.cpp QueryRunner.cpp)
add_executable(StringDictionaryTest StringDictionaryTest.cpp)
add_executable(PlanTest PlanTest.cpp)
add_executable(ProfileTest ProfileTest.cpp)

if(ENABLE_CUDA)
  set(MAPD_DEFINITIONS -DHAVE_CUDA)
  add_custom_command(
    DEPENDS ProfileUtils.cu ProfileTest.h
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/libProfileUtils.so
    COMMAND nvcc
    ARGS
        ${MAPD_HOST_COMPILER_FLAG}
        -Xcompiler -fPIC
        -O3
        -D_FORCE_INLINES
        ${MAPD_DEFINITIONS}
        -arch sm_30
        -std=c++11
        -shared
        ${CMAKE_CURRENT_SOURCE_DIR}/ProfileUtils.cu
        -o ${CMAKE_CURRENT_BINARY_DIR}/libProfileUtils.so
    )
  add_custom_target(ProfileUtils DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/libProfileUtils.so)
  add_dependencies(ProfileTest ProfileUtils)
  set(PROF_LIBRARIES ${CMAKE_CURRENT_BINARY_DIR}/libProfileUtils.so)
endif()

add_executable(ResultSetTest ResultSetTest.cpp ResultSetTestUtils.cpp)
add_executable(ResultSetBaselineRadixSortTest ResultSetBaselineRadixSortTest.cpp ResultSetTestUtils.cpp)
add_executable(UtilTest UtilTest.cpp)
add_executable(StorageTest StorageTest.cpp PopulateTableRandom.cpp ScanTable.cpp)
add_executable(StoragePerfTest StoragePerfTest.cpp PopulateTableRandom.cpp ScanTable.cpp)
add_executable(ImportTest ImportTest.cpp)

target_link_libraries(PlanTest gtest ${MAPD_LIBRARIES} ${Boost_LIBRARIES} ${Glog_LIBRARIES} ${CMAKE_DL_LIBS} ${CUDA_LIBRARIES} ${LLVM_LINKER_FLAGS} ${CURSES_LIBRARIES})
target_link_libraries(ProfileTest gtest ${MAPD_LIBRARIES} ${Boost_LIBRARIES} ${Glog_LIBRARIES} ${CMAKE_DL_LIBS} ${CUDA_LIBRARIES} ${PROF_LIBRARIES} ${LLVM_LINKER_FLAGS} ${CURSES_LIBRARIES})
target_link_libraries(UtilTest Utils gtest ${Boost_LIBRARIES})
target_link_libraries(StorageTest gtest ${MAPD_LIBRARIES} ${Boost_LIBRARIES} ${Glog_LIBRARIES} ${CMAKE_DL_LIBS} ${CUDA_LIBRARIES} ${LLVM_LINKER_FLAGS} ${CURSES_LIBRARIES})
target_link_libraries(StoragePerfTest gtest ${MAPD_LIBRARIES} ${Boost_LIBRARIES} ${Glog_LIBRARIES} ${CMAKE_DL_LIBS} ${CUDA_LIBRARIES} ${LLVM_LINKER_FLAGS} ${CURSES_LIBRARIES})
set(EXECUTE_TEST_LIBS gtest ${MAPD_LIBRARIES} ${Boost_LIBRARIES} ${Glog_LIBRARIES} ${CMAKE_DL_LIBS} ${CUDA_LIBRARIES} ${LLVM_LINKER_FLAGS} ${CURSES_LIBRARIES})
list(APPEND EXECUTE_TEST_LIBS Calcite)
target_link_libraries(ExecuteTest ${EXECUTE_TEST_LIBS})
target_link_libraries(ResultSetTest ${EXECUTE_TEST_LIBS})
target_link_libraries(ResultSetBaselineRadixSortTest ${EXECUTE_TEST_LIBS})
target_link_libraries(RunQueryLoop ${EXECUTE_TEST_LIBS})
target_link_libraries(StringDictionaryTest StringDictionary gtest ${Boost_LIBRARIES})
target_link_libraries(ImportTest gtest ${MAPD_LIBRARIES})

set(TEST_ARGS "--gtest_output=xml:../")
add_test(PlanTest PlanTest ${TEST_ARGS})
add_test(UtilTest UtilTest ${TEST_ARGS})
add_test(ExecuteTest ExecuteTest ${TEST_ARGS})
add_test(ResultSetTest ResultSetTest ${TEST_ARGS})
add_test(ResultSetBaselineRadixSortTest ResultSetBaselineRadixSortTest ${TEST_ARGS})
add_test(RunQueryLoop RunQueryLoop ${TEST_ARGS})
add_test(StringDictionaryTest StringDictionaryTest ${TEST_ARGS})
add_test(StorageTest StorageTest ${TEST_ARGS})
add_test(StoragePerfTest StoragePerfTest ${TEST_ARGS})
add_test(ImportTest ImportTest ${TEST_ARGS})

add_custom_target(sanity_tests
    COMMAND mkdir -p ${TEST_BASE_PATH}
    COMMAND initdb -f ${TEST_BASE_PATH}
    COMMAND ${CMAKE_CTEST_COMMAND} --verbose --tests-regex "\"(PlanTest|ExecuteTest|ResultSetTest|ResultSetBaselineRadixSortTest|StorageTest|ImportTest)\""
    DEPENDS PlanTest ExecuteTest ResultSetTest ResultSetBaselineRadixSortTest StorageTest ImportTest)

add_custom_target(all_tests
    COMMAND mkdir -p ${TEST_BASE_PATH}
    COMMAND initdb -f ${TEST_BASE_PATH}
    COMMAND ${CMAKE_CTEST_COMMAND} --verbose
    DEPENDS PlanTest ProfileTest UtilTest ExecuteTest ResultSetTest ResultSetBaselineRadixSortTest RunQueryLoop StringDictionaryTest StorageTest StoragePerfTest ImportTest)

add_custom_target(storage_perf_tests
    COMMAND mkdir -p ${TEST_BASE_PATH}
    COMMAND initdb -f ${TEST_BASE_PATH}
    COMMAND ${CMAKE_CTEST_COMMAND} --verbose --tests-regex "\"(StoragePerfTest)\""
    DEPENDS StoragePerfTest)
