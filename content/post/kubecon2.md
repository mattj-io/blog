---
title: "Kubecon2"
date: 2018-05-04T22:30:57+01:00
draft: false
---

Yesterday’s theme of deja vu continues this morning at day 2 of KubeCon EU 2018. When you’ve been around in the open source world for a long time, it’s not uncommon for trends to rise, fall, and resurface later, and this period is no different.

Aparna Sinha, product manager from Google, gave an update on the Kubernetes project itself in the [first keynote](https://kccncna17.sched.com/event/CU7S/panel-kubernetes-cloud-native-and-the-public-cloud-b-moderated-by-dan-kohn-cloud-native-computing-foundation). Security was one of the big topics, and Aparna made the point that containers don’t contain very well, and many enterprises are still concerned about harder separation. For those of us who’ve been at the front line with customers in many different verticals including financial services and telco, this is no surprise and has been a continuous discussion since the initial rise of the container trend.

{{< youtube 2eAOx8E6-5Q >}}

This leads naturally back to a non-shared kernel, running on some kind of hypervisor, essentially back to virtual machines. Yesterday I caught Anne Bertucio from the OpenStack Foundation, and Samuel Ortiz from Intel, introducing [Kata containers](https://kccncna17.sched.com/event/CU81/kata-containers-hypervisor-based-container-runtime-xu-wang-hyperhq-samuel-ortiz-intel). Kata is a new project under the governance of the OpenStack Foundation that leverages underlying CPU hardware virtualization to provide a low overhead virtual environment designed for containers, with the fast boot times and low memory impact of containers along with the hard separation of virtual machines. The Kata project is fully compliant with runc and can be dropped into a Kubernetes environment. Google yesterday open sourced their own efforts in this space, with a microkernel based approach called [GVisor](https://github.com/google/gvisor), which runs in user space and provides a hypervisor layer for containers.

Aparna Sinha also introduced application operators in Kubernetes, identifying the need for complex applications to have an automated way for deploying and managing their lifecycle, and demo’ed the first release of the Spark operator, which has experimental support on Kubernetes in Spark 2.3. This concept is also central to the design of DC/OS, with two level scheduling at the heart of each of our frameworks.

Brandon Philips from CoreOS/RedHat then talked about the [challenges of maintaining catalogs](https://kccncna17.sched.com/event/CUFM/keynote-manage-the-app-on-kubernetes-brandon-philips-cto-coreos) of operators, and inserting them into clusters. We wholeheartedly agree, which is exactly why DC/OS has the [Service Catalog](https://mesosphere.com/service-catalog/) at the center of our project, enabling users to choose from over 100 frameworks via a single CLI command or a single click in the UI.

{{< youtube 8iQRJXJHiZ8 >}}

All this just goes to prove that what goes around comes around!
