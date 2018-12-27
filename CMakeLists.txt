cmake_minimum_required(VERSION 3.3)

project(cairo-dpf-example LANGUAGES C CXX)
include(FindPkgConfig)

###########
# Options #
###########
set(DEPS_STATIC_LINK_DEFAULT OFF)
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
  set(DEPS_STATIC_LINK_DEFAULT ON)
endif()
if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
endif()
option(DEPS_STATIC_LINK "Whether to link dependencies statically" "${DEPS_STATIC_LINK_DEFAULT}")

############
# Find DPF #
############
set(DPF_PATH "${PROJECT_SOURCE_DIR}/../DPF" CACHE STRING "DPF path")
if(NOT EXISTS "${DPF_PATH}/distrho")
  message(FATAL_ERROR "DPF was not found under the directory DPF_PATH: ${DPF_PATH}")
endif()

#############
# Build DPF #
#############
add_library(dpf STATIC EXCLUDE_FROM_ALL
  "${DPF_PATH}/dgl/src/Application.cpp"
  "${DPF_PATH}/dgl/src/Color.cpp"
  "${DPF_PATH}/dgl/src/Geometry.cpp"
  # "${DPF_PATH}/dgl/src/Image.cpp"
  # "${DPF_PATH}/dgl/src/ImageWidgets.cpp"
  # "${DPF_PATH}/dgl/src/NanoVG.cpp"
  "${DPF_PATH}/dgl/src/Resources.cpp"
  "${DPF_PATH}/dgl/src/Widget.cpp"
  "${DPF_PATH}/dgl/src/Window.cpp")
target_compile_definitions(dpf PUBLIC "HAVE_DCAIRO=1")
target_include_directories(dpf
  PUBLIC "${DPF_PATH}/distrho" "${DPF_PATH}/dgl" "${DPF_PATH}/dgl/src")
if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  target_compile_options(dpf PRIVATE $<$<COMPILE_LANGUAGE:CXX>:-x objective-c++>)
endif()
pkg_check_modules(CAIRO "cairo" REQUIRED)
target_include_directories(dpf PUBLIC ${CAIRO_INCLUDE_DIRS})
if(NOT DEPS_STATIC_LINK)
  target_link_libraries(dpf PUBLIC ${CAIRO_LIBRARIES})
else()
  if(NOT DEFINED CAIRO_STATIC_LIBRARIES)
    execute_process(
      COMMAND "${PKG_CONFIG_EXECUTABLE}" "--libs" "--static" "cairo"
      OUTPUT_VARIABLE CAIRO_STATIC_LIBRARIES)
    set(CAIRO_STATIC_LIBRARIES "${CAIRO_STATIC_LIBRARIES}" CACHE STRING "cairo static libraries")
  endif()
  target_link_libraries(dpf PUBLIC ${CAIRO_STATIC_LIBRARIES})
endif()
set_target_properties(dpf PROPERTIES POSITION_INDEPENDENT_CODE ON)

########################
# Dependencies for DPF #
########################
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  find_library(COCOA_FRAMEWORK "Cocoa")
  target_link_libraries(dpf PUBLIC "${COCOA_FRAMEWORK}")
else()
  find_package(X11 REQUIRED)
  target_include_directories(dpf PUBLIC "${X11_INCLUDE_DIR}")
  target_link_libraries(dpf PUBLIC "${X11_X11_LIB}")
endif()

###########
# Helpers #
###########
macro(add_plugin ID)
  add_library("${ID}" MODULE ${ARGN})
  if(DEPS_STATIC_LINK)
    set_property(TARGET "${ID}" APPEND_STRING PROPERTY LINK_FLAGS " -static")
  endif()
  set_target_properties("${ID}" PROPERTIES
    PREFIX "" OUTPUT_NAME "${PROJECT_NAME}" LIBRARY_OUTPUT_DIRECTORY "${ID}")
endmacro()

########################
# Build common sources #
########################
add_library(coreplugin STATIC "sources/PluginMain.cc")
add_library(coreui STATIC
  "sources/PluginUI.cc"
  "sources/DemoWidgetClickable.cc"
  "sources/DemoWidgetBanner.cc")
foreach(target coreplugin;coreui)
  target_include_directories("${target}" PUBLIC "sources")
  target_link_libraries("${target}" PUBLIC dpf)
  set_target_properties("${target}" PROPERTIES POSITION_INDEPENDENT_CODE ON)
endforeach()

#############
# Build VST #
#############
add_plugin(VST
  "${DPF_PATH}/distrho/DistrhoPluginMain.cpp"
  "${DPF_PATH}/distrho/DistrhoUIMain.cpp")
target_compile_definitions(VST PRIVATE "DISTRHO_PLUGIN_TARGET_VST=1")
target_link_libraries(VST PRIVATE coreplugin coreui)

#############
# Build LV2 #
#############
add_plugin(LV2 "${DPF_PATH}/distrho/DistrhoPluginMain.cpp")
add_plugin(LV2ui "${DPF_PATH}/distrho/DistrhoUIMain.cpp")
foreach(target LV2;LV2ui)
  target_compile_definitions("${target}" PRIVATE "DISTRHO_PLUGIN_TARGET_LV2=1")
  set_target_properties("${target}" PROPERTIES LIBRARY_OUTPUT_DIRECTORY "LV2/${PROJECT_NAME}.lv2")
endforeach()
target_link_libraries(LV2 PRIVATE coreplugin)
target_link_libraries(LV2ui PRIVATE coreui)
set_target_properties(LV2ui PROPERTIES OUTPUT_NAME "${PROJECT_NAME}_ui")
if(NOT CMAKE_CROSSCOMPILING)
  find_program(LV2_TTL_GENERATOR "lv2_ttl_generator")
  if(NOT LV2_TTL_GENERATOR)
    message(WARNING "cannot find lv2_ttl_generator")
  else()
    add_custom_command(TARGET LV2 POST_BUILD
      COMMAND "${LV2_TTL_GENERATOR}" "./$<TARGET_FILE_NAME:LV2>"
      WORKING_DIRECTORY "$<TARGET_FILE_DIR:LV2>")
  endif()
endif()

##############
# Build Jack #
##############
pkg_check_modules(JACK "jack")
if(JACK_FOUND)
  add_executable(JACK
    "${DPF_PATH}/distrho/DistrhoPluginMain.cpp"
    "${DPF_PATH}/distrho/DistrhoUIMain.cpp")
  target_compile_definitions(JACK PRIVATE "DISTRHO_PLUGIN_TARGET_JACK=1")
  target_include_directories(JACK PRIVATE ${JACK_INCLUDE_DIRS})
  target_link_libraries(JACK PRIVATE coreplugin coreui ${JACK_LIBRARIES})
  set_target_properties(JACK PROPERTIES
    OUTPUT_NAME "${PROJECT_NAME}" RUNTIME_OUTPUT_DIRECTORY "JACK")
endif()
