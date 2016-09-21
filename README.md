# CMSSWToBigData

## About

This repository is a collection of examples which take c++ CMS software (CMSSW) framework [ROOT](https://root.cern.ch)-based event data format (`MiniAOD`) and converts it to the [Avro](https://avro.apache.org) data format.

## Overview

The following steps are required:
- **Install the Avro-C package**
- **Link Avro-C to CMSSW**
- **Create Avro file!**

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

## Setting up CMSSW and linking the built Avro-C libraries

_n.b. This recipe is setup to work at cmslpc-sl6.fnal.gov_

First set up your CMSSW enviroment, in this case we work in CMSSW releast `8_0_19`:
```
cmsrel CMSSW_8_0_19
cd CMSSW_8_0_19/src
git clone git@github.com:nhanvtran/CMSSWToAvro.git Demo
```

Copy your Avro-C `include` and `lib` into `Demo/externalAvro`.  In this repository are libraries and headers which work at cmslpc-sl6.fnal.gov.
Next, copy the `avro.xml` file into your CMSSW configuration directory to properly link and tell CMSSW about it.
```
cp Demo/AvroProducer/externalAvro/avro.xml config/toolbox/$SCRAM_ARCH/tools/selected/.
scram setup avro
```

## Creating an Avro file

In this example, we take two jet collections from `MiniAOD` and write them in Avro format.  We choose this particular example because of the non-trivial (non-flat) structure that is sometimes desired in particle physics.  We write vectors of jet object records, which vary in size from event-to-event, for two different jet object collections which can contain different information.  

The steps are as follows:
- define schema using JSON format
- fill schema
- write out file

The entirety of the example is given in [AvroProducer.cc](https://github.com/nhanvtran/CMSSWToAvro/blob/master/AvroProducer/plugins/AvroProducer.cc).  Below, we walk through step-by-step this example.

### Defining schema

In this example, we define the schema directly in the source code. In the future, we plan to expand the examples to read in an external `JSON` file which can then be converted with native CMSSW `yaml` libraries or by directly reading in `yaml` files.  However, in this case, including the schema, in JSON format, directly into the source code makes the example as transparent as possible.




