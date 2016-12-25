+++
date = "2015-07-09T23:21:55-04:00"
title = "Monitoring Munki with Logstash"

+++

![img](https://i.imgur.com/W2TIDjx.png)

This summer I set out to build a monitoring system for our infrastructure. The [ELK stack](https://www.elastic.co/webinars/introduction-elk-stack) is a great solution for log collection and analysis. One of the first things that I began monitoring with ELK is my Munki Install log. Here is how I'm using Logstash and Elasticsearch to monitor Munki accross all my clients. 

# Step 1: Client devices
## Ship logs from clients to logstash.
To transport logs, we will use [logstash-forwarder](https://github.com/elastic/logstash-forwarder), which is a CLI utility that will send logs over the network and use TLS for security. This is not the only way, but it is the most simple if you care about security. 

Unfortunately, there is no OS X binary for logstash-forwarder, but I wrote a Makefile to create a pkg installer [here](https://github.com/whitby/mac-scripts/tree/master/logstash_forwarder).  
The configuration for logstash-forwarder is found under `/etc/logstash-forwarder/config/` and split into multiple `.conf` file. 

First, we create a configuration file to connect with logstash:

```
# /etc/logstash-forwarder/config/00-base.conf
{
	"network": {
		"servers": [ "logstash.example.net:31415" ],
			"ssl ca": "/etc/logstash-forwarder/ca.crt",
			"timeout": 15
	},

		"files": [{}]
}
```
Then, we need to create a `.conf` file for each log file or group of logs that we want to monitor. In this case, we want to monitor `/Library/Managed Installs/Logs/Install.log` for changes.

```
# /etc/logstash-forwarder/config/01-munki.conf
{

	"files": [
	{
		"paths": [ "/Library/Managed Installs/Logs/Install.log" ],
			"fields": { "type": "munki_install" }
	}
	]
}
```

Once we have our configuration, we can create a [LaunchDaemon](https://github.com/whitby/mac-scripts/blob/master/logstash_forwarder/pkgroot/Library/LaunchDaemons/com.elastic.logstash-forwarder.plist) to run the `logstash-forwarder -config /etc/logstash-forwarder/config` command.

# Step 2
## Collect and parse logs with logstash

Assuming we have a logstash server at `logstash.example.net` we can configure it to listen to incoming logs from logstash-forwarder. Again, the configuration on the server is split into multiple `.conf` files.
I've chosen port 31415, but the port number can be any available port that you choose.

```
# /etc/conf.d/logstash/00-input-lumberjack.conf
input {
	lumberjack {
		# The port to listen on
		port => 31415

			# The paths to your ssl cert and key
			ssl_certificate => "/private/ssl/logstash.crt"
			ssl_key => "/private/ssl/logstash.key"

	}
}
```

By far, the hardest part of the process is figuring out a grok filter that will parse your Install.log file. I've used both [http://grokdebug.herokuapp.com/](http://grokdebug.herokuapp.com/) and [http://grokconstructor.appspot.com/](http://grokconstructor.appspot.com/) to build a grok filter, This is the second part of our logstash config:

For reference, this is what Install.log looks like:
```
Jun 10 2015 12:25:05 -0400 Removal of iPhoto: SUCCESSFUL
Jun 11 2015 12:34:14 -0400 Install of Adobe Flash Player-18.0.0.160: SUCCESSFUL
Jun 14 2015 11:59:35 -0400 Install of Google Chrome-43.0.2357.124: SUCCESSFUL
Jul 01 2015 09:07:10 -0400 Install of Google Chrome-43.0.2357.130: SUCCESSFUL
Jul 01 2015 09:11:24 -0400 Apple Software Update install of OS X Update-10.10.4: SUCCESSFUL
```
What we are doing bellow, is taking any log line from Install.log and parsing it into several components:

* A timestamp
* An action, this is either an `Install`, `Removal` or `Apple Software Update`
* The name of the software, ex. "Google Chrome"
* A version string, ex. 43.0.2357.130
* A result, this is either `SUCCESSFUL` or `FAILURE`

```
# /etc/conf.d/logstash/01-munki_install.conf
filter {
	if [type] == "munki_install" {
		grok {
			match => { "message" => "(?<time>%{CISCOTIMESTAMP} %{ISO8601_TIMEZONE}) %{DATA:action} of %{DATA:software}(-%{GREEDYDATA:version})?: %{WORD:result}" }
			add_tag => [ "grokked" ]
		}

		date {
			match => [ "time", "MMM dd YYYY HH:mm:ss Z" ]
				remove_field => [ "time" ]
				add_tag => [ "dated" ]

		}
	}
}
```

Once we have parsed the log line, we can send it to elasticsearch. 

```
# /etc/conf.d/logstash/99-out-elastic.conf
output {
	elasticsearch {
		host => "elasticsearch.local"
	}
}
```

Now we can run logstash with the command `logstash -f /etc/conf.d/logstash`. Logstash will combine all the `.conf` files in the conf.d directory and begin listening on port 31415 for new logs.

# Step 3
## Create a pretty graph in Kibana

Now that we have indexed logs in Elasticsearch we can visualize this in a live dashboard. I chose to represent the data in a bar graph that has 3 columns: `Install`, `Removal` and `Apple Software Update`. Each bar shows `SUCCESSFUL` or `FAILURE` counts. 

![img](https://i.imgur.com/WI5Uode.png)

Once we have the above visualization, we can combine it with other graphs in a dashboard that updates every minute and shows us live data. 

# Next steps

Implementing ELK to monitor Munki logs might seems too complex, but there are several big advantages. If we have ELK set up, we can centralize logs from multiple system. For example, we can configure [osquery](https://osquery.io/) to send logs to ELK. 

Another advantage is that we can have multiple outputs for our log data. For example, we can send each Install or Failure counter to a timeseries database like [InfluxDB](influxdb.com) or [Prometheus](prometheus.io). Now, if we have our munki repo in version control, we can correlate each git commit timestamp with an install counter. Plotting both on the same graph lets us detect which change caused a failure at a glance.
