---
title: "KubeCon 2018: Looking Back and Forward"
date: 2018-05-03T22:23:23+01:00
draft: true
---

And so KubeCon 2018 begins. The biggest gathering of the community so far, and the rise of Kubernetes continues. In the course of my career I’ve now seen three major open source projects reach dizzying heights that we in the community would have considered to be impossible for many years before: Linux now runs the world; OpenStack built the largest collaborative software project the world had ever seen; and now Kubernetes looks to be on a similar trajectory. When an open source project gets this kind of traction, the only thing most can do is jump onboard or get out of the way, as the scale of the community turns the wheels even faster.

![image 1](/images/kubecon1.jpg)

It’s clear as with the rise of OpenStack in the previous 5 or 6 years that there is still an irresistible pull toward a standard API for application infrastructure and deployment, and there remain many similarities, despite Kubernetes having moved the focus up the stack. To many veterans of the virtualization space, it sometimes feels like we’re trying to solve the same problems we solved there, with storage and network automation continuing to be headaches for many enterprises and users. This is reflected in the KubeCon schedule, with an entire track on service meshes, although this does beg the question of how much service mesh one project needs!

From a commercial perspective, the vendor landscape looks a little thinner than at peak OpenStack, which is reflected in the KubeCon exhibition hall. Although many of the same big names remain, the spend level also looks smaller than during the OpenStack glory days. This is likely a result of the failure of many to unlock significant revenue from that particular revolution, despite huge market penetration and commercial focus, and it will be interesting to see if anyone other than the biggest players can achieve that this time around.

Operating the underlying Kubernetes system is again a major challenge, with high availability, upgrades and elasticity at the top of many minds. Outside of the public cloud platforms, lifecycle orchestration is the only real way to achieve this, and many in Europe feel they can only comply with data privacy regulations with on-premise deployment. The burden of Kubernetes operation is alleviated through Mesos’ non-monolithic scheduling model, and DC/OS streamlines Kubernetes operations in exactly the same way it does for the many other distributed systems we support.

![image 2](/images/kubecon2.jpg)

As Kubernetes has become a juggernaut, the cloud native landscape has exploded, as evidenced by the increasingly unreadable infographics from the CNCF. There’s a risk, as with the OpenStack Big Tent approach, that development resources get diluted across too many projects, ending up with a wide array of projects where many remain unsuitable for production. Dan Kohn’s keynote explored some of the challenges around this space, emphasising the rapidly growing code base and the need for focusing on test and CI processes, in order to continue to deliver high quality code across the growing project base. Kelsey Hightower followed up on this topic, with tongue firmly in cheek, proposing his new no-code movement as a potential solution to the problem.

Time will also tell if Kubernetes remains the standard, given the inevitable rise of serverless architectures already starting to take hold, and many in the Kubernetes community are already predicting a move towards transient workloads becoming the majority, complicating further the requirements for cluster management. The serverless track on Friday will explore this in more depth, looking at both Function As A Service on top of k8s, and building lower level primitives to support more transient work.

With those thoughts in mind, I’m looking forward to digging deeper into the work going on across the community this week, and to discussions with friends, colleagues and users. The Kubernetes train is still gathering speed, and it promises to be an interesting ride!
