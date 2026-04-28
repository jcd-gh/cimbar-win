cmake_minimum_required(VERSION 3.10)

if(WIN32)
	add_library(gui STATIC)
	target_sources(gui PRIVATE
		${libcimbar_SOURCE_DIR}/src/third_party_lib/glad/src/gles2.c
	)
	target_include_directories(gui PUBLIC
		${libcimbar_SOURCE_DIR}/src/third_party_lib/glad/include
	)
else()
	add_library(gui INTERFACE)
endif()
