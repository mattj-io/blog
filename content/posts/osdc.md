---
title: "OSDC 2018"
date: 2018-06-24T22:49:48+01:00
draft: false
---

Earlier this month I attended the Open Source Data Center conference at the Mercure MOA in Berlin. [OSDC](https://osdc.de/) is an annual event, organised by [Netways]( https://www.netways.de/home/ ), a German IT services company and long term supporters of open source software. Netways is also a Mesos and Marathon user, having built an application delivery platform for their customers, which we’ll hopefully be covering in a future blog post. The theme of the conference is simplifying IT infrastructures with open source, and the aim is to present innovative strategies, forward looking developments, and new perspectives in dealing with large data centers.

### OSDC Addressing Rising Complexity

Overall the message from many of the speakers was the challenges of dealing with rising complexity in infrastructure, illustrated clearly by the opening keynote from Mitchell Hashimoto.

Unless you’ve been living under a stone for the last few years, Mitchell’s name will be very familiar as the creator of some of the most important workflow and automation tools of recent times including Vagrant, Packer, Terraform and Vault, as well as finding time to start his own company [Hashicorp] ( https://www.hashicorp.com/ ), where he’s the CEO. At Mesosphere we’re big fans of Hashicorp’s tools, using Terraform as the basis for the latest iteration of our installer, and Vault forms the backbone of the secrets service in DC/OS. Mitchell talked about the journey we’ve been on over the last two decades from single machines to the datacenter-as-a-computer, and how adding capabilities to our systems often means that we also add complexity. He also talked about how Terraform has evolved as a general platform for automation which should be viewed as being less about multi-cloud and more about multi-services, citing some of the many providers that now exist and are not directly about controlling infrastructure.

### Monitoring And Logging

One of the big challenges in operating distributed systems is always monitoring, and the topic was well represented at OSDC. Distributed systems produce a LOT of output across monitoring points, metrics and logs, and without systems that can make sense of this it’s easy to drown in the noise. Within the DC/OS user base, Prometheus is a very popular choice for monitoring, and DC/OS now has native Prometheus endpoints deployed in each agent, removing the requirement to install any additional agents. With Prometheus available as a package within the service catalog, you can also easily deploy Prometheus itself within your DC/OS cluster, and it will automatically configure collection from all your cluster endpoints.

Conrad Hoffman from Soundcloud talked about their journey in migrating their monitoring infrastructure almost entirely to Prometheus. As many will know, Prometheus started life at Soundcloud, and is now the most widely used cloud native monitoring system, under the governance of the CNCF. Soundcloud’s large on-prem estate had over 15 separate interlinked monitoring systems, causing major headaches for their infrastructure team in tracing issues. Through leveraging custom exporters for Prometheus, the team reduced this down significantly, improving their operational efficiency and workflows.

One of the other complex problems with monitoring in distributed systems is in tracing the root causes of issues, when a single request may go through multiple machines and services. Gianluca Arbezzano from [InfluxData] ( https://www.influxdata.com/ ) talked about this problem space, and how tracing is not really standardised yet. He then introduced the [OpenTracing project]( http://opentracing.io/ ), under the governance of the CNCF, and showed how this can be integrated into your applications to provide a standardised way of instrumenting distributed systems. Despite being a relatively new project, OpenTracing has a rapidly growing ecosystem and many real world implementations, and Gianluca described how InfluxData is using it within their own application stack.

### OSDC Configuration Management
It was also interesting to see the configuration management space represented by both Puppet and Salt. I’ve not been following the config management space very closely over the last couple of years, and to some extent the rise of immutable containers and orchestration systems has made many of the goals of config management redundant. Having said that, looking at some of the code presented, I was also reminded that I’ve looked at some Dockerfiles in the last couple of years which have definitely made me miss more structured declarative configuration languages.

Mike Place, project lead for [Salt] ( https://saltstack.com/ ) gave an update on the Salt project. Salt is probably the least well known of the configuration management projects, although it has a dedicated user base. I was particularly interested in Salt’s event driven automation, providing a common network bus to transmit events, and then being able to define actions to be triggered on those events.

Walter Gildersleeve from [Puppet] ( https://puppet.com/ ) introduced new features in the product,  and it was noteworthy that the company’s focus has diversified. I’ve written a lot of Puppet code in my working life, and used to run a Puppet shop, so have always had a soft spot for Puppet as a language and community. The rise of Ansible a few years ago had a big impact on the Puppet user base, due in part to it’s easier syntax, Python codebase, and better support for ad-hoc task execution. Walter introduced some new features and products within the Puppet ecosystem, although my impression was that they were more me-too than genuine innovation in the space. Puppet Discovery is an infrastructure discovery tool aimed at auto-discovery of hosts on your network, very similar to functionality which has existed in tools like Foreman for several years. Puppet Pipelines is a CI/CD tool in the same space as Spinnaker, and Puppet Tasks is an ad-hoc task execution framework. I hope Puppet can build on these new directions and stay relevant.

### The Influence Of Compute Technology On Application Architecture

![image 1](/images/osdc1.png)

On the second day I talked about how changes in compute technology have influenced application architecture, and how emerging patterns in cloud native applications are increasing operational complexity. I then introduced Mesos and DC/OS, describing how these kinds of cluster orchestration systems can help to reduce this complexity. I was pleased to see a number of Mesos and DC/OS users in the room, and had some good questions and discussion with folks in the hallway track around this topic. Many people expressed an interest in DC/OS as a platform for running multiple Kubernetes clusters, a pattern we see repeated wherever we engage with users and customers.

![image 2](/images/osdc2.png)

I also got the opportunity to learn about Apache Ignite ( https://ignite.apache.org/ ) , a memory-centric distributed database, caching, and processing platform, aimed at very high performance and scalability. Akmal Chaudhri talked about the many different use cases for this project, from speeding up traditional RDBMS, acting alone as a distributed ACID database in a similar vein to CockroachDB, through to new features for doing machine learning directly within the platform, alleviating the requiring for Extract, Transform and Load processes. Interesting stuff, although tools that seem to want to do everything somewhat conflict with my grounding in the Unix philosophy of doing one thing really well.

### OSDC Punches Above Its Weight

For a relatively small conference OSDC really was punching above its weight in terms of content and quality of speakers, with a really interesting range of technologies and use cases covered. I didn’t get to attend all the sessions in person as there were two tracks, but will definitely be catching up on the videos once they go online. Thanks to Netways and the OSDC team for inviting me to speak, and I’d love to come back in 2019.
