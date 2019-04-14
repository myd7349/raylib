# Setup the project and settings
project(examples)

# Get the sources together
set(example_dirs models)
set(CMAKE_REQUIRED_DEFINITIONS -D_POSIX_C_SOURCE=199309L)
  include(CheckSymbolExists)
  check_symbol_exists(CLOCK_MONOTONIC time.h HAVE_CLOCK_MONOTONIC)
  check_symbol_exists(QueryPerformanceCounter windows.h HAVE_QPC)
set(CMAKE_REQUIRED_DEFINITIONS)
if(HAVE_QPC OR HAVE_CLOCK_MONOTONIC)
  set(example_dirs ${example_dirs} physac)
endif()

set(example_resources)
foreach(example_dir ${example_dirs})
  # Get the .c files
  file(GLOB sources ${example_dir}/*.c)
  list(APPEND example_sources ${sources})

  # Any any resources
  file(GLOB resources ${example_dir}/resources/*)
  list(APPEND example_resources ${resources})
endforeach()

include(CheckIncludeFiles)
set(OUTPUT_EXT)
list(REMOVE_ITEM example_sources ${CMAKE_CURRENT_SOURCE_DIR}/others/rlgl_standalone.c)

include_directories(BEFORE SYSTEM others/external/include)

if (NOT TARGET raylib)
  find_package(raylib 2.0 REQUIRED)
endif()

# Do each example
foreach(example_source models/models_rlgl_solar_system.c)
  # Create the basename for the example
  get_filename_component(example_name ${example_source} NAME)
  string(REPLACE ".c" "${OUTPUT_EXT}" example_name ${example_name})

  # Setup the example
  add_executable(${example_name} ${example_source})

  target_link_libraries(${example_name} raylib)

  string(REGEX MATCH ".*/.*/" resources_dir ${example_source})
  string(APPEND resources_dir "resources")

  if(${PLATFORM} MATCHES "Web" AND EXISTS ${resources_dir})
    # The local resources path needs to be mapped to /resources virtual path
    string(APPEND resources_dir "@resources")
    set_target_properties(${example_name} PROPERTIES LINK_FLAGS "--preload-file ${resources_dir}")
  endif()
endforeach()

if (${PLATFORM} MATCHES "Desktop")
  # rlgl_standalone can't be linked with raylib because of duplicate rlgl symbols
  foreach (example_source "others/rlgl_standalone.c")
    # Create the basename for the example
    get_filename_component(example_name ${example_source} NAME)
    string(REPLACE ".c" "${OUTPUT_EXT}" example_name ${example_name})
    add_executable(${example_name} ${example_source})
    add_dependencies(${example_name} raylib)
    target_link_libraries(${example_name} ${raylib_LDFLAGS})
    target_include_directories(${example_name} PRIVATE ${raylib_INCLUDE_DIRS})

  endforeach()
endif()

# Copy all of the resource files to the destination
file(COPY ${example_resources} DESTINATION "resources/")

execute_process(
    COMMAND ${CMAKE_C_COMPILER} -E ${CMAKE_CURRENT_SOURCE_DIR}/models/models_rlgl_solar_system.c -I ${CMAKE_CURRENT_SOURCE_DIR}/others/external/include -I ${CMAKE_CURRENT_LIST_DIR}/../src
)
