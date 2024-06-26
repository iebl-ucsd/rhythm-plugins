cmake_minimum_required(VERSION 3.5.0)
if (NOT DEFINED GUI_BASE_DIR)
	if (DEFINED ENV{GUI_BASE_DIR})
		set(GUI_BASE_DIR $ENV{GUI_BASE_DIR})
	else()
		set(GUI_BASE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../plugin-GUI)
	endif()
endif()

get_filename_component(PROJECT_FOLDER ${CMAKE_CURRENT_SOURCE_DIR} ABSOLUTE)
get_filename_component(PLUGIN_NAME ${PROJECT_FOLDER} NAME)

set(CMAKE_OSX_ARCHITECTURES "x86_64" CACHE STRING "Build architecture for Mac OS X" FORCE)

project(OE_PLUGIN_${PLUGIN_NAME})
set(CMAKE_SHARED_LIBRARY_PREFIX "")

set_property(DIRECTORY APPEND PROPERTY COMPILE_DEFINITIONS
	OEPLUGIN
	"$<$<PLATFORM_ID:Windows>:JUCE_API=__declspec(dllimport)>"
	$<$<PLATFORM_ID:Windows>:_CRT_SECURE_NO_WARNINGS>
	$<$<PLATFORM_ID:Linux>:JUCE_DISABLE_NATIVE_FILECHOOSERS=1>
	$<$<CONFIG:Debug>:DEBUG=1>
	$<$<CONFIG:Debug>:_DEBUG=1>
	$<$<CONFIG:Release>:NDEBUG=1>
	)


set(SOURCE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/Source)

if(BUILD_TESTS)
set(TESTS_PATH ${CMAKE_CURRENT_SOURCE_DIR}/Tests)
endif()

file(GLOB_RECURSE SRC_FILES LIST_DIRECTORIES false "${SOURCE_PATH}/*.cpp" "${SOURCE_PATH}/*.h")

if(BUILD_TESTS)
file(GLOB_RECURSE TESTS_FILES LIST_DIRECTORIES false "${TESTS_PATH}/*.cpp")
endif()

set(GUI_COMMONLIB_DIR ${GUI_BASE_DIR}/installed_libs)

set(CMAKE_CXX_STANDARD 20)

set(CONFIGURATION_FOLDER $<$<CONFIG:Debug>:Debug>$<$<NOT:$<CONFIG:Debug>>:Release>)

list(APPEND CMAKE_PREFIX_PATH ${GUI_COMMONLIB_DIR} ${GUI_COMMONLIB_DIR}/${CONFIGURATION_FOLDER})

if (APPLE)
	add_library(${PLUGIN_NAME} MODULE ${SRC_FILES})
else()
	add_library(${PLUGIN_NAME} SHARED ${SRC_FILES})
endif()

add_dependencies(${PLUGIN_NAME} open-ephys)

if(BUILD_TESTS)
add_library(${PLUGIN_NAME}_testable SHARED ${SRC_FILES})
endif()

target_compile_features(${PLUGIN_NAME} PUBLIC cxx_auto_type cxx_generalized_initializers)
target_include_directories(${PLUGIN_NAME} PUBLIC ${GUI_BASE_DIR}/JuceLibraryCode ${GUI_BASE_DIR}/JuceLibraryCode/modules ${GUI_BASE_DIR}/Plugins/Headers ${GUI_COMMONLIB_DIR}/include)

if(BUILD_TESTS)
target_compile_features(${PLUGIN_NAME}_testable PUBLIC cxx_auto_type cxx_generalized_initializers)
target_include_directories(${PLUGIN_NAME}_testable PUBLIC ${GUI_BASE_DIR}/JuceLibraryCode ${GUI_BASE_DIR}/JuceLibraryCode/modules ${GUI_BASE_DIR}/Plugins/Headers ${GUI_COMMONLIB_DIR}/include)


add_executable(
		${PLUGIN_NAME}_tests
		${TESTS_FILES}
)
target_compile_features(${PLUGIN_NAME}_tests PRIVATE cxx_std_17)
endif()


set(GUI_BIN_DIR ${CMAKE_BINARY_DIR}/plugin-GUI/${CONFIGURATION_FOLDER})

if(BUILD_TESTS)
set(GUI_TEST_BIN_COMMON_DIR ${GUI_BASE_DIR}/Build/TestBin/common)
set(GUI_TEST_HELPERS_DIR ${GUI_BASE_DIR}/TestHelpers)
endif()

if (NOT CMAKE_LIBRARY_ARCHITECTURE)
	if (CMAKE_SIZEOF_VOID_P EQUAL 8)
		set(CMAKE_LIBRARY_ARCHITECTURE "x64")
	else()
		set(CMAKE_LIBRARY_ARCHITECTURE "x86")
	endif()
endif()

#Libraries and compiler options
if(MSVC)
	target_link_libraries(${PLUGIN_NAME} ${GUI_BIN_DIR}/open-ephys.lib)
	target_compile_options(${PLUGIN_NAME} PRIVATE /sdl- /W0)
	
	if(BUILD_TESTS)
	target_compile_options(${PLUGIN_NAME}_testable PRIVATE /sdl- /W0)
	endif()

	set(INSTALL_PATH  ${GUI_BIN_DIR}/plugins)	
	install(TARGETS ${PLUGIN_NAME} RUNTIME DESTINATION ${GUI_BIN_DIR}/plugins  CONFIGURATIONS ${CMAKE_CONFIGURATION_TYPES})
elseif(LINUX)
	target_link_libraries(${PLUGIN_NAME} GL X11 Xext Xinerama asound dl freetype pthread rt)
	set_property(TARGET ${PLUGIN_NAME} APPEND_STRING PROPERTY LINK_FLAGS
		"-fvisibility=hidden -fPIC -rdynamic -Wl,-rpath='$ORIGIN/../shared' -Wl,-rpath='$ORIGIN/../shared-api8'")
	target_compile_options(${PLUGIN_NAME} PRIVATE -fPIC -rdynamic)
	target_compile_options(${PLUGIN_NAME} PRIVATE -O3) #enable optimization for linux debug
	
	set(INSTALL_PATH  ${GUI_BIN_DIR}/plugins)	
	install(TARGETS ${PLUGIN_NAME} LIBRARY DESTINATION ${GUI_BIN_DIR}/plugins)
elseif(APPLE)
	set_target_properties(${PLUGIN_NAME} PROPERTIES BUNDLE TRUE)
	set_property(TARGET ${PLUGIN_NAME} APPEND_STRING PROPERTY LINK_FLAGS
	"-undefined dynamic_lookup -rpath @loader_path/../../../../shared-api8")
	
	if(BUILD_TESTS)
	set_property(TARGET ${PLUGIN_NAME}_testable APPEND_STRING PROPERTY LINK_FLAGS
	"-undefined dynamic_lookup -rpath @loader_path/../../../../shared-api8")
	endif()

	set(INSTALL_PATH  $ENV{HOME}/Library/Application\ Support/open-ephys/plugins-api8)
	
	add_custom_command(TARGET ${PLUGIN_NAME} POST_BUILD COMMAND
    install_name_tool -change "@loader_path/../Frameworks/libokFrontPanel.dylib"
	"@rpath/libokFrontPanel.dylib" $<TARGET_FILE:${PLUGIN_NAME}>)

	install(TARGETS ${PLUGIN_NAME} DESTINATION $ENV{HOME}/Library/Application\ Support/open-ephys/plugins-api8)
endif()


if(APPLE)
       add_custom_command(TARGET ${PLUGIN_NAME} POST_BUILD COMMAND ${CMAKE_COMMAND} -E make_directory ${INSTALL_PATH}/$<TARGET_BUNDLE_DIR_NAME:${PLUGIN_NAME}>) 
       add_custom_command(TARGET ${PLUGIN_NAME} POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy_directory $<TARGET_BUNDLE_DIR:${PLUGIN_NAME}> ${INSTALL_PATH}/$<TARGET_BUNDLE_DIR_NAME:${PLUGIN_NAME}>)
elseif(MSVC OR LINUX)
       add_custom_command(TARGET ${PLUGIN_NAME} POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:${PLUGIN_NAME}> ${INSTALL_PATH})
endif()

if(BUILD_TESTS)
add_dependencies(${PLUGIN_NAME}_testable gui_testable_source test_helpers)
add_dependencies(${PLUGIN_NAME}_tests ${PLUGIN_NAME}_testable)
endif()


if(MSVC)	
	set(FRONT_PANEL_LIBRARY "${CMAKE_CURRENT_SOURCE_DIR}/Resources/okFrontPanel.lib" )
	set(FRONT_PANEL_RUNTIME "${CMAKE_CURRENT_SOURCE_DIR}/Resources/okFrontPanel.dll" )
elseif(APPLE)
	set(FRONT_PANEL_LIBRARY "${CMAKE_CURRENT_SOURCE_DIR}/Resources/libokFrontPanel.dylib" )
elseif(LINUX)
	set(FRONT_PANEL_LIBRARY "${CMAKE_CURRENT_SOURCE_DIR}/Resources/libokFrontPanel.so" )
endif()

#need to change the linked libraries search path in the test .exe
if(APPLE)
	if(BUILD_TESTS)
    add_custom_command(TARGET ${PLUGIN_NAME}_testable POST_BUILD COMMAND
    install_name_tool -change "@loader_path/../Frameworks/libokFrontPanel.dylib"
    ${FRONT_PANEL_LIBRARY} $<TARGET_FILE:${PLUGIN_NAME}_testable>)

    add_custom_command(TARGET ${PLUGIN_NAME}_tests POST_BUILD COMMAND
    install_name_tool -change "@loader_path/../Frameworks/libokFrontPanel.dylib"
    ${FRONT_PANEL_LIBRARY} $<TARGET_FILE:${PLUGIN_NAME}_tests>)
	endif()
endif()

# Need to copy Open Ephys testing libraries to test .exe directory
if(BUILD_TESTS)
if(MSVC)
       add_custom_command(TARGET ${PLUGIN_NAME}_tests POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:gui_testable_source> $<TARGET_FILE_DIR:${PLUGIN_NAME}_tests>)
       add_custom_command(TARGET ${PLUGIN_NAME}_tests POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:test_helpers> $<TARGET_FILE_DIR:${PLUGIN_NAME}_tests>)
endif()

target_compile_definitions(${PLUGIN_NAME}_testable PRIVATE -DBUILD_TESTS)
target_link_libraries(${PLUGIN_NAME}_testable PRIVATE ${FRONT_PANEL_LIBRARY} gui_testable_source test_helpers)
target_compile_definitions(${PLUGIN_NAME}_tests PRIVATE -DBUILD_TESTS -DTEST_RUNNER)
target_link_libraries(${PLUGIN_NAME}_tests PRIVATE ${PLUGIN_NAME}_testable gtest_main test_helpers ${FRONT_PANEL_LIBRARY} PUBLIC gui_testable_source)
target_include_directories(${PLUGIN_NAME}_tests PRIVATE ${GUI_TEST_HELPERS_DIR}/include ${GUI_BASE_DIR}/Source)
add_test(NAME ${PLUGIN_NAME}_tests  COMMAND ${PLUGIN_NAME}_tests)
endif()


#create filters for vs and xcode

foreach( src_file IN ITEMS ${SRC_FILES})
	get_filename_component(src_path "${src_file}" PATH)
	file(RELATIVE_PATH src_path_rel "${SOURCE_PATH}" "${src_path}")
	string(REPLACE "/" "\\" group_name "${src_path_rel}")
	source_group("${group_name}" FILES "${src_file}")
endforeach()

add_library(rhythm-plugins::okFrontPanel SHARED IMPORTED GLOBAL)
#additional libraries
if (MSVC)
	set_property(TARGET rhythm-plugins::okFrontPanel PROPERTY
				 IMPORTED_LOCATION "${CMAKE_CURRENT_SOURCE_DIR}/Resources/okFrontPanel.dll")
	set_property(TARGET rhythm-plugins::okFrontPanel PROPERTY
				 IMPORTED_IMPLIB "${CMAKE_CURRENT_SOURCE_DIR}/Resources/okFrontPanel.lib")
elseif(LINUX)
	set_property(TARGET rhythm-plugins::okFrontPanel PROPERTY
			IMPORTED_LOCATION "${CMAKE_CURRENT_SOURCE_DIR}/Resources/libokFrontPanel.so")
elseif(APPLE)
	set_property(TARGET rhythm-plugins::okFrontPanel PROPERTY
			IMPORTED_LOCATION "${CMAKE_CURRENT_SOURCE_DIR}/Resources/libokFrontPanel.dylib")
endif()

target_link_libraries(${PLUGIN_NAME} rhythm-plugins::okFrontPanel)
target_link_libraries(${PLUGIN_NAME} ug3-electrode-model)

if(BUILD_TESTS)
target_link_libraries(${PLUGIN_NAME}_testable PRIVATE ug3-electrode-model)
target_link_libraries(${PLUGIN_NAME}_tests PRIVATE ug3-electrode-model)
endif()

#additional libraries
if (MSVC)
	install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/Resources/okFrontPanel.dll DESTINATION ${GUI_BIN_DIR}/shared CONFIGURATIONS ${CMAKE_CONFIGURATION_TYPES})
	add_custom_command(TARGET ${PLUGIN_NAME} POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/Resources/okFrontPanel.dll ${GUI_BIN_DIR}/shared)
elseif(LINUX)
	install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/Resources/libokFrontPanel.so DESTINATION ${GUI_BIN_DIR}/shared)
elseif(APPLE)
	if(BUILD_TESTS)
	add_custom_command(TARGET ${PLUGIN_NAME}_tests POST_BUILD
 				COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/Resources/libokFrontPanel.dylib ${CMAKE_BINARY_DIR}/lib/$<CONFIG>)	
	endif()
	
	add_custom_command(TARGET ${PLUGIN_NAME} POST_BUILD
 				COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/Resources/libokFrontPanel.dylib  $ENV{HOME}/Library/Application\ Support/open-ephys/shared-api8)
endif()
