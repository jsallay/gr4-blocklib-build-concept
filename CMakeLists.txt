cmake_minimum_required(VERSION 3.28)
project(MyProject LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 23)

option(ENABLE_SPLIT "Enable splitting of generated files (pass --split to the parser)" ON)

# --------------------------------------------------------------------------
# Configure the rest of the project
# --------------------------------------------------------------------------

# add core functionality such as registry
add_subdirectory(include)
add_subdirectory(src)

function(create_block_targets LIB_NAME)
    set(options "")
    set(oneValueArgs HEADER_FILE BLOCK_NAME)
    set(multiValueArgs TYPES_LIST)
    cmake_parse_arguments(CBT "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})

    # directory where parse_registrations will output .cpp files.
    set(GEN_DIR "${CMAKE_BINARY_DIR}/generated_plugins/${LIB_NAME}")
    file(MAKE_DIRECTORY "${GEN_DIR}")

    # Generate a random 20 digit number
    string(RANDOM LENGTH 20 ALPHABET "0123456789" RANDOM_INTEGER)

    set(GEN_CPP_LIST "")
    message(STATUS "${CBT_HEADER_FILE} ${CBT_BLOCK_NAME} ${CBT_TYPES_LIST}")
    get_filename_component(ABS_HDR "${CBT_HEADER_FILE}" ABSOLUTE)
    get_filename_component(BASENAME "${CBT_HEADER_FILE}" NAME_WE)
    list(LENGTH CBT_TYPES_LIST TYPES_LENGTH)
    message(STATUS "What: ${TYPES_LENGTH}")
    if (${TYPES_LENGTH} EQUAL 0)
        # NON TEMPLATE BLOCK
        message(STATUS "RUT RO: ${BASENAME} ${CBT_TYPES_LIST}")
        configure_file(${CMAKE_SOURCE_DIR}/cmake/block_nt.cpp.in ${GEN_DIR}/${BASENAME}.cpp )
        list(APPEND GEN_CPP_LIST ${GEN_DIR}/${BASENAME}.cpp)
    else()
        foreach(BLOCK_TYPES ${CBT_TYPES_LIST})
            message(STATUS "Yo ${BLOCK_TYPES}")

            configure_file(${CMAKE_SOURCE_DIR}/cmake/block.cpp.in ${GEN_DIR}/${BASENAME}_${BLOCK_TYPES}.cpp )
            list(APPEND GEN_CPP_LIST ${GEN_DIR}/${BASENAME}_${BLOCK_TYPES}.cpp)
        endforeach()
    endif()
    message(STATUS ${GEN_CPP_LIST})

    # APPEND the generated files to our source list
    target_sources(${LIB_NAME} PRIVATE ${GEN_CPP_LIST})

    # These are repeated, could be moved out, but I don't think it harms anything here.
    target_include_directories(${LIB_NAME} PUBLIC
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
            $<INSTALL_INTERFACE:include>
            "${CMAKE_SOURCE_DIR}/include"
    )
    set_target_properties(${LIB_NAME} PROPERTIES CXX_STANDARD 23)
    install(TARGETS ${LIB_NAME} LIBRARY DESTINATION lib)
endfunction()


# define global block building command
function(create_block_library LIB_NAME)
    set(options "")
    set(oneValueArgs "")
    set(multiValueArgs HEADERS SOURCES TYPES_LIST)
    cmake_parse_arguments(CBL "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})

    # directory where parse_registrations will output .cpp files.
    set(GEN_DIR "${CMAKE_BINARY_DIR}/generated_plugins/${LIB_NAME}")
    file(MAKE_DIRECTORY "${GEN_DIR}")

    set(GEN_CPP_LIST "")
    foreach(HDR IN LISTS CBL_HEADERS)
        get_filename_component(ABS_HDR "${HDR}" ABSOLUTE)
        get_filename_component(BASENAME "${HDR}" NAME_WE)

        # remove any previously generated .cpp files for this header.
        file(GLOB OLD_FILES "${GEN_DIR}/${BASENAME}*.cpp")
        if(OLD_FILES)
            message(STATUS "deleting old generated files ${OLD_FILES}")
            file(REMOVE ${OLD_FILES})
        endif()

        if(ENABLE_SPLIT)
            set(PARSER_SPLIT_OPTION "--split")
        else()
            set(PARSER_SPLIT_OPTION "")
        endif()

        message(STATUS "Generating code for header: ${ABS_HDR} -> ${GEN_DIR} with PARSER_SPLIT_OPTION=${PARSER_SPLIT_OPTION}")

        # run the parser tool immediately at configuration time.
        execute_process(
                COMMAND ${PARSER_EXECUTABLE} "${ABS_HDR}" "${GEN_DIR}" ${PARSER_SPLIT_OPTION}
                RESULT_VARIABLE gen_res
                OUTPUT_VARIABLE gen_out
                ERROR_VARIABLE gen_err
                OUTPUT_STRIP_TRAILING_WHITESPACE
                ERROR_STRIP_TRAILING_WHITESPACE
        )
        message(STATUS "Output from parse_registrations for ${ABS_HDR}:\n${gen_out}")
        if(NOT gen_res EQUAL 0)
            message(FATAL_ERROR "Error running parse_registrations on ${HDR}: ${gen_err}")
        endif()

        # Glob for the generated .cpp files.
        file(GLOB GENERATED_FILES "${GEN_DIR}/${BASENAME}*.cpp")
        if(NOT GENERATED_FILES)
            # If no .cpp files were generated, create a dummy file.
            set(DUMMY_CPP "${GEN_DIR}/dummy_${BASENAME}.cpp")
            file(WRITE "${DUMMY_CPP}" "// No macros or expansions found for '${BASENAME}'\n")
            list(APPEND GENERATED_FILES "${DUMMY_CPP}")
        endif()

        list(APPEND GEN_CPP_LIST ${GENERATED_FILES})
    endforeach()

    add_library(${LIB_NAME} SHARED
            ${GEN_CPP_LIST}
            ${CBL_SOURCES}
    )
    target_include_directories(${LIB_NAME} PUBLIC
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
            $<INSTALL_INTERFACE:include>
            "${CMAKE_SOURCE_DIR}/include"
    )
    set_target_properties(${LIB_NAME} PROPERTIES CXX_STANDARD 23)
    install(TARGETS ${LIB_NAME} LIBRARY DESTINATION lib)
endfunction()

# --------------------------------------------------------------------------
# 4. Add subdirectories defining libraries via create_block_library
# --------------------------------------------------------------------------
add_subdirectory(blocks)

# Optionally build aggregator library.
if (TARGET grBasic AND TARGET grMath)
    add_library(grBlockLib SHARED src/dummy.cpp)
    target_link_libraries(grBlockLib PUBLIC grBasic grMath)
    set_target_properties(grBlockLib PROPERTIES CXX_STANDARD 23)
    install(TARGETS grBlockLib LIBRARY DESTINATION lib)
endif()

add_subdirectory(example)
