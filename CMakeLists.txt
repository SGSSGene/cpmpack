# SPDX-FileCopyrightText: 2024 Simon Gene Gottlieb
# SPDX-License-Identifier: MIT

cmake_minimum_required(VERSION 3.31 FATAL_ERROR)

function(read_entry ITEM FIELD PREFIX VAR_OUT WITH_QUOTES AS_LIST)
    set(RET_VALUE)
    string(JSON TYPE_VALUE ERROR_VARIABLE JSON_ERROR TYPE ${ITEM} ${FIELD})

    set(QUOTES "")
    if (WITH_QUOTES)
        set(QUOTES "\"")
    endif()

    # If not found, check if lower case can be found
    if (NOT ${JSON_ERROR} MATCHES "^NOTFOUND$")
        string(TOLOWER ${FIELD} FIELD)
        string(JSON TYPE_VALUE ERROR_VARIABLE JSON_ERROR TYPE ${ITEM} ${FIELD})
    endif()

    if (NOT ${JSON_ERROR} MATCHES "^NOTFOUND$")
        # nothing to do
    elseif (${TYPE_VALUE} MATCHES "^STRING$")
        string(JSON VALUE GET ${ITEM} ${FIELD})
        if (NOT ${VALUE} MATCHES "^$")
            set(RET_VALUE "${PREFIX}${QUOTES}${VALUE}${QUOTES}")
        endif()
    elseif (${TYPE_VALUE} MATCHES "^BOOLEAN$")
        string(JSON VALUE GET ${ITEM} ${FIELD})
        if (${VALUE} MATCHES "ON")
            set(RET_VALUE "${PREFIX}ON")
        else()
            set(RET_VALUE "${PREFIX}OFF")
        endif()

    elseif (${TYPE_VALUE} MATCHES "^ARRAY$")
        string(JSON ARRAY_LENGTH LENGTH ${ITEM} ${FIELD})
        if (${ARRAY_LENGTH} EQUAL 0)
            return()
        endif()
        MATH(EXPR ARRAY_LENGTH "${ARRAY_LENGTH}-1")

        set(RET_VALUE ${PREFIX})
        set(SEP " ")
        if (AS_LIST)
            set(SEP ";")
        endif()

        foreach (IDX RANGE ${ARRAY_LENGTH})
            string(JSON VALUE GET ${ITEM} ${FIELD} ${IDX})
            set(RET_VALUE "${RET_VALUE}${SEP}${QUOTES}${VALUE}${QUOTES}")
        endforeach()
    else()
        message(FATAL_ERROR "Unexpected JSON type ${TYPE_VALUE} at position ${IDX} ${FIELD}")
    endif()

    set(${VAR_OUT} "${RET_VALUE}" PARENT_SCOPE)
endfunction()

function(read_packages_section FILE_AS_STRING)
    # Iterate over it and declare all dependencies
    string(JSON number_of_objects LENGTH ${FILE_AS_STRING})
    MATH(EXPR number_of_objects "${number_of_objects}-1")
    foreach (IDX RANGE ${number_of_objects})
        string(JSON TYPE_VALUE TYPE ${FILE_AS_STRING} ${IDX})
        string(JSON ITEM GET ${FILE_AS_STRING} ${IDX})
        if (${TYPE_VALUE} MATCHES "^OBJECT$")
            #!TODO Many options missing
            read_entry(${ITEM} IF "" DEP_IF NO NO)
            if ("${DEP_IF}" MATCHES ".")
                set(TEST_IF "
                    if (${DEP_IF})
                        set(MUST_LOAD_THIS_PACKAGE ON)
                    else()
                        set(MUST_LOAD_THIS_PACKAGE OFF)
                    endif()"
                )
                cmake_language(EVAL CODE "${TEST_IF}")
                if (NOT ${MUST_LOAD_THIS_PACKAGE})
                    continue()
                endif()
            endif ()

            read_entry(${ITEM} NAME "" DEP_NAME NO NO)
            if ("${DEP_NAME}" EQUAL "")
                message(FATAL_ERROR "CPM dependency entry must hav a \"NAME\" or \"name\" field")
            endif()
            read_entry(${ITEM} FORCE "FORCE " DEP_FORCE NO NO)
            read_entry(${ITEM} VERSION "VERSION " DEP_VERSION YES NO)
            read_entry(${ITEM} VERSION "" DEP_RAW_VERSION NO NO)
            read_entry(${ITEM} GIT_TAG "GIT_TAG " DEP_GIT_TAG YES NO)
            read_entry(${ITEM} DOWNLOAD_ONLY "DOWNLOAD_ONLY " DEP_DOWNLOAD_ONLY NO NO)
            read_entry(${ITEM} PATCHES "PATCHES " DEP_PATCHES YES NO)
            read_entry(${ITEM} GITHUB_REPOSITORY "GITHUB_REPOSITORY " DEP_GITHUB_REPOSITORY NO NO)
            read_entry(${ITEM} BITBUCKET_REPOSITORY "BITBUCKET_REPOSITORY " DEP_BITBUCKET_REPOSITORY NO NO)
            read_entry(${ITEM} GITLAB_REPOSITORY "GITLAB_REPOSITORY " DEP_GITLAB_REPOSITORY NO NO)
            read_entry(${ITEM} GIT_REPOSITORY "GIT_REPOSITORY " DEP_GIT_REPOSITORY NO NO)
            read_entry(${ITEM} SOURCE_DIR "SOURCE_DIR " DEP_SOURCE_DIR YES NO)
            read_entry(${ITEM} FIND_PACKAGE_ARGUMENTS "FIND_PACKAGE_ARGUMENTS " DEP_FIND_PACKAGE_ARGUMENTS YES NO)
            read_entry(${ITEM} NO_CACHE "NO_CACHE " DEP_NO_CACHE NO NO)
            read_entry(${ITEM} SYSTEM "SYSTEM " DEP_SYSTEM NO NO)
            read_entry(${ITEM} GIT_SHALLOW "GIT_SHALLOW " DEP_GIT_SHALLOW NO NO)
            read_entry(${ITEM} EXCLUDE_FROM_ALL "EXCLUDE_FROM_ALL " DEP_EXCLUDE_FROM_ALL NO NO)
            read_entry(${ITEM} SOURCE_SUBDIR "SOURCE_SUBDIR " DEP_SOURCE_SUBDIR YES NO)
            read_entry(${ITEM} CUSTOM_CACHE_KEY "CUSTOM_CACHE_KEY " DEP_CUSTOM_CACHE_KEY YES NO)
            read_entry(${ITEM} URL "URL " DEP_URL YES NO)
            read_entry(${ITEM} OPTIONS "OPTIONS " DEP_OPTIONS YES NO)
            read_entry(${ITEM} DOWNLOAD_COMMAND "DOWNLOAD_COMMAND " DEP_DOWNLOAD_COMMAND YES NO)
            read_entry(${ITEM} CPMADDPACKAGE_COMMAND "" DEP_${DEP_NAME}_CPMADDPACKAGE_COMMAND NO NO)
            read_entry(${ITEM} CMAKE_COMMANDS "" DEP_${DEP_NAME}_CMAKE_COMMANDS NO YES)

            string(REPLACE "{VERSION}" "${DEP_RAW_VERSION}" DEP_GIT_TAG "${DEP_GIT_TAG}")

            if (NOT DEP_SYSTEM)
                set(DEP_SYSTEM "SYSTEM YES")
            endif()
            if (NOT DEP_EXCLUDE_FROM_ALL)
                set(DEP_EXCLUDE_FROM_ALL "EXCLUDE_FROM_ALL YES")
            endif()

            set(${DEP_NAME}_declare_command
                "${DEP_NAME}
                NAME ${DEP_NAME}
                ${DEP_FORCE}
                ${DEP_VERSION}
                ${DEP_GIT_TAG}
                ${DEP_DOWNLOAD_ONLY}
                ${DEP_PATCHES}
                ${DEP_GITHUB_REPOSITORY}
                ${DEP_BITBUCKET_REPOSITORY}
                ${DEP_GITLAB_REPOSITORY}
                ${DEP_GIT_REPOSITORY}
                ${DEP_SOURCE_DIR}
                ${DEP_FIND_PACKAGE_ARGUMENTS}
                ${DEP_NO_CACHE}
                ${DEP_SYSTEM}
                ${DEP_GIT_SHALLOW}
                ${DEP_EXCLUDE_FROM_ALL}
                ${DEP_SOURCE_SUBDIR}
                ${DEP_CUSTOM_CACHE_KEY}
                ${DEP_URL}
                ${DEP_OPTIONS}
                ${DEP_DOWNLOAD_COMMAND}"
                PARENT_SCOPE
            )
            set(DEP_${DEP_NAME}_CMAKE_COMMANDS "${DEP_${DEP_NAME}_CMAKE_COMMANDS}" PARENT_SCOPE)
            list(APPEND PackageNames ${DEP_NAME})
        else()
            message(FATAL_ERROR "Unexpected JSON type ${TYPE_VALUE} at position ${IDX}")
        endif()
    endforeach()
    set(PackageNames "${PackageNames}" PARENT_SCOPE)
endfunction()

function(parseDependencyFile file)
    # Add file to CMAKE_CONFIGURE_DEPENDS so changes trigger a rerun of cmake
    set_property(
        DIRECTORY
        APPEND
        PROPERTY CMAKE_CONFIGURE_DEPENDS
        "${file}"
    )
    file(READ ${file} FILE_AS_STRING)

    # Read format_version
    string(JSON FORMAT_VERSION ERROR_VARIABLE JSON_ERROR GET ${FILE_AS_STRING} "format_version")
    if (NOT ${JSON_ERROR} MATCHES "^NOTFOUND$")
        message(FATAL_ERROR "JSON Parsing Error: ${JSON_ERROR}")
    endif()

    if ("${FORMAT_VERSION}" MATCHES "1")
        # Read package section
        string(JSON ITEM ERROR_VARIABLE JSON_ERROR GET ${FILE_AS_STRING} "packages")
        if (${JSON_ERROR} MATCHES "^NOTFOUND$")
            read_packages_section(${ITEM})
            foreach (DEP_NAME ${PackageNames})
                set(${DEP_NAME}_declare_command "${${DEP_NAME}_declare_command}" PARENT_SCOPE)
                set(DEP_${DEP_NAME}_CMAKE_COMMANDS "${DEP_${DEP_NAME}_CMAKE_COMMANDS}" PARENT_SCOPE)
            endforeach()
            set(PackageNames "${PackageNames}" PARENT_SCOPE)
        endif()

        # Read other dependency files
        string(JSON ITEM ERROR_VARIABLE JSON_ERROR GET ${FILE_AS_STRING} "dependency_files")
        if (${JSON_ERROR} MATCHES "^NOTFOUND$")
            string(JSON number_of_objects LENGTH ${ITEM})
            MATH(EXPR number_of_objects "${number_of_objects}-1")
            foreach (IDX RANGE ${number_of_objects})
                string(JSON FILE GET ${ITEM} ${IDX})

                cmake_path(IS_RELATIVE FILE is_relative)
                if (${is_relative})
                    set(path ${file})
                    cmake_path(HAS_PARENT_PATH path has_parent)

                    cmake_path(GET path PARENT_PATH parent_path)
                    set(FILE "${parent_path}/${FILE}")
                endif()
                parseDependencyFile(${FILE})
                foreach (DEP_NAME ${PackageNames})
                    set(${DEP_NAME}_declare_command "${${DEP_NAME}_declare_command}" PARENT_SCOPE)
                    set(DEP_${DEP_NAME}_CMAKE_COMMANDS "${DEP_${DEP_NAME}_CMAKE_COMMANDS}" PARENT_SCOPE)
                endforeach()
                set(PackageNames "${PackageNames}" PARENT_SCOPE)
            endforeach()
        endif()
    else()
        message(FATAL_ERROR "unknown \"format_version\" ${FORMAT_VERSION}")
    endif()
endfunction()

macro(loadDependenciesFile file)
    set(PackageNames)
    parseDependencyFile(${file})
    message(VERBOSE "CPM.dependencies:")
    foreach (DEP_NAME ${PackageNames})
        message(VERBOSE "- ${DEP_NAME}")
        string(REGEX REPLACE "[\n\r \t]+" " " decl "${${DEP_NAME}_declare_command}")
        cmake_language(EVAL CODE CPMDeclarePackage(${decl}))
    endforeach()

    foreach (DEP_NAME ${PackageNames})
        if (NOT DEP_${DEP_NAME}_CPMADDPACKAGE_COMMAND)
            CPMGetPackage("${DEP_NAME}")
        else()
            cmake_language(CALL ${DEP_${DEP_NAME}_CPMADDPACKAGE_COMMAND} ${declaration})
        endif()
        if (${DEP_NAME}_ADDED)
            foreach (cmd ${DEP_${DEP_NAME}_CMAKE_COMMANDS})
                set(CPM_CURRENT_SOURCE_DIR "${${DEP_NAME}_SOURCE_DIR}")
                cmake_language(EVAL CODE "${cmd}")
            endforeach()
        endif()

    endforeach()
endmacro()

function(fancyAdd)
    set(options)
    set(oneValueArgs LIBRARY EXECUTABLE NAMESPACE HEADER_PATH_PREFIX CPP_VERSION TEST)
    set(multiValueArgs SOURCE_FILES HEADER_FILES DEPENDENCIES)
    cmake_parse_arguments(PARSE_ARGV 0 arg
        "${options}" "${oneValueArgs}" "${multiValueArgs}"
    )

    string(MAKE_C_IDENTIFIER "${arg_NAME}" "arg_SAFENAME")

    if (NOT "${arg_LIBRARY}" STREQUAL "")
        set(arg_NAME "${arg_LIBRARY}")
        if ("${arg_SOURCE_FILES}" STREQUAL "")
            set (TARGET_TYPE "INTERFACE")
            add_library(${arg_NAME} INTERFACE)
        else()
            set (TARGET_TYPE "PUBLIC")
            set (CPM_LINK_LIB_${arg_NAME} "STATIC" CACHE STRING "link library as STATIC or SHARED")
            add_library(${arg_NAME} ${CPM_LINK_LIB_${arg_NAME}}
                ${arg_SOURCE_FILES}
            )
        endif()
    elseif (NOT "${arg_EXECUTABLE}" STREQUAL "")
        set(arg_NAME "${arg_EXECUTABLE}")
        set(TARGET_TYPE "PUBLIC")
        add_executable(${arg_NAME}
            ${arg_SOURCE_FILES}
        )
        if (NOT "${arg_TEST}" STREQUAL "")
            add_test(NAME ${arg_NAME} COMMAND ${arg_NAME})
        endif()
    else()
        message(FATAL_ERROR "Unknown compilation type")
    endif()

    if (NOT "${arg_HEADER_PATH_PREFIX}" STREQUAL "")
        target_sources(${arg_NAME} ${TARGET_TYPE} FILE_SET ${arg_SAFENAME}_headers TYPE HEADERS
            BASE_DIRS ${arg_HEADER_PATH_PREFIX}
            FILES ${arg_HEADER_FILES}
        )
    endif()
    if (NOT "${arg_CPP_VERSION}" STREQUAL "")
        target_compile_features(${arg_NAME} ${TARGET_TYPE} ${arg_CPP_VERSION})
    endif()
    if (NOT "${arg_HEADER_PATH_PREFIX}" STREQUAL "")
        target_include_directories(${arg_NAME} ${TARGET_TYPE}
            $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/${arg_HEADER_PATH_PREFIX}>
            $<INSTALL_INTERFACE:${arg_HEADER_PATH_PREFIX}>
        )
    endif()
    if (NOT "${arg_DEPENDENCIES}" STREQUAL "")
        target_link_libraries(${arg_NAME} ${TARGET_TYPE}
            ${arg_DEPENDENCIES}
        )
    endif()
    if (NOT "${arg_NAMESPACE}" STREQUAL "")
        add_library(${arg_NAMESPACE}::${arg_NAME} ALIAS ${arg_NAME})
    endif()

    # Install library related stuff
    if (NOT "${arg_LIBRARY}" STREQUAL "")
        include(GNUInstallDirs)
        include(CMakePackageConfigHelpers)

        # Install headers
        install(TARGETS ${arg_NAME}
            EXPORT ${arg_NAME}
            FILE_SET ${arg_SAFENAME}_headers
            DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
        )

        # Install a generated cmake file
        install(EXPORT ${arg_NAME}
            FILE ${arg_NAME}.cmake
            DESTINATION "${CMAKE_INSTALL_DATADIR}/cmake/${arg_NAME}"
            NAMESPACE "${arg_NAMESPACE}::"
        )

        execute_process(
            COMMAND git describe --tags
            OUTPUT_VARIABLE arg_NAME_VERSION
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
            ERROR_VARIABLE VERSION_ERROR
            OUTPUT_STRIP_TRAILING_WHITESPACE
        )

        if(${arg_NAME_VERSION} STREQUAL "")
            message(FATAL_ERROR "cannot generate version string")
        endif()

        message("${arg_NAME} version: " ${arg_NAME_VERSION})
        set(VERSION ${arg_NAME_VERSION})

        # Install pkgconfig file
        configure_file(${CMAKE_CURRENT_LIST_DIR}/template_pkgfile.pc.in ${arg_NAME}.pc @ONLY)
        install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${arg_NAME}.pc
            DESTINATION ${CMAKE_INSTALL_DATADIR}/pkgconfig
        )

        # Install license files
        install(DIRECTORY LICENSES/
            DESTINATION ${CMAKE_INSTALL_DATADIR}/licenses/${PROJECT_NAME}
        )
    endif()
endfunction()

function(fancyAdd_StdLayout)
    set(options)
    set(oneValueArgs LIBRARY EXECUTABLE TEST CPP_VERSION)
    set(multiValueArgs DEPENDENCIES)
    cmake_parse_arguments(PARSE_ARGV 0 arg
        "${options}" "${oneValueArgs}" "${multiValueArgs}"
    )

    if (NOT "${arg_LIBRARY}" STREQUAL "")
        set(arg_NAME "${arg_LIBRARY}")
        set(TYPE "LIBRARY")
    elseif (NOT "${arg_EXECUTABLE}" STREQUAL "")
        set(arg_NAME "${arg_EXECUTABLE}")
        set(TYPE "EXECUTABLE")
    endif()

    if (NOT "${arg_TEST}" STREQUAL "")
        set(TEST "TEST YES")
    endif()

    string(REPLACE "::" ";" NAME_LIST ${arg_NAME})
    list(LENGTH NAME_LIST len)
    if (len EQUAL 1)
        set(NAMESPACE)
        list(GET NAME_LIST 0 arg_NAME)
    elseif (len EQUAL 2)
        list(GET NAME_LIST 0 NAMESPACE)
        set(NAMESPACE "NAMESPACE ${NAMESPACE}")
        list(GET NAME_LIST 1 arg_NAME)
    else()
        message(FATAL_ERROR "invalid name ${arg_NAME}")
    endif()


    file(GLOB_RECURSE SOURCE_FILES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "src/${arg_NAME}/*.cpp")
    file(GLOB_RECURSE HEADER_FILES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "src/${arg_NAME}/*.h")
    file(GLOB_RECURSE HEADER_FILES_INCL RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "include/${arg_NAME}/*.h")

    if (NOT "${SOURCE_FILES}" STREQUAL "")
        set(SOURCE_FILES "SOURCE_FILES ${SOURCE_FILES}")
    endif()

    set(HEADER_PATH_PREFIX src)
    if (NOT "${HEADER_FILES_INCL}" STREQUAL "")
        set(HEADER_FILES "${HEADER_FILES_INCL}")
        set(HEADER_PATH_PREFIX include)
    endif()
    if (NOT "${HEADER_FILES}" STREQUAL "")
        set(HEADER_FILES "HEADER_FILES ${HEADER_FILES}")
    endif()

    set(CPP_VERSION)
    if (NOT "${arg_CPP_VERSION}" STREQUAL "")
        set(CPP_VERSION "CPP_VERSION ${arg_CPP_VERSION}")
    endif()
    set(DEPENDENCIES)
    if (NOT "${arg_DEPENDENCIES}" STREQUAL "")
        set(DEPENDENCIES "DEPENDENCIES ${arg_DEPENDENCIES}")
    endif()

    cmake_language(EVAL CODE "fancyAdd(
        ${TYPE} ${arg_NAME} ${NAMESPACE}
        ${SOURCE_FILES}
        HEADER_PATH_PREFIX ${HEADER_PATH_PREFIX}
        ${HEADER_FILES}
        ${CPP_VERSION}
        ${DEPENDENCIES}
        ${TEST}
    )")
endfunction()

function(fancy_headerTest)
    set(options)
    set(oneValueArgs NAME)
    set(multiValueArgs)
    cmake_parse_arguments(PARSE_ARGV 0 arg
        "${options}" "${oneValueArgs}" "${multiValueArgs}"
    )

    string(REPLACE "::" ";" NAME_LIST ${arg_NAME})
    list(LENGTH NAME_LIST len)
    if (len EQUAL 1)
        set(NAMESPACE)
        list(GET NAME_LIST 0 arg_NAME)
    elseif (len EQUAL 2)
        list(GET NAME_LIST 0 NAMESPACE)
        set(NAMESPACE "NAMESPACE ${NAMESPACE}")
        list(GET NAME_LIST 1 arg_NAME)
    else()
        message(FATAL_ERROR "invalid name ${arg_NAME}")
    endif()


    string(MAKE_C_IDENTIFIER "${arg_NAME}" "arg_SAFENAME")

    set(library_path "src/${arg_NAME}")

    set(target_name "${arg_NAME}_test_header")

    file(GLOB_RECURSE header_files LIST_DIRECTORIES false RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}/${library_path} ${library_path}/*.h)

    set(files)
    foreach (header ${header_files})
        string (REPLACE "/" "__" test_name "${header}")
        set(cpp_file test_header_${target_name}_${test_name}.cpp)
        configure_file(${CMAKE_CURRENT_LIST_DIR}/test.cpp.in ${cpp_file})
        set(files ${files} ${cpp_file})
    endforeach()

    add_library(${target_name} ${files})
    target_include_directories(${target_name} PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/${library_path})
    if ("${arg_NAMESPACE}" STREQUAL "")
        target_link_libraries(${target_name} PUBLIC ${arg_NAME})
    else()
        target_link_libraries(${target_name} PUBLIC ${NAMESPACE}::${arg_NAME})
    endif()
endfunction()

function(read_translationset_section FILE_AS_STRING)
    # Iterate over it and declare all dependencies
    string(JSON number_of_objects LENGTH ${FILE_AS_STRING})
    MATH(EXPR number_of_objects "${number_of_objects}-1")
    foreach (IDX RANGE ${number_of_objects})
        string(JSON TYPE_VALUE TYPE ${FILE_AS_STRING} ${IDX})
        string(JSON ITEM GET ${FILE_AS_STRING} ${IDX})
        if (${TYPE_VALUE} MATCHES "^OBJECT$")
            #!TODO Many options missing
            read_entry(${ITEM} IF "" DEP_IF NO NO)
            if ("${DEP_IF}" MATCHES ".")
                set(TEST_IF "
                    if (${DEP_IF})
                        set(MUST_LOAD_THIS_PACKAGE ON)
                    else()
                        set(MUST_LOAD_THIS_PACKAGE OFF)
                    endif()"
                )
                cmake_language(EVAL CODE "${TEST_IF}")
                if (NOT ${MUST_LOAD_THIS_PACKAGE})
                    continue()
                endif()
            endif ()

            read_entry(${ITEM} NAME "" DEP_NAME NO NO)
            if ("${DEP_NAME}" EQUAL "")
                message(FATAL_ERROR "translation set must have a \"NAME\" or \"name\" field")
            endif()
            read_entry(${ITEM} TYPE "" DEP_TYPE NO NO)
            read_entry(${ITEM} LANGUAGE "" DEP_LANGUAGE NO NO)
            read_entry(${ITEM} DEPENDENCIES "" DEP_DEPENDENCIES NO YES)
            string(TOLOWER "${DEP_TYPE}" DEP_TYPE)
            if ("${DEP_TYPE}" STREQUAL "executable")
                set(TARGET "EXECUTABLE")
            elseif("${DEP_TYPE}" STREQUAL "library")
                set(TARGET "LIBRARY")
            else()
                message(FATAL_ERROR "unknown type ${DEP_TYPE}")
            endif()
            read_entry(${ITEM} TEST "TEST " DEP_TEST NO NO)
            read_entry(${ITEM} TEST_HEADER "" DEP_TEST_HEADER NO NO)
            if ("${DEP_TEST_HEADER}")
                cmake_language(EVAL CODE "
                    fancy_headerTest(
                        NAME ${DEP_NAME}
                    )
                ")
            endif()

            cmake_language(EVAL CODE "
                fancyAdd_StdLayout(
                    ${TARGET} ${DEP_NAME}
                    CPP_VERSION ${DEP_LANGUAGE}
                    DEPENDENCIES ${DEP_DEPENDENCIES}
                    ${DEP_TEST}
                )
            ")
        else()
            message(FATAL_ERROR "Unexpected JSON type ${TYPE_VALUE} at position ${IDX}")
        endif()
    endforeach()
endfunction()

function(read_options_section FILE_AS_STRING)
    # Iterate over it and declare all dependencies
    string(JSON number_of_objects LENGTH ${FILE_AS_STRING})
    MATH(EXPR number_of_objects "${number_of_objects}-1")
    foreach (IDX RANGE ${number_of_objects})
        string(JSON TYPE_VALUE TYPE ${FILE_AS_STRING} ${IDX})
        string(JSON ITEM GET ${FILE_AS_STRING} ${IDX})
        if (${TYPE_VALUE} MATCHES "^OBJECT$")
            #!TODO Many options missing
            read_entry(${ITEM} IF "" DEP_IF NO NO)
            if ("${DEP_IF}" MATCHES ".")
                set(TEST_IF "
                    if (${DEP_IF})
                        set(MUST_LOAD_THIS_PACKAGE ON)
                    else()
                        set(MUST_LOAD_THIS_PACKAGE OFF)
                    endif()"
                )
                cmake_language(EVAL CODE "${TEST_IF}")
                if (NOT ${MUST_LOAD_THIS_PACKAGE})
                    continue()
                endif()
            endif ()

            read_entry(${ITEM} NAME "" DEP_NAME NO NO)
            if ("${DEP_NAME}" EQUAL "")
                message(FATAL_ERROR "opiton must have a \"NAME\" or \"name\" field")
            endif()
            read_entry(${ITEM} DESCRIPTION "" DEP_DESCRIPTION YES NO)
            read_entry(${ITEM} DEFAULT "" DEP_DEFAULT NO NO)
            set(OPTION_DEFAULT_VALUE OFF)
            if ("${DEP_DEFAULT}" MATCHES ".")
                set(CODE "
                    if (${DEP_DEFAULT})
                        set(OPTION_DEFAULT_VALUE ON)
                    endif()"
                )
                cmake_language(EVAL CODE "${CODE}")
            endif()
            message("option(${DEP_NAME} ${DEP_DESCRIPTION} ${OPTION_DEFAULT_VALUE})")

            cmake_language(EVAL CODE "
                option(${DEP_NAME} ${DEP_DESCRIPTION} ${OPTION_DEFAULT_VALUE})
            ")
        else()
            message(FATAL_ERROR "Unexpected JSON type ${TYPE_VALUE} at position ${IDX}")
        endif()
    endforeach()
endfunction()


function(fancyAddFromFile file)
    file(READ ${file} FILE_AS_STRING)

    # Read format_version
    string(JSON FORMAT_VERSION ERROR_VARIABLE JSON_ERROR GET ${FILE_AS_STRING} "format_version")
    if (NOT ${JSON_ERROR} MATCHES "^NOTFOUND$")
        message(FATAL_ERROR "JSON Parsing Error: ${JSON_ERROR}")
    endif()
    if ("${FORMAT_VERSION}" MATCHES "1")
        # Read translation sect section
        string(JSON ITEM ERROR_VARIABLE JSON_ERROR GET ${FILE_AS_STRING} "translationsets")
        if (${JSON_ERROR} MATCHES "^NOTFOUND$")
            read_translationset_section(${ITEM})
        endif()
    endif()
endfunction()
function(optionsFromFile file)
    file(READ ${file} FILE_AS_STRING)
    # Read format_version
    string(JSON FORMAT_VERSION ERROR_VARIABLE JSON_ERROR GET ${FILE_AS_STRING} "format_version")
    if (NOT ${JSON_ERROR} MATCHES "^NOTFOUND$")
        message(FATAL_ERROR "JSON Parsing Error: ${JSON_ERROR}")
    endif()
    if ("${FORMAT_VERSION}" MATCHES "1")
        # Read translation sect section
        string(JSON ITEM ERROR_VARIABLE JSON_ERROR GET ${FILE_AS_STRING} "options")
        if (${JSON_ERROR} MATCHES "^NOTFOUND$")
            read_options_section(${ITEM})
        endif()
    endif()
endfunction()

macro(loadCPMPack file)
    message("Loading cpmpack: ${file}")
    optionsFromFile("${file}")
    loadDependenciesFile("${file}")
    fancyAddFromFile("${file}")

endmacro()
