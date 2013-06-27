###############################################################################
#                                                                             #
#  Template for CMakeList.txt                                                 #
#                                                                             #
#  * Author: Chaoya Li <harry75369@gmail.com>                                 #
#  * Update date: Apr, 2013                                                   #
#  * Reference: http://www.itk.org/Wiki/CMakeMacroParseArguments              #
#                                                                             #
###############################################################################
CMAKE_MINIMUM_REQUIRED(VERSION 2.8)

#####################################################################
#                                                                   #
#                         Macro Definitions                         #
#                                                                   #
#####################################################################
MACRO(CAR var)
  SET(${var} ${ARGV1})
ENDMACRO(CAR)

MACRO(CDR var junk)
  SET(${var} ${ARGN})
ENDMACRO(CDR)

MACRO(PARSE_ARGUMENTS prefix arg_names option_names)
  SET(DEFAULT_ARGS)
  FOREACH(arg_name ${arg_names})
    SET(${prefix}_${arg_name})
  ENDFOREACH(arg_name)
  FOREACH(option ${option_names})
    SET(${prefix}_${option} FALSE)
  ENDFOREACH(option)

  SET(current_arg_name DEFAULT_ARGS)
  SET(current_arg_list)
  FOREACH(arg ${ARGN})
    SET(larg_names ${arg_names})
    LIST(FIND larg_names "${arg}" is_arg_name)
    IF (is_arg_name GREATER -1)
      SET(${prefix}_${current_arg_name} ${current_arg_list})
      SET(current_arg_name ${arg})
      SET(current_arg_list)
    ELSE (is_arg_name GREATER -1)
      SET(loption_names ${option_names})
      LIST(FIND loption_names "${arg}" is_option)
      IF (is_option GREATER -1)
        SET(${prefix}_${arg} TRUE)
      ELSE (is_option GREATER -1)
        SET(current_arg_list ${current_arg_list} ${arg})
      ENDIF (is_option GREATER -1)
    ENDIF (is_arg_name GREATER -1)
  ENDFOREACH(arg)
  SET(${prefix}_${current_arg_name} ${current_arg_list})
ENDMACRO(PARSE_ARGUMENTS)

MACRO(CONFIG_EXE EXENAME LIBNAME)
  INCLUDE_DIRECTORIES( ${${LIBNAME}_INCLUDE_DIR} )
  TARGET_LINK_LIBRARIES( ${EXENAME} ${${LIBNAME}_LIBRARY} )
ENDMACRO(CONFIG_EXE)

MACRO(ADD_EXE)
  SET(lists   SOURCECODE_LIST LIBRARY_LIST
              MOC_LIST UIC_LIST RCC_LIST)
  SET(options USE_MPI USE_CUDA USE_BOOST)
  PARSE_ARGUMENTS(EXE "${lists}" "${options}" ${ARGN})
  CAR(EXE_NAME ${EXE_DEFAULT_ARGS})

  IF (EXE_MOC_LIST)
    MESSAGE(STATUS "EXE_MOC_LIST = ${EXE_MOC_LIST}")
    MESSAGE(STATUS "MOC_FILES = ${MOC_FILES}")
    FOREACH (moc ${EXE_MOC_LIST})
      MESSAGE(STATUS "moc = ${moc}")
      STRING(REGEX REPLACE "([^.]*).([^.]*)\$"
        "${CMAKE_CURRENT_BINARY_DIR}/moc_\\1.cxx" moc_file ${moc})
      MESSAGE(STATUS "moc_file = ${moc_file}")
      LIST(FIND "${MOC_FILES}" "${moc_file}" moc_found)
      MESSAGE(STATUS "moc_found = ${moc_found}")
      IF (moc_found)
        SET(EXE_SOURCECODE_LIST ${EXE_SOURCECODE_LIST} ${moc_file})
      ENDIF (moc_found)
    ENDFOREACH (moc)
    MESSAGE(STATUS "${EXE_SOURCECODE_LIST}")
  ENDIF (EXE_MOC_LIST)

  IF (EXE_UIC_LIST)
    INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR})
    FOREACH (uic ${EXE_UIC_LIST})
      STRING(REGEX REPLACE "([^.]*).([^.]*)\$"
        "${CMAKE_CURRENT_BINARY_DIR}/ui_\\1.h" uic_file ${uic})
      LIST(FIND "${UIC_FILES}" "${uic_file}" uic_found)
      IF (uic_found)
        SET(EXE_SOURCECODE_LIST ${EXE_SOURCECODE_LIST} ${uic_file})
      ENDIF (uic_found)
    ENDFOREACH (uic)
  ENDIF (EXE_UIC_LIST)

  IF (EXE_RCC_LIST)
    FOREACH (rcc ${EXE_RCC_LIST})
      STRING(REGEX REPLACE "([^.]*).([^.]*)\$"
        "${CMAKE_CURRENT_BINARY_DIR}/qrc_\\1.cxx" rcc_file ${rcc})
      LIST(FIND "${RCC_FILES}" "${rcc_file}" rcc_found)
      IF (rcc_found)
        SET(EXE_SOURCECODE_LIST ${EXE_SOURCECODE_LIST} ${rcc_file})
      ENDIF (rcc_found)
    ENDFOREACH (rcc)
  ENDIF (EXE_RCC_LIST)

  IF (EXE_USE_CUDA)
    IF (EXE_USE_BOOST)
      MESSAGE(STATUS "Direct use of Boost in CUDA is not supported. While "
      "indirect use is OK by linking a Boost library if configured properly")
    ENDIF (EXE_USE_BOOST)
    SET(EXE_CUDA_LIST)
    SET(EXE_LIBS_LIST)
    FOREACH(CODE ${EXE_SOURCECODE_LIST})
      IF (CODE MATCHES ".cu$")
        SET(EXE_CUDA_LIST ${EXE_CUDA_LIST} ${CODE})
      ELSE ()
        SET(EXE_LIBS_LIST ${EXE_LIBS_LIST} ${CODE})
      ENDIF ()
    ENDFOREACH ()
    IF (NOT EXE_CUDA_LIST)
      MESSAGE(FATAL_ERROR "No CUDA source code found!")
    ENDIF ()
    CUDA_ADD_EXECUTABLE(${EXE_NAME} ${EXE_CUDA_LIST})
    IF (EXE_LIBS_LIST)
      ADD_LIBRARY(${EXE_NAME}_libs ${EXE_LIBS_LIST})
      TARGET_LINK_LIBRARIES(${EXE_NAME} ${EXE_NAME}_libs)
    ENDIF ()
  ELSE (EXE_USE_CUDA)
    ADD_EXECUTABLE(${EXE_NAME} ${EXE_SOURCECODE_LIST})
  ENDIF (EXE_USE_CUDA)

  FOREACH(LIBRARY ${EXE_LIBRARY_LIST})
    CONFIG_EXE(${EXE_NAME} ${LIBRARY})
  ENDFOREACH(LIBRARY)

  IF (EXE_USE_BOOST)
    MESSAGE(STATUS "Configuring Boost for ${EXE_NAME}...")
    TARGET_LINK_LIBRARIES(${EXE_NAME} ${Boost_LIBRARIES})
  ENDIF (EXE_USE_BOOST)

  IF (EXE_USE_MPI)
    MESSAGE(STATUS "Configuring MPI for ${EXE_NAME}...")
    INCLUDE_DIRECTORIES(${MPI_INCLUDE_PATH})
    TARGET_LINK_LIBRARIES(${EXE_NAME} ${MPI_LIBRARIES})
    SET_TARGET_PROPERTIES(${EXE_NAME} PROPERTIES COMPILE_FLAGS "${MPI_COMPILE_FLAGS}")
    SET_TARGET_PROPERTIES(${EXE_NAME} PROPERTIES LINK_FLAGS "${MPI_LINK_FLAGS}")
  ENDIF (EXE_USE_MPI)
ENDMACRO(ADD_EXE)

#####################################################################
#                                                                   #
#                      Project Configurations                       #
#                                                                   #
#####################################################################
PROJECT(PortableMap)
#SET(${PROJECT_NAME}_VERSION_MAJOR 1)
#SET(${PROJECT_NAME}_VERSION_MINOR 0)
#CONFIGURE_FILE("${PROJECT_SOURCE_DIR}/Config.hpp.in"
               #"${PROJECT_SOURCE_DIR}/Config.hpp")
#SET(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)
#IF (WIN32)
  #SET(CMAKE_INCLUDE_PATH ${CMAKE_INCLUDE_PATH} "D:/libraries/eigen3")
  #SET(CMAKE_INCLUDE_PATH ${CMAKE_INCLUDE_PATH} "D:/libraries/boost_1_53_0")
  #IF (CMAKE_GENERATOR MATCHES "64")
    #SET(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "D:/libraries/boost_1_53_0/lib/win64")
	#SET(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "C:/Program\ Files/MPICH2/lib")
  #ELSE ()
    #SET(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "D:/libraries/boost_1_53_0/lib/win32")
	#SET(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "C:/Program\ Files\ (x86)/MPICH2/lib")
  #ENDIF ()
  #MESSAGE(STATUS "CMAKE_INCLUDE_PATH = ${CMAKE_INCLUDE_PATH}")
  #MESSAGE(STATUS "CMAKE_LIBRARY_PATH = ${CMAKE_LIBRARY_PATH}")
#ENDIF(WIN32)

#####################################################################
#                                                                   #
#                       Libray Configurations                       #
#                                                                   #
#####################################################################
# OpenSceneGraph
#FIND_PACKAGE(osg REQUIRED)
#FIND_PACKAGE(osgDB)
#FIND_PACKAGE(osgGA)
#FIND_PACKAGE(osgUtil)
#FIND_PACKAGE(osgViewer)
#FIND_PACKAGE(osgQt)
#FIND_PACKAGE(osgWidget)

# Maya
#FIND_PACKAGE(Maya REQUIRED)

# Qt
#FIND_PACKAGE(Qt4)
#FIND_PACKAGE(osgQt)
#FIND_PACKAGE(osgWidget)

# Bullet
#FIND_PACKAGE(Bullet)

# Cuda
#FIND_PACKAGE(CUDA)
#SET(CUDA_NVCC_FLAGS "-arch=sm_20")
#INCLUDE_DIRECTORIES(common/cuda)

# OpenMP
#FIND_PACKAGE(OpenMP)

# MPI
#FIND_PACKAGE(MPI)

# Boost
#IF (WIN32)
#  SET(Boost_USE_STATIC_LIBS        ON) # don't ask me why, we just use static libs under windows
#  SET(Boost_USE_MULTITHREADED      ON)
#  SET(Boost_USE_STATIC_RUNTIME    OFF)
#ENDIF (WIN32)
#FIND_PACKAGE(Boost REQUIRED COMPONENTS regex filesystem system random)
#MESSAGE(STATUS "Boost_INCLUDE_DIR = ${Boost_INCLUDE_DIR}")
#INCLUDE_DIRECTORIES(${Boost_INCLUDE_DIR})

# Eigen
#FIND_PACKAGE(Eigen3 REQUIRED)
#MESSAGE(STATUS "EIGEN3_INCLUDE_DIR = ${EIGEN3_INCLUDE_DIR}")
#INCLUDE_DIRECTORIES(${EIGEN3_INCLUDE_DIR})

# ZeroMQ
#FIND_PACKAGE(ZeroMQ)
#IF (ZEROMQ_FOUND)
#  MESSAGE(STATUS "Found ZeroMQ include = ${ZEROMQ_INCLUDE_DIR}")
#  MESSAGE(STATUS "Found ZeroMQ library = ${ZEROMQ_LIBRARY}")
#ENDIF ()

# Json-cpp
#INCLUDE_DIRECTORIES("${PROJECT_SOURCE_DIR}/common/jsoncpp/include/")
#ADD_LIBRARY(jsoncpp STATIC
#  "${PROJECT_SOURCE_DIR}/common/jsoncpp/src/lib_json/json_reader.cpp"
#  "${PROJECT_SOURCE_DIR}/common/jsoncpp/src/lib_json/json_writer.cpp"
#  "${PROJECT_SOURCE_DIR}/common/jsoncpp/src/lib_json/json_value.cpp"
#)
#SET(JSONCPP_INCLUDE_DIR "${PROJECT_SOURCE_DIR}/common/jsoncpp/include/")
#SET(JSONCPP_LIBRARY jsoncpp)

#####################################################################
#                                                                   #
#                        Precompile Qt files                        #
#                                                                   #
#####################################################################
SET(MOC_HEADER_LIST)
SET(UIC_FORM_LIST)
SET(RCC_RESOURCE_LIST)
SET(MOC_FILES)
SET(UIC_FILES)
SET(RCC_FILES)
IF (MOC_HEADER_LIST)
  MESSAGE(STATUS "Using Qt moc to process ${MOC_HEADER_LIST}")
  QT4_WRAP_CPP(MOC_FILES ${MOC_HEADER_LIST})
ENDIF (MOC_HEADER_LIST)
IF (UIC_FORM_LIST)
  MESSAGE(STATUS "Using Qt uic to process ${UIC_FORM_LIST}")
  INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR})
  QT4_WRAP_UI(UIC_FILES ${UIC_FORM_LIST})
ENDIF (UIC_FORM_LIST)
IF (RCC_RESOURCE_LIST)
  MESSAGE(STATUS "Using Qt rcc to process ${RCC_RESOURCE_LIST}")
  QT4_ADD_RESOURCES(RCC_FILES "${RCC_RESOURCE_LIST}")
ENDIF (RCC_RESOURCE_LIST)

#####################################################################
#                                                                   #
#                        Project executables                        #
#                                                                   #
#####################################################################


#######################################
# portable map test
ADD_EXE(testPortableMap
  SOURCECODE_LIST test/testPortableMap.cc
)

