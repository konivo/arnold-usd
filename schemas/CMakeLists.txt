set(SCHEMAS_OUT "${CMAKE_CURRENT_BINARY_DIR}/${ARNOLD_VERSION}/${USD_VERSION}")
set(SCHEMAS_SOURCE_OUT "${SCHEMAS_OUT}/source")

set(SCHEMAS_TOKENS "${SCHEMAS_SOURCE_OUT}/tokens.cpp")
set(SCHEMAS_PLUGINFO "${SCHEMAS_SOURCE_OUT}/plugInfo.json")
set(SCHEMAS_WRAP_MODULE "${SCHEMAS_OUT}/wrapModule.cpp")
set(SCHEMAS_USDA "${SCHEMAS_OUT}/schema.usda")
set(CREATE_SCHEMA_PY "${CMAKE_CURRENT_SOURCE_DIR}/createSchemaFile.py")

set(_PYTHONPATHS "${ARNOLD_PYTHON_DIR}")
set(_LIBRARY_PATHS "${ARNOLD_BINARY_DIR}")
list(APPEND _PYTHONPATHS "$ENV{PYTHONPATH}")
if (USD_LIBRARY_DIR)
    if (EXISTS ${USD_LIBRARY_DIR}/python)
        list(APPEND _PYTHONPATHS "${USD_LIBRARY_DIR}/python")
    endif ()
    list(APPEND _LIBRARY_PATHS "${USD_LIBRARY_DIR}")
endif ()
if (USD_BINARY_DIR)
    list(APPEND _LIBRARY_PATHS "${USD_BINARY_DIR}")
endif ()

if (WIN32)
    list(APPEND _LIBRARY_PATHS "$ENV{PATH}")
    string(JOIN "\;" LIBRARY_PATHS ${_LIBRARY_PATHS})
    string(JOIN "\;" PYTHONPATHS ${_PYTHONPATHS})
else ()
    list(APPEND _LIBRARY_PATHS "$ENV{LD_LIBRARY_PATH}:$ENV{PATH}")
    string(JOIN ":" LIBRARY_PATHS ${_LIBRARY_PATHS})
    string(JOIN ":" PYTHONPATHS ${_PYTHONPATHS})
endif ()

if (NOT USD_GENSCHEMA)
    message(WARNING "usdGenSchema was not found, the schemas won't be generated with it")
endif()

# Custom command for generating an arnold schema from the arnold api
add_custom_command(OUTPUT ${SCHEMAS_USDA}
    COMMAND
    ${CMAKE_COMMAND} -E env
    "PYTHONPATH=${PYTHONPATHS}"
    "LD_LIBRARY_PATH=${LIBRARY_PATHS}"
    "PATH=${LIBRARY_PATHS}"
    ${PYTHON_EXECUTABLE} "${CREATE_SCHEMA_PY}" "${SCHEMAS_OUT}"
    DEPENDS "${CREATE_SCHEMA_PY}"
)

# Custom target for generating an arnold schema from the arnold api
add_custom_target(schemaUsda
    DEPENDS ${SCHEMAS_USDA}
)

add_custom_target(schemaSourceOut
    BYPRODUCTS "${SCHEMAS_SOURCE_OUT}"
    COMMAND ${CMAKE_COMMAND} -E make_directory "${SCHEMAS_SOURCE_OUT}"
)

if (BUILD_USDGENSCHEMA_ARNOLD)
    # Running our custom usdGenSchemaArnold on the schema file
    add_custom_command(OUTPUT "${SCHEMAS_SOURCE_OUT}/generatedSchema.usda"
        COMMAND
        ${CMAKE_COMMAND} -E env
        "$<TARGET_FILE:usdGenSchemaArnold>" "${SCHEMAS_USDA}" "${SCHEMAS_SOURCE_OUT}"
        WORKING_DIRECTORY ${SCHEMAS_SOURCE_OUT}
        DEPENDS schemaSourceOut schemaUsda usdGenSchemaArnold
    )
else()
    # Running usdGenSchema on the schema file
    add_custom_command(OUTPUT "${SCHEMAS_SOURCE_OUT}/generatedSchema.usda"
        COMMAND
        ${CMAKE_COMMAND} -E env
        "PYTHONPATH=${PYTHONPATHS}"
        "LD_LIBRARY_PATH=${LIBRARY_PATHS}"
        "PATH=${LIBRARY_PATHS}"
        "${PYTHON_EXECUTABLE}" "${USD_GENSCHEMA}" "${SCHEMAS_USDA}"
        WORKING_DIRECTORY ${SCHEMAS_SOURCE_OUT}
        DEPENDS schemaSourceOut schemaUsda 
    )
endif()

add_custom_target(generatedSchema
    DEPENDS "${SCHEMAS_SOURCE_OUT}/generatedSchema.usda"
)

# Fixing the plugInfo.json.in file in post.
add_custom_target(updatedPlugInfo
    BYPRODUCTS "${SCHEMAS_PLUGINFO}"
    # UsdGenSchema doesn't setup the library name properly in the file and it's not compatible with
    # configure_file (in older USD versions), so we are manually editing the file using a simple python command.
    COMMAND
    ${PYTHON_EXECUTABLE} "${CMAKE_CURRENT_SOURCE_DIR}/updatePlugInfo.py" "${SCHEMAS_PLUGINFO}"
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/tools"
    DEPENDS generatedSchema
)

# The final target is named schemas, it has the plugInfo updated and the generated schemas
# It is always built if this file was included.
add_custom_target(schemas ALL
    DEPENDS generatedSchema updatedPlugInfo
)

install(FILES "${SCHEMAS_SOURCE_OUT}/generatedSchema.usda"
    DESTINATION "${PREFIX_SCHEMA}/usdArnold/resources")

install(FILES "${SCHEMAS_SOURCE_OUT}/generatedSchema.usda"
    DESTINATION "${PREFIX_SCHEMA}/usdArnold/resources/usdArnold")

install(FILES "${SCHEMAS_USDA}"
    DESTINATION "${PREFIX_SCHEMA}/usdArnold/resources/usdArnold")

install(FILES "${SCHEMAS_PLUGINFO}"
    DESTINATION "${PREFIX_SCHEMA}/usdArnold/resources")

if (ENABLE_HYDRA_IN_USD_PROCEDURAL and BUILD_PROCEDURAL and BUILD_WITH_USD_STATIC)
    # TODO we should also copy these files to the usd proc build folder
    install(FILES "${SCHEMAS_PLUGINFO}"
        DESTINATION "${PREFIX_PROCEDURAL}/usd/usdArnold/resources")
    install(FILES "${SCHEMAS_SOURCE_OUT}/generatedSchema.usda"
        DESTINATION "${PREFIX_PROCEDURAL}/usd/usdArnold/resources")
