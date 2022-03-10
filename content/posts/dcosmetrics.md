---
title: "Getting custom metrics into the DC/OS metrics service"
date: 2018-09-06T23:02:34+01:00
draft: false
---

DC/OS ships with a comprehensive metrics service, providing metrics from DC/OS cluster hosts and from containers running on those hosts. These metrics are then made available via the DC/OS metrics API, allowing for easy integration with a wide range of monitoring solutions. It's also possible to add your own custom metrics from your applications to the metrics service, and in this blog I'll show you how to do exactly that from a Python application.

DC/OS metrics listens for [statsd](https://github.com/b/statsd_spec) metrics from every app running with the [Mesos containerizer](http://mesos.apache.org/documentation/latest/container-image/). This works by exposing a statsd server for each container, which allows us to tag all metrics by origin. The address of the statsd server is made available to the application by injecting the standard environment variables STATSD_UDP_HOST and STATSD_UDP_PORT into each container.

All the code in this blog is available in my GitHub repository at https://github.com/mattj-io/dcos_metrics

To try this out, I've written a simple API server in Python, using Flask ( http://flask.pocoo.org/ )

```
$ cat app.py
#!/usr/bin/env python
from flask import Flask, Response
from metrics import setup_metrics
app = Flask(__name__)
setup_metrics(app)
@app.route('/test/')
def test():
"""
Test API endpoint
"""
return 'My first REST API'
if __name__ == '__main__':
app.run()
```

In this code, we set up a Flask app, with one test endpoint on /test, which will simply return a text string.

In the same folder, I have a Python module called metrics.py, which is imported into our app, and then the app passes the Flask instance to the setup_metrics function from that module. Let's also take a look at that code :

```
$ cat metrics.py
import time
import sys
import os
from flask import request
import statsd
statsd_host = os.getenv('STATSD_UDP_HOST', 'localhost')
statsd_port = os.getenv('STATSD_UDP_PORT', 8125)
c = statsd.StatsClient(statsd_host, statsd_port, prefix='testapp')
def start_timer():
request.start_time = time.time()
def stop_timer(response):
resp_time = time.time() - request.start_time
sys.stderr.write("Response time: %ss\n" % resp_time)
c.timing('response.latency', float(resp_time))
return response
def setup_metrics(app):
app.before_request(start_timer)
app.after_request(stop_timer)
```

Here we're using the Python statsd module ( https://pypi.org/project/statsd/ ) to set up a statsd connection, using the environment variables which are provided by DC/OS, and setting a default prefix of ‘testapp'.

We then define a couple of functions - a function that sets a start time, and a function that sets an end time, calculates the total time between start and end, and pushes the metrics into a gauge called ‘response.latency' on the statsd server.

Finally we have a function setup_metrics, which uses Flask callbacks to call these methods on our app as it receives requests. At the start of a particular request to our API server, the start_timer method gets called, then after the request is completed, stop_timer is called.

In order to test this code, we're going to deploy it in a Docker container, which I've already prebuilt and pushed to Dockerhub. The Dockerfile is in the repository if you want to rebuilt that. To deploy it to DC/OS, we need some Marathon configuration in JSON :

```
$ cat api_server.json
{
  "id": "apiserver",
  "instances": 1,
  "cpus": 0.1,
  "mem": 16,
  "cmd": "./api_server.py",
  "portDefinitions": [
    {
      "protocol": "tcp",
      "port": 5000
    }
  ],
  "container": {
    "type": "MESOS",
    "docker": {
      "image": "mattjarvis/api_server",
      "forcePullImage": true,
      "privileged": false
    }
  },
  "requirePorts": true
}
```

So here we are going to deploy one instance of a task called apiserver, which is simply going to run our api_server python script and we're going to expose port 5000 externally. Now let's go ahead and deploy that to our DC/OS cluster :

```
$ dcos marathon app add apiserver.json
Created deployment ec216ba1-21f9-4556-8ff9-88a379b7d8e8
```

Now in order to generate some metrics, I also wrote a simple Python script which will poll a URL. This is also in the github repository, so let's take a quick look at the code :

```
$ cat browser.py
#!/usr/bin/env python
import time
import requests
from argparse import ArgumentParser
parser = ArgumentParser()
parser.add_argument("-u", "--url", dest="url",
required=True, help="API server URL")
parser.add_argument("-f", "--frequency", dest="freq",
default=5, help="Frequency to poll the API server")
args = parser.parse_args()
while True:
r = requests.get(args.url)
time.sleep(float(args.freq))
```

As we can see, this simply uses the python-requests library to get a particular URL in a loop with a configurable sleep time. Again, I've built this into a Docker image, and the Dockerfile is in the repository.

The Marathon configuration JSON we are going to use to deploy this is very simple, it just calls the browser.py script with the URL of our apiserver test API - we're using the Mesos DNS ( https://docs.mesosphere.com/1.11/networking/DNS/mesos-dns/ ) name for the apiserver process.

```
$ cat browser.json
{
  "id": "browser",
  "instances": 1,
  "cpus": 0.1,
  "mem": 16,
  "cmd": "./browser.py -u http://apiserver.marathon.mesos:5000/test/",
  "container": {
    "type": "MESOS",
    "docker": {
      "image": "mattjarvis/browser",
      "forcePullImage": true,
      "privileged": false
    }
  },
  "requirePorts": false
}
```

Let's go ahead and deploy that too :

```
$ dcos marathon app add browser.json
Created deployment 3b0cfeb7-cf09-43a3-8c09-e49e1c2d5fc2
```

Now both of our test elements are deployed, let's see if we have metrics in the DC/OS metrics API. First we'll need the task ID for our apiserver process :

```
$ dcos task
NAME HOST USER STATE ID MESOS ID REGION ZONE
apiserver 10.0.0.21 root R apiserver.5c4f6f44-9b18-11e8-a8d7-723f5859f1f0 e0323652-8918-449a-ae1d-e08cd6b2903c-S4 --- ---
browser 10.0.0.21 root R browser.e46bcb85-9b18-11e8-a8d7-723f5859f1f0 e0323652-8918-449a-ae1d-e08cd6b2903c-S4 --- ---
```

And then we'll use that task ID to query the metrics API. Remember from our apiserver code, we're looking for a metric called response.latency which will be prefixed with testapp, so here we can pipe the JSON output from the DC/OS CLI into grep to search for the correct section :

```
$ dcos task metrics details --json apiserver.5c4f6f44-9b18-11e8-a8d7-723f5859f1f0 | grep -A4 -B1 testapp.response.latency
{
  "name": "testapp.response.latency",
  "timestamp": "2018-08-08T14:44:34Z",
  "unit": "",
  "value": 0.09094
},
```

So there we have it, custom metrics into the DC/OS metrics API using Python ! You can learn more about the metrics API in the DC/OS docs ( https://docs.mesosphere.com/1.11/metrics/ ) and check out the source at https://github.com/dcos/dcos-metrics . The metrics API is under heavy development at the minute, with some major internal architecture changes likely in the DC/OS 1.12 release, so I'll revisit this topic at some point in the future to expand on this blog.
