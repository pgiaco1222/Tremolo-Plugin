cmake_minimum_required(VERSION 3.25 FATAL_ERROR)
set(CMAKE_POLICY_DEFAULT_CMP0141        NEW      CACHE STRING "" FORCE)
set(CMAKE_OSX_DEPLOYMENT_TARGET "10.11" CACHE STRING "" FORCE)
set(CMAKE_OSX_ARCHITECTURES "x86_64;arm64" CACHE STRING "" FORCE)
project(JucePluginDevelopmentCourseTremoloPlugin)
if(PROJECT_IS_TOP_LEVEL)
  if (NOT CMAKE_CXX_STANDARD)
    set(CMAKE_CXX_STANDARD 23)
  endif()
  set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
  set(CMAKE_MSVC_DEBUG_INFORMATION_FORMAT "$<$<CONFIG:Debug,RelWithDebInfo>:Embedded>" CACHE STRING "" FORCE)
endif()
set(LIB_DIR ${CMAKE_CURRENT_SOURCE_DIR}/libs)
include(cmake/cpm.cmake)
cpmaddpackage(
  NAME
    JUCE
  GIT_TAG
    8.0.12
  GITHUB_REPOSITORY
    juce-framework/JUCE
  SOURCE_DIR
    ${LIB_DIR}/juce
)
include(cmake/CompilerWarnings.cmake)

if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
  set_directory_properties(PROPERTIES JUCE_VST3_COPY_DIR "$ENV{LOCALAPPDATA}/Programs/Common/VST3")
endif()
set(TREMOLO_PLUGIN_NAME "Tremolo")
juce_add_plugin(
  TremoloCoursePlugin
  PRODUCT_NAME
    ${TREMOLO_PLUGIN_NAME}
  VERSION
    1.0.0
  COMPANY_NAME
    "Paul G Sounds"
  PLUGIN_MANUFACTURER_CODE
    Pjgs
  PLUGIN_CODE
    Trcp
  FORMATS
    Standalone
    AU
    VST3
  IS_SYNTH
    FALSE
  NEEDS_MIDI_INPUT
    FALSE
  NEEDS_MIDI_OUTPUT
    FALSE
  VST3_AUTO_MANIFEST
    FALSE
  COPY_PLUGIN_AFTER_BUILD
    FALSE
)
target_compile_definitions(TremoloCoursePlugin
  PRIVATE
    JUCE_WEB_BROWSER=0
    JUCE_USE_CURL=0
    JUCE_VST3_CAN_REPLACE_VST2=0
)

juce_add_binary_data(tremolo_plugin_assets
  HEADER_NAME
    "TremoloPluginAssets.h"
  NAMESPACE
    tremolo::assets
  SOURCES
    assets/Background.png
    assets/Logo.png
    assets/Inter-Bold.ttf
    assets/Inter-Medium.ttf
)

add_library(tremolo::tremolo_plugin_assets ALIAS tremolo_plugin_assets)

juce_add_module(tremolo_plugin ALIAS_NAMESPACE tremolo)
target_compile_definitions(tremolo_plugin
  INTERFACE
    TREMOLO_PLUGIN_NAME="${TREMOLO_PLUGIN_NAME}"
)
target_compile_options(tremolo_plugin INTERFACE "${PROJECT_WARNINGS_CXX}")

target_link_libraries(tremolo_plugin INTERFACE tremolo::tremolo_plugin_assets)

target_link_libraries(
  TremoloCoursePlugin
  PRIVATE
    tremolo::tremolo_plugin
  PUBLIC
    juce::juce_recommended_config_flags
    juce::juce_recommended_lto_flags
    juce::juce_recommended_warning_flags
)
get_target_property(VST3_BUNDLE_PATH TremoloCoursePlugin_VST3 JUCE_PLUGIN_ARTEFACT_FILE)
cmake_path(GET VST3_BUNDLE_PATH PARENT_PATH VST3_BUNDLE_OUTPUT_DIR)
add_custom_command(
  TARGET TremoloCoursePlugin_VST3
  POST_BUILD
  WORKING_DIRECTORY ${VST3_BUNDLE_OUTPUT_DIR}
  COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_CURRENT_SOURCE_DIR}/assets/README.txt" ${VST3_BUNDLE_OUTPUT_DIR}
  COMMAND ${CMAKE_COMMAND} -E tar cvf "${VST3_BUNDLE_OUTPUT_DIR}/Tremolo.zip" --format=zip -- "${VST3_BUNDLE_PATH}" README.txt
)
source_group(TREE ${CMAKE_CURRENT_SOURCE_DIR}/..)

option(BUILD_EXAMPLES "When on, the supplied example apps will be added as targets and built" OFF)
if(BUILD_EXAMPLES)
  add_subdirectory(examples)
endif()

option(BUILD_TESTS "When on, will download the GoogleTest framework and add the automated tests project" OFF)
if(BUILD_TESTS)
  enable_testing()
  add_subdirectory(test)
endif()
