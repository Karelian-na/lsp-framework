cmake_minimum_required(VERSION 3.12)

project(lsp)

include(utils.cmake)
lsp_read_version()

option(LSP_USE_SANITIZERS "Use address and undefined behavior sanitizers" ON)
option(LSP_INSTALL "Configure lsp install configuration" ON)

set(CMAKE_CXX_STANDARD 20)

if(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
	add_compile_options(/W4 /bigobj)
else()
	add_compile_options(-Wall -Wextra -pedantic)

	if(LSP_USE_SANITIZERS)
		add_compile_options(
			$<$<CONFIG:DEBUG>:-fsanitize=address,undefined>
			$<$<CONFIG:DEBUG>:-fno-omit-frame-pointer>
		)
		add_link_options($<$<CONFIG:DEBUG>:-fsanitize=address,undefined>)
	endif()

	if(WIN32 AND CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
		add_compile_options(-Wa,-mbig-obj)
	endif()
endif()

add_executable(lspgen lspgen/lspgen.cpp lsp/json/json.cpp lsp/str.cpp)
target_include_directories(lspgen PRIVATE .)

set(LSP_SRC_DIR lsp/)

set(LSP_HEADERS
	# lsp
	connection.h
	error.h
	fileuri.h
	messagebase.h
	messagedispatcher.h
	messagehandler.h
	nullable.h
	requesthandler.h
	serialization.h
	str.h
	util.h
	# io
	io/standardio.h
	# json
	json/json.h
	# jsonrpc
	jsonrpc/jsonrpc.h
)
list(TRANSFORM LSP_HEADERS PREPEND ${LSP_SRC_DIR})

set(LSP_SOURCES
	# lsp
	connection.cpp
	messagedispatcher.cpp
	requesthandler.cpp
	str.cpp
	fileuri.cpp
	# io
	io/standardio.cpp
	# json
	json/json.cpp
	# jsonrpc
	jsonrpc/jsonrpc.cpp
)
list(TRANSFORM LSP_SOURCES PREPEND ${LSP_SRC_DIR})

set(LSP_GENERATED_FILES_DIR ${CMAKE_CURRENT_BINARY_DIR}/generated)
file(MAKE_DIRECTORY ${LSP_GENERATED_FILES_DIR}/lsp)

set(LSP_GENERATED_HEADERS
	${LSP_GENERATED_FILES_DIR}/lsp/types.h
	${LSP_GENERATED_FILES_DIR}/lsp/messages.h
)

set(LSP_GENERATED_SOURCES
	${LSP_GENERATED_FILES_DIR}/lsp/types.cpp
	${LSP_GENERATED_FILES_DIR}/lsp/messages.cpp
)

set(LSP_METAMODEL_JSON ${CMAKE_CURRENT_SOURCE_DIR}/lspgen/lspmetamodel.json)

add_custom_command(
	COMMAND lspgen ${LSP_METAMODEL_JSON}
	WORKING_DIRECTORY ${LSP_GENERATED_FILES_DIR}/lsp
	OUTPUT ${LSP_GENERATED_HEADERS} ${LSP_GENERATED_SOURCES}
	DEPENDS lspgen ${LSP_METAMODEL_JSON}
	COMMENT "Generating lsp types from meta model..."
)

add_library(lsp STATIC)
set_target_properties(
	lsp PROPERTIES
	DEBUG_POSTFIX d
)
target_include_directories(lsp PUBLIC 
	"$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}>" 
	"$<BUILD_INTERFACE:${LSP_GENERATED_FILES_DIR}>"
	"$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>"
)
target_sources(lsp PRIVATE ${LSP_HEADERS} ${LSP_SOURCES} ${LSP_GENERATED_HEADERS} ${LSP_GENERATED_SOURCES})

if (DEFINED LSP_INSTALL)
	include(GNUInstallDirs)
	include(CMakePackageConfigHelpers)

	set(EXPORT_CMAKE_CONFIG_DIR "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}/")

	# write and install pkgConfig file
	configure_file(
		"${CMAKE_CURRENT_SOURCE_DIR}/lsp.pc.in"
		"${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.pc"
		@ONLY
	)
	install(
		FILES "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.pc"
		DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig/"
	)

	# write and install config files
	write_basic_config_version_file(
		"${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
		VERSION ${LSP_VERSION}
		COMPATIBILITY SameMajorVersion
	)
	configure_package_config_file(
		"${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}Config.cmake.in"
		"${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
		INSTALL_DESTINATION 
		${EXPORT_CMAKE_CONFIG_DIR}
	)
	install(
		EXPORT lsp
		DESTINATION ${EXPORT_CMAKE_CONFIG_DIR}
		NAMESPACE lsp::
		FILE "${PROJECT_NAME}ConfigTargets.cmake"
	)
	install(
		FILES
			"${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
			"${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
		DESTINATION ${EXPORT_CMAKE_CONFIG_DIR}
	)

	# install headers
	install(
		FILES ${LSP_GENERATED_HEADERS}
		DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/lsp"
	)
	foreach(RELATIVE_HEADER_FILE ${LSP_HEADERS})
		get_filename_component(RELATIVE_PATH ${RELATIVE_HEADER_FILE} PATH)
		# 安装文件到对应的相对路径下
		install(FILES ${RELATIVE_HEADER_FILE} DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/${RELATIVE_PATH}")
	endforeach()

	# install libs
	install(
		TARGETS lsp
		EXPORT lsp
		ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
		LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
		RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
	)
endif()