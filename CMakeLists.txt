PROJECT(lfs C)
CMAKE_MINIMUM_REQUIRED(VERSION 2.8)


INCLUDE(CMakeDependentOption)
SET(CPACK_SET_DESTDIR TRUE)
INCLUDE(CPack)

# Make sure CMake can find our non-standard modules
SET(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/CMakeModules/" ${CMAKE_MODULE_PATH})

FIND_PACKAGE(Lua53)

INCLUDE_DIRECTORIES(${LUA_INCLUDE_DIR})


IF(APPLE AND IOS)
	OPTION(WANTS_BUILD_MODULE_LIBRARY "Set to ON to build loadable module." OFF)
	OPTION(WANTS_BUILD_STATIC_LIBRARY "Set to ON to build static library." ON)
ELSE()
	OPTION(WANTS_BUILD_MODULE_LIBRARY "Set to ON to build loadable module." ON)
	OPTION(WANTS_BUILD_STATIC_LIBRARY "Set to ON to build static library." OFF)
ENDIF()

IF(APPLE)
	SET(MODULE_LINK_FLAGS "-undefined dynamic_lookup")
ENDIF(APPLE)



SET(lfs_SOURCE_FILES
		${lfs_SOURCE_DIR}/src/lfs.c
)

#SET(lfs_RESOURCE_FILES
#		${lfs_SOURCE_DIR}/re.lua
#)

IF(WIN32)
	LIST(APPEND lfs_RESOURCE_FILES ${lfs_SOURCE_DIR}/src/lfs.def)
ENDIF(WIN32)

IF(WANTS_BUILD_MODULE_LIBRARY)
	ADD_LIBRARY(lfs_library_module MODULE 
		${lfs_SOURCE_FILES} 
#		${lfs_RESOURCE_FILES}
	)

	IF(WIN32)
		TARGET_LINK_LIBRARIES(lfs_library_module ${LUA_LIBRARY} ${MODULE_LINK_FLAGS})
	ELSE()
		TARGET_LINK_LIBRARIES(lfs_library_module ${MODULE_LINK_FLAGS})
	ENDIF()

	IF(ANDROID)
		# The problem seems to be that Android won't externalize .so modules into /data/data/packagename/lib unless
		# the module starts with a "lib" prefix, e.g. liblfs.so instead of lfs.so. 
		# But simply renaming the file to liblfs.so causes internal assumptions to fail,
		# such as then needing to require('liblfs') instead of require('lfs').
		# But then if we do that, Lua tries to call luaopen_liblfs instead of luaopen_lfs.
		# So we can avoid this mess by prefixing lib to the name in the cpath.
		# REMEMBER: We must ship modules on Android with the lib prefix (liblfs.so).
		# But users don't need to know about this in their code.
		# Also note that Java System.loadLibrary implicitly assumes the lib prefix,
		# so this is important for that too.
		SET_TARGET_PROPERTIES(lfs_library_module PROPERTIES 
			# call the thing lfs
			OUTPUT_NAME "lfs"
		)
	ELSE()
		SET_TARGET_PROPERTIES(lfs_library_module PROPERTIES 
			# make sure "lib" doesn't become a prefix
			PREFIX ""
			# call the thing lfs
			OUTPUT_NAME "lfs"
		)
	ENDIF()

	IF(APPLE)
		INCLUDE(XcodeDefaults)
		BLURRR_CONFIGURE_XCODE_DEFAULTS(lfs_library_module)
	ENDIF()

	INSTALL(TARGETS
		lfs_library_module
		#	RUNTIME DESTINATION bin
		LIBRARY DESTINATION lib
		ARCHIVE DESTINATION lib
		#	ARCHIVE DESTINATION lib/static
	)

ENDIF(WANTS_BUILD_MODULE_LIBRARY)

IF(WANTS_BUILD_STATIC_LIBRARY)
	ADD_LIBRARY(lfs_library_static STATIC 
		${lfs_SOURCE_FILES} 
		${lfs_RESOURCE_FILES}
	)
#	TARGET_LINK_LIBRARIES(lfs_library_static ${LUA_LIBRARY})
	# FIXME: Might cause a name clash in Visual Studio
	SET_TARGET_PROPERTIES(lfs_library_static PROPERTIES OUTPUT_NAME "lfs")
	INSTALL(TARGETS
		lfs_library_static
		#	RUNTIME DESTINATION bin
		LIBRARY DESTINATION lib
		ARCHIVE DESTINATION lib
		#	ARCHIVE DESTINATION lib/static
	)

	IF(APPLE)
		INCLUDE(XcodeDefaults)
		BLURRR_CONFIGURE_XCODE_DEFAULTS(lfs_library_static)
	ENDIF()


ENDIF(WANTS_BUILD_STATIC_LIBRARY)

#IF(LUA_INCLUDE_DIR AND EXISTS "${LUA_INCLUDE_DIR}/lua.h")
#	file(STRINGS "${LUA_INCLUDE_DIR}/lua.h" LUA_VERSION_MAJOR_LINE REGEX "^#define[ \t]+LUA_VERSION_MAJOR[ \t]+\"[0-9]+\"$")
#	file(STRINGS "${LUA_INCLUDE_DIR}/lua.h" LUA_VERSION_MINOR_LINE REGEX "^#define[ \t]+LUA_VERSION_MINOR[ \t]+\"[0-9]+\"$")
#	string(REGEX REPLACE "^#define[ \t]+LUA_VERSION_MAJOR[ \t]+\"([0-9]+)\"$" "\\1" LUA_VERSION_MAJOR "${LUA_VERSION_MAJOR_LINE}")
#	string(REGEX REPLACE "^#define[ \t]+LUA_VERSION_MINOR[ \t]+\"([0-9]+)\"$" "\\1" LUA_VERSION_MINOR "${LUA_VERSION_MINOR_LINE}")

#	SET(LUAPACKAGE_CDIR "lib/lua/${LUA_VERSION_MAJOR}.${LUA_VERSION_MINOR}" CACHE STRING "Path for Lua packaged platform specific things.")
#	SET(LUAPACKAGE_LDIR "share/lua/${LUA_VERSION_MAJOR}.${LUA_VERSION_MINOR}" CACHE STRING "Path for Lua packages platform independent things.")
#ENDIF()
#INSTALL(FILES ${lfs_RESOURCE_FILES} DESTINATION ${LUAPACKAGE_LDIR})




# For uninstall (needs cmake_uninstall.cmake.in in the top-level directory)
CONFIGURE_FILE(
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
  IMMEDIATE @ONLY)
ADD_CUSTOM_TARGET(uninstall
  "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")



