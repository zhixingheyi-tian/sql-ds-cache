# Developer Guide

This document is a supplement to the whole [OAP Developer Guide](OAP-Developer-Guide.md) for SQL Index and Data Source Cache.
After following that document, you can continue more details for SQL Index and Data Source Cache.

* [Building](#Building)
* [Enabling NUMA binding for Intel® Optane™ DC Persistent Memory in Spark](#enabling-numa-binding-for-pmem-in-spark)

## Building

### Building SQL DS Cache 

Building with [Apache Maven\*](http://maven.apache.org/).

Before building, install PMem-Common locally:

```
git clone -b <tag-version> https://github.com/oap-project/pmem-common.git
cd pmem-common
mvn clean install -DskipTests
```

Build the SQL DS Cache package:

```
git clone -b <tag-version> https://github.com/oap-project/sql-ds-cache.git
cd sql-ds-cache
mvn clean -DskipTests package
```

### Running Tests

Run all the tests:
```
mvn clean test
```
Run a specific test suite, for example `OapDDLSuite`:
```
mvn -DwildcardSuites=org.apache.spark.sql.execution.datasources.oap.OapDDLSuite test
```
**NOTE**: Log level of unit tests currently default to ERROR, please override oap-cache/oap/src/test/resources/log4j.properties if needed.

### Building with Intel® Optane™ DC Persistent Memory Module

#### Prerequisites for building with PMem support

Install the required packages on the build system:

- [cmake](https://help.directadmin.com/item.php?id=494)
- [memkind](https://github.com/memkind/memkind/tree/v1.10.1)
- [vmemcache](https://github.com/pmem/vmemcache)
- [Plasma](http://arrow.apache.org/blog/2017/08/08/plasma-in-memory-object-store/)

####  memkind installation

The memkind library depends on `libnuma` at the runtime, so it must already exist in the worker node system. Build the latest memkind lib from source:

```
git clone -b v1.10.1 https://github.com/memkind/memkind
cd memkind
./autogen.sh
./configure
make
make install
``` 
#### vmemcache installation

To build vmemcache library from source, you can (for RPM-based linux as example):
```
git clone https://github.com/pmem/vmemcache
cd vmemcache
mkdir build
cd build
cmake .. -DCMAKE_INSTALL_PREFIX=/usr -DCPACK_GENERATOR=rpm
make package
sudo rpm -i libvmemcache*.rpm
```
#### Plasma installation

To use optimized Plasma cache with OAP, you need following components:  

   (1) `libarrow.so`, `libplasma.so`, `libplasma_java.so`: dynamic libraries, will be used in Plasma client.   
   (2) `plasma-store-server`: executable file, Plasma cache service.  
   (3) `arrow-plasma-3.0.0.jar`: will be used when compile oap and spark runtime also need it. 

- `.so` file and binary file  
  Clone code from Arrow repo and run following commands, this will install `libplasma.so`, `libarrow.so`, `libplasma_java.so` and `plasma-store-server` to your system path(`/usr/lib64` by default). And if you are using Spark in a cluster environment, you can copy these files to all nodes in your cluster if the OS or distribution are same, otherwise, you need compile it on each node.
  
```
cd /tmp
git clone https://github.com/oap-project/arrow.git
cd arrow && git checkout arrow-3.0.0-oap
cd cpp
mkdir release
cd release
#build libarrow, libplasma, libplasma_java
cmake -DCMAKE_INSTALL_PREFIX=/usr/ -DCMAKE_BUILD_TYPE=Release -DARROW_BUILD_TESTS=on -DARROW_PLASMA_JAVA_CLIENT=on -DARROW_PLASMA=on -DARROW_DEPENDENCY_SOURCE=BUNDLED  ..
make -j$(nproc)
sudo make install -j$(nproc)
```

- arrow-plasma-3.0.0.jar  
  Run following command, this will install arrow jars to your local maven repo. Besides, you need copy arrow-plasma-3.0.0.jar to `$SPARK_HOME/jars/` dir, cause this jar is needed when using external cache.
   
```
cd /tmp/arrow/java
mvn clean -q -pl plasma -am -DskipTests install
```


#### Building the package
You need to add `-Ppersistent-memory` to build with PMem support. For `noevict` cache strategy, you also need to build with `-Ppersistent-memory` parameter.
```
cd <path>/pmem-common
mvn clean install -Ppersistent-memory -DskipTests 
cd <path>/sql-ds-cache
mvn clean -DskipTests package
```

For vmemcache cache strategy, please build with command:

```
cd <path>/pmem-common
mvn clean install -Pvmemcache -DskipTests
cd <path>/sql-ds-cache
mvn clean -DskipTests package
```

Build with this command to use all of them:

```
cd <path>/pmem-common
mvn clean install -Ppersistent-memory -Pvmemcache -DskipTests
cd <path>/sql-ds-cache
mvn clean -DskipTests package
```

## Enabling NUMA binding for PMem in Spark

### Rebuilding Spark packages with NUMA binding patch 

When using PMem as a cache medium apply the [NUMA](https://www.kernel.org/doc/html/v4.18/vm/numa.html) binding patch [numa-binding-spark-3.0.0.patch](./numa-binding-spark-3.0.0.patch) to Spark source code for best performance.

1. Download src for [Spark-3.0.0](https://archive.apache.org/dist/spark/spark-3.0.0/spark-3.0.0.tgz) and clone the src from github.

2. Apply this patch and [rebuild](https://spark.apache.org/docs/latest/building-spark.html) the Spark package.

```
git apply  numa-binding-spark-3.0.0.patch
```

3. Add these configuration items to the Spark configuration file $SPARK_HOME/conf/spark-defaults.conf to enable NUMA binding.


```
spark.yarn.numa.enabled true 
```
**NOTE**: If you are using a customized Spark, you will need to manually resolve the conflicts.

###### \*Other names and brands may be claimed as the property of others.
