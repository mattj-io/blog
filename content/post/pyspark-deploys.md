---
title: "Deploying PySpark jobs using DC/OS"
date: 2017-10-11T19:22:46+01:00
draft: true
---

I've recently been working with PySpark, building a natural language processing pipeline demo for DC/OS. This has been a great learning experience, and PySpark provides an easier entry point into the world of Spark programming for a systems guy like myself than having to learn Java or Scala

When you're developing Spark jobs, testing locally is a very different environment from deploying to a cluster, so it's not always straightforward working out how to deploy to a system like DC/OS once you think you've got a working job. The Spark docs aren't particularly clear with respect to Python, especially where you've got dependent libraries involved - a very different case from the Java world of uploading a jar file. There's also seen to be a bunch of different approaches to doing this 

So let's look at the problem space. I have a PySpark job which needs a couple of external Python libraries, numpy and kafka, and also needs an additional Spark module, spark-sql-kafka. 

When I'm running it locally, I have the Python libraries installed, in this case in a virtualenv, and I supply the --package argument on the Spark to command line to add the spark-sql-kafka package at runtime. 

```
spark-submit --packages org.apache.spark:spark-sql-kafka-0-10_2.11:2.2.0 spark_kafka.py
```

When I move it to DC/OS, my first problem is how do I include my dependent Python libraries. Spark has a CLI argument --py-files which takes a comma separated list, but for even a small module this would be pretty unwieldy. If we look at the contents of our two dependent modules, we can see they are made up of a bunch of different files :

```
Mattbook-Pro:site-packages matt$ ls kafka*
kafka:
__init__.py     client_async.py     codec.py        conn.py         context.pyc     future.py       producer        structs.pyc     version.py
__init__.pyc        client_async.pyc    codec.pyc       conn.pyc        coordinator     future.pyc      protocol        util.py         version.pyc
client.py       cluster.py      common.py       consumer        errors.py       metrics         serializer      util.pyc
client.pyc      cluster.pyc     common.pyc      context.py      errors.pyc      partitioner     structs.py      vendor

kafka-1.3.5.dist-info:
DESCRIPTION.rst INSTALLER   METADATA    RECORD      WHEEL       metadata.json   top_level.txt
Mattbook-Pro:site-packages matt$ ls numpy*
numpy:
__config__.py       _distributor_init.py    _import_tools.py    compat          distutils       f2py            ma          polynomial      testing
__config__.pyc      _distributor_init.pyc   _import_tools.pyc   core            doc         fft         matlib.py       random          tests
__init__.py     _globals.py     add_newdocs.py      ctypeslib.py        dual.py         lib         matlib.pyc      setup.py        version.py
__init__.pyc        _globals.pyc        add_newdocs.pyc     ctypeslib.pyc       dual.pyc        linalg          matrixlib       setup.pyc       version.pyc

numpy-1.13.3.dist-info:
DESCRIPTION.rst INSTALLER   METADATA    RECORD      WHEEL       metadata.json   top_level.txt
```

So obviously we need a different approach. There appears to be some [support](https://issues.apache.org/jira/browse/SPARK-13587) for virtualenv's directly in Spark, but the only documentation I could find looked very specific to Yarn, and given my limited knowledge of Spark I didn't think I could make it work in the time I had available.

I then came across [this](https://developerzen.com/best-practices-writing-production-grade-pyspark-jobs-cb688ac4d20f) great post on best practices with PySpark, which although slightly too much for my small project, did give me a clue as to the direction I should be going in. Python can treat a zip file as a directory and import modules and functions from it just like any other directory. 

Now, with my --py-files argument, and a small code change to my Spark job, I can include the modules I want to and deploy them to my DC/OS cluster. 

So, let's start at the start and work through our development pipeline.

First, we'll create a virtualenv :

```
Mattbook-Pro:pyspark matt$ virtualenv venv
New python executable in /Users/matt/pyspark/venv/bin/python
Installing setuptools, pip, wheel...done.
```

Now we'll activate it :

```
Mattbook-Pro:pyspark matt$ source venv/bin/activate
(venv) Mattbook-Pro:pyspark matt$
```

Now let's deploy our required libraries into our new virtualenv :

```
(venv) Mattbook-Pro:pyspark matt$ pip install kafka
Collecting kafka
  Using cached kafka-1.3.5-py2.py3-none-any.whl
Installing collected packages: kafka
Successfully installed kafka-1.3.5
(venv) Mattbook-Pro:pyspark matt$ pip install numpy
Collecting numpy
  Using cached numpy-1.13.3-cp27-cp27m-macosx_10_6_intel.macosx_10_9_intel.macosx_10_9_x86_64.macosx_10_10_intel.macosx_10_10_x86_64.whl
Installing collected packages: numpy
Successfully installed numpy-1.13.3
```

Virtualenv has installed those libraries into venv/lib/python2.7/site-packages/ so let's find those packages and add them to a zip file. We don't need anything else out of our virtualenv, just the two deployed packages. 

```
(venv) Mattbook-Pro:pyspark matt$ cd venv/lib/python2.7/site-packages/
(venv) Mattbook-Pro:site-packages matt$ zip -r libs.zip numpy kafka
```

So if we supply our libs.zip along with our Spark job using the --py-files argument, the zip file will be deployed to the same directory as the Spark job itself. In order for our Spark job to include those libraries we need to make a small change to our PySpark code to get it to add that directory to its path :

```
import sys
import os
if os.path.exists('libs.zip'):
    sys.path.insert(0, 'libs.zip')
```

We make this change at the top of our Spark job, before any of the imports contained in libs.zip, basically saying if this zip file exists then add it to the path using sys.path.insert

The DC/OS Spark CLI extension supports the --py-files argument, so we can simply deploy our libs.zip somewhere accessible and include the URL in our DC/OS command line. In my case, I'm going to use Github to host both the job and the libs.zip, so the start of my command will be something like :

```
dcos spark run --submit-args="--py-files=https://raw.githubusercontent.com/mattj-io/spark_nlp/master/libs.zip https://raw.githubusercontent.com/mattj-io/spark_nlp/master/spark_kafka.py"
```

Note that any arguments for Spark directly precede the job on the DC/OS CLI, arguments after the job are interpreted as to be passed to the job itself.

So my next problem is that I need to include a Spark library, spark-sql-kafka, which I only have a Maven co-ordinate for. The DC/OS Spark CLI extension doesn't have the --packages switch, so how can I pass this into Spark ? Yes, I could directly go into the cluster, start running things manually, but my aim here is automation so I want to just be exercising the DC/OS CLI directly. 

After a lot of hair pulling and googling, I noticed the DC/OS CLI has a --submit-args switch to set Spark configuration values :

```
   --conf=PROP=VALUE ...     Custom Spark configuration properties.
```

This then led me to the Spark docs, and after trawling through the Spark configuration [reference](https://spark.apache.org/docs/latest/configuration.html), I found the configuration entry spark.jars.package :

```
Comma-separated list of Maven coordinates of jars to include on the driver and executor classpaths. The coordinates should be groupId:artifactId:version. If spark.jars.ivySettings is given artifacts will be resolved according to the configuration in the file, otherwise artifacts will be searched for in the local maven repo, then maven central and finally any additional remote repositories given by the command-line option --repositories. For more details, see Advanced Dependency Management.
```

This looked like it was what I needed, so now my DC/OS CLI string looked like :

```
dcos spark run --submit-args="--py-files=https://raw.githubusercontent.com/mattj-io/spark_nlp/master/libs.zip --conf=spark.jars.packages=org.apache.spark:spark-sql-kafka-0-10_2.11:2.2.0 https://raw.githubusercontent.com/mattj-io/spark_nlp/master/spark_kafka.py"
```

When I ran this, it actually worked ! Well, nearly. After a few minutes, my job died with the Spark logs complaining about no Spark Context. Heading back to the Spark docs, I realised that the default for driver memory allocation is only 1GB, so I uppped that to 2GB in my command line, and the next run worked perfectly !

My final CLI looked like this :

```
dcos spark run --submit-args="--driver-memory 2048M --py-files=https://raw.githubusercontent.com/mattj-io/spark_nlp/master/libs.zip --conf=spark.jars.packages=org.apache.spark:spark-sql-kafka-0-10_2.11:2.2.0 https://raw.githubusercontent.com/mattj-io/spark_nlp/master/spark_kafka.py"
```

So there we have it, a working method for deploying PySpark jobs to DC/OS, along with dependent libraries and additional Spark packages. 

