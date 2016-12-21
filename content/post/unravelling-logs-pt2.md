+++
description = "Making sense of logs in distributed systems"
date = "2015-11-11T08:12:35Z"
title = "Unravelling Logs - Part 2"

+++

##### 2016 UPDATE

log-courier came into existence as a fork of the original [logstash-forwarder](https://github.com/elastic/logstash-forwarder), which had become fairly unmaintained, and didn't support some features the community needed. Elastic have now introduced [filebeat](https://www.elastic.co/products/beats/filebeat), which is probably the best way to do this now.

I also never did manage to complete pt3 of this blog set, which was supposed to cover using Riemann for automated log file analysis. At the time we seemed to be one of the few people using it, and it's a black art with config files written in [Clojure](https://clojure.org/). 

James Turnbull's [The Art of Monitoring](https://www.artofmonitoring.com/) has been released in the meantime which makes major use of Riemann, so I should really revisit this at some point.  

--------------------------------------------


In my first post in this series, I talked about the ELK stack of Elasticsearch, Logstash and Kibana and how they provide the first steps into automated logfile analysis. 

In this post, I’m going to deal with the first step in this process, how we get logs into this system in the first place. I’m not going to go into the process of installing logstash and elasticsearch, that’s pretty well covered elsewhere on the internet, so this post assumes you’ve already done that.

Logstash can accept logs in a number of different ways, one of which is plain old syslog format. The first thing to integrate and probably the easiest, is syslog which is where the majority of logging happens anyway. We started off doing this by configuring rsyslog in exactly the way we would to centralise logging to another rsyslog server.

In /etc/rsyslog.conf ( or in a separate config file in /etc/rsyslog.d/ depending on your distribution of choice ) you’d use one of the following :

```
# Provides UDP forwarding. The IP is the server's IP address
*.* @192.168.1.1:514

# Provides TCP forwarding.
*.* @@192.168.1.1:514
```

On the logstash server, we then need to define an input to handle syslog :

```
input {
  syslog {
    port => 514
    type => "native_syslog"
  }
}
```

The type entry is arbitary, this just adds a tag to any incoming entries on this input, but because the input itself is defined as a syslog input, logstash will use it’s built in filters for syslog in order to structure the data when it’s pushed into ElasticSearch.

The traditional method of doing this would be to send logs over UDP, which has less of a cpu overhead than TCP. But as the saying goes – "I’d tell you a joke about UDP but you might not get it" … 

Once we start to treat our log data as something we need to see in real time, as opposed to a historical record that may or may not be useful, then not receiving it is critical. We found that rsyslog would occasionally stop sending logs over the network when using UDP, and we had no way of automatically detecting if it was broken or not.

Switching to TCP guarantees delivery at the network layer, but had it’s own set of problems – we hit another situation where if rsyslog had a problem sending logs, it could also hang without logging locally. 

From an auditability and compliance perspective it’s critical that we continue to log locally, so architecturally we decided we needed to split the log generation from the log shipping so that the two things can’t possibly interfere with each other, and we’re guaranteed local logging under all circumstances.

You can actually install logstash directly on your clients and use that as your log shipper, but a more lightweight alternative is to use [log-courier](https://github.com/driskell/log-courier). 

It’s designed for minimal resource usage, is secured using certs, and allows you to add arbitary fields to log entries as it ships them. 

Since it uses TCP, we also wrote a Nagios NRPE check which uses netstat to confirm that logstash-forwarder is connected to our logstash server, and we get alerted if there any problems with the connectivity.

The logstash-forwarder configuration file is in JSON, and is fairly simple, although you do need to understand a bit about SSL certs in order to configure it. We just set up the server details, certs and define which logs to ship :

```
{
  "network": {
    "servers": [ "logstash.yourdomain.com:55515" ],
    "ssl certificate": "/var/lib/puppet/ssl/certs/yourhost.yourdomain.com.pem",
    "ssl key": "/var/lib/puppet/ssl/private_keys/yourhost.yourdomain.com.pem",
    "ssl ca": "/var/lib/puppet/ssl/certs/ca.pem",
    "timeout": 15
  },

  "files": [
  {
    "paths": [ "/var/log/syslog" ],
    "fields": {"shipper":"logstash-forwarder","type":"syslog"}
  },
  ]
}
```

In the syslog section, you can see we’ve added two arbitary fields to each entry that gets shipped, one which defines the shipper used, and one which tags the entries with a type of syslog. These make it easy for us to identify the type of the log for further processing later in the chain.

On the logstash server side, our configuration is slightly different, the input is defined as a lumberjack input, which is the name of the protocol used for transport, and we define the certs :

```
input {
  lumberjack {
    port => 55515
    ssl_certificate => "/var/lib/puppet/ssl/certs/logstash.yourdomain.com.pem"
    ssl_key => "/var/lib/puppet/ssl/private_keys/logstash.yourdomain.com.pem"
    type => "lumberjack"
  }
}
```

As we’re not using the syslog input, we also need to tell logstash how to split up the log data for this data type. We do that by using logstash’s built in filters, in this case a grok and a date filter, in combination with the tags we added when we shipped the log entries. 

One important thing to note here is that logstash processes the config file in order, so you need to have your filter section after your input section for incoming data to flow through the filter.

```
filter {
  if [type] == "syslog" {
    grok {
      match => { "message" => "<%{POSINT:syslog_pri}>%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:[%{POSINT:syslog_pid}])?: %{GREEDYDATA:syslog_message}" }
      add_field => [ "received_at", "%{@timestamp}" ]
      add_field => [ "received_from", "%{host}" ]
      add_field => [ "program", "%{syslog_program}" ]
      add_field => [ "timestamp", "%{syslog_timestamp}" ]
    }
    date {
      match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
    }
  }
}
```

What this filter does is look for messages tagged with the tag we added when we shipped the entry with logstash forwarder, and if it finds that tag, pass the message on through the filter chain.

The grok filter basically defines the layout of the message, and the fields which logstash should assign the sections to. We’re also adding a few extra fields which we use internally at DataCentred, and finally we pass through a date filter, which tells logstash what format the timestamp is in so it can set the internal timestamp correctly. The final step is very important so you have consistent timestamps across all your different log formats.

Once we’ve configured both sides, we just start the logstash-forwarder service and logstash-forwarder will watch the files we’ve defined, and send any new entries on over the network to our logstash server, which now knows how to process them. 

In the next post in this series I’ll talk about some of the more advanced filtering and munging we can do in logstash, and also introduce [Riemann](http://riemann.io/), a fantastic stream processing engine which can be combined with logstash to do much more complex real time analysis.
