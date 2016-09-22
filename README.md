# CMSSWToBigData

## About

This repository is an example which take c++ CMS software (CMSSW) framework [ROOT](https://root.cern.ch)-based event data format (`MiniAOD`) and converts it to the [Avro](https://avro.apache.org) data format.  This is part of a proof-of-principle workflow which converts `MiniAOD` to Avro and performs the analysis in Spark/Scala.

## Overview

The following steps are required:
- **Install the Avro-C package**
- **Link Avro-C to CMSSW**
- **Create Avro file!**

## Installing the Avro C package

The build of Avro-C has been discussed in this ticket: https://issues.apache.org/jira/browse/AVRO-1844.

First download the latest stable release from this [Apache page](http://www.apache.org/dyn/closer.cgi/avro/).  We used this [one](http://apache.mirrors.pair.com/avro/stable/c/avro-c-1.8.1.tar.gz).  Follow the build README.

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

More details and examples can be found at the [Avro-C documentation](https://avro.apache.org/docs/current/api/c/).

The steps are as follows:
- define schema using JSON format
- fill schema
- write out file

The entirety of the example is given in [AvroProducer.cc](https://github.com/nhanvtran/CMSSWToAvro/blob/master/AvroProducer/plugins/AvroProducer.cc).  Below, we walk through step-by-step this example.

### Defining a schema

In this example, we define the schema directly in the source code. In the future, we plan to expand the examples to read in an external `JSON` file which can then be converted with native CMSSW `yaml` libraries or by directly reading in `yaml` files.  However, in this case, including the schema, in JSON format, directly into the source code makes the example as transparent as possible.

First declare the schema object, the output file, and the event interface in the header:
```
avro_schema_t event_schema;
avro_file_writer_t db;
avro_value_iface_t *avroEventInterface;
avro_value_t avroEvent;
```     
The `avro_value_t` and `avro_value_iface_t` gives you record interface for a given event.

Then in the `beginJob()` method, define the schema:
```    
    const char EVENT_SCHEMA[] =
     "{\"type\": \"record\",\n \
     \"name\": \"Event\", \n \
     \"fields\": [ \n \
         {\"name\": \"ak4chsjets\", \n \
          \"type\": {\"type\": \"array\", \"items\": \n \
                   {\"type\": \"record\", \n \
                    \"name\": \"AK4CHSJet\", \n \
                    \"fields\": [ \n \
                        {\"name\": \"pt\", \"type\": \"double\"}, \n \
                        {\"name\": \"eta\", \"type\": \"double\"}, \n \
                        {\"name\": \"phi\", \"type\":\"double\"}]}}}, \n \
        {\"name\": \"ak4pupjets\", \n \
          \"type\": {\"type\": \"array\", \"items\": \n \
                   {\"type\": \"record\", \n \
                    \"name\": \"AK4PUPJet\", \n \
                    \"fields\": [ \n \
                        {\"name\": \"pt\", \"type\": \"double\"}, \n \
                        {\"name\": \"eta\", \"type\": \"double\"}, \n \
                        {\"name\": \"phi\", \"type\":\"double\"},  \n \
                        {\"name\": \"mass\", \"type\":\"double\"}]}}} \n \
                        ]}";

```
This schema has an event record with fields for arrays of jet records (or physics objects).  The jet objects then are their own records with the following fields `{pt,eta,phi}` and `{pt,eta,phi,mass}` for the `ak4chsjets` and `ak4pupjets` collections respectively.

Then, still in the `beginJob()` create the file and the schema interface and object:
```	
const char *dbname = "jets.avro";
rval = avro_file_writer_create_with_codec(dbname, event_schema, &db, "null", 0);

avroEventInterface = avro_generic_class_from_schema(event_schema);
avro_generic_value_new(avroEventInterface,&avroEvent);
```

### Filling the schema

The filling of the Avro schema is done by index of objects in the schema and therefore should be done with care.  This is the most complex part of the example.

Each field of the schema is represented by an `avro_value_t`.  Each jet collection is attached to the event by index using `avro_value_get_by_index`.  Then reset the jet collection array.  Usually this should be done for all `avro_value_t` but is necessary for arrays and maps.  Then, loop through the jet collection and fill the Avro-C records.  For an array, append each jet to the jet array.


```
avro_value_t JetsCHSAK4;
avro_value_t JetsPUPAK4;
avro_value_get_by_index(&avroEvent,0,&JetsCHSAK4,0);
avro_value_get_by_index(&avroEvent,1,&JetsPUPAK4,0);

avro_value_reset(&JetsCHSAK4);
for (unsigned int i = 0; i < patJetsCHSAK4->size(); ++i){
  avro_value_t JetCHSAK4;
  avro_value_append(&JetsCHSAK4,&JetCHSAK4,0);
  avro_value_t JetCHSAK4_pt;
  avro_value_t JetCHSAK4_eta;
  avro_value_t JetCHSAK4_phi;
  avro_value_get_by_index(&JetCHSAK4,0,&JetCHSAK4_pt,0);
  avro_value_get_by_index(&JetCHSAK4,1,&JetCHSAK4_eta,0);
  avro_value_get_by_index(&JetCHSAK4,2,&JetCHSAK4_phi,0);
  avro_value_set_double(&JetCHSAK4_pt,patJetsCHSAK4->at(i).pt());
  avro_value_set_double(&JetCHSAK4_eta,patJetsCHSAK4->at(i).eta());
  avro_value_set_double(&JetCHSAK4_phi,patJetsCHSAK4->at(i).phi());
}

avro_value_reset(&JetsPUPAK4);
for (unsigned int i = 0; i < patJetsPUPAK4->size(); ++i){
  avro_value_t JetPUPAK4;
  avro_value_append(&JetsPUPAK4,&JetPUPAK4,0);
  avro_value_t JetPUPAK4_pt;
  avro_value_t JetPUPAK4_eta;
  avro_value_t JetPUPAK4_phi;
  avro_value_t JetPUPAK4_m;
  avro_value_get_by_index(&JetPUPAK4,0,&JetPUPAK4_pt,0);
  avro_value_get_by_index(&JetPUPAK4,1,&JetPUPAK4_eta,0);
  avro_value_get_by_index(&JetPUPAK4,2,&JetPUPAK4_phi,0);
  avro_value_get_by_index(&JetPUPAK4,3,&JetPUPAK4_m,0);
  avro_value_set_double(&JetPUPAK4_pt,patJetsPUPAK4->at(i).pt());
  avro_value_set_double(&JetPUPAK4_eta,patJetsPUPAK4->at(i).eta());
  avro_value_set_double(&JetPUPAK4_phi,patJetsPUPAK4->at(i).phi());
  avro_value_set_double(&JetPUPAK4_m,patJetsPUPAK4->at(i).mass());
}  
```

At the end of the event, append the event record to the file.
```  
avro_file_writer_append_value(db,&avroEvent);
```

### Write out file

Finally write out the file:
```
avro_file_writer_close(db);
```





