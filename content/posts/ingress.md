---
title: "Kubernetes Ingress Controllers on DC/OS"
date: 2018-07-13T23:02:34+01:00
draft: false
---

Kubernetes handles East-West connectivity for services internally within our Kubernetes cluster, by assigning an cluster-internal IP which can be reached by all services within the cluster. When it comes to external, or North-South, connectivity, there are a number of different methods we can use to achieve this. 

Firstly, we can define a NodePort Service type. This exposes the service on a static port at each node’s IP. Any traffic sent to that port on a node is forwarded to the relevant service. 

For platforms which support it, such as AWS or GCP, we can define a LoadBalancer service, which will configure an external load balancer specifically for our service. Whilst this works well on cloud platforms, it can also be very expensive, as we may end up with many load balancer services. 

The most flexible, although often the most confusing for new users, is the Ingress Controller. An Ingress Controller can sit in front of many services within our cluster, routing traffic to them and depending on the implementation, can also add functionality like SSL termination, path rewrites or name based virtual hosts. From an architecture perspective, ingress controllers are generally an automation layer integrated with a backend proxy service, and can generally operate at both layer 4 and layer 7. 

There is a growing ecosystem of ingress controllers, some leveraging well known load balancers and proxies, and some new cloud native implementations. There are ingress controllers for most of the familiar tools in this space, like HAProxy and NGinx, alongside new Kubernetes native implementations like [Ambassador] ( https://www.getambassador.io/ ) and [Contour] ( https://github.com/heptio/contour ), both of which leverage the [Envoy] ( https://www.envoyproxy.io/ ) proxy. There are also implementations to use other cloud native proxy tools like [Traefik] ( https://traefik.io/ ) and [Kong] ( https://getkong.org/ ), along with controllers which integrate with hardware load balancers such as those from F5.

The ingress type is relatively new, and the space is developing very rapidly, so for the purposes of this blog we’re going to look at one of the most mature implementations, which is the NGinx ingress controller. It’s worth noting there are a couple of different implementations of an Nginx ingress controller, with the two most noteworthy being Nginx’s own implementation, and the Kubernetes community implementation. You can find the differences between them [here] ( “https://github.com/nginxinc/kubernetes-ingress/blob/master/docs/nginx-ingress-controllers.md “ )

We’re going to be using the community implementation, which you can find the code for [here] ( “https://github.com/kubernetes/ingress-nginx” )

The first step we’re going to need to do is to deploy a default backend service. This is used by the Nginx ingress controller in the event that services are unavailable. The default backend image needs to satisfy two requirements :

* Serve a 404 page at /
* Respond with a 200 for requests to /healthz

The ingress-nginx project provides us with a [template yaml] ( https://github.com/kubernetes/ingress-nginx/blob/master/deploy/default-backend.yaml ) to use for this purpose, and the only change we’ve made from the defaults is to remove the namespace. 

```
---

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: default-http-backend
  labels:
    app: default-http-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: default-http-backend
  template:
    metadata:
      labels:
        app: default-http-backend
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: default-http-backend
        # Any image is permissible as long as:
        # 1. It serves a 404 page at /
        # 2. It serves 200 on a /healthz endpoint
        image: gcr.io/google_containers/defaultbackend:1.4
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
---

apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
  labels:
    app: default-http-backend
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: default-http-backend

```

As we can see, this is going to deploy a single replica of the defaultbackend container, which we are getting from gcr.io, and our service will route port 80 requests to port 8080 on the target. 

```
Mattbook-Pro:ingress matt$ kubectl create -f default-backend.yaml 
deployment "default-http-backend" created
service "default-http-backend" created
```

Now we have our default backend, we can go ahead and deploy our nginx ingress controller. This will deploy both Nginx and the additional code for Kubernetes to control it. In the interests of clarity, we’re going to deploy a very simple version of this configuration, it’s worth reading the [docs] ( “https://kubernetes.github.io/ingress-nginx/” ) to understand the full scope of the functionality.

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata: 
  name: nginx-ingress-controller
spec: 
  replicas: 1
  revisionHistoryLimit: 3
  template: 
    metadata: 
      labels: 
        k8s-app: ingress-nginx
    spec: 
      containers: 
        - args: 
            - /nginx-ingress-controller
            - "--default-backend-service=$(POD_NAMESPACE)/default-http-backend"
          env: 
            - name: POD_NAME
              valueFrom: 
                fieldRef: 
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom: 
                fieldRef: 
                  fieldPath: metadata.namespace
          image: "quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.15.0"
          imagePullPolicy: Always
          livenessProbe: 
            httpGet: 
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            timeoutSeconds: 5
          name: nginx-ingress-controller
          ports: 
            - containerPort: 80
              hostPort: 80
              name: http
              protocol: TCP
      terminationGracePeriodSeconds: 60
      nodeSelector:
          kubernetes.dcos.io/node-type: public
      tolerations:
        - key: "node-type.kubernetes.dcos.io/public"
          operator: "Exists"
          effect: "NoSchedule"
---
```
```
Mattbook-Pro:ingress matt$ kubectl create -f nginx-controller-simple.yaml 
deployment "nginx-ingress-controller" created
```

There are a few things worth noting in this configuration, which are specific to deploying this in DC/OS. Firstly, we need to ensure that this is deployed to our public agents, which is defined by :

```
  nodeSelector:
          kubernetes.dcos.io/node-type: public
```

We have also defined some tolerations, which define the behaviour we want to ensure if that condition can’t be met. In this case, we are defining that unless we have a public node available, then we’re not going to schedule the deployment.

```
 tolerations:
        - key: "node-type.kubernetes.dcos.io/public"
          operator: "Exists"
          effect: "NoSchedule"
```

We’ve also bound the deployment container directly to port 80 on the host it is deployed onto :

```
          ports: 
            - containerPort: 80
              hostPort: 80
              name: http
              protocol: TCP
```

We could define a Service, and then have the ports allocated automatically by Kubernetes using a NodePort type, but DC/OS already exposes port 80 on public agents, so this simplifies our configuration. In a production environment, this may not be the best option, since we need to manually ensure that no other processes are using port 80 on any of our DC/OS public agents. This includes any scenario where you are running marathon-lb in it’s default configuration, since marathon-lb binds to port 80 on public agents and so will prevent this ingress configuration from working correctly.

If we wanted to use the NodePort type, we would remove the hostPort line from the above configuration, and then define a Service which would look something like :

```
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      protocol: TCP
  selector:
      k8s-app: ingress-nginx
```

In the NodePort configuration we would need to find the ports the ingress controller is actually using by running :

```
Mattbook-Pro:ingress matt$ kubectl describe svc ingress-nginx | grep NodePort
Type:			NodePort
NodePort:		http	31585/TCP
```

Once we have the port, we would need to ensure we can connect externally to those ports, by checking firewall rules and access controls. We would then connect to our endpoint using :

```
Mattbook-Pro:ingress matt$ curl 54.171.202.99:31585
```

Now back to our example implementation. Once our ingress controller is running, we can test connectivity to our public IP address. In order to find the public IP’s of your DC/OS cluster, you can refer to the relevant [documentation] ( https://docs.mesosphere.com/1.11/administering-clusters/locate-public-agent/ ) If you’re running in AWS as I am, you can also use the handy DC/OS AWS [CLI extension] ( https://github.com/dcos-labs/dcos-aws-cli ) for doing exactly this. I only have a single public agent in my cluster, so using the CLI extension only returns me one result. 

```
Mattbook-Pro:ingress matt$ dcos dcos-aws-cli publicIPs
54.171.202.99
```

Once we have our public IP, we can use curl to connect to port 80 on this host. 

```
Mattbook-Pro:ingress matt$ curl 54.171.202.99
default backend - 404
```

We can see from the response code that we’re hitting our default backend, which is exactly what the expected behaviour is since we don’t yet have any ingress configuration. 

Let’s go ahead and deploy a simple service we can use for our first test of the ingress controller. We’re going to use 

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  labels:
    app: hello-world
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: echo-server
        image: hashicorp/http-echo
        args:
        - -listen=:80
        - -text="Hello from Kubernetes!"
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  selector:
    app: hello-world
  ports:
    - port: 80
      targetPort: 80
```

Lets go ahead and deploy that :

```
Mattbook-Pro:ingress matt$ kubectl create -f helloworld.yaml
deployment "hello-world" created
service "hello-world" created
```

We now need to define an Ingress for the ingress controller to configure external access to our test application. 

```
Mattbook-Pro:ingress matt$ cat helloworld-ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hello-world-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: hello-world
          servicePort: 80
```

In this configuration, we use an annotation to define which ingress class Kubernetes should use, we define the backend we should use, in this case our hello-world service, and we define the host, path and port which the ingress should be used on. 

```
Mattbook-Pro:ingress matt$ kubectl create -f helloworld-ingress.yaml 
ingress "hello-world-ingress" created
```

In order to test this, we need to do some local configuration on our kubectl host to map api.example.com to the public IP of our DC/OS cluster. This will depend on your operating system, but on MacOS and Linux you will want to add an entry in your /etc/hosts file as below :

```
Mattbook-Pro:ingress matt$ cat /etc/hosts
##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1	localhost
255.255.255.255	broadcasthost
::1             localhost
54.171.202.99 api.example.com
```

With that entry in our /etc/hosts file, we can now resolve api.example.com to the public IP of our DC/OS cluster, so we can use curl once again to test out the ingress rule. 

```
Mattbook-Pro:ingress matt$ curl api.example.com
"Hello from Kubernetes!"
```

Success ! We see that our request has been routed to the correct service. We can also test that if we use the IP address instead of the hostname, then we hit the default backend with a 404 response, as expected.

```
Mattbook-Pro:ingress matt$ curl 54.171.202.99
default backend - 404
```

Let’s add a second service, which does something slightly different :

```
Mattbook-Pro:ingress matt$ cat holamundo.yaml 
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hola-mundo
  labels:
    app: hola-mundo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hola-mundo
  template:
    metadata:
      labels:
        app: hola-mundo
    spec:
      containers:
      - name: echo-server
        image: hashicorp/http-echo
        args:
        - -listen=:80
        - -text="Hola de Kubernetes!"
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hola-mundo
spec:
  selector:
    app: hola-mundo
  ports:
    - port: 80
      targetPort: 80

Mattbook-Pro:ingress matt$ kubectl create -f holamundo.yaml 
deployment "hola-mundo" created
service "hola-mundo" created
```

And now let’s configure ingress to this on a different path to the same host :

```
Mattbook-Pro:ingress matt$ cat holamundo-ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: holamundo-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /hola
        backend:
          serviceName: hola-mundo
          servicePort: 80


Mattbook-Pro:ingress matt$ kubectl create -f holamundo-ingress.yaml 
ingress "holamundo-ingress" created
```

Now when we curl our endpoints, we can see that on the root we have our first service, and on the /hola endpoint we have our new service. 

```
Mattbook-Pro:ingress matt$ curl api.example.com
"Hello from Kubernetes!"
Mattbook-Pro:ingress matt$ curl api.example.com/hola
"Hola de Kubernetes!"
```

Let’s try something different, first let’s add another couple of host entries to our /etc/hosts file :

```
Mattbook-Pro:ingress matt$ cat /etc/hosts
##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1	localhost
255.255.255.255	broadcasthost
::1             localhost
54.171.202.99 api.example.com
54.171.202.99 es.example.com
54.171.202.99 gb.example.com
```

And now lets add some host based routing to our ingress configuration :

```
Mattbook-Pro:ingress matt$ cat hosts-ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hosts-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: gb.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: hello-world
          servicePort: 80
  - host: es.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: hola-mundo
          servicePort: 80
Mattbook-Pro:ingress matt$ kubectl create -f hosts-ingress.yaml 
ingress "hosts-ingress" created
```

With this configuration we basically have name based virtual hosts, where depending on the host requested we route the traffic differently. Here we are just using the root path, but of course, we can also combine this with paths for a ton of flexibility. 

```
Mattbook-Pro:ingress matt$ curl gb.example.com
"Hello from Kubernetes!"
Mattbook-Pro:ingress matt$ curl es.example.com
"Hola de Kubernetes!"
```

We can even do path rewrites in our ingress rules :

```
Mattbook-Pro:ingress matt$ cat rewrite.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: rewrite
  annotations:
    ingress.kubernetes.io/rewrite-target: /old
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: "api.example.com"
    http:
      paths:
      - path: /new
        backend:
          serviceName: hello-world
          servicePort: 80
```

Using the ingress.kubernetes.io/rewrite-target annotation, we can ensure that any requests to /old will be rewritten to direct to /new.

I’ve actually only covered a small subset of the functionality that the NGinx ingress controller can provide, including sticky sessions, TLS termination and proxy protocol, so it’s well worth digging into the docs for further reading. 

Hopefully we can now see that ingress controllers provide us with a powerful resource to control access to our applications running in Kubernetes on DC/OS. This is a space that’s moving very fast, so we’ll likely see more innovation around ingress over the coming months, and I’ll be trying to cover more of the options for ingress into DC/OS in future blog posts. 



