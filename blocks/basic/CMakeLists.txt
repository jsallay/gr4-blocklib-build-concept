# ./blocks/basic/CMakeLists.txt

# list module block headers (absolute or relative)
set(MY_HEADERS
        include/gnuradio-4.x/basic/block0.hpp
        include/gnuradio-4.x/basic/block1.hpp
        include/gnuradio-4.x/basic/block2.hpp
        include/gnuradio-4.x/basic/block3.hpp
        include/gnuradio-4.x/basic/blockN.hpp
        # add more as necessary
)

# if there are additional C++ sources required for the build e.g. in blocks/basic/src
file(GLOB MY_SRCS "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp")

add_library(grBasic SHARED)
create_block_targets(grBasic 
        HEADER_FILE include/gnuradio-4.x/basic/block0.hpp
        BLOCK_NAME "gr::basic::Block0"
        TYPES_LIST "")
create_block_targets(grBasic 
        HEADER_FILE include/gnuradio-4.x/basic/block1.hpp
        BLOCK_NAME "gr::basic::Block1"
        TYPES_LIST "float;double")
create_block_targets(grBasic 
        HEADER_FILE include/gnuradio-4.x/basic/block2.hpp
        BLOCK_NAME "gr::basic::Block2"
        TYPES_LIST "int,double;float,double")
create_block_targets(grBasic 
        HEADER_FILE include/gnuradio-4.x/basic/block3.hpp
        BLOCK_NAME "gr::basic::Block3"
        TYPES_LIST "int")
create_block_targets(grBasic 
        HEADER_FILE include/gnuradio-4.x/basic/blockN.hpp
        BLOCK_NAME "gr::basic::BlockN"
        TYPES_LIST "float,short,3UZ;float,int,3UZ;float,long,3UZ;float,long long,3UZ;double,short,3UZ;double,int,3UZ;double,long,3UZ;double,long long,3UZ;")

# optionally:
install(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/include/" DESTINATION include/gnuradio-4.x/basic FILES_MATCHING PATTERN "*.hpp")

if(ENABLE_TESTING)
    add_subdirectory(test)
endif()
