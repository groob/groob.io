+++
date = "2015-08-02T21:08:24-04:00"
draft = false
title = "A practical intro to Prometheus"

+++
![img](https://i.imgur.com/zvvp4Zw.png)

There are two terms that are used to describe monitoring - whitebox and blackbox. An example of blackbox monitoring are Nagios checks, like pinging a gateway to see if it responds. It's called a blackbox because we can probe at a program, but we don't really have any control over it's internal state, or how it interacts with the rest of your system.  
Whitebox monitoring on the other hand means having data about the internal state of your program. An example of this is adding a counter like `requests_processed` in your webserver and making that available to a time series database. 
A great introduction to whitebox vs blackbox monitoring is Jamie Wilkinson's [talk](https://www.youtube.com/watch?v=eq4CnIzw-pE) at PuppetConf '12. 

Prometheus is both a time series database and a framework for instrumenting your code. First, you have to make the metrics available over http in a format that Prometheus server will understand. Then prometheus will scrape that server at a specified interval and store the metrics in it's database. Once Prometheus has the data, you can analyse it, create alerts or expose certain stats to a dashboard.   
For example, here are some metrics that prometheus exposes itself at http://prometheus/metrics:
```
# HELP http_requests_total Total number of HTTP requests made.
# TYPE http_requests_total counter
http_requests_total{code="200",handler="graph",method="get"} 6
http_requests_total{code="200",handler="label_values",method="get"} 6
```

Unless you're writing an application yourself, it is unlikely that it exposes metrics in a prometheus format. Luckily, there are dozens of '[exporters](http://prometheus.io/docs/instrumenting/exporters/)' out there that will convert data from another format, into something that prometheus can understand. Today, I'll focus on **caching_exporter**, an exporter that I wrote to monitor OS X Caching server, and **mtail**, the utility that it's based on. 

# mtail
[mtail](https://github.com/google/mtail) is a daemon that will tail a log file and expose various metrics over HTTP, so that Prometheus can scrape them. 
To use mtail, we first need to write a set of rules that we can use to process the log file. 
Here is a basic mtail rule from the project README:
```
# ~/linecounter.mtail
# simple line counter
counter line_count
/$/ {
  line_count++
}
```

All the above rule says is: when we encounter an end of line anchor(`$`), increment the counter line_count by one.

Now that we have a filter, we can start mtail. 
```
mtail --progs linecounter.mtail --logs /Library/Server/Caching/Logs/Debug.log
```
Now if we open our browser and go to `http://localhost:3903/metrics` we should see:
```
# TYPE line_count counter
line_count{prog="linecounter.mtail",instance="mylaptop.example.net"} 1124
```
The counter on the right will start at 0 and increment by one every time there's a new line in Debug.log. 

The above example is simple, but it also doesn't do anything useful. Let's write a more complicated rule. If we look at Debug.log, we will see something like this:

```
2015-08-02 21:01:14.932 Cleanup succeeded.
2015-08-02 21:01:15.483 Request for registration from https://lcdn-registration.apple.com/lcdn/register succeeded
2015-08-02 21:01:15.488 This server has 0 peers
2015-08-02 21:05:17.364 #leDkqrU0GiHl Request by "itunesstored/1.0" for http://a1443.phobos.apple.com/us/r1000/169/Purple1/v4/4d/2c/16/4d2c169d-7aa6-df87-1c86-ff1f37251be5/hlw1128731461172829049.D2.pd.ipa
```
All of the above information can be turned into useful metrics with some regex magic. 
```
# caching.mtail
counter caching_cleanups
counter caching_registrations
counter caching_requests by request_source, file_type

/^(?P<date>\d+-\d+-\d+ \d+:\d+:\d+\.\d+)/{
    strptime($date, "2006-01-02 15:04:05.000")
    caching_parsed_log_lines++

    # Registration
    /(\bRegistration\b \bsucceeded\b\.)/ {
    caching_registrations++
    }
    # Cleanup
    /(\bCleanup\b \bsucceeded\b\.)/ {
    caching_cleanups++
    }
    
    # requests
    /\#.* (Request by \"(?P<request_source>\w+\/\d+\.\d+)\" for http:\/\/).*\/[\w\.]+\.(?P<file_type>\w+)/ {
    caching_requests[$request_source][$file_type]++
    }
}
```
Now this is what prometheus sees at `http://caching.example.net:3903/metrics`:
```
# TYPE caching_parsed_log_lines counter
caching_parsed_log_lines{prog="caching.mtail",instance="mylaptop.example.net"} 1124
# TYPE caching_registrations counter
# TYPE caching_cleanups counter
caching_cleanups{prog="caching.mtail",instance="mylaptop.example.net"} 182
# TYPE caching_requests counter
caching_requests{file_type="ipa",request_source="itunesstored/1.0",prog="caching.mtail",instance="mylaptop.example.net"} 2
```

# caching_exporter
With a bit more work, we can turn everything in Debug.log into useful data. This post is specificaly about prometheus and mtail, but the above can also be achieved with Logstash and another metrics database. However, if you're familiar with Caching Server, you know that some of the useful information is also stored in
`/Library/Server/Caching/Config/Config.plist` and, `/Library/Server/Caching/Logs/LastState.plist` 
To get the data that I wanted out of the two plist files, I modified `mtail` a little and created my own exporter: https://github.com/groob/caching_exporter  

It's still mtail, and obeys the same command line flags and same *.mtail rule files, but also gets some counters from the plists. 
Here are a few metrics exposed from the Config plist:
```
# HELP caching_data data cached by server.
# TYPE caching_data gauge
caching_data{type="Books"} 3.9385975e+07
caching_data{type="Mac Software"} 7.3859516e+07
caching_data{type="Movies"} 0
# HELP caching_status_active whether caching server is currently running
# TYPE caching_status_active gauge
caching_status_active 1
```

# Prometheus
Now that we have an exporter running, let's get configure Prometheus. The official repo provides binaries for linux and OS X, but I prefer the docker container:
```
docker pull prom/prometheus
docker run -d --name prometheus -p 9090:9090 -v $(pwd)/config:/prometheus-config prom/prometheus -config.file=/prometheus-config/prometheus.yml
```
And a sample config file:
```YAML
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.
  evaluation_interval: 15s # By default, scrape targets every 15 seconds.
  # scrape_timeout is set to the global default (10s).

  # Attach these extra labels to all timeseries collected by this Prometheus instance.
  labels:
    monitor: 'devbox'

scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'caching-server'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 30s
    scrape_timeout: 10s

    target_groups:
      - targets: ['caching-server-url:3903']
```
