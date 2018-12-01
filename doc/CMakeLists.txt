find_package(Doxygen)

if(NOT DOXYGEN_FOUND)
  return()
endif()

# The Doxygen module in CMake 3.9+ includes a function to generate the
# Doxyfile for us. It uses CMake variables with a DOXYGEN_ prefix to
# set the configuration options in the Doxyfile.
#
# For compatibility with this in the future, we use variables with DOXYGEN_
# prefix when generating the Doxyfile with configure_file().

# Qt help generator (.qch)
find_program(qhg_executable qhelpgenerator)
if(qhg_executable)
  set(DOXYGEN_GENERATE_QHP YES)
  set(DOXYGEN_QHG_LOCATION ${qhg_executable})
endif()
unset(qhg_executable)

# HTML Help compiler (.chm)
find_package(HTMLHelp)
if(HTML_HELP_COMPILER)
  set(DOXYGEN_GENERATE_HTMLHELP YES)
  set(DOXYGEN_SEARCHENGINE NO)
  set(DOXYGEN_HHC_LOCATION ${HTML_HELP_COMPILER})
endif()

if(DOXYGEN_DOT_FOUND)
  set(DOXYGEN_HAVE_DOT YES)
endif()

configure_file(Doxyfile.in Doxyfile @ONLY)

add_custom_target(doc
  ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
  COMMENT "Generating API documentation with Doxygen"
  VERBATIM
)
