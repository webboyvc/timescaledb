set(INSTALL_FILE ${PROJECT_NAME}--${PROJECT_VERSION_MOD}.sql)

# Source files that define the schemas and tables for our metadata
set(CATALOG_SOURCE_FILES
  schemas.sql # Must be first
  tables.sql
)

# The rest of the source files defining mostly functions
set(SOURCE_FILES
  chunk.sql
  ddl_internal.sql
  util_time.sql
  util_internal_table_ddl.sql
  chunk_constraint.sql
  partitioning.sql
  schema_info.sql
  ddl_api.sql
  ddl_triggers.sql
  bookend.sql
  time_bucket.sql
  version.sql
  size_utils.sql
  histogram.sql
  cache.sql
)

# These files should be pre-pended to update scripts so that they are
# executed before anything else during updates
set(PRE_UPDATE_FILES
  updates/pre-global.sql
)

# These files represent the modifications that happen in each version,
# excluding new objects or updates to functions. We use them to build
# a path (update script) from every historical version to the current
# version. Note that not all of these files may exist on disk, in case
# they would have no contents. There still needs to be an entry here
# to build an update script for that version. Thus, for every new
# release, an entry should be added here.
set(MOD_FILES
  updates/0.1.0--0.2.0.sql
  updates/0.2.0--0.3.0.sql
  updates/0.3.0--0.4.0.sql
  updates/0.4.0--0.4.1.sql
  updates/0.4.1--0.4.2.sql
  updates/0.4.2--0.5.0.sql
  updates/0.5.0--0.6.0.sql
  updates/0.6.0--0.6.1.sql
  updates/0.6.1--0.7.0.sql
  updates/0.7.0--0.7.1.sql
  updates/0.7.1--0.8.0.sql
  updates/0.8.0--0.9.0.sql
  updates/0.9.0--0.9.1.sql
  updates/0.9.1--0.9.2.sql
  updates/0.9.2--0.10.0-dev.sql
)

set(MODULE_PATHNAME "$libdir/timescaledb-${PROJECT_VERSION_MOD}")

# Function to replace @MODULE_PATHNAME@ in source files, producing an
# output file in the build dir
function(version_files SRC_FILE_LIST OUTPUT_FILE_LIST)
  set(result "")
  foreach(unversioned_file ${SRC_FILE_LIST})
    set(versioned_file ${unversioned_file})
    list(APPEND result ${CMAKE_CURRENT_BINARY_DIR}/${versioned_file})
    if (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${unversioned_file})
      configure_file(${unversioned_file} ${versioned_file} @ONLY)
    endif ()
  endforeach(unversioned_file)
  set(${OUTPUT_FILE_LIST} "${result}" PARENT_SCOPE)
endfunction()

# Create versioned files (replacing MODULE_PATHNAME) in the build
# directory of all our source files
version_files("${PRE_UPDATE_FILES}" PRE_UPDATE_FILES_VERSIONED)
version_files("${CATALOG_SOURCE_FILES}" CATALOG_SOURCE_FILES_VERSIONED)
version_files("${SOURCE_FILES}" SOURCE_FILES_VERSIONED)
version_files("${MOD_FILES}" MOD_FILES_VERSIONED)

# Function to concatenate all files in SRC_FILE_LIST into file OUTPUT_FILE
function(cat_files SRC_FILE_LIST OUTPUT_FILE)
  if (WIN32)
    # Make list of files into string of files separated by "+"
    # to make Windows copy concatenate them

    file(TO_NATIVE_PATH "${SRC_FILE_LIST}" SRC_FILE_LIST_NATIVE)
    string(REPLACE ";" ";+" SQL_LIST_JOINED "${SRC_FILE_LIST_NATIVE}")

    file(TO_NATIVE_PATH "${OUTPUT_FILE}" OUTPUT_FILE_NATIVE)

    set(CAT_CMD copy /B /y ${SQL_LIST_JOINED} "\"${OUTPUT_FILE_NATIVE}\"" >NUL)
  else ()
    set(CAT_CMD cat ${SRC_FILE_LIST} > ${OUTPUT_FILE})
  endif ()
  add_custom_command(
    OUTPUT ${OUTPUT_FILE}
    DEPENDS ${SRC_FILE_LIST}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    COMMAND ${CAT_CMD}
    COMMENT "Generating ${OUTPUT_FILE}"
  )
endfunction()

# Generate the extension file used with CREATE EXTENSION
cat_files("${CATALOG_SOURCE_FILES_VERSIONED};${SOURCE_FILES_VERSIONED}" ${CMAKE_CURRENT_BINARY_DIR}/${INSTALL_FILE})
add_custom_target(sqlfile ALL DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${INSTALL_FILE})

# Generate the update files used with ALTER EXTENSION <name> UPDATE
set(MOD_FILE_REGEX "([0-9]+\\.[0-9]+\\.*[0-9]*)--([0-9]+\\.[0-9]+\\.*[0-9]*[-dev]*).sql")

# We'd like to process the updates in reverse (descending) order
set(MOD_FILES_LIST ${MOD_FILES_VERSIONED})
list(REVERSE MOD_FILES_LIST)

# Variable that will hold the list of update scripts from every
# previous version to the current version
set(UPDATE_SCRIPTS "")

# A list of current modfiles. We append to this list for every
# previous version that moves us further away from the current
# version, thus making the update path longer as we move back in
# history
set(CURR_MOD_FILES "")

# Now loop through the modfiles and generate the update files
foreach(transition_mod_file ${MOD_FILES_LIST})

  if (NOT (${transition_mod_file} MATCHES ${MOD_FILE_REGEX}))
    message(FATAL_ERROR "Cannot parse update file name ${mod_file}")
  endif ()

  set(START_VERSION ${CMAKE_MATCH_1})
  set(END_VERSION ${CMAKE_MATCH_2})
  set(PRE_FILES ${PRE_UPDATE_FILES_VERSIONED})

  # Check for version-specific update code with fixes
  if (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/updates/${START_VERSION}.sql)
    version_files("updates/${START_VERSION}.sql" ORIGIN_MOD_FILE)
    list(APPEND PRE_FILES ${ORIGIN_MOD_FILE})
  endif ()

  # There might not have been any changes in the modfile, in which
  # case the modfile need not be present
  if (EXISTS ${transition_mod_file})
    # Prepend the modfile as we are moving through the versions in
    # descending order
    list(INSERT CURR_MOD_FILES 0 ${transition_mod_file})
  endif ()

  set(UPDATE_SCRIPT ${CMAKE_CURRENT_BINARY_DIR}/timescaledb--${START_VERSION}--${PROJECT_VERSION_MOD}.sql)
  list(APPEND UPDATE_SCRIPTS ${UPDATE_SCRIPT})
  if (CURR_MOD_FILES)
    cat_files("${PRE_FILES};${CURR_MOD_FILES};${SOURCE_FILES_VERSIONED}" ${UPDATE_SCRIPT})
  else ()
    cat_files("${PRE_FILES};${SOURCE_FILES_VERSIONED}" ${UPDATE_SCRIPT})
  endif ()
endforeach(transition_mod_file)

add_custom_target(sqlupdatescripts ALL DEPENDS ${UPDATE_SCRIPTS})

# Install target for the extension file and update scripts
install(
  FILES ${CMAKE_CURRENT_BINARY_DIR}/${INSTALL_FILE}
  ${UPDATE_SCRIPTS}
  DESTINATION "${PG_SHAREDIR}/extension")
