##############################################
# Requirements
find_package(PythonComplete QUIET)
find_package(Cython REQUIRED)





##############################################
# Definitions
set(cymod_src "${CMAKE_CURRENT_SOURCE_DIR}/mvdtool.pyx")
set(default_cybindings_cpp "${CMAKE_CURRENT_SOURCE_DIR}/mvdtool.cpp")
set(cybindings_cpp "${CMAKE_CURRENT_BINARY_DIR}/mvdtool.cpp")
set(local_include_dir "${CMAKE_CURRENT_SOURCE_DIR}/include")
set(cy_sources
  mvdtool.pyx
  include/boost.pxd
  include/MVD2.pxd
  include/MVD3.pxd
  include/MVD_MVDType.pxd
  include/MVD.pxd
  include/std.pxd)

##############################################
# Rules

# .pyx -> .cpp
add_custom_command(
    OUTPUT ${cybindings_cpp}
    COMMAND PYTHONPATH=${CMAKE_CURRENT_SOURCE_DIR} ${CYTHON_EXECUTABLE} ${CYTHON_FLAGS} --fast-fail --cplus -I${local_include_dir} --output-file ${cybindings_cpp} ${cymod_src}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    # No dependencies, except own sources
    DEPENDS ${cy_sources}
    COMMENT "Cythonizing ${cymod_src}"
)

# .cpp -> shared_lib
add_definitions("-std=c++11" "-Wfatal-errors" "-Wno-strict-aliasing")
python_add_module(mvdtool ${cybindings_cpp})
target_include_directories(mvdtool PRIVATE ${PYTHON_INCLUDE_DIRS} ${local_include_dir})
target_link_libraries(mvdtool MVDTool)


##############################################
# Python packaging

get_target_property(_include_dirs mvd-tool INCLUDE_DIRECTORIES)
set(_inc_str ${local_include_dir})
foreach(dir ${_include_dirs})
    set(_inc_str "${_inc_str}:${dir}")
endforeach()

# Wheel
add_custom_target(pydist
    COMMAND python setup.py build --build-base=${CMAKE_CURRENT_BINARY_DIR}  build_ext "-I${_inc_str}" bdist_wheel --dist-dir ${CMAKE_BINARY_DIR}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS ${cybindings_cpp}
    COMMENT "Building distributable Python wheel in ${CMAKE_BINARY_DIR}"
)

# Source dist - still requires include dirs
add_custom_target(pysrcdist
    COMMAND python setup.py build --build-base=${CMAKE_CURRENT_BINARY_DIR} sdist --dist-dir ${CMAKE_BINARY_DIR}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS ${cybindings_cpp}
    COMMENT "Building distributable Python wheel in ${CMAKE_BINARY_DIR}"
)


##############################################
# Install
#
install(TARGETS mvdtool
        LIBRARY DESTINATION ${CMAKE_INSTALL_FULL_PYTHON_PACKAGES} )


##############################################
# TESTS
#
add_test(NAME python_read_mvd2
         COMMAND bash -c "PYTHONPATH=${CMAKE_CURRENT_BINARY_DIR} python test/read_mvd2.py"
         WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)

add_test(NAME python_read_mvd3
         COMMAND bash -c "PYTHONPATH=${CMAKE_CURRENT_BINARY_DIR} python test/read_mvd3.py"
         WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)

