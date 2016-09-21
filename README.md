# CMSSWToBigData

## About

This repository is a collection of examples which take c++ CMS software (CMSSW) framework [ROOT](https://root.cern.ch)-based event data format (`MiniAOD`) and converts it to the [Avro](https://avro.apache.org) data format.

## Overview

The following steps are required:
- Install the Avro-C package
- Link Avro-C to CMSSW
- Create Avro file!

## Installing the Avro C package

The build of Avro-C has been discussed in this ticket: https://issues.apache.org/jira/browse/AVRO-1844.

Here we copy the relevant fix needed to properly include Jansson.

Replace the following lines in CMakeLists.txt: 
```
#pkg_check_modules(JANSSON jansson>=2.3)
#if (JANSSON_FOUND)
#    set(JANSSON_PKG libjansson)
#    include_directories(${JANSSON_INCLUDE_DIR})
#    link_directories(${JANSSON_LIBRARY_DIRS})
#else (JANSSON_FOUND)
#    message(FATAL_ERROR "libjansson >=2.3 not found")
#endif (JANSSON_FOUND)
```
with:
```
# - Try to find Jansson
# Once done this will define
#
#  JANSSON_FOUND - system has Jansson
#  JANSSON_INCLUDE_DIRS - the Jansson include directory
#  JANSSON_LIBRARIES - Link these to use Jansson
#
#  Copyright (c) 2011 Lee Hambley <lee.hambley@gmail.com>
#
#  Redistribution and use is allowed according to the terms of the New
#  BSD license.
#  For details see the accompanying COPYING-CMAKE-SCRIPTS file.
#

if (JANSSON_LIBRARIES AND JANSSON_INCLUDE_DIRS)
  # in cache already
  set(JANSSON_FOUND TRUE)
else (JANSSON_LIBRARIES AND JANSSON_INCLUDE_DIRS)
  find_path(JANSSON_INCLUDE_DIR
    NAMES
      jansson.h
    PATHS
      /usr/include
      /usr/local/include
      /opt/local/include
      /sw/include
			${JANSSON_PATH}/include
  )

find_library(JANSSON_LIBRARY
    NAMES
      jansson
    PATHS
      /usr/lib
      /usr/local/lib
      /opt/local/lib
      /sw/lib
			${JANSSON_PATH}/lib
  )

set(JANSSON_INCLUDE_DIRS
  ${JANSSON_INCLUDE_DIR}
  )

if (JANSSON_LIBRARY)
  set(JANSSON_LIBRARIES
    ${JANSSON_LIBRARIES}
    ${JANSSON_LIBRARY}
    )
endif (JANSSON_LIBRARY)

  include(FindPackageHandleStandardArgs)
  find_package_handle_standard_args(Jansson DEFAULT_MSG
    JANSSON_LIBRARIES JANSSON_INCLUDE_DIRS)

  # show the JANSSON_INCLUDE_DIRS and JANSSON_LIBRARIES variables only in the advanced view
  mark_as_advanced(JANSSON_INCLUDE_DIRS JANSSON_LIBRARIES)

endif (JANSSON_LIBRARIES AND JANSSON_INCLUDE_DIRS)


    include_directories(${JANSSON_INCLUDE_DIR})
    link_directories(${JANSSON_LIBRARY_DIRS})
```
This allows the user to specify -DJANSSON_PATH as a command-line option to cmake, and this worked to build Avro.

