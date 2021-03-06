# Copyright (C) 2007-2013 LuaDist.
# Created by Peter Drahoš, Peter Kapec
# Redistribution and use of this file is allowed according to the terms of the MIT license.
# For details see the COPYRIGHT file distributed with LuaDist.
# Please note that the package source code is licensed under its own license.

if (BIICODE)
  project ( lua C )
  cmake_minimum_required ( VERSION 2.8 )
  include ( cmake/dist.cmake )
  
  # Initializes block variables.
  INIT_BIICODE_BLOCK()

  ## CONFIGURATION
  # Default configuration (we assume POSIX by default)
  set ( LUA_PATH "LUA_PATH" CACHE STRING "Environment variable to use as package.path." )
  set ( LUA_CPATH "LUA_CPATH" CACHE STRING "Environment variable to use as package.cpath." )
  set ( LUA_INIT "LUA_INIT" CACHE STRING "Environment variable for initial script." )
  
  # Nous Xiong: default build static lib
  option (LUA_ENABLE_STATIC "Build static liblua" ON)
  if (LUA_ENABLE_STATIC)
    set (BII_LIB_TYPE STATIC)
  else ()
    set (BII_LIB_TYPE SHARED)
  endif ()

  option ( LUA_ANSI "Use only ansi features." OFF )
  option ( LUA_USE_RELATIVE_LOADLIB "Use modified loadlib.c with support for relative paths on posix systems." ON )
  option ( LUA_COMPAT_ALL "Enable backwards compatibility options." ON )
  set ( LUA_IDSIZE 60 CACHE NUMBER "gives the maximum size for the description of the source." )

  #2DO: LUAI_* and LUAL_* settings, for now defaults are used.
  set ( LUA_DIRSEP "/" )
  set ( LUA_MODULE_SUFFIX ${CMAKE_SHARED_MODULE_SUFFIX} )
  set ( LUA_LDIR ${INSTALL_LMOD} )
  set ( LUA_CDIR ${INSTALL_CMOD} )

  if ( LUA_USE_RELATIVE_LOADLIB )
    # This will set up relative paths to lib
    string ( REGEX REPLACE "[^!/]+" ".." LUA_DIR "!/${INSTALL_BIN}/" )
  else ( )
    # Direct path to installation
    set ( LUA_DIR ${CMAKE_INSTALL_PREFIX} CACHE STRING "Destination from which modules will be resolved. See INSTALL_LMOD and INSTALL_CMOD." )
  endif ( )

  set ( LUA_PATH_DEFAULT "./?.lua;${LUA_DIR}${LUA_LDIR}/?.lua;${LUA_DIR}${LUA_LDIR}/?/init.lua" )
  set ( LUA_CPATH_DEFAULT "./?${LUA_MODULE_SUFFIX};${LUA_DIR}${LUA_CDIR}/?${LUA_MODULE_SUFFIX};${LUA_DIR}${LUA_CDIR}/loadall${LUA_MODULE_SUFFIX}" )

  if ( WIN32 AND NOT CYGWIN )
    # Windows systems
    option ( LUA_WIN "Windows specific build." ON )
    option ( LUA_BUILD_WLUA "Build wLua interpretter without console output." ON )
    
    # Nous Xiong set win dll or lib
    if (LUA_ENABLE_STATIC)
      option ( LUA_BUILD_AS_DLL "Build Lua library as Dll." OFF )
    else ()
      option ( LUA_BUILD_AS_DLL "Build Lua library as Dll." ON )
    endif ()
    
    # Paths (Double escapes ne  option needed)
    set ( LUA_DIRSEP "\\\\" )
    string ( REPLACE " /" ${LUA_DIRSEP} LUA_DIR "${LUA_DIR}" )
    string ( REPLACE "/" ${LUA_DIRSEP} LUA_LDIR "${LUA_LDIR}" )
    string ( REPLACE "/" ${LUA_DIRSEP} LUA_CDIR "${LUA_CDIR}" )
    string ( REPLACE "/" ${LUA_DIRSEP} LUA_PATH_DEFAULT "${LUA_PATH_DEFAULT}" )
    string ( REPLACE "/" ${LUA_DIRSEP} LUA_CPATH_DEFAULT "${LUA_CPATH_DEFAULT}" )
  else ( )
    # Posix systems (incl. Cygwin)
    option ( LUA_USE_POSIX "Use POSIX features." ON )
    option ( LUA_USE_DLOPEN "Use dynamic linker to load modules." ON )
    option ( LUA_USE_MKSTEMP "Use mkstep." ON )
    option ( LUA_USE_ISATTY "Use tty." ON )
    option ( LUA_USE_POPEN "Use popen." ON )
    option ( LUA_USE_ULONGJMP "Use ulongjmp" ON )
    option ( LUA_USE_GMTIME_R "Use GTIME_R" ON )
    # Apple and Linux specific
    if ( LINUX OR APPLE )
      option ( LUA_USE_STRTODHEX "Assume 'strtod' handles hexa formats" ON )
      option ( LUA_USE_AFORMAT "Assume 'printf' handles 'aA' specifiers" ON )
      option ( LUA_USE_LONGLONG "Assume support for long long" ON )
    endif ( )
  endif ( )

  ## SETUP
  # Optional libraries
  find_package ( Readline )
  if ( READLINE_FOUND )
    option ( LUA_USE_READLINE "Use readline in the Lua CLI." ON )
  endif ( )

  find_package ( Curses )
  if ( CURSES_FOUND )
    option ( LUA_USE_CURSES "Use curses in the Lua CLI." ON )
  endif ( )

  # Setup needed variables and libraries
  if ( LUA_USE_POSIX )
    # On POSIX Lua links to standard math library "m"
    list ( APPEND LIBS m )
  endif ( )

  if ( LUA_USE_DLOPEN )
    # Link to dynamic linker library "dl"
    find_library ( DL_LIBRARY NAMES dl )
    if ( DL_LIBRARY )
      list ( APPEND LIBS ${DL_LIBRARY} )
    endif ( )
  endif ( )

  if ( LUA_USE_READLINE )
    # Add readline
    include_directories ( ${READLINE_INCLUDE_DIR} )
    list ( APPEND LIBS ${READLINE_LIBRARY} )
  endif ( )

  if ( LUA_USE_CURSES )
    # Add curses
    include_directories ( ${CURSES_INCLUDE_DIR} )
    list ( APPEND LIBS ${CURSES_LIBRARY} )
  endif ( )

  ## SOURCES
  # Generate luaconf.h
  configure_file ( src/luaconf.h.in ${CMAKE_CURRENT_BINARY_DIR}/luaconf.h )

  # Sources and headers
  include_directories ( src ${CMAKE_CURRENT_BINARY_DIR} )
  set ( SRC_CORE src/lapi.c src/lcode.c src/lctype.c src/ldebug.c src/ldo.c src/ldump.c 
    src/lfunc.c src/lgc.c src/llex.c src/lmem.c src/lobject.c src/lopcodes.c src/lparser.c 
    src/lstate.c src/lstring.c src/ltable.c src/ltm.c src/lundump.c src/lvm.c src/lzio.c )
  set ( SRC_LIB src/lauxlib.c src/lbaselib.c src/lbitlib.c src/lcorolib.c src/ldblib.c 
    src/liolib.c src/lmathlib.c src/loslib.c src/lstrlib.c src/ltablib.c src/linit.c )
  set ( SRC_LUA src/lua.c )
  set ( SRC_LUAC src/luac.c )

  if ( LUA_USE_RELATIVE_LOADLIB )
    # Use modified loadlib
    list ( APPEND SRC_LIB src/loadlib_rel.c )
  else ( )
    list ( APPEND SRC_LIB src/loadlib.c )
  endif ( )
  
  # Nous Xiong: set lib and exe src
  set (BII_LIB_SRC ${SRC_CORE} ${SRC_LIB} ${LUA_DLL_RC} ${LUA_DEF})
  set (BII_src_lua_SRC ${SRC_LUA} src/lua.rc)
  set (BII_src_luac_SRC ${SRC_CORE} ${SRC_LIB} ${SRC_LUAC} src/luac.rc)
  if (LUA_BUILD_WLUA)
    set (BII_src_wmain_SRC src/wmain.c ${SRC_LUA} src/lua.rc)
    # Nous Xiong: manual adding src_wmain exe
    list (APPEND BII_BLOCK_EXES src_wmain)
  else ()
    list (REMOVE_ITEM BII_BLOCK_EXES 
      src_wmain
      )
  endif ()

  # Actually create targets: EXEcutables and libraries.
  ADD_BIICODE_TARGETS()
  
  # Nous Xiong: add include dirs for Lua Library
  target_include_directories (${BII_BLOCK_TARGET} INTERFACE src ${CMAKE_CURRENT_BINARY_DIR})
  
  # Nous Xiong: set readline and curses
  if ( LUA_USE_READLINE )
    # Add readline
    target_include_directories ( ${BII_BLOCK_TARGET} INTERFACE ${READLINE_INCLUDE_DIR} )
  endif ()

  if ( LUA_USE_CURSES )
    # Add curses
    target_include_directories ( ${BII_BLOCK_TARGET} INTERFACE ${CURSES_INCLUDE_DIR} )
  endif ()

  ## BUILD
  # Create lua library
  if (LIBS)
    target_link_libraries ( ${BII_LIB_TARGET} PUBLIC ${LIBS} )
  endif ()
  set_target_properties ( ${BII_LIB_TARGET} PROPERTIES OUTPUT_NAME lua CLEAN_DIRECT_OUTPUT 1 )
  if ( LUA_BUILD_AS_DLL )
    set_target_properties ( ${BII_LIB_TARGET} PROPERTIES COMPILE_DEFINITIONS LUA_BUILD_AS_DLL )
  endif ()

  # Nous Xiong: set lua exe
  target_link_libraries ( ${BII_src_lua_TARGET} PUBLIC ${BII_LIB_TARGET} )

  # Nous Xiong: set luac exe
  target_link_libraries ( ${BII_src_luac_TARGET} PUBLIC ${BII_LIB_TARGET} )

  # On windows a variant of the lua interpreter without console output needs to be built
  if ( LUA_BUILD_WLUA )
    # Nous Xiong: set wlua exe
    target_link_libraries ( ${BII_src_wmain_TARGET} PUBLIC ${BII_LIB_TARGET} )
  endif ( )
  
else ()
  include (CMakeLists_original.txt)
endif ()