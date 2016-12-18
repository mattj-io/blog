+++
title = "Storage Myths"
description = "Time to change the way we think about storage - five myths we should put out to pasture"
date = "2015-01-21T10:56:37Z"

+++

This article was originally published by Tech Week Europe, the original article can be found [here](http://www.silicon.co.uk/data-storage/5-storage-myths-put-pasture-159933). As of the end of 2016, many of these concepts have already penetrated deeply into the enterprise space - object storage and distributed storage architectures are rapidly becoming the standard approaches. Storage class memory will fundamentally alter the storage space, and potentially the computing architectures which we've essentially followed for the last 50 years, and we're starting to see [mainstream](http://www.intel.co.uk/content/www/uk/en/architecture-and-technology/non-volatile-memory.html) products appear in that space. Revisiting this piece, I'm still most interested in the final two points, around how we interact with data and the potential for storage to become transformative pipelines. I expect to see both of these spaces continue to produce innovation over the next few years. 

-----------------

The biggest and most performant storage architecture technologies were developed by companies like Google, Amazon and Facebook, which grew out of university research projects in the 1990s. The business and operational drivers which initially led these hyper-scale web operators to build such architectures were, of course, very different to those of a typical enterprise. The sheer volume of data those high-growth internet businesses were dealing with at the time forced them to look outside the box in a way that smaller enterprises were not required to do.

Given the rapidly increasing data volumes all businesses are experiencing today, is it now time to rethink how we manage data across all other enterprise too?

##### Myth 1: We need to backup key data

###### New reality: Stop thinking about backups. Design systems are so redundant we don’t need to backup.

Above a certain size, it’s physically impossible to backup all that data within a typical backup window, so the concept of daily backups becomes impossible to achieve. Object storage architectures provide for distributed redundancy which, combined with clusters across geographic locations, handles the requirements for failure protection. Since object storage has no concept of file modification, only write and replace, the natural paradigm becomes versioning as opposed to keeping backup copies, and snapshotting into the cluster provides for rollback.

##### Myth 2: Disk failure is a problem we need to deal with.

###### New Reality: Expect failure – it’s inevitable and it doesn’t matter.

Typical enterprise architecture relies on RAID arrays to protect against drives failure and as drives get larger, RAID rebuild times get longer. When dealing with a large number of drives, this becomes unworkable as the average time to reboot means the system will be in an error or performance limited state for too long.

The bigger your infrastructure gets, the more you must expect failure. Self-healing systems face this reality head on. We can learn from Netflix here: its use of its ChaosMonkey application to test its infrastructure offers some insight into how we need to start rethinking storage. It randomly switches things off and breaks things to ensure that failure does not affect the user experience of the Netflix service. In well-designed distributed object stores, all of the components are redundant and, because the data is copied multiple times, a huge amount of the cluster can fail without any interruption to service or data loss.

##### Myth 3: True global namespaces are impossible to achieve

###### New Reality: Decoupling the namespace from the underlying hardware makes for far less management overhead and simpler capacity upgrades

Almost every enterprise is fighting with the storage silos problem. Proprietary vendors have offered many different solutions to this, including storage virtualisation or overlay management systems, but this is really just hiding the issue. With large scale object storage architectures, your data is presented as a global namespace wherever you view it. Multiple datacentres across the world are seamlessly integrated into a single view of your storage, and adding capacity is as simple as adding new servers into the cluster.

##### Myth 4: Users really need files.

###### New Reality: Files are largely irrelevant for most people.

Very little we do now as consumers involves us using files as we tend to interact through applications. We view photos on Facebook or Flickr, share workflows on Trello or Basecamp and watch videos on YouTube or Vimeo. Behind most of those applications is an object store, showing how, for consumers at least, the file metaphor is outdated.

Increasingly, enterprise users will adopt applications to interact with data. In the past we may have used spreadsheets to generate business information from files of raw data. Increasingly enterprises are using analytics applications to re-present their data in new and more visual ways, and a new breed of applications is emerging which automatically take spreadsheets and raw data and turn them into graphs and charts.
The file paradigm is increasingly an unnecessary intermediate step and is merely a legacy of how computer technology has evolved. As human beings we are far more naturally engaged with visual than numerical information – users in the future will interact with pre-processed information through analytic systems or other applications.

##### Myth 5: Storage is a bit like a bucket – a repository where I keep everything.

###### New Reality: Storage is more like a pipeline. A pipeline of transformation.

It is becoming increasingly apparent that there is a lot of unused processing power in a storage system and there is the potential to convert or process raw data while it sits in the storage system.

Object stores are ideally placed for doing this because they are built out of general purpose servers. This blurs the lines between storage and computation and presents a different paradigm: instead of thinking of our storage system as a bucket, we can think of it as a pipeline that data moves through in order for people to use it.

The open source object storage system, Ceph, is architected with this in mind and offers many hooks in the code for users to write custom plugins to execute on write or read. This kind of pre-processing is already happening in many industries, for example adding watermarks to images on ingest in the media industry, or first stage analysis of survey data in the oil and gas industries.

It’s time to change the way we think about storage.

The growth in data requirements is now outpacing the growth in drive capacity. The bigger drives get, the slower they become, and emerging technologies like Shingled Magnetic Recording, designed to keep drive sizes increasing, will continue that trend. Unless there is a technological breakthrough we are going to have to rethink the way we architect existing hardware.

Object storage is the most obvious and common sense answer: highly performant, self-healing, highly scalable and cheap. 
