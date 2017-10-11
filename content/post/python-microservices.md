+++
title = "Developing native microservices for Marathon in Python"
date = "2017-10-11T18:33:03+01:00"
description = ""
+++
Deploying microservices in Docker containers is a great way of ensuring immutability in the operating environment that your microservice needs, giving you exactly the dependencies and runtime versions that your application expects. However, as microservices get smaller, the overhead of running additional filesystems, libraries, binaries and runtimes just to run a few lines of code can seem like overkill. In many cases, Docker containers can end up very large, which somewhat defeats the point of containerisation in the first place.

Mesos supports its own native [containerizer](http://mesos.apache.org/documentation/latest/mesos-containerizer/), which in conjunction with [Marathon](https://mesosphere.github.io/marathon/), is capable of containerising individual binaries, sandboxed on the host filesystem, and for very small microservices this can be an ideal way to run things. Obviously there are limitations with this approach, particularly in terms of library dependencies, and typically statically compiled languages like Go are a good fit for this approach. 

Go is great, but many of us are more used to working in older languages like Python, so how might we fit Python applications into this paradigm ?

Typically people think of Python as an interpreted language, which requires libraries and an interpreter in the filesystem to run. Luckily the Python ecosystem contains a number of great tools which when used together allow us to take our Python code, and statically compile it into a binary which contains the Python interpreter itself, and all of our library dependencies. By using this toolchain we can end up with a binary which doesn't require Python or any libraries to be installed, and we can simply deploy our binary standalone to a native Mesos container using Marathon.

The first thing we need to do when developing in this way is to use [virtualenv](https://virtualenv.pypa.io/en/stable/). Virtualenv provides a mechanism for creating fully isolated Python environments, seperated from your system Python environment. 

The easiest way to install virtualenv is to use [pip](https://pip.pypa.io/en/stable/installing/)

```
pip install virtualenv
```

Then we can create our isolated environment to develop our microservice:

```
Mattbook-Pro:microdev matt$ virtualenv venv
New python executable in /Users/matt/microdev/venv/bin/python
Installing setuptools, pip, wheel...done.
```

The venv argument in the snippet above is arbitary, simply a name for the folder which will contain our virtualenv. In the venv directory we now have a folder layout containing the Python runtime, along with a set of folders for installing into. 

Before we can use the virtualenv, we need to activate it, which will set all of our paths to point to the installed virtualenv, and not to the system directories.

```
Mattbook-Pro:microdev matt$ source venv/bin/activate
(venv) Mattbook-Pro:microdev matt$ 
```

Once we've activated our environment, you can see our prompt changes to indicate we are activated. To deactivate it and return all of our paths back to the system defaults :

```
(venv) Mattbook-Pro:microdev matt$ deactivate
Mattbook-Pro:microdev matt$ 
```

Whilst creating, activating and deactivating are super simple, and the most common things you might do with virtualenv, it's also a lot more powerful than that. There are a ton of different options so it's well worth reading the docs. 

Now we are in our isolated environment we can install Python packages using pip, and they will be installed directly into our virtualenv and not into the system locations. In my case I have a simple Python app which downloads some JSON, and writes it into Kafka:

```
#!/usr/bin/env python
"""
Load data for demo
"""

import urllib2
import json
import ssl
from kafka import KafkaProducer

def main():
    """
    Load JSON from file into Kafka
    """
    kafka_topic = 'ingest'

    producer = KafkaProducer(value_serializer=lambda m: json.dumps(m).encode('utf-8'),
                             bootstrap_servers='localhost:9092')

    ctx = ssl.create_default_context()
    ctx.check_hostname = False
    ctx.verify_mode = ssl.CERT_NONE
    data_url = 'https://raw.githubusercontent.com/mattj-io/spark_nlp/master/stack.json'

    data_file = urllib2.urlopen(data_url, context=ctx)

    for line in data_file:
        producer.send(kafka_topic, json.loads(line))

if __name__ == "__main__":
    main()
```

Other than standard Python builtins, this script requires the python-kafka package, so we'll install that using pip.

```
(venv) Mattbook-Pro:microdev matt$ pip install kafka
Collecting kafka
  Using cached kafka-1.3.5-py2.py3-none-any.whl
Installing collected packages: kafka
Successfully installed kafka-1.3.5
(venv) Mattbook-Pro:microdev matt$ pip list
DEPRECATION: The default format will switch to columns in the future. You can use --format=(legacy|columns) (or define a format=(legacy|columns) in your pip.conf under the [list] section) to disable this warning.
kafka (1.3.5)
pip (9.0.1)
setuptools (36.5.0)
wheel (0.30.0)
(venv) Mattbook-Pro:microdev matt$ 
```

As we can see, other than the kafka package we just installed, we have a very basic list of pip packages installed - pip itself, setuptools and wheel - just enough to bootstrap any other packages we need. 

Now we've got our dependent libraries installed, our microservice will run quite happily inside this virtualenv, but we want to eventually run it in a native Mesos container without any of our dependencies or even a Python runtime which brings us to the next piece of our toolchain.

[PyInstaller](https://pyinstaller.readthedocs.io/en/stable/) bundles together a Python application and all of it's dependencies into a single binary package. It's important to remember the resulting executable is not cross-platform compatible, so you need to run this whole process on the platform that you intend to run the finished binary on. For the purposes of this blog post, I'm doing this on my Macbook. In reality for me to run this on my Mesos cluster, I need this to be a Linux binary, so I would probably do all of this in a Linux VM booted using Vagrant - the steps are identical, at least on Ubuntu. 

First let's use pip to install pyinstaller.

```
(venv) Mattbook-Pro:microdev matt$ pip install pyinstaller
Collecting pyinstaller
Collecting macholib>=1.8 (from pyinstaller)
  Using cached macholib-1.8-py2.py3-none-any.whl
Collecting pefile>=2017.8.1 (from pyinstaller)
Requirement already satisfied: setuptools in ./venv/lib/python2.7/site-packages (from pyinstaller)
Collecting dis3 (from pyinstaller)
  Using cached dis3-0.1.1-py2-none-any.whl
Collecting altgraph>=0.13 (from macholib>=1.8->pyinstaller)
  Using cached altgraph-0.14-py2.py3-none-any.whl
Collecting future (from pefile>=2017.8.1->pyinstaller)
Installing collected packages: altgraph, macholib, future, pefile, dis3, pyinstaller
Successfully installed altgraph-0.14 dis3-0.1.1 future-0.16.0 macholib-1.8 pefile-2017.9.3 pyinstaller-3.3
```

Now we have pyinstaller installed we can go ahead and build our self-contained binary. This is super simple, although as with virtualenv, pyinstaller has a ton of different options. I would recommend a thorough read of the docs.

```
(venv) Mattbook-Pro:microdev matt$ pyinstaller --onefile ingester.py 
167 INFO: PyInstaller: 3.3
168 INFO: Python: 2.7.10
178 INFO: Platform: Darwin-16.7.0-x86_64-i386-64bit
178 INFO: wrote /Users/matt/microdev/ingester.spec
184 INFO: UPX is not available.
185 INFO: Extending PYTHONPATH with paths
['/Users/matt/microdev', '/Users/matt/microdev']
185 INFO: checking Analysis
185 INFO: Building Analysis because out00-Analysis.toc is non existent
185 INFO: Initializing module dependency graph...
186 INFO: Initializing module graph hooks...
228 INFO: running Analysis out00-Analysis.toc
233 INFO: Caching module hooks...
236 INFO: Analyzing /Users/matt/microdev/ingester.py
2756 INFO: Processing pre-safe import module hook   _xmlplus
2950 INFO: Processing pre-find module path hook   distutils
2952 INFO: distutils: retargeting to non-venv dir '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/distutils'
3857 INFO: Loading module hooks...
3857 INFO: Loading module hook "hook-distutils.py"...
3861 INFO: Loading module hook "hook-xml.py"...
3922 INFO: Loading module hook "hook-httplib.py"...
3923 INFO: Loading module hook "hook-encodings.py"...
4440 INFO: Looking for ctypes DLLs
4442 INFO: Analyzing run-time hooks ...
4445 INFO: Including run-time hook 'pyi_rth_multiprocessing.py'
4453 INFO: Looking for dynamic libraries
4633 INFO: Looking for eggs
4633 INFO: Python library not in binary dependencies. Doing additional searching...
4635 INFO: Using Python library /Users/matt/microdev/venv/bin/../.Python
4639 INFO: Warnings written to /Users/matt/microdev/build/ingester/warningester.txt
4662 INFO: Graph cross-reference written to /Users/matt/microdev/build/ingester/xref-ingester.html
4765 INFO: checking PYZ
4765 INFO: Building PYZ because out00-PYZ.toc is non existent
4765 INFO: Building PYZ (ZlibArchive) /Users/matt/microdev/build/ingester/out00-PYZ.pyz
5131 INFO: Building PYZ (ZlibArchive) /Users/matt/microdev/build/ingester/out00-PYZ.pyz completed successfully.
5219 INFO: checking PKG
5219 INFO: Building PKG because out00-PKG.toc is non existent
5219 INFO: Building PKG (CArchive) out00-PKG.pkg
7563 INFO: Building PKG (CArchive) out00-PKG.pkg completed successfully.
7577 INFO: Bootloader /Users/matt/microdev/venv/lib/python2.7/site-packages/PyInstaller/bootloader/Darwin-64bit/run
7577 INFO: checking EXE
7577 INFO: Building EXE because out00-EXE.toc is non existent
7577 INFO: Building EXE from out00-EXE.toc
7577 INFO: Appending archive to EXE /Users/matt/microdev/dist/ingester
7587 INFO: Fixing EXE for code signing /Users/matt/microdev/dist/ingester
7591 INFO: Building EXE from out00-EXE.toc completed successfully.
```

Basically what is happening here, is that pyinstaller is analysing all the calls our Python code will make, gathering up all the required libraries, and then outputting a binary. In this case, it will create a folder called dist, and in that folder will be a binary with the name of the original script. 

```
(venv) Mattbook-Pro:microdev matt$ ls dist
ingester
(venv) Mattbook-Pro:microdev matt$ file dist/ingester 
dist/ingester: Mach-O 64-bit executable x86_64
```

I now have a statically compiled binary of my Python script, which I can deploy in a native Mesos container using Marathon. My Marathon job definition for this would be very simple, I just need to get the binary and run it - in my case, I'm going to host the binary on Github, so I can reference it in my job definition directly from there.


```
{
  "id": "ingester",
  "description": "ingest test data",
  "run": {
    "cpus": 0.1,
    "mem": 32,
    "cmd": "curl -s -L https://raw.githubusercontent.com/mattj-io/spark_nlp/master/ingester_linux -o ingester && chmod u+x ingester && ./ingester"
  }
}
```

So there we have it, it's not just shiny new languages like Go which can be used to build standalone microservices, we can also leverage Python with its rich ecosystem of supporting libraries. Who needs oversized Docker containers when we can simply deploy and run a single binary, properly isolated in its own container.  




