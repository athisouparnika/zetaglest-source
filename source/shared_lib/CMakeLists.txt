#########################################################################################
# streflop

OPTION(FORCE_EMBEDDED_LIBS "Force use of embedded libraries' code." OFF)

add_definitions("-DDATADIR=${INSTALL_DIR_DATA}")

# On Travis, the build fails on osx and Ubuntu trusty using
# clang 5.0 unless the -std=c++11 if used
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall")

if(WANT_USE_STREFLOP)
	# IMPORTANT: should come BEFORE finding packages
	find_package(PkgConfig REQUIRED)
	IF(FORCE_STREFLOP_SOFTWRAPPER)
		pkg_search_module(STREFLOP streflop-soft)
	ELSE()
		IF(HAS_SSE_EXTENSIONS AND NOT ${FORCE_MAX_SSE_LEVEL} MATCHES "0")
			pkg_search_module(STREFLOP streflop-sse)
		ELSE()
			IF(HAS_X87_SUPPORT)
				pkg_search_module(STREFLOP streflop-x87)
			ELSE()
				pkg_search_module(STREFLOP streflop-soft)
			ENDIF()
		ENDIF()
	ENDIF()
	IF(NOT STREFLOP_FOUND)
	    pkg_search_module(STREFLOP streflop)
	ENDIF()
	MESSAGE(STATUS "Search for Library STREFLOP result = ${STREFLOP_FOUND} libs: ${STREFLOP_LIBRARIES} include dirs: ${STREFLOP_INCLUDE_DIRS}")
	IF(FORCE_EMBEDDED_LIBS)
		SET(STREFLOP_FOUND OFF)
		MESSAGE(STATUS "FORCING USE of EMBEDDED Libraries...")
	ENDIF()
	IF(NOT STREFLOP_FOUND)
		ADD_SUBDIRECTORY(sources/streflop)
	ELSE()
		INCLUDE_DIRECTORIES(${STREFLOP_INCLUDE_DIRS} ${STREFLOP_INCLUDE_DIRS}/streflop)
		SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${STREFLOP_LIBRARIES})
		ADD_DEFINITIONS("-DUSE_STREFLOP_PKG")
	ENDIF()
ENDIF()

#########################################################################################
# common libraries

SET(EXTERNAL_LIBS "")
SET(TARGET_NAME "libzetaglest")

IF(BUILD_MODEL_VIEWER OR BUILD_MAP_EDITOR OR BUILD_ZETAGLEST)
	MESSAGE(STATUS "Building ZetaGlest shared library...")

	INCLUDE (CheckIncludeFiles)

	IF(WIN32)
		SET(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH}
					${PROJECT_SOURCE_DIR}/source/win32_deps/lib
					${PROJECT_SOURCE_DIR}/source/win32_deps/xerces-c-src_2_8_0/lib
					${PROJECT_SOURCE_DIR}/source/win32_deps/wxWidgets-2.8.10/lib)

		INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/source/win32_deps/include)
		INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/source/win32_deps/openal-soft-1.12.854/include)
		INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/source/win32_deps/openal-soft-1.12.854)
		INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/source/win32_deps/xerces-c-src_2_8_0/include)
		INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/source/win32_deps/libogg-1.2.1/include)
		INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/source/win32_deps/lua-5.1/src)
		INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/source/win32_deps/jpeg-8b)
		INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/source/win32_deps/lpng141)
		INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/source/win32_deps/zlib-1.2.5)
		INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/source/win32_deps/${SDL_WINDOWS_DIR_DINC}/include)
		INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/source/win32_deps/wxWidgets-2.8.10/include)

		link_directories(${PROJECT_SOURCE_DIR}/source/win32_deps/lib)
		link_directories(${PROJECT_SOURCE_DIR}/source/win32_deps/xerces-c-src_2_8_0/lib)
		link_directories(${PROJECT_SOURCE_DIR}/source/win32_deps/wxWidgets-2.8.10/lib)

		INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/source/win32_deps/include)
		INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/source/shared_lib/include/platform/posix)
		INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/source/shared_lib/include/platform/win32)
	ENDIF()

	IF(WANT_USE_OpenSSL)
		FIND_PACKAGE(OpenSSL)
		IF(OPENSSL_FOUND)
		    INCLUDE_DIRECTORIES(${OPENSSL_INCLUDE_DIR})
		ELSE()
		    MESSAGE(STATUS "**NOTE: OpenSSL is usually not required.")
		ENDIF()
	ENDIF()

	include(FindCURL)
	INCLUDE_DIRECTORIES(${CURL_INCLUDE_DIRS})
	IF(UNIX)
		IF(NOT OPENSSL_FOUND)
		    # default
		    SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${CURL_LIBRARIES})
		ELSE()
		    # mageia
		    SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${CURL_LIBRARIES} ${OPENSSL_LIBRARIES})
		ENDIF()
	ENDIF()

	FIND_PACKAGE(${SDL_VERSION_NAME} REQUIRED)
	INCLUDE_DIRECTORIES(${${SDL_VERSION_NAME}_INCLUDE_DIR})
	IF(UNIX)
		SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${${SDL_VERSION_NAME}_LIBRARY})
	ENDIF()

	FIND_PACKAGE(OpenGL REQUIRED)
	INCLUDE_DIRECTORIES(${OPENGL_INCLUDE_DIR})
	SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${OPENGL_LIBRARY})

	FIND_PACKAGE(X11 REQUIRED)
	INCLUDE_DIRECTORIES(${X11_INCLUDE_DIR})
	SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${X11_LIBRARIES})

	include(FindLua)
	INCLUDE_DIRECTORIES(${LUA_INCLUDE_DIR})
	SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${LUA_LIBRARIES} ${CMAKE_DL_LIBS})

	FIND_PACKAGE(JPEG REQUIRED)
	INCLUDE_DIRECTORIES(${JPEG_INCLUDE_DIR})
	SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${JPEG_LIBRARY})

	FIND_PACKAGE(OpenAL REQUIRED)
	INCLUDE_DIRECTORIES(${OPENAL_INCLUDE_DIR})
	SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${OPENAL_LIBRARY})

	FIND_PACKAGE(PNG REQUIRED)
	INCLUDE_DIRECTORIES(${PNG_INCLUDE_DIR})
	SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${PNG_LIBRARY})

	IF(WANT_USE_XercesC)
		FIND_PACKAGE(XercesC REQUIRED)
		INCLUDE_DIRECTORIES(${XERCESC_INCLUDE_DIR})
		IF(UNIX)
			SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${XERCESC_LIBRARY})
		ENDIF()
	ENDIF()

	OPTION(WANT_USE_FontConfig "use the library fontconfig" ON)
	IF(WANT_USE_FontConfig)
		FIND_PACKAGE(FontConfig)
		IF(FONTCONFIG_FOUND)
			MESSAGE(STATUS "**NOTE: FontConfig support was detected and enabled.")
			SET(HAVE_FONTCONFIG 1)
			ADD_DEFINITIONS(-DHAVE_FONTCONFIG)
			INCLUDE_DIRECTORIES( ${FONTCONFIG_INCLUDE_DIR} )
			SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${FONTCONFIG_LIBRARIES} )
		ENDIF(FONTCONFIG_FOUND)
	ENDIF()

	OPTION(WANT_USE_FTGL "Use libFTGL for on-screen fonts (found on your system)" ON)
	find_package( Freetype REQUIRED)
	include_directories( ${FREETYPE_INCLUDE_DIRS} )
	FIND_PACKAGE(FTGL)
	IF(WANT_USE_FTGL)
		if (FTGL_FOUND)
			MESSAGE(STATUS "**NOTE: FTGL font support was detected and enabled.")
			ADD_DEFINITIONS(-DUSE_FTGL)
			INCLUDE_DIRECTORIES(${FTGL_INCLUDE_DIR})
			IF(UNIX)
				SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${FTGL_LIBRARY})
			ENDIF()
		else()
			MESSAGE("**NOTE: Warning! FTGL has been enabled but not detected. Your compilation will probably break. Turn off FTGL support by setting WANT_USE_FTGL to false, or visit http://ftgl.wiki.sourceforge.net/ for help on installing FTGL.")
		endif()
	ELSE()
		IF(FTGL_FOUND)
			MESSAGE(STATUS "**NOTE:  Warning: FTGL font support was detected but NOT enabled. You can enable it by setting WANT_USE_FTGL=true")
		ELSE()
			MESSAGE(STATUS "**NOTE:  Warning: FTGL font support was not detected. Visit http://ftgl.wiki.sourceforge.net/ for help on installing FTGL.")
		ENDIF()
	ENDIF()

	IF(NOT FREETYPE_FOUND)
		FIND_PACKAGE(Freetype REQUIRED)
		INCLUDE_DIRECTORIES(${FREETYPE_INCLUDE_DIRS})
	ENDIF()
	IF(UNIX)
		SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${FREETYPE_LIBRARY})
	ENDIF()

	include(FindGLEW)
	INCLUDE_DIRECTORIES(${GLEW_INCLUDE_DIRS})
	IF(UNIX)
		SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${GLEW_LIBRARIES})
	ENDIF()

	if(WANT_USE_FriBiDi)
		find_package( FriBiDi )
		if(FRIBIDI_FOUND)
			add_definitions(-DHAVE_FRIBIDI)

			include_directories( ${FRIBIDI_INCLUDE_DIR} )
			SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${FRIBIDI_LIBRARIES})

		else()
			message("Could not find FriBiDi. Disabling FriBiDi support.")
		endif()
	endif()

	if(WANT_USE_GoogleBreakpad)
		find_package( GoogleBreakpad )
		if(BREAKPAD_EXCEPTION_HANDLER_FOUND AND BREAKPAD_EXCEPTION_HANDLER_LIBRARIES)
			add_definitions(-DHAVE_GOOGLE_BREAKPAD)

			#include_directories( ${BREAKPAD_EXCEPTION_HANDLER_INCLUDE_DIR} )
			#SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${BREAKPAD_EXCEPTION_HANDLER_LIBRARIES})

		else()
			message(STATUS "** Message: Could not find GoogleBreakpad. Disabling GoogleBreakpad support.")
		endif()
	endif()

	message(STATUS "Searching for zlib")
	include(FindZLIB)
	IF (FORCE_EMBEDDED_LIBS OR NOT ZLIB_FOUND)
		SET(DIRS_WITH_SRC
			${DIRS_WITH_SRC}
			# miniz is a library included with the ZG distribution. Not as
			# developed as zlib, but smaller, and uses the same function names
			# so basically a replacement for miniz. We may one day replace
			# miniz with the zlib sources
			miniz)
	ELSE()
		if (ZLIB_FOUND)
			message(STATUS "Using external zlib")
			SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${ZLIB_LIBRARIES})
			INCLUDE_DIRECTORIES(${ZLIB_INCLUDE_DIRS})
			add_definitions("-DHAVE_ZLIB")
		ENDIF()
	ENDIF()

	SET(DIRS_WITH_SRC
		${DIRS_WITH_SRC}
		compression
		feathery_ftp
		glew
		graphics
		graphics/gl
		lua
		map
		platform/common
		sound
		sound/openal
		util
		xml)

	IF(WANT_USE_STREFLOP AND NOT STREFLOP_FOUND)
		SET(DIRS_WITH_SRC
            ${DIRS_WITH_SRC}
			streflop
			streflop/libm_flt32_source
            streflop/softfloat)
	ENDIF()

	IF(FORCE_EMBEDDED_LIBS)
		SET(FORCE_USE_EMBEDDED_Miniupnpc ON)
	ELSE()
		OPTION(FORCE_USE_EMBEDDED_Miniupnpc "Use embedded to source code version of miniupnpc" OFF)
	ENDIF()
	IF(NOT FORCE_USE_EMBEDDED_Miniupnpc)
		FIND_PACKAGE(Miniupnpc)
	ENDIF()

	if(NOT MINIUPNP_FOUND)
		if(NOT FORCE_USE_EMBEDDED_Miniupnpc)
			MESSAGE(STATUS "*** Using EMBEDDED miniupnpc since dev system does not have it... MINIUPNP_FOUND [${MINIUPNP_FOUND}]")
		else()
			MESSAGE(STATUS "*** Using EMBEDDED miniupnpc because it was forced by option")
		endif()

		if (NOT WIN32)
			add_definitions (-DMINIUPNPC_SET_SOCKET_TIMEOUT)
			add_definitions (-D_BSD_SOURCE)

			if (CMAKE_SYSTEM_NAME STREQUAL "Darwin")
					add_definitions (-DMACOSX -D_DARWIN_C_SOURCE)
			 else()
					add_definitions (-D_POSIX_C_SOURCE=200809L)
			endif ()
		endif()

		SET(DIRS_WITH_SRC ${DIRS_WITH_SRC} platform/miniupnpc)
	else()
		MESSAGE(STATUS "*** Using miniupnpc lib found in [${MINIUPNP_INCLUDE_DIR}] MINIUPNP_FOUND [${MINIUPNP_FOUND}] MINIUPNP_LIBRARY [${MINIUPNP_LIBRARY}]")

		INCLUDE_DIRECTORIES(${MINIUPNP_INCLUDE_DIR})
		SET(EXTERNAL_LIBS ${EXTERNAL_LIBS} ${MINIUPNP_LIBRARY})
	endif()

	IF(MINIUPNPC_VERSION_PRE1_5)
	        ADD_DEFINITIONS(-DMINIUPNPC_VERSION_PRE1_5)
                message(STATUS "Adding macro for miniupnpc version: pre v1.5")
	ENDIF()
	IF(MINIUPNPC_VERSION_PRE1_6)
		ADD_DEFINITIONS(-DMINIUPNPC_VERSION_PRE1_6)
                message(STATUS "Adding macro for miniupnpc version: pre v1.6")
	ENDIF()
	IF(MINIUPNPC_VERSION_PRE1_7)
		ADD_DEFINITIONS(-DMINIUPNPC_VERSION_PRE1_7)
                message(STATUS "Adding macro for miniupnpc version: pre v1.7")
	ENDIF()

	IF(NOT MINIUPNPC_VERSION_PRE1_5 AND NOT MINIUPNPC_VERSION_PRE1_6 AND NOT MINIUPNPC_VERSION_PRE1_7)
		message(STATUS "**No macros required for miniupnpc version")
	ENDIF()

	IF(APPLE)
		SET(DIRS_WITH_SRC ${DIRS_WITH_SRC} platform/${SDL_VERSION_SNAME})
		SET(DIRS_WITH_SRC ${DIRS_WITH_SRC} platform/posix)
		SET(DIRS_WITH_SRC ${DIRS_WITH_SRC} platform/macosx)
	ELSEIF(WIN32)
		SET(DIRS_WITH_SRC ${DIRS_WITH_SRC} platform/win32)
		SET(DIRS_WITH_SRC ${DIRS_WITH_SRC} ${PROJECT_SOURCE_DIR}/source/win32_deps/src)
	ELSE()
		SET(DIRS_WITH_SRC ${DIRS_WITH_SRC} platform/${SDL_VERSION_SNAME})
		SET(DIRS_WITH_SRC ${DIRS_WITH_SRC} platform/posix)
		SET(DIRS_WITH_SRC ${DIRS_WITH_SRC} platform/unix)
	ENDIF()

	SET(ZG_INCLUDES_ROOT "include/")
	SET(ZG_SOURCES_ROOT "sources/")
	SET(ZG_INCLUDE_FILES "")
	SET(ZG_SOURCE_FILES "")
	INCLUDE_DIRECTORIES( ${ZG_INCLUDES_ROOT}platform/${SDL_VERSION_SNAME} )
        INCLUDE_DIRECTORIES( ${ZG_INCLUDES_ROOT}xml/rapidxml )
        INCLUDE_DIRECTORIES("sources/")

	IF(WIN32)
		SET(ZG_SOURCE_FILES ${ZG_SOURCE_FILES} ${PROJECT_SOURCE_DIR}/source/shared_lib/sources/platform/posix/socket.cpp)
		SET(ZG_SOURCE_FILES ${ZG_SOURCE_FILES} ${PROJECT_SOURCE_DIR}/source/shared_lib/sources/platform/posix/miniftpserver.cpp)
		SET(ZG_SOURCE_FILES ${ZG_SOURCE_FILES} ${PROJECT_SOURCE_DIR}/source/shared_lib/sources/platform/posix/miniftpclient.cpp)
		SET(ZG_SOURCE_FILES ${ZG_SOURCE_FILES} ${PROJECT_SOURCE_DIR}/source/shared_lib/sources/platform/${SDL_VERSION_SNAME}/gl_wrap.cpp)
		SET(ZG_SOURCE_FILES ${ZG_SOURCE_FILES} ${PROJECT_SOURCE_DIR}/source/shared_lib/sources/platform/${SDL_VERSION_SNAME}/thread.cpp)
		SET(ZG_SOURCE_FILES ${ZG_SOURCE_FILES} ${PROJECT_SOURCE_DIR}/source/shared_lib/sources/platform/${SDL_VERSION_SNAME}/window.cpp)
		SET(ZG_SOURCE_FILES ${ZG_SOURCE_FILES} ${PROJECT_SOURCE_DIR}/source/shared_lib/sources/platform/${SDL_VERSION_SNAME}/window_gl.cpp)
		SET(ZG_SOURCE_FILES ${ZG_SOURCE_FILES} ${PROJECT_SOURCE_DIR}/source/win32_deps/src/glprocs.c)
	ENDIF()

	FOREACH(DIR IN LISTS DIRS_WITH_SRC)
		set(INCLUDE_DIR_TO_GLOB ${ZG_INCLUDES_ROOT}${DIR})
		INCLUDE_DIRECTORIES( ${INCLUDE_DIR_TO_GLOB} )
		IF(APPLE)
			#MESSAGE(STATUS "INCLUDE_DIR_TO_GLOB: ${INCLUDE_DIR_TO_GLOB}")
			FILE(GLOB INC_FILES_FROM_THIS_DIR ${INCLUDE_DIR_TO_GLOB}/*.h)
			#MESSAGE(STATUS "INCLUDE FILES GLOBBED: ${INC_FILES_FROM_THIS_DIR}")
			SET(ZG_INCLUDE_FILES ${ZG_INCLUDE_FILES} ${INC_FILES_FROM_THIS_DIR})
		ENDIF(APPLE)

		set(SRC_DIR_TO_GLOB ${ZG_SOURCES_ROOT}${DIR})
		#MESSAGE(STATUS "SRC_DIR_TO_GLOB: ${SRC_DIR_TO_GLOB}")
		FILE(GLOB SRC_FILES_FROM_THIS_DIR ${SRC_DIR_TO_GLOB}/*.cpp)
		#MESSAGE(STATUS "SRC_FILES_FROM_THIS_DIR: ${SRC_FILES_FROM_THIS_DIR}")
		SET(ZG_SOURCE_FILES ${ZG_SOURCE_FILES} ${SRC_FILES_FROM_THIS_DIR})

		set(SRC_DIR_TO_GLOB ${ZG_SOURCES_ROOT}${DIR})
		#MESSAGE(STATUS "SRC_DIR_TO_GLOB: ${SRC_DIR_TO_GLOB}")
		FILE(GLOB SRC_FILES_FROM_THIS_DIR ${SRC_DIR_TO_GLOB}/*.c)
		#MESSAGE(STATUS "SRC_FILES_FROM_THIS_DIR: ${SRC_FILES_FROM_THIS_DIR}")
		SET(ZG_SOURCE_FILES ${ZG_SOURCE_FILES} ${SRC_FILES_FROM_THIS_DIR})

		IF(APPLE)
			FILE(GLOB SRC_FILES_FROM_THIS_DIR ${SRC_DIR_TO_GLOB}/*.mm)
			#MESSAGE(STATUS "SRC_FILES_FROM_THIS_DIR: ${SRC_FILES_FROM_THIS_DIR}")
			SET(ZG_SOURCE_FILES ${ZG_SOURCE_FILES} ${SRC_FILES_FROM_THIS_DIR})
			FILE(GLOB SRC_FILES_FROM_THIS_DIR ${SRC_DIR_TO_GLOB}/*.m)
			#MESSAGE(STATUS "SRC_FILES_FROM_THIS_DIR: ${SRC_FILES_FROM_THIS_DIR}")
			SET(ZG_SOURCE_FILES ${ZG_SOURCE_FILES} ${SRC_FILES_FROM_THIS_DIR})
		ENDIF(APPLE)
	ENDFOREACH(DIR)

	#MESSAGE(STATUS "Source files: ${ZG_INCLUDE_FILES}")
	#MESSAGE(STATUS "Source files: ${ZG_SOURCE_FILES}")
	#MESSAGE(STATUS "Include dirs: ${INCLUDE_DIRECTORIES}")

	IF(APPLE)
		SET(PLATFORM_SPECIFIC_DEFINES "-DHAVE_SYS_IOCTL_H")
	ELSEIF(WIN32)
		SET(PLATFORM_SPECIFIC_DEFINES "-DX11_AVAILABLE")
	ELSE()
		SET(PLATFORM_SPECIFIC_DEFINES "-DX11_AVAILABLE -DHAVE_SYS_IOCTL_H")
	ENDIF()

	SET_SOURCE_FILES_PROPERTIES(${ZG_SOURCE_FILES} PROPERTIES COMPILE_FLAGS
		"${PLATFORM_SPECIFIC_DEFINES} ${STREFLOP_PROPERTIES} ${CXXFLAGS}")

	SET_SOURCE_FILES_PROPERTIES(${ZG_INCLUDE_FILES} PROPERTIES HEADER_FILE_ONLY 1)

	IF(NOT WIN32)
		ADD_LIBRARY(${TARGET_NAME} STATIC ${ZG_SOURCE_FILES} ${ZG_INCLUDE_FILES})
		set_target_properties(libzetaglest PROPERTIES PREFIX "")
		#MESSAGE(STATUS "Building shared game library as a STATIC library and saving to [${LIBRARY_OUTPUT_PATH}]")
	ELSE()
		SET(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/data/glest_game)
		ADD_LIBRARY(${TARGET_NAME} STATIC ${ZG_SOURCE_FILES} ${ZG_INCLUDE_FILES})
		MESSAGE(STATUS "Building shared game library as a SHARED library and saving to [${LIBRARY_OUTPUT_PATH}]")
	ENDIF()

	MESSAGE(STATUS "EXTERNAL_LIBS = [${EXTERNAL_LIBS}]")

	OPTION(MINIUPNP_DEBUG_BUILD "Build ZetaGlest with UPNP DEBUG enabled" OFF)
	MARK_AS_ADVANCED(MINIUPNP_DEBUG_BUILD)
	IF(MINIUPNP_DEBUG_BUILD)
		MESSAGE(STATUS "Building ZetaGlest shared library with DEBUG info for UPNP...")
		ADD_DEFINITIONS("-DDEBUG")
	ENDIF()

	IF(WANT_USE_STREFLOP AND NOT STREFLOP_FOUND)
		TARGET_LINK_LIBRARIES(${TARGET_NAME} streflop)
	ENDIF()
	TARGET_LINK_LIBRARIES(${TARGET_NAME} ${EXTERNAL_LIBS})

	#IF(MINIUPNP_DEBUG_BUILD)
		# REMOVE_DEFINITIONS("-DDEBUG")
	#ENDIF()

ENDIF()
