#
# Copyright (c) 2015-2023 CNRS INRIA
# Copyright (c) 2015 Wandercraft, 86 rue de Paris 91400 Orsay, France.
#

include(${JRL_CMAKE_MODULES}/python-helpers.cmake)

# --- PYTHON TARGET --- #
SET(PYWRAP ${PROJECT_NAME}_pywrap)
SET(PYWRAP ${PYWRAP} PARENT_SCOPE)

# --- COMPILE WRAPPER
REMOVE_PATH_FROM_LIST(${PROJECT_NAME}_BINDINGS_PYTHON_SOURCES
  "bindings/python/"
  ${PROJECT_NAME}_BINDINGS_PYTHON_SOURCES)
PREPEND_PATH_FROM_LIST(${PROJECT_NAME}_BINDINGS_PYTHON_PUBLIC_HEADERS
  "../../"
  ${PROJECT_NAME}_BINDINGS_PYTHON_PUBLIC_HEADERS)
MAKE_DIRECTORY("${${PROJECT_NAME}_BINARY_DIR}/bindings/python/${PROJECT_NAME}")
SET(${PYWRAP}_SOURCES ${${PROJECT_NAME}_BINDINGS_PYTHON_SOURCES})
SET(${PYWRAP}_HEADERS ${${PROJECT_NAME}_BINDINGS_PYTHON_PUBLIC_HEADERS})

IF(BUILD_PYTHON_INTERFACE)
  ADD_CUSTOM_TARGET(python)
  SET_TARGET_PROPERTIES(python PROPERTIES EXCLUDE_FROM_DEFAULT_BUILD True)

  PYTHON_BUILD_GET_TARGET(python_build_target)
  add_dependencies(python ${python_build_target})

  SET(PKG_CONFIG_PYWRAP_REQUIRES "eigenpy >= 2.6.2")
  ADD_LIBRARY(${PYWRAP} SHARED ${${PYWRAP}_SOURCES} ${${PYWRAP}_HEADERS})
  MODERNIZE_TARGET_LINK_LIBRARIES(${PYWRAP} SCOPE PUBLIC
    TARGETS eigenpy::eigenpy)
  IF(BUILD_WITH_OPENMP_SUPPORT)
    TARGET_COMPILE_OPTIONS(${PYWRAP} PRIVATE ${OpenMP_CXX_FLAGS})
    TARGET_COMPILE_DEFINITIONS(${PYWRAP} PRIVATE -DPINOCCHIO_PYTHON_INTERFACE_WITH_OPENMP)
    TARGET_INCLUDE_DIRECTORIES(${PYWRAP} SYSTEM PRIVATE ${OpenMP_CXX_INCLUDE_DIR})
    IF(LINK_PYTHON_INTERFACE_TO_OPENMP)
      TARGET_LINK_LIBRARIES(${PYWRAP} PRIVATE ${OpenMP_CXX_LIBRARIES})
    ENDIF(LINK_PYTHON_INTERFACE_TO_OPENMP)
  ENDIF(BUILD_WITH_OPENMP_SUPPORT)
  ADD_DEPENDENCIES(python ${PYWRAP})

  # Do not report:
  #  -Wconversion as the BOOST_PYTHON_FUNCTION_OVERLOADS implicitly converts.
  #  -Wcomment as latex equations have multi-line comments.
  #  -Wself-assign-overloaded as bp::self operations trigger this (Clang only)
  CXX_FLAGS_BY_COMPILER_FRONTEND(
    GNU -Wno-conversion -Wno-comment -Wno-self-assign-overloaded
    OUTPUT PRIVATE_OPTIONS
    FILTER)
  TARGET_COMPILE_OPTIONS(${PYWRAP} PRIVATE ${PRIVATE_OPTIONS})

  SET_TARGET_PROPERTIES(${PYWRAP} PROPERTIES VERSION ${PROJECT_VERSION})
  IF(BUILD_WITH_COMMIT_VERSION)
    TAG_LIBRARY_VERSION(${PYWRAP})
  ENDIF(BUILD_WITH_COMMIT_VERSION)
  ADD_HEADER_GROUP(${PYWRAP}_HEADERS)
  ADD_SOURCE_GROUP(${PYWRAP}_SOURCES)

  TARGET_LINK_LIBRARIES(${PYWRAP} PUBLIC ${PROJECT_NAME})
  IF(BUILD_WITH_HPP_FCL_PYTHON_BINDINGS)
    TARGET_COMPILE_DEFINITIONS(${PYWRAP} PRIVATE -DPINOCCHIO_PYTHON_INTERFACE_WITH_HPP_FCL_PYTHON_BINDINGS)
  ENDIF(BUILD_WITH_HPP_FCL_PYTHON_BINDINGS)
  IF(WIN32)
    TARGET_LINK_LIBRARIES(${PYWRAP} PUBLIC ${PYTHON_LIBRARY})
  ENDIF(WIN32)

  IF(IS_ABSOLUTE ${PYTHON_SITELIB})
    SET(ABSOLUTE_PYTHON_SITELIB ${PYTHON_SITELIB})
  ELSE()
    SET(ABSOLUTE_PYTHON_SITELIB ${CMAKE_INSTALL_PREFIX}/${PYTHON_SITELIB})
  ENDIF()
  SET(${PYWRAP}_INSTALL_DIR ${ABSOLUTE_PYTHON_SITELIB}/${PROJECT_NAME})

  SET_TARGET_PROPERTIES(${PYWRAP}
    PROPERTIES
    PREFIX ""
    SUFFIX ${PYTHON_EXT_SUFFIX}
    OUTPUT_NAME "${PYWRAP}"
    LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bindings/python/${PROJECT_NAME}"
    # On Windows, shared library are treat as binary
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bindings/python/${PROJECT_NAME}"
    )

  IF(UNIX)
    GET_RELATIVE_RPATH(${${PYWRAP}_INSTALL_DIR} ${PYWRAP}_INSTALL_RPATH)
    SET_TARGET_PROPERTIES(${PYWRAP} PROPERTIES INSTALL_RPATH "${${PYWRAP}_INSTALL_RPATH}")
  ENDIF()

  if(GENERATE_PYTHON_STUBS)
      include("${CMAKE_SOURCE_DIR}/cmake/stubs.cmake")
      LOAD_STUBGEN()
      # Set PYWRAP and PROJECT_NAME as stubs dependencies.
      # PROJECT_NAME is mandatory (even if it's a PYWRAP dependency)
      # to find PROJECT_NAME name DLL on windows.
      GENERATE_STUBS(${CMAKE_CURRENT_BINARY_DIR} ${PROJECT_NAME} ${ABSOLUTE_PYTHON_SITELIB}
        ${PYWRAP} ${PROJECT_NAME})
  endif(GENERATE_PYTHON_STUBS)

  INSTALL(
    TARGETS ${PYWRAP}
    EXPORT ${TARGETS_EXPORT_NAME}
    DESTINATION ${${PYWRAP}_INSTALL_DIR}
    )

  # --- INSTALL SCRIPTS
  SET(PYTHON_FILES
    __init__.py
    deprecated.py
    deprecation.py
    utils.py
    robot_wrapper.py
    romeo_wrapper.py
    explog.py
    shortcuts.py
    windows_dll_manager.py
    )

  FOREACH(python ${PYTHON_FILES})
    PYTHON_BUILD(${PROJECT_NAME} ${python})
    INSTALL(FILES
      "${${PROJECT_NAME}_SOURCE_DIR}/bindings/python/pinocchio/${python}"
      DESTINATION ${${PYWRAP}_INSTALL_DIR})
  ENDFOREACH(python)

  # --- INSTALL VISUALIZATION SCRIPTS
  SET(PYTHON_VISUALIZE_FILES
    __init__.py
    base_visualizer.py
    gepetto_visualizer.py
    meshcat_visualizer.py
    panda3d_visualizer.py
    rviz_visualizer.py
    )

  FOREACH(python ${PYTHON_VISUALIZE_FILES})
    PYTHON_BUILD(${PROJECT_NAME}/visualize ${python})
    INSTALL(FILES
      "${${PROJECT_NAME}_SOURCE_DIR}/bindings/python/pinocchio/visualize/${python}"
      DESTINATION ${${PYWRAP}_INSTALL_DIR}/visualize)
  ENDFOREACH(python)

  # --- PACKAGING --- #

  # Format string
  SET(_PKG_CONFIG_PYWRAP_LIBDIR ${${PYWRAP}_INSTALL_DIR})
  SET(_PKG_CONFIG_PYWRAP_BINDIR ${${PYWRAP}_INSTALL_DIR})
  SET(_PKG_CONFIG_PYWRAP_CONFLICTS)
  SET(_PKG_CONFIG_PYWRAP_REQUIRES "${PROJECT_NAME}")
  FOREACH(dep ${PKG_CONFIG_PYWRAP_REQUIRES})
    SET(_PKG_CONFIG_PYWRAP_REQUIRES "${_PKG_CONFIG_PYWRAP_REQUIRES}, ${dep}")
  ENDFOREACH(dep ${PKG_CONFIG_PYWRAP_REQUIRES})

  SET(_PKG_CONFIG_PYWRAP_LIBS "-L\${libdir} -l${PYWRAP}")
  IF(APPLE)
    SET(_PKG_CONFIG_PYWRAP_LIBS "${_PKG_CONFIG_PYWRAP_LIBS} -Wl,-undefined,dynamic_lookup,${Boost_${UPPERCOMPONENT}_LIBRARY}")
  ELSE(APPLE)
    SET(_PKG_CONFIG_PYWRAP_LIBS "${_PKG_CONFIG_PYWRAP_LIBS} ${LIBINCL_KW}boost_python")
  ENDIF(APPLE)

  SET(_PKG_CONFIG_PYWRAP_CFLAGS "-I\${includedir} -I/usr/local/include")
  SET(_PKG_CONFIG_PYWRAP_CFLAGS "${_PKG_CONFIG_PYWRAP_CFLAGS} -I${PYTHON_INCLUDE_DIRS}")
  FOREACH(cflags ${CFLAGS_DEPENDENCIES})
    SET(_PKG_CONFIG_PYWRAP_CFLAGS "${_PKG_CONFIG_PYWRAP_CFLAGS} ${cflags}")
  ENDFOREACH(cflags ${CFLAGS_DEPENDENCIES})

  CONFIGURE_FILE(
     "${CMAKE_CURRENT_SOURCE_DIR}/pinocchiopy.pc.cmake"
     "${CMAKE_CURRENT_BINARY_DIR}/pinocchiopy.pc")

  INSTALL(
      FILES "${CMAKE_CURRENT_BINARY_DIR}/pinocchiopy.pc"
      DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
      PERMISSIONS OWNER_READ GROUP_READ WORLD_READ OWNER_WRITE)

  IF(DOXYGEN_FOUND AND DOXYGEN_VERSION VERSION_GREATER 1.8.17)
    SET(DOXYGEN_GENERATE_HTML YES)
    SET(DOXYGEN_GENERATE_LATEX NO)
    SET(DOXYGEN_PROJECT_NAME "Pinocchio PyBind11 helpers.")
    SET(_source_headers_root "../../include/${PROJECT_NAME}/bindings/python")
    DOXYGEN_ADD_DOCS(doc_pybind11
      ${_source_headers_root}/pybind11.hpp
      ${_source_headers_root}/pybind11-all.hpp
      USE_STAMP_FILE
      COMMENT "Generating documentation of the PyBind11 helpers.")
  ENDIF()
ENDIF(BUILD_PYTHON_INTERFACE)
